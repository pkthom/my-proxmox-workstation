# Proxmox ワークステーション README（Ryzen 9 7900 / B650M-A II 構成）

この README は、以下の構成で **Proxmox VE** を導入し、用途別 VM（開発用 Ubuntu／RKE2 学習用クラスター／Windows 動画編集用）を同時運用するための手順をまとめたものです。

## ゴール

* **Proxmox VE** を SATA SSD ミラー（ZFS）にインストール
* NVMe ミラー（ZFS）を **VM/ISO/バックアップ** に分離して高速・安全運用
* **開発用 Ubuntu VM（Docker/CURSOR）** と **RKE2 学習クラスター（6ノード）**
* **Windows 11 4K編集 VM（プロキシ編集）**
* 将来 **NVIDIA GPU パススルー（RTX 5070想定）** に備えた設定
* バックアップ／メンテとトラブル対処

> **メモ**: メモリ64GBでも動きますが、同時実行の余裕を重視するなら **128GB** 化が最も効果的です。

---

## 0. 想定ハード（そのまま流用可）

* **CPU**: AMD Ryzen 9 7900（12C/24T, iGPU内蔵）
* **M/B**: ASUS PRIME B650M‑A II CSM（mATX）
* **RAM**: Crucial Pro DDR5‑5600 32GB×2 = 64GB（将来128GB化）
* **CPUクーラー**: Thermalright Peerless Assassin 120 SE
* **GPU**: なし（将来 MSI GeForce RTX 5070 12GB を追加予定）
* **OS用**: SATA 2.5" SSD 250GB ×2（ZFS mirror）
* **データ用**: NVMe 2TB ×2（ZFS mirror）
* **PSU**: Corsair RM750e（ATX 3.1 / 750W）
* **Case**: Corsair 3000D TG Airflow（前120mm×2 + **後120mm×1**）
* **NIC**: オンボード Realtek 2.5GbE（当面1GbEハブ）。不安定なら **Intel I210‑T1** を増設

---

## 1. 組み立て & 配線（5分チェック）

1. SATA 250GB×2 → OS用、NVMe 2TB×2 → データ用
2. **NVMeヒートシンクは“マザボ純正 or SSD付属のどちらか一方”**（重ね付けNG）
3. **FAN: フロント×2吸気 + リア×1排気**（正圧で埃に強い）
4. 電源24pin/8pin/（将来）12V‑2×6を整線。
5. 最小構成で起動 → **POST/温度/認識** を確認

---

## 2. BIOS/UEFI 設定（ASUS系の一般例）

* Load Optimized Defaults
* **SVM(AMD‑V)=Enabled / IOMMU=Enabled**
* **Above 4G Decoding=Enabled / Resizable BAR=Enabled**（将来GPU向け）
* Primary Display = iGPU（dGPUはVMへ渡す想定）
* **CSM=Disabled（UEFI前提）**
* **メモリは最初は Auto/JEDEC（安定確認後に EXPO を試す）**

---

## 3. Proxmox VE インストール（OS: SATA 250GB×2 / ZFS mirror）

1. 公式ISOでブート → Target で **ZFS (RAID1/mirror)** を選択し 2台のSATA SSDを指定
2. タイムゾーン **Asia/Tokyo**、root PW、管理NICはDHCPでもOK
3. 再起動 → Web UI `https://<PVE-IP>:8006/`

**初期調整（No‑Subscription & 更新）**

```bash
sudo sed -i 's/^deb/# deb/g' /etc/apt/sources.list.d/pve-enterprise.list
echo 'deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription' | \
  sudo tee /etc/apt/sources.list.d/pve-no-subscription.list
sudo apt update && sudo apt -y full-upgrade
```

* **NTP有効化**（Datacenter → Time）
* （任意）通知先メール設定（Datacenter → Notifications）

---

## 4. データ用 ZFS プール（NVMe 2TB×2 / ZFS mirror）

