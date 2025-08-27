# Proxmox ワークステーション README（Ryzen 9 7900 / B650M-A II 構成）

この README は、以下の構成で **Proxmox VE** を導入し、用途別 VM（開発用 Ubuntu／RKE2 学習用クラスター／Windows 動画編集用）を同時運用するための手順をまとめたものです。

* CPU: AMD Ryzen 9 7900（iGPU 内蔵）
* M/B: ASUS PRIME B650M-A II CSM
* RAM: Crucial Pro DDR5-5600 32GB ×2（計 64GB） ※将来 128GB へ増設想定
* OS 用 SSD: SATA 2.5" 250GB ×2（RAID1 / ZFS mirror）
* データ用 SSD: NVMe 2TB ×2（ZFS mirror）
* PSU: Corsair RM750e（ATX 3.1 / 750W）
* Case: Corsair 3000D TG Airflow（前面 120mm×2 搭載済）
* 追加ファン: 120mm PWM ×1（リア排気）
* NIC: オンボード 2.5GbE（当面 1GbE ハブ運用）
* 将来拡張: GeForce RTX 5070（PCIe パススルー）、メモリ 128GB 化、Intel NIC 追加

> **目的**
>
> * 開発用 Ubuntu VM（Docker / CURSOR）
> * 学習用 RKE2（最小構成 6 ノード）
> * Windows VM（iPhone16 の 4K 素材を **プロキシ編集**）

---

## 0. 組み立てと配線チェック

1. **ストレージ配置**

   * SATA 250GB ×2 → マザーボードの SATA ポートへ（OS 用）
   * NVMe 2TB ×2 → M.2 スロットへ（データ用）
2. **ファン**

   * フロント: 120mm×2 吸気（ケース同梱）
   * リア: 120mm×1 排気（追加）
3. **電源配線**

   * 24pin / 8pin CPU、将来 GPU 用に 12V-2×6 ケーブルを整線しておく
4. **ネットワーク**

   * オンボード 2.5GbE を 1GbE ハブへ接続（LAN ケーブルは Cat6A 以上推奨）

---

## 1. BIOS/UEFI 設定（初期）

マザーボード起動後、UEFI で以下を設定：

* **Load Optimized Defaults** を適用
* **SVM（AMD-V） = Enabled**
* **IOMMU = Enabled**（`SVM Mode` と別項目の場合あり）
* **Above 4G Decoding = Enabled**
* **Resizable BAR = Enabled**（将来 GPU パススルー用）
* **Primary Display = iGPU**（将来 dGPU を VM へ渡すため）
* **CSM = Disabled**（UEFI 前提）
* メモリは **EXPO プロファイル有効**（安定しない場合は Auto へ戻す）

> 将来 dGPU パススルー時もこの方針のまま使えます。

---

## 2. Proxmox VE のインストール（OS 用 SATA 2台で ZFS mirror）

1. 公式 ISO でブート → **ZFS (RAID1 / mirror)** を選択
2. 2台の SATA 250GB を選択し、プールを作成（標準設定で可）
3. 管理 IP（DHCP で可）、root パスワード、タイムゾーン（Asia/Tokyo）を設定
4. インストール完了後、Web UI（`https://<PVE-IP>:8006/`）へアクセス

> **注意:** OS 用 ZFS は容量に余裕が少ないため、ISO やバックアップは基本的に **NVMe プール** または外部へ置く運用を推奨。

---

## 3. 初期セットアップ（リポジトリ／更新／時刻・通知）

### 3.1 No-Subscription リポジトリ（任意）

Enterprise リポを無効化し、No-Subscription を利用する場合：

```bash
# Enterprise リポをコメントアウト
sudo sed -i 's/^deb/# deb/g' /etc/apt/sources.list.d/pve-enterprise.list

# No-Subscription リポを追加（bookworm の例）
echo 'deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription' | sudo tee /etc/apt/sources.list.d/pve-no-subscription.list

sudo apt update && sudo apt -y full-upgrade
```

### 3.2 NTP/時刻同期

* Datacenter → **Time** で NTP を有効にし、`time.google.com` などを設定

