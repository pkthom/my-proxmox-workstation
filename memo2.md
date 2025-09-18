了解！上のパーツ構成で Proxmox を入れて、狙いどおりの 3 つの VM・ZFS・固定 IP・Samba 共有・GPU パススルーまでを、一気に最後までできるように手順を「そのまま実行できる形」でまとめました。
（以下、例の LAN は `192.168.1.0/24`、Proxmox ホストは `192.168.1.10` を仮定。数字だけ置き換えてください）

---

# 0. BIOS 設定（最初に一度）

ASUS TUF B650-PLUS WIFI（Ryzen 9 7900）

1. 「Advanced」→「CPU Configuration」

   * `SVM Mode` → **Enabled**（仮想化）
2. 「Advanced」→「AMD PBS / PCI Subsystem Settings」

   * `IOMMU` → **Enabled**
   * `Above 4G Decoding` → **Enabled**
   * `Re-Size BAR Support` → **Enabled**（任意、4060 は有効で OK）
3. 起動順序：USB を最優先に
4. 変更保存して再起動、USB から起動

---

# 1. Proxmox を ZFS ミラー（SATA 250GB×2）へインストール

1. インストーラ起動 → **Install Proxmox VE**
2. ターゲットディスク：SATA SSD 2 台を選択し、**ZFS (RAID1)** を選ぶ

   * `ashift=12`（NVMe/HDD と混在でも無難）
3. ロケール・パスワード・メール設定
4. ネットワーク：

   * Management Interface（例: `enp...`）を選択
   * ホスト名：`pve.local` など
   * IPv4：**Static**（例）`192.168.1.10/24`、Gateway `192.168.1.1`、DNS `1.1.1.1` など
5. 再起動 → WebUI にアクセス：`https://192.168.1.10:8006`

> **OS スナップショット/ロールバック**
> Proxmox は ZFS なので、OS もスナップショット可。大きな変更前に：

```bash
# 例：root プール名が rpool の場合
zfs snapshot -r rpool/ROOT/pve-1@pre-change
# ロールバック
zfs rollback -r rpool/ROOT/pve-1@pre-change
```

---

# 2. 物理ディスク確認とプール作成（NVMe×2、HDD×2）

Proxmox シェル（WebUI > ノード > Shell）で：

```bash
# デバイス確認
lsblk -o NAME,SIZE,MODEL,SERIAL
```

## 2-1. NVMe 1TB の 1 枚目 → VM 保管プール `nvme1`

```bash
# 例: デバイスが /dev/nvme0n1
zpool create -o ashift=12 nvme1 /dev/nvme0n1
# VM/コンテナ向けに atime off・圧縮 on
zfs set atime=off nvme1
zfs set compression=zstd nvme1
```

## 2-2. NVMe 1TB の 2 枚目 → 編集素材＆キャッシュ `nvme2`

```bash
# 例: /dev/nvme1n1
zpool create -o ashift=12 nvme2 /dev/nvme1n1
zfs set atime=off nvme2
zfs set compression=zstd nvme2
```

### ZVOL 作成（編集用 VM 向け D/E）

* **D（素材）**：大きい連続読み書き → `volblocksize=64K`
* **E（キャッシュ）**：小さめランダム多め → `volblocksize=16K` か `32K`（ここでは 16K）

```bash
# D: 700GB / 64K
zfs create -V 700G -o volblocksize=64K -o compression=zstd nvme2/d_vol
# E: 300GB / 16K
zfs create -V 300G -o volblocksize=16K -o compression=zstd nvme2/e_vol
```

## 2-3. HDD 4TB×2 → ミラー `hdds`

```bash
# 例: /dev/sda /dev/sdb
zpool create -o ashift=12 hdds mirror /dev/sda /dev/sdb
zfs set compression=zstd hdds
zfs create hdds/pictures
zfs create hdds/documents
```

---

# 3. ストレージを Proxmox に登録（VM 保存先）

WebUI → Datacenter → Storage → **Add → ZFS**

