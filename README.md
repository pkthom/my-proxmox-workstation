# Proxmox Workstation – README（改訂版・手順書）

**目的**

* 本リポジトリの構成例に基づき、以下の要件を安定・高性能に満たすための“実運用レベル”の手順をまとめます。

  * Proxmox VE（以下 PVE）を **NVMe 1TB×2 の ZFS ミラー**にインストール（OS/VM 共用）
  * **HDD 4TB×2 の ZFS ミラー**を写真保管（共有）用に構築
  * VM 構成：

    * Windows（開発用：Cursor + Docker Desktop）
    * Windows（動画編集用：プロキシ編集）
    * Ubuntu ×6（RKE2 学習用クラスター）
  * 将来：**dGPU（例：RTX 5070）を Windows VM にパススルー**

> ハード構成（想定）
>
> * CPU: Ryzen 9 7900（iGPU あり）
> * MB: ASUS ROG STRIX B650-A GAMING WIFI（ECC UDIMM 対応）
> * RAM: DDR5 ECC UDIMM 32GB（→後日 32GB 追加で計 64GB 推奨）
> * NVMe: Crucial T500 1TB ×2（ZFS mirror：`rpool`）
> * HDD: WD Red Plus 4TB ×2（ZFS mirror：`tank`）
> * PSU: RM850e (ATX3.1/12V-2x6)
> * Case: Corsair 3000D（前 120mm×2 付属 + 後 120mm×1 追加）

---

## 0. 物理組み立てと配線のポイント（GPU 増設前提）

* **M.2 スロット**：NVMe は **M.2\_1 + M.2\_2** に装着（将来 PCIEX16\_2 を使っても M.2\_3 影響を受けにくい）
* **ファン気流**：フロント 2 吸気 → リア 1 排気（後で必要ならトップ排気を追加）
* **電源ケーブル**：将来の dGPU 用に \*\*12V-2x6（新規格）\*\*を空けておく

---

## 1. BIOS 設定（最初に必ず）

1. **BIOS 更新**（最新へ）
2. **メモリ関連**

   * EXPO（DOCP）を有効化（対応速度で安定動作）
   * ECC：*Auto/Enabled*（項目がある場合）
3. **仮想化/パススルー**

   * *SVM*（AMD-V）= Enabled
   * *IOMMU*（SVM-IOMMU/AMD-IOV）= Enabled
   * *Above 4G Decoding* = Enabled
   * *Resizable BAR* = Enabled（GPU パススルー安定化に寄与）
   * *CSM* = Disabled（UEFI ブート）
4. **iGPU 出力**：ホスト表示は iGPU（HDMI/DP）を使用（将来 dGPU を丸ごと VM へ）

> メモ：BIOS 名称はバージョン差があります。該当が見つからない場合は *Advanced → PCI Subsystem / NBIO / CPU* 周辺を探してください。

---

## 2. Proxmox VE のインストール（ZFS ミラー）

### 2-1. インストールメディア作成

* 公式 ISO を取得し、Ventoy/Rufus 等で USB 作成

### 2-2. インストーラの選択

* **Target File System: ZFS (RAID1)** を選択し、2 本の NVMe をミラーに
* オプション（任意）

  * *ashift=12*（4K 物理セクタ前提）
  * *compression=zstd*（後からでも可）
  * *swap* はデフォルトで OK（メモリダンプや OOM 対策）

### 2-3. 初期設定

* ホスト名例：`pve.lab.local`
* 管理アドレス：固定 IP 推奨（例：`192.168.1.10/24`）
* `vmbr0` に物理 NIC をブリッジ（デフォルト）

---

## 3. 初回ログイン後のベース設定

SSH または Web UI（https\://\<PVE\_IP>:8006）で実施。

```bash
# 3-1. リポジトリ調整（無償利用の場合は enterprise を外し、no-subscription を追加）
sed -i 's/^deb/#deb/' /etc/apt/sources.list.d/pve-enterprise.list
cat >/etc/apt/sources.list.d/pve-no-subscription.list <<'EOF'
deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription
EOF

apt update && apt full-upgrade -y

# 3-2. 時刻・言語（必要に応じて）
timedatectl set-timezone Asia/Tokyo

# 3-3. 必須ツール
apt install -y qemu-guest-agent sudo vim htop iftop iotop smartmontools

# 3-4. ZFS 圧縮を既定化（任意）
zfs set compression=zstd rpool
```