### 3.3 通知（任意）

* Datacenter → **Notifications** でメール通知先を設定すると便利

---

## 4. データ用 NVMe プール作成（ZFS mirror）

NVMe 2TB ×2 で ZFS プール（例: `tank`）を作成します。**by-id** 名で実行してください。

```bash
# デバイス確認（例）
ls -l /dev/disk/by-id/

# プール作成（4K セクタ最適化 / TRIM 有効 / 圧縮）
sudo zpool create -f \
  -o ashift=12 \
  -o autotrim=on \
  tank mirror \
  /dev/disk/by-id/nvme-<NVMe0-ID> \
  /dev/disk/by-id/nvme-<NVMe1-ID>

# 代表的なデータセット（VM/ISO/バックアップ分離）
sudo zfs create -o compression=zstd -o atime=off -o xattr=sa -o acltype=posixacl tank/vm
sudo zfs create -o compression=zstd -o atime=off tank/iso
sudo zfs create -o compression=zstd -o atime=off tank/backup
```

Proxmox にストレージ登録：

* **Datacenter → Storage → Add → ZFS**

  * ID: `tank-vm`
  * ZFS Pool: `tank`（または `tank/vm`）
  * Thin provision: **Yes**
* **Add → Directory**（ISO 用）

  * ID: `tank-iso`
  * Directory: `/tank/iso`（マウントは ZFS が管理）
  * Content: ISO image / Container template
* **Add → Directory**（バックアップ用）

  * ID: `tank-bak`
  * Directory: `/tank/backup`
  * Content: VZDump backup file

> **運用ヒント**: `tank/vm` に VM ディスク、`tank/iso` に ISO/テンプレ、`tank/backup` にバックアップを分けると整理しやすい。

---

## 5. ネットワーク（ブリッジ）

* 既定の `vmbr0` を **ブリッジ接続**にし、物理 NIC（例: `enp…`）をスレーブにする（インストーラ既定）
* 家庭内ルータの **DHCP** で VM に IP を配布する運用が簡単
* Realtek 2.5GbE は 1GbE ハブでも自動で 1G リンクします

> **不安定な場合** は、後日 Intel NIC（i225/i226 系）へ変更を検討

---

## 6. ISO／テンプレの準備

1. **Ubuntu 24.04 LTS cloud image**（`ubuntu-24.04-live-server` でも可）を `tank-iso` へアップロード
2. **Windows 11 ISO** を `tank-iso` へアップロード
3. **virtio-win ISO**（Windows 用 VirtIO ドライバ）を `tank-iso` へアップロード

---

## 7. 開発用 Ubuntu VM（Docker/CURSOR）

### 7.1 作成パラメータ（例）

* Name: `dev-ubuntu`
* Node: この PVE ホスト
* ISO: `ubuntu-24.04-server`（または Cloud-Init で自動化）
* Machine: **q35** / BIOS: **OVMF (UEFI)**
* Storage: `tank-vm`
* Disk: 100–200GB（用途に応じて） / **SCSI (VirtIO SCSI single) + IOThread**
* CPU: **host**, 4 vCPU
* RAM: 8–16GB（作業量に応じて）
* Network: VirtIO (paravirtualized) / Bridge: `vmbr0`

### 7.2 初期セットアップ（例）

```bash
# パッケージ更新
sudo apt update && sudo apt -y upgrade

# Docker
sudo apt-get -y install ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo $UBUNTU_CODENAME) stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update && sudo apt -y install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo usermod -aG docker $USER

# 再ログインして 'docker run hello-world' で確認
```

> **CURSOR** は VSCode 派生のエディタです。GUI が必要な場合は Remmina + xrdp などで接続、またはローカル端末から SSH Remote を利用してください。

---

## 8. RKE2 学習用クラスター（6 ノード）

### 8.1 VM 構成例

* `rke2-srv01`（コントロールプレーン/サーバ）: 2 vCPU / 4GB / 40GB
* `rke2-agt01`〜`rke2-agt05`（エージェント）: 各 1–2 vCPU / 2–3GB / 30GB
* いずれも **Ubuntu 24.04 Server** / `tank-vm` に作成