* `nvme1` を **Disk image / ISO / Container** 用として登録
* `nvme2` は ZVOL を **Raw LUN** として VM に直付けするので登録不要（後で VM にアタッチ）
* `local-zfs`（インストール時の rpool）は OS 用。VM は `nvme1` に置く

---

# 4. Samba 共有（hdds を家中から見えるように）

```bash
apt update
apt install -y samba
```

`/etc/samba/smb.conf` の末尾に追記（Proxmox ホストで共有）：

```ini
[pictures]
   path = /hdds/pictures
   browseable = yes
   read only = no
   create mask = 0664
   directory mask = 0775
   valid users = editor viewer
   write list = editor

[documents]
   path = /hdds/documents
   browseable = yes
   read only = no
   create mask = 0664
   directory mask = 0775
   valid users = editor viewer
   write list = editor
```

ユーザー作成（Linux ユーザー＋Samba パス）：

```bash
# Linuxユーザー
adduser --disabled-password --gecos "" editor
adduser --disabled-password --gecos "" viewer
# Samba パスワード
smbpasswd -a editor
smbpasswd -a viewer
systemctl restart smbd
```

> **Windows 側マッピング**
> エクスプローラー → 右クリック「ネットワーク ドライブの割り当て」
> `X:` → `\\192.168.1.10\pictures`（ユーザー: editor / viewer）

---

# 5. ネットワークの固定 IP

## 5-1. Proxmox ホスト（すでに静的ならスキップ）

WebUI → ノード → System → Network で `vmbr0` を静的 `192.168.1.10/24` に。

## 5-2. 各 VM

* **Windows**：アダプタの IPv4 を手動
* **Ubuntu**：`/etc/netplan/*.yaml` 例

```yaml
network:
  version: 2
  ethernets:
    ens18:
      addresses: [192.168.1.21/24]
      gateway4: 192.168.1.1
      nameservers:
        addresses: [1.1.1.1, 8.8.8.8]
```

`sudo netplan apply`

（DHCP 予約が使えるルーターなら、そちらで MAC 予約管理でも OK）

---

# 6. VM 作成（共通のコツ）

* **Firmware**：`OVMF (UEFI)`
* **Machine**：`q35`
* **SCSI Controller**：`VirtIO SCSI`
* **ディスク**：`SCSI` で `nvme1` を選択（Windows は後で VirtIO ドライバ ISO をマウント）
* **NIC**：`VirtIO (paravirtualized)`
* **Display**：GPU パススルーする編集 VM は `None` 推奨（Parsec 前提）

---

# 7. VM-1：開発用 Windows（RDP 用）

1. WebUI → **Create VM**

   * Storage：`nvme1`
   * Disk 80–120GB 目安
   * CD/DVD：Windows ISO と **virtio-win ISO** を 2 枚（後から 2 台目マウントでも可）
2. インストール途中のドライバ画面で「ドライバーの読み込み」→ virtio ISO 内の `viostor/win10(11)` を選択
3. 追加で `NetKVM`（ネットワーク）`Balloon`（任意）も入れておく
4. Windows 起動後：**RDP 有効化**、ユーザー作成、固定 IP 設定
5. 開発環境（Cursor 等）導入

---

# 8. VM-2：自作サイト本体（Ubuntu＋Docker＋cloudflared）

1. Create VM → ISO：Ubuntu Server → Disk：`nvme1`（40–80GB）
2. 初期設定＆固定 IP
3. Docker / Compose：

```bash
sudo apt update
sudo apt install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
 | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo \
"deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
 https://download.docker.com/linux/ubuntu $(. /etc/os-release; echo $UBUNTU_CODENAME) stable" \
| sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
sudo usermod -aG docker $USER
```

4. cloudflared（トンネル公開）※ Cloudflare アカウント側のガイドに沿って `docker run cloudflared` で認証 → `docker compose` で常駐化。

   * 公開したい内部サービス（例：`http://127.0.0.1:8000`）をトンネルの `ingress` に設定