> **ZFS ARC（メモリキャッシュ）**：初期 32GB の期間は `zfs_arc_max` を絞ると安定します（例：8GB）。
>
> ```bash
> echo 'options zfs zfs_arc_max=8589934592' >/etc/modprobe.d/zfs.conf
> update-initramfs -u -k all
> reboot
> ```

---

## 4. ディスク構成：HDD ミラー（写真保管）

HDD 4TB×2 を `tank` というミラープールにします。

```bash
# 物理デバイス確認
lsblk -o NAME,SIZE,MODEL | grep -iE 'wd|disk'

# 例: /dev/sdb と /dev/sdc が HDD の場合
zpool create -o ashift=12 tank mirror /dev/disk/by-id/<HDD1-id> /dev/disk/by-id/<HDD2-id>

# おすすめプロパティ
zfs set compression=zstd tank
zfs set atime=off tank
zfs set xattr=sa tank

# 写真用データセット
zfs create -o recordsize=1M tank/photos

# バックアップ保管用（後述の vzdump 保存先）
zfs create -o recordsize=1M tank/backups
```

> 以後、`tank` 側に週次スクラブを設定：
>
> ```bash
> cat >/etc/cron.weekly/zpool-scrub <<'EOF'
> #!/bin/sh
> /sbin/zpool scrub rpool
> /sbin/zpool scrub tank
> EOF
> chmod +x /etc/cron.weekly/zpool-scrub
> ```

### 4-1. Proxmox ストレージ登録

* Web UI → **Datacenter → Storage → Add → Directory**

  * ID: `tank-backups` / Directory: `/tank/backups` / Content: **VZDump backup file**
  * ID: `tank-photos` / Directory: `/tank/photos` / Content: **ISO image, VZDump backup file, VZDisk image**（*共有用途なら ISO/VZDisk は外しても良い*）

---

## 5. rpool 上のデータセット設計（VM/編集向け）

NVMe ミラー（`rpool`）にワーク用のデータセットを分け、性能と整理性を両立します。

```bash
# VM 用（デフォルトの local-zfs を使う場合は必須ではない）
zfs create rpool/vm
zfs set compression=zstd rpool/vm

# プロキシ素材用（大きめの連続 I/O を想定）
zfs create -o recordsize=128K rpool/proxy
zfs set atime=off rpool/proxy
```

> Web UI → Datacenter → Storage → Add → **Directory** で `/rpool/vm` と `/rpool/proxy` を登録（Content: **Disk image, Container**）。

---

## 6. 写真共有（Samba） – Windows から参照

Proxmox ホストで簡易に SMB 共有します（専用ファイルサーバ VM を立てる場合はスキップ）。

```bash
apt install -y samba

cat >>/etc/samba/smb.conf <<'EOF'
[photos]
   path = /tank/photos
   browseable = yes
   read only = no
   guest ok = no
   valid users = photosuser
   create mask = 0644
   directory mask = 0755
EOF

# 共有ユーザー作成（Linux ユーザー + SMB パスワード）
useradd -m photosuser
passwd photosuser
smbpasswd -a photosuser
systemctl restart smbd
```

Windows から：`\\<PVE_IP>\photos` を **ネットワークドライブ割り当て**。

---

## 7. Windows 11 VM（動画編集用）

### 7-1. ISO 準備

* Windows 11 ISO と **virtio-win ISO** を `local` または `local-zfs` の ISO ストレージへアップロード

### 7-2. VM 作成（推奨設定）

* **OS**: Microsoft Windows 11
* **Machine**: q35 / **BIOS**: OVMF (UEFI) / **Display**: *Default*
* **TPM**: v2.0（自動作成）
* **SCSI Controller**: VirtIO SCSI（Single）
* **Disk**: SCSI, **VirtIO** ブロック, `rpool/vm` に 200–500GB（プロジェクト規模で調整）
* **CPU**: type `host`, 8–10 vCPU
* **Memory**: 24–32GB（プロキシ編集前提）
* **Network**: VirtIO (paravirtualized)