### 8.2 インストール（RKE2）

**Server ノード**（`rke2-srv01`）

```bash
curl -sfL https://get.rke2.io | sudo sh -
sudo systemctl enable rke2-server --now

# Kubeconfig（ユーザ用に権限付与）
sudo mkdir -p ~/.kube
sudo cp /etc/rancher/rke2/rke2.yaml ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config
export KUBECONFIG=~/.kube/config

# トークン確認（エージェント登録に使用）
sudo cat /var/lib/rancher/rke2/server/node-token
```

**Agent ノード**（`rke2-agtXX`）

```bash
curl -sfL https://get.rke2.io | sudo INSTALL_RKE2_TYPE="agent" sh -
echo "server: https://<SRV01-IP>:9345" | sudo tee /etc/rancher/rke2/config.yaml
# 上記に token: '<出力されたトークン>' も追記
sudo bash -lc 'echo token: "$(cat /tmp/rke2_token)" >> /etc/rancher/rke2/config.yaml' # ← 例（事前に /tmp に保存しておく）

sudo systemctl enable rke2-agent --now
```

**kubectl 確認**

```bash
kubectl get nodes -o wide
```

> まずは学習目的なので、CPU/メモリを控えめにしてデプロイしてください。必要に応じて vCPU/RAM を増やします。

---

## 9. Windows 11 動画編集 VM（プロキシ前提）

### 9.1 作成パラメータ（初期）

* Name: `win11-edit`
* Machine: **q35** / BIOS: **OVMF (UEFI)** / **TPM v2** 有効
* Storage: `tank-vm`
* Disk: System 150GB（SCSI / VirtIO SCSI single / IOThread）
* 追加 Disk: Scratch 500GB〜1TB（同上。プロジェクト・キャッシュ用）
* CPU: **host**, 8 vCPU（必要に応じ増減）
* RAM: 16–32GB（32GB 推奨）
* Display: **Default**（将来 GPU パススルーで None に変更）
* NIC: VirtIO（準仮想化）
* ISO: Windows 11 ISO + `virtio-win` ISO を CD/DVD2 に追加

### 9.2 インストール手順

1. ブート → ストレージ選択画面で **ドライバの読み込み** → `virtio-win` から **SCSI/ネットワーク** ドライバを導入
2. OS 導入後、`virtio-win-guest-tools` を実行し VirtIO ドライバ一式を導入
3. 電源設定で **高パフォーマンス**、不要なバックグラウンド同期を OFF
4. 編集ソフト（例: DaVinci / Premiere 等）で **プロキシ生成の既定保存先** を Scratch ディスクに設定

> **iPhone 4:2:2 / HEVC** はプロキシ編集が安定。最初の運用としては dGPU なしでも快適性十分です。

---

## 10. 将来: NVIDIA GPU パススルー（RTX 5070 予定）

### 10.1 物理増設と UEFI 設定

* GPU を実装、補助電源（12V-2×6）を接続
* **Primary Display = iGPU** を維持
* **Above 4G / Resizable BAR = Enabled**（すでに設定済み）

### 10.2 Proxmox ホスト設定

`/etc/default/grub` に IOMMU を有効化（AMD）

```bash
GRUB_CMDLINE_LINUX_DEFAULT="quiet amd_iommu=on iommu=pt"
sudo update-grub
```

`/etc/modules` に vfio 関連を追加

```bash
sudo tee -a /etc/modules <<'EOF'
vfio
vfio_iommu_type1
vfio_pci
vfio_virqfd
EOF
sudo update-initramfs -u
```

**（任意）NVIDIA ドライバをホストで読み込ませない**

```bash
echo -e 'blacklist nvidia\nblacklist nvidia_drm\nblacklist nvidia_uvm' | \
  sudo tee /etc/modprobe.d/blacklist-nvidia.conf
```

**再起動** 後、GPU のデバイス ID を確認して VFIO にバインド：

```bash
lspci -nnk | grep -A3 -i nvidia
# 例: 0000:03:00.0 VGA [10de:xxx] / 0000:03:00.1 Audio [10de:yyy]

echo 'options vfio-pci ids=10de:xxxx,10de:yyyy' | \
  sudo tee /etc/modprobe.d/vfio-pci-ids.conf
sudo update-initramfs -u && sudo reboot
```