**目的**: VM・ISO・バックアップをOSプールと分離して高速＆安全運用

1. デバイス確認（**by‑id** を使うのが安全）

```bash
ls -l /dev/disk/by-id/
```

2. プール作成（例: `tank`）

```bash
sudo zpool create -f \
  -o ashift=12 -o autotrim=on \
  tank mirror \
  /dev/disk/by-id/nvme-<NVMe0-ID> \
  /dev/disk/by-id/nvme-<NVMe1-ID>
```

3. 用途別データセット

```bash
sudo zfs create -o compression=zstd -o atime=off -o xattr=sa -o acltype=posixacl tank/vm
sudo zfs create -o compression=zstd -o atime=off tank/iso
sudo zfs create -o compression=zstd -o atime=off tank/backup
```

4. Proxmox へ登録（GUI）

* **ZFS**: ID=`tank-vm` / Pool=`tank`（または `tank/vm`）/ Thin=Yes（推奨）
* **Directory**: ISO → ID=`tank-iso` / Dir=`/tank/iso`（Content=ISO/Template）
* **Directory**: Backup → ID=`tank-bak` / Dir=`/tank/backup`（Content=VZDump）

**64GB運用でのメモリ余裕確保（ZFS ARC制限）**

```bash
echo "options zfs zfs_arc_max=8589934592" | sudo tee /etc/modprobe.d/zfs.conf
sudo update-initramfs -u
sudo reboot
```

**TRIM（Discard）の徹底**

* プール作成時 `autotrim=on` に加え、**各VMのディスク設定で Discard=on / SSD emulation=on** を有効化
* ゲストOS側でも `fstrim.timer` を有効化（週1など）

> **非ECC × ZFS の注意**: 月1 `zpool scrub` と **外部バックアップ（3‑2‑1）** 推奨。停電対策にUPSも。

---

## 5. ネットワーク（ブリッジ）

* 既定の `vmbr0` を物理NICにブリッジ（インストーラ既定）
* 家庭内DHCPでVMにIP配布が簡単
* **Realtek 2.5GbE** で不安定が出たら **Intel I210‑T1** に差替え（設定は概ねそのまま）
* 大容量コピーやNAS運用が増えたら、将来 **2.5GbEスイッチ** に更新

---

## 6. ISO / テンプレ準備

`tank-iso` に以下をアップロード：

* Ubuntu 24.04 LTS（cloud‑image または live‑server）
* Windows 11 ISO
* `virtio-win` ISO（Windows用 VirtIO ドライバ）

---

## 7. Ubuntu Cloud‑Init テンプレ（量産用）

テンプレVM（例: ID=9000）を作成し、以後Cloneで量産します。

```bash
qm create 9000 --name ubuntu-2404-cloud --memory 4096 --cores 2 \
  --net0 virtio,bridge=vmbr0 --scsihw virtio-scsi-single --agent enabled=1 \
  --bios ovmf --machine q35

qm importdisk 9000 /tank/iso/ubuntu-24.04-server-cloudimg-amd64.img tank
qm set 9000 --scsi0 tank:vm-9000-disk-0
qm set 9000 --ide2 tank-iso:cloudinit --boot c --bootdisk scsi0
qm set 9000 --serial0 socket --vga serial0
qm template 9000
```

**最小 user‑data 例**

```yaml
#cloud-config
hostname: dev-ubuntu
users:
  - name: ubuntu
    sudo: ALL=(ALL) NOPASSWD:ALL
    groups: users, admin
    shell: /bin/bash
    ssh_authorized_keys:
      - ssh-ed25519 AAAA... your-public-key ...
package_update: true
package_upgrade: true
timezone: Asia/Tokyo
runcmd:
  - apt-get update
  - apt-get -y install qemu-guest-agent
  - systemctl enable --now qemu-guest-agent
```

---

## 8. 開発用 Ubuntu VM（Docker/CURSOR）

**推奨**: 4 vCPU / 8–16GB RAM / 100–200GB（`tank-vm`）