> インストール時に「ドライバが見つかりません」と出たら、virtio ISO の `vioscsi/w11/amd64` を読み込んで続行。ネットワークは `NetKVM/w11/amd64`。

### 7-3. プロキシ編集の配置

* **プロキシ素材/プロジェクト**を `Z:`（例）として `rpool/proxy` を SMB 共有して割り当てる
* **生素材（4K/HEVC 等）は `\<PVE_IP>\photos`** から読み取り（書き込みは必要に応じて）

> 画面転送は RDP でも可。低遅延・高画質が必要なら **Sunshine + Moonlight** または **Parsec** を検討。

---

## 8. Windows 11 VM（開発用：Cursor + Docker Desktop）

* **CPU**: 4–6 vCPU、**Memory**: 8–12GB（プロジェクト規模で調整）
* **Nested Virtualization（WSL2 用）**

  * ホストで有効化：

    ```bash
    echo 'options kvm_amd nested=1' >/etc/modprobe.d/kvm_amd.conf
    update-initramfs -u -k all
    reboot
    ```
  * VM 設定：CPU type `host`、*Hyper-V* 関連フラグ（Proxmox UI の **Processors → Advanced → Enable Hyper-V**）を有効
* 初回起動後に Windows 機能で **仮想化基盤**と **WSL** をオン → 再起動 → Docker Desktop / WSL2 を導入

---

## 9. Ubuntu テンプレート & RKE2 学習用 6 ノード

### 9-1. Ubuntu クラウドイメージからテンプレート作成

1. **Cloud-Init ISO** を使わず、Proxmox の **Cloud-Init 機能**でパラメータ注入
2. `ubuntu-24.04-live-server` から通常 VM を作ってクリーンに整える or 公式 cloud image（qcow2）をインポート

**（簡便）通常インストールからテンプレート化の例）**

```bash
# 初期 VM: vCPU 2 / RAM 2GB / Disk 20GB / Net: VirtIO
# パッケージ整備
apt update && apt -y upgrade
apt install -y qemu-guest-agent curl vim
systemctl enable qemu-guest-agent --now
# 余計なログ減らしなど（任意）
```

* シャットダウン → **Convert to Template**
* **クローン**で 6 台作成（`rke2-ctrl-1` + `rke2-wkr-[1-5]` など）

### 9-2. RKE2 導入（最小構成）

**コントロールプレーン**（1 台）

```bash
curl -sfL https://get.rke2.io | sudo sh -
sudo systemctl enable rke2-server --now

# kubeconfig 取得
sudo mkdir -p ~/.kube
sudo cp /etc/rancher/rke2/rke2.yaml ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config
```

**ワーカ**（残り 5 台）

```bash
curl -sfL https://get.rke2.io | sudo INSTALL_RKE2_TYPE="agent" sh -
# server の IP を指定
sudo mkdir -p /etc/rancher/rke2/
echo 'server: https://<CTRL_IP>:9345' | sudo tee /etc/rancher/rke2/config.yaml
sudo systemctl enable rke2-agent --now
```

> 事前に **swapoff** しておくこと（`sudo swapoff -a`、`/etc/fstab` から swap 行をコメントアウト）。

---

## 10. GPU パススルー（将来の dGPU 追加時）

> いったん **ホスト表示は iGPU** に固定し、dGPU をまるごと Windows 編集 VM へ。

### 10-1. カーネルパラメータ（systemd-boot 前提）

```bash
# 値を確認
cat /etc/kernel/cmdline
# 末尾に以下を追記（スペース区切り）
# amd_iommu=on iommu=pt video=efifb:off pcie_acs_override=downstream,multifunction

nano /etc/kernel/cmdline
proxmox-boot-tool refresh
reboot
```

> *pcie\_acs\_override* は不要な環境もあります。IOMMU グループが分離されない場合のみ有効化。

### 10-2. VFIO バインド