### 10.3 VM 側設定

* `win11-edit` の **Hardware → Add → PCI Device** で GPU（VGA/Audio）を追加

  * Options: **All Functions**, **PCI-Express**, **Primary GPU**（必要に応じ）
* Display: **None**（QXL を外す）
* 起動後、Windows に **NVIDIA ドライバ** をインストール

> 近年の Proxmox + OVMF/q35 では **Code 43 回避設定不要** なケースが多いです。

---

## 11. バックアップ＆メンテ

### 11.1 バックアップ（VZDump）

* **Datacenter → Backup** で ジョブ追加

  * Storage: `tank-bak`（可能なら外部/NAS を推奨）
  * Schedule: 週 1〜3 回（深夜）
  * Mode: Stop（または Snapshot）
  * Retention: 直近 3〜6 世代

### 11.2 ZFS メンテ

```bash
# 月例 scrub（cron）
sudo crontab -e
# 毎月 1 日の 03:00 に scrub
0 3 1 * * /sbin/zpool scrub tank

# 健康確認
zpool status -x
zpool list
zfs list
```

### 11.3 アップデート

* 月 1 回程度、`apt update && apt full-upgrade`（再起動計画の上で）

> **ECC なし + ZFS** でも問題なく運用可能ですが、**定期バックアップ** は必須です。

---

## 12. リソース割当の目安（同時運用想定）

| VM            | vCPU |     RAM |                   Disk | 備考         |
| ------------- | ---: | ------: | ---------------------: | ---------- |
| dev-ubuntu    |    4 |  8–16GB |              100–200GB | Docker/開発用 |
| rke2-srv01    |    2 |     4GB |                   40GB | コントロールプレーン |
| rke2-agt01〜05 | 各1–2 |  各2–3GB |                  各30GB | 学習用エージェント  |
| win11-edit    |    8 | 16–32GB | 150GB + Scratch 500GB〜 | プロキシ編集     |

> 64GB でも可能ですが、**128GB 化**で余裕が大きく改善します。

---

## 13. 省コスト運用のコツ

* **GPUは後回し**：まずはプロキシ編集で十分。必要性が明確になってから導入
* **NIC も後回し**：安定性に問題が出た時点で Intel NIC に換装（できれば 2.5GbE）
* **ファンは最小構成**：リア 120mm×1 のままで OK。温度が高い時のみ増設
* **OS プールに ISO を溜めない**：`tank/iso` を活用

---

## 14. トラブル時のヒント

* **Realtek 2.5GbE が不安定**：ケーブル交換 → スイッチ交換 → Intel NIC への換装
* **Windows の I/O が重い**：SCSI (VirtIO) + IOThread を確認、Scratch を別仮想ディスクに
* **ZFS 容量逼迫**：`zfs list -o space` で使用状況確認、不要スナップショット削除
* **GPU パススルーで起動しない**：IOMMU 設定／All Functions／Display None／ドライバ競合（blacklist）を再確認

---

## 15. 付録：Cloud-Init 最小 user-data（Ubuntu）

```yaml
#cloud-config
hostname: dev-ubuntu
users:
  - name: ubuntu
    sudo: ALL=(ALL) NOPASSWD:ALL
    groups: users, admin
    shell: /bin/bash
    ssh_authorized_keys:
      - ssh-ed25519 AAAA... あなたの公開鍵 ...
package_update: true
package_upgrade: true
timezone: Asia/Tokyo
runcmd:
  - apt-get update
  - apt-get -y install qemu-guest-agent
  - systemctl enable --now qemu-guest-agent
```

> Proxmox で Cloud-Init を有効にし、`CI CD-ROM` を追加して適用してください。

---

### 完了

この README のとおりに進めれば、**Proxmox 上で「開発 + RKE2 + Windows 編集」を同時運用**できます。将来は **メモリ 128GB 化**と **RTX 5070 パススルー**を追加して、さらに快適に拡張してください。