```bash
sudo apt update && sudo apt -y upgrade
# Docker 公式手順
sudo apt-get -y install ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo $UBUNTU_CODENAME) stable" | \
sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update && sudo apt -y install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo usermod -aG docker $USER
# 再ログイン後
docker run hello-world
```

> Cloud‑Initを使わない作成方法でも、**`qemu-guest-agent` を必ず導入**してください。

---

## 9. RKE2 学習用クラスター（6ノード）

**VM構成例**

* `rke2-srv01`（サーバ/CP）: 2 vCPU / 4GB / 40GB
* `rke2-agt01`〜`rke2-agt05`（エージェント）: 各1–2 vCPU / 2–3GB / 30GB

**全ノード共通の前提設定**

```bash
# swap無効
sudo swapoff -a
sudo sed -i.bak '/ swap / s/^/#/' /etc/fstab

# モジュールとsysctl
sudo modprobe br_netfilter overlay
cat <<'EOF' | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables=1
net.ipv4.ip_forward=1
net.bridge.bridge-nf-call-ip6tables=1
EOF
sudo sysctl --system
```

**Server（rke2-srv01）**

```bash
curl -sfL https://get.rke2.io | sudo INSTALL_RKE2_TYPE=server sh -
sudo systemctl enable rke2-server --now
mkdir -p ~/.kube
sudo cp /etc/rancher/rke2/rke2.yaml ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config
# エージェント接続用トークン確認
sudo cat /var/lib/rancher/rke2/server/node-token
```

**Agent（rke2-agtXX）**

```bash
curl -sfL https://get.rke2.io | sudo INSTALL_RKE2_TYPE="agent" sh -
sudo mkdir -p /etc/rancher/rke2
sudo tee /etc/rancher/rke2/config.yaml >/dev/null <<EOF
server: "https://<SRV01-IP>:9345"
token:  "<上で表示したトークン>"
EOF
sudo systemctl enable rke2-agent --now
```

**確認**

```bash
kubectl get nodes -o wide
```

---

## 10. Windows 11 4K編集 VM（プロキシ前提）

**推奨**: 8 vCPU / 16–32GB RAM / System 150GB + **Scratch 500GB〜1TB**（`tank-vm`）

**作成ポイント**

* Machine=`q35` / BIOS=`OVMF(UEFI)` / **TPM v2有効**
* Disk = **SCSI (VirtIO SCSI single) + IOThread + Cache=Write‑back**
* NIC = VirtIO
* ISO = Windows 11 + `virtio-win`（CD/DVD2）

**インストール**

1. ディスク選択画面で **ドライバ読み込み** → `virtio-win` から SCSI/NetKVM を導入
2. OS導入後 `virtio-win-guest-tools` 実行（VirtIO一式＋QGA）
3. **電源プラン=高パフォーマンス**、プロキシ生成・キャッシュは **Scratch** へ
4. 共有が要るなら LXC の Samba で `/tank/media` をSMB共有

**Balloonメモリ**

* 重い編集時は **Balloonを無効化（固定メモリ）** で安定

---

## 11. 将来: NVIDIA GPU パススルー（RTX 5070想定）

**前提**: 2章で SVM/IOMMU/Above4G/ResizableBAR を有効化済み

**Proxmoxホスト**

```bash
# IOMMU有効化（AMD）
sudo sed -i 's/^GRUB_CMDLINE_LINUX_DEFAULT=.*/GRUB_CMDLINE_LINUX_DEFAULT="quiet amd_iommu=on iommu=pt"/' /etc/default/grub
sudo update-grub

# vfio モジュール
sudo tee -a /etc/modules <<'EOF'
vfio
vfio_iommu_type1
vfio_pci
vfio_virqfd
EOF
sudo update-initramfs -u

# （任意）ホストでNVIDIAドライバを使わない
printf 'blacklist nvidia
blacklist nvidia_drm
blacklist nvidia_uvm
' | \
  sudo tee /etc/modprobe.d/blacklist-nvidia.conf

# 再起動後、GPUのデバイスIDを vfio-pci にバインド
echo 'options vfio-pci ids=10de:xxxx,10de:yyyy' | \
  sudo tee /etc/modprobe.d/vfio-pci-ids.conf
sudo update-initramfs -u && sudo reboot
```