```bash
# デバイス ID を確認（例：GPU と HDMI Audio）
lspci -nn | grep -i 'vga\|audio'
# 例: 10de:xxxx（GPU）, 10de:yyyy（Audio）

cat >/etc/modprobe.d/vfio.conf <<EOF
options vfio-pci ids=10de:xxxx,10de:yyyy disable_vga=1
EOF

# ブラックリスト（必要時）
cat >/etc/modprobe.d/blacklist-nvidia.conf <<'EOF'
blacklist nvidia
blacklist nouveau
EOF

update-initramfs -u -k all
reboot
```

### 10-3. VM への割り当て

* VM 設定 → **Add → PCI Device** で GPU 本体と Audio 関連を **両方** 追加
* オプション：`Primary GPU` チェック *オフ*（基本は仮想ディスプレイ併用 → RDP/Parsec で入る）
* Windows で GeForce ドライバ導入 → 再起動

> エラーが出る場合：CSM 無効、4G/REBAR 有効、IOMMU グループ再確認、ROM ファイル指定（必要時）などを確認。

---

## 11. バックアップ（vzdump）

* Web UI → **Datacenter → Backup** で **`tank-backups` へ週次**スケジュール

  * Mode: Snapshot / Compression: `zstd` / Prune: `keep-last=3, keep-weekly=4, keep-monthly=3`
* リストア手順の確認（年 1 回は DR リハーサルを推奨）

---

## 12. 運用チューニング/メンテ

* **ZFS スクラブ**：週次（前述の cron）
* **SMART テスト**：月次短時間テスト

  ```bash
  smartctl -t short /dev/sdX
  smartctl -a /dev/sdX | less
  ```
* **ログローテ/不要サービス停止**：`journalctl --vacuum-time=14d` など
* **アップデート**：月 1 回 `apt update && apt full-upgrade`（スナップショット取得後）

---

## 13. 参考の vCPU/メモリ割り当て（同時運用）

* **初期 32GB のとき**

  * Dev Win: 4 vCPU / 8–10GB
  * Edit Win: 8–10 vCPU / 16–24GB
  * K8s：必要時のみ（2GB×数台は厳しい）
* **64GB へ増設後**（推奨）

  * Dev Win: 6 vCPU / 12GB
  * Edit Win: 10 vCPU / 24–32GB
  * RKE2: 2 vCPU / 2GB ×6（合計 12GB + α）
  * 残りはホスト & ARC に回す

---

## 14. よくあるつまずき

* **インストール ISO が見えない/起動しない**：CSM を無効、UEFI Only、Secure Boot は OVMF/TPM と併用
* **Windows でディスクが見えない**：virtio SCSI ドライバを読み込む
* **GPU パススルーでブラックスクリーン**：`video=efifb:off` 入れ忘れ、ROM 指定、Primary GPU の扱い確認
* **RKE2 が Join しない**：`/etc/rancher/rke2/config.yaml` の server URL と FW、swap 無効化を確認

---

## 15. 変更履歴（本 README への主な修正）

* BIOS～GPU パススルーまでを **ワンストップ手順**として整理
* ZFS の \*\*データセット設計（rpool/proxy / tank/photos）\*\*を明確化
* **Samba 共有**で Windows から写真/プロキシを扱う実手順を追加
* **Nested Virtualization**（WSL2）と **Hyper-V フラグ**の具体手順を追加
* **バックアップ/スクラブ/SMART**など運用タスクの定例化

---

## 付録 A：チートシート

```bash
# ZFS 使用量
zfs list -o name,used,avail,compressratio

# IOMMU グループ確認
find /sys/kernel/iommu_groups/ -type l

# Proxmox ストレージ一覧
pvesm status

# VM の live 移動（同一ホスト内でディスクの移動）
qm move_disk <VMID> scsi0 tank-backups:images/<VMID>/vm-<VMID>-disk-0.raw
```

---

## 付録 B：推奨フォルダ構成（Windows 編集 VM）

* `Z:\Projects\<ProjectName>\` … プロジェクト/プロキシ（`rpool/proxy`）
* `Y:\Photos\` … 参照用の素材（`tank/photos`）

> プロジェクト終了後は **プロキシを削除**し、写真・完成品のみ長期保管（`tank`）。

---

以上。実際のリポジトリ構成に合わせてセクション名やコマンドブロックを微調整してください。必要なら “最小構成のファイルサーバ VM” 方式（Samba をホストではなく VM/LXC で動かす）版も追補できます。