---

# 9. GPU パススルー（RTX 4060 を編集用 Windows VM へ）

## 9-1. ホストは iGPU（Ryzen 9 7900）で表示

特別な設定なしで `amdgpu` が使われます（必要に応じて `nomodeset` を外すなどは不要）。

## 9-2. IOMMU/VFIO 設定（Proxmox ホスト）

```bash
# GRUB に IOMMU を追記
echo 'GRUB_CMDLINE_LINUX_DEFAULT="quiet amd_iommu=on iommu=pt"' \
 | sudo tee /etc/default/grub.d/iommu.cfg
update-grub

# VFIO モジュール
cat << 'EOF' | sudo tee /etc/modules-load.d/vfio.conf
vfio
vfio_pci
vfio_iommu_type1
EOF

# NVIDIA ドライバをホストで使わない（ブラックリスト）
cat << 'EOF' | sudo tee /etc/modprobe.d/blacklist-nvidia.conf
blacklist nvidia
blacklist nvidiafb
blacklist nouveau
EOF

update-initramfs -u
reboot
```

再起動後、デバイス ID 確認：

```bash
lspci -nn | grep -E "VGA|Audio"
# 例: GPU 0000:01:00.0 [10de:xxx], HDMI Audio 0000:01:00.1 [10de:yyy]
```

VFIO にバインド（例の ID を置換）：

```bash
echo "options vfio-pci ids=10de:XXXX,10de:YYYY disable_vga=1" \
 | sudo tee /etc/modprobe.d/vfio-pci-ids.conf
update-initramfs -u
reboot
```

## 9-3. 編集用 Windows VM 作成・割り当て

1. Create VM（Windows ISO）→ Disk は `nvme1` へ（C: 用 120–200GB）
2. **D/E を直付け**：作成済み ZVOL をアタッチ

   * VM を停止 → Hardware → **Add → Disk** → **Use existing volume** → `nvme2/d_vol` を **SCSI** で追加
   * 同様に `nvme2/e_vol` も追加
3. **GPU 割り当て**：Hardware → **Add → PCI Device** → 4060（VGA）と同じ関係の **Audio** デバイスも追加

   * オプション：`All Functions`、`Primary GPU`、`PCI-Express` を有効化
4. OVMF/TPM/UEFI、Machine q35 を確認 → 起動
5. Windows で NVIDIA ドライバを入れる → **Parsec** を導入

> **Defender 除外（速度優先）**

```powershell
Add-MpPreference -ExclusionPath "D:\"
Add-MpPreference -ExclusionPath "E:\"
```

> **Adobe**
> Premiere/After Effects のメディアキャッシュを \**E:\** に設定

---

# 10. 編集用ワークフロー（D→X へワンキー移行）

## 10-1. Windows PowerShell スクリプト（デスクトップに保存）

`Move-ToXByMonth.ps1`（rclone 必須。`rclone.exe` を PATH に置く）