**VM側**

* Hardware → Add → **PCI Device** で GPU（**VGA と HD Audio の2関数**）を追加（All Functions / PCI‑Express）
* Display は **最初は残す**（トラブルシュート用）。**ドライバ導入後** に **Display=None** へ
* Windows 内で公式ドライバを導入

---

## 12. バックアップ & メンテ

**VZDump（GUI）**

* Storage=`tank-bak`（可能なら外部/NASへ二重化）
* Mode=`Snapshot` or `Stop` / **Compression=`ZSTD`** / Schedule=深夜
* 保持=直近3〜6世代

**ZFSメンテ**

```bash
# 毎月1日の03:00にscrub（rootのcrontab例）
0 3 1 * * /sbin/zpool scrub tank
# 健康確認
zpool status -x
zpool list
zfs list
```

**更新**

```bash
apt update && apt full-upgrade -y
```

**UPS推奨**

* ZFSは瞬断に弱いため、小型UPSで安全終了を確保

---

## 13. リソース割当（同時運用の目安）

| VM            | vCPU |     RAM |                   Disk | 備考         |
| ------------- | ---: | ------: | ---------------------: | ---------- |
| dev-ubuntu    |    4 |  8–16GB |              100–200GB | Docker/開発  |
| rke2-srv01    |    2 |     4GB |                   40GB | コントロールプレーン |
| rke2-agt01〜05 | 各1–2 |  各2–3GB |                  各30GB | 学習用エージェント  |
| win11-edit    |    8 | 16–32GB | 150GB + Scratch 500GB〜 | プロキシ編集     |

> **64GBでも可能**。ただし余裕重視なら **128GB 化** が最大効果。

---

## 14. トラブル時のヒント

* **Realtek不安定**: ケーブル/スイッチ確認 → Intel I210‑T1 へ。`dmesg`/`ethtool -S` で統計確認
* **Windows I/Oが重い**: SCSI+IOThread+Write‑back、Scratchを別ディスクに
* **K8s NotReady**: NTP/時刻ズレ、swap無効化、CNIモジュール
* **ZFS容量逼迫**: `zfs list -o space`、不要スナップショット削除
* **GPUパススルー不可**: IOMMU/All Functions/Display Noneの切替タイミング/ドライバblacklist を再確認

---

## 付録A：便利コマンド

```bash
# Proxmox更新
apt update && apt full-upgrade -y

# ZFS状態/スクラブ
zpool status
zpool scrub tank

# TRIM（手動）
fstrim -av

# バックアップ（手動例）
vzdump 100 --mode snapshot --compress zstd --storage tank-bak

# VMクローン（テンプレ9000→VM 101）
qm clone 9000 101 --name=dev-ubuntu --full 1 --storage tank

# VMリソース変更
qm set 101 --cores 4 --memory 8192
```

## 付録B：Samba LXCでの簡易共有（任意）

```bash
sudo apt update && sudo apt -y install samba
sudo mkdir -p /srv/media
# /etc/samba/smb.conf に共有定義を追加
[media]
   path = /srv/media
   read only = no
   browsable = yes
   guest ok = no
sudo smbpasswd -a <username>
sudo systemctl enable --now smbd
```

---

### ライセンス

このREADMEは MIT ライセンスとして扱って構いません（必要なら変更してください）。

---

これで、組立 → インストール → 開発VM → RKE2 → Windows編集 → 将来GPU → バックアップ運用 まで **1本で完走**できます。コミット前に、あなたの環境のホスト名や固定IP、公開鍵などを差し込んでください。