```powershell
# 設定
$SourceRoot = "D:\"
$DestRoot   = "X:\"   # ネットワークドライブ（\\192.168.1.10\pictures）
$RcloneExe  = "rclone"  # 例: rclone が PATH にある場合

# D配下のファイルを走査
$files = Get-ChildItem -Path $SourceRoot -File -Recurse
foreach ($f in $files) {
    $dt = $f.LastWriteTime
    $yyyyMM = "{0}{1:D2}" -f $dt.Year, $dt.Month
    $destDir = Join-Path $DestRoot $yyyyMM

    if (-not (Test-Path $destDir)) {
        New-Item -ItemType Directory -Path $destDir | Out-Null
    }

    # まずコピー（ベリファイ付き）。成功時のみ削除したいので --checksum/--size-only は用途次第で
    & $RcloneExe copy --progress --immutable --multi-thread-streams 4 --transfers 4 `
        ("`"$($f.FullName)`"") ("`"$destDir`"")

    if ($LASTEXITCODE -eq 0) {
        # コピー成功確認（サイズ一致など簡易チェック）
        $destFile = Join-Path $destDir $f.Name
        if ((Test-Path $destFile) -and ((Get-Item $destFile).Length -eq $f.Length)) {
            Remove-Item -Force ("`"$($f.FullName)`"")
        }
    }
}
```

* デスクトップのショートカット作成：
  対象 `powershell.exe -ExecutionPolicy Bypass -File "C:\Users\<あなた>\Desktop\Move-ToXByMonth.ps1"`

## 10-2. Mac から最初の 1TB を直接 X へ

1. Finder → 「移動」→「サーバへ接続」→ `smb://192.168.1.10/pictures` → `editor` で接続
2. 写真アプリから一括書き出し（ファイルへ）
3. **年月フォルダ分け Python スクリプト（macOS）**
   書き出し元（例：`~/temp_export`）→ マウント済み X（例：`/Volumes/pictures`）へ

```bash
python3 << 'PY'
import shutil, os, pathlib, time
SRC = os.path.expanduser("~/temp_export")
DST = "/Volumes/pictures"
for root, _, files in os.walk(SRC):
    for name in files:
        src = os.path.join(root, name)
        try:
            mtime = os.path.getmtime(src)
        except:
            continue
        t = time.localtime(mtime)
        yyyyMM = f"{t.tm_year}{t.tm_mon:02d}"
        dstdir = os.path.join(DST, yyyyMM)
        os.makedirs(dstdir, exist_ok=True)
        dst = os.path.join(dstdir, name)
        if not os.path.exists(dst):
            shutil.copy2(src, dst)
PY
```

---

# 11. SMART 監視 & ZFS スクラブを自動化

```bash
apt install -y smartmontools

# smartd 有効化（自動監視・メールは任意設定）
systemctl enable --now smartd
# 週1の簡易テストをsmartd.confに入れたい場合は編集:
# /etc/smartd.conf の例:
# DEVICESCAN -a -o on -s (S/../../7/02) -m root
# → 毎週日曜 02:00 にショートテスト

# ZFS スクラブ（systemd タイマー）
systemctl enable --now zfs-scrub@hdds.timer   # 週1
systemctl enable --now zfs-scrub@nvme1.timer  # 月1（デフォ周期）
systemctl enable --now zfs-scrub@nvme2.timer  # 月1
```

---

# 12. スナップショットの自動化（便利）

頻繁に戻したいなら：

```bash
apt install -y zfs-auto-snapshot
# デフォ: hourly/daily/weekly/monthly が有効化される
# 適用対象を必要なデータセットに限定する例：
zfs set com.sun:auto-snapshot=true hdds
zfs set com.sun:auto-snapshot=true hdds/pictures
zfs set com.sun:auto-snapshot=true hdds/documents
```

---

# 13. 最後の仕上げ（細部のチェックリスト）

* [ ] **Windows（編集用）**

  * Parsec 導入・ログイン
  * NVIDIA ドライバ最新化・NVENC/AV1 有効（Parsec 設定でも H.264/AV1 を選択可）
  * `D:\`（素材）`E:\`（キャッシュ）を確認、Defender 除外済み
* [ ] **Windows（開発用）**

  * RDP 有効・固定 IP
  * Cursor / Docker Desktop など好みの開発ツール
* [ ] **Ubuntu（本体）**

  * Docker/Compose 導入
  * cloudflared トンネルで公開（ドメイン・ゼロトラスト設定）
* [ ] **Samba**

  * `\\192.168.1.10\pictures` / `\documents` に `editor`/`viewer` でアクセス検証
  * 編集用 VM で `X:` マウント
* [ ] **バックアップ**

  * `hdds` の外部バックアップ（外付けやクラウド）検討
* [ ] **電源管理**

  * Proxmox の APCI シャットダウン、停電対策（UPS 任意）

---

## もし IP 例・サイズの「私仕様」に置き換えたテンプレが必要なら、あなたのサブネットや希望 IP を教えてください。すぐにその値で書き直します。
