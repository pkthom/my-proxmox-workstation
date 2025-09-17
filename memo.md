# 0) ProxmoxインストーラーUSBを作る

[ISO置き場](http://proxmox.com/en/downloads/proxmox-virtual-environment/iso)から、
[最新版](https://enterprise.proxmox.com/iso/proxmox-ve_9.0-1.iso)をダウンロード

[公式 インストールメディアの作り方](https://pve.proxmox.com/wiki/Prepare_Installation_Media)の中の、
[MacOS版](https://pve.proxmox.com/wiki/Prepare_Installation_Media#_instructions_for_macos)を参照

```
pkthom@MacBook-Pro Downloads % hdiutil convert proxmox-ve_9.0-1.iso -format UDRW -o proxmox-ve_9.0-1.dmg
Reading Driver Descriptor Map (DDM : 0)…
Reading PVE                              (Apple_ISO : 1)…
Reading Apple (Apple_partition_map : 2)…
Reading PVE                              (Apple_ISO : 3)…
Reading Gap0 (ISO9660_data : 4)…
.
Reading HFSPLUS_Hybrid (Apple_HFS : 5)…
......................................................................................................................................................
Reading Gap1 (ISO9660_data : 6)…
......................................................................................................................................................
Elapsed Time:  3.124s
Speed: 501.1MB/s
Savings: 0.0%
created: /Users/pkthom/Downloads/proxmox-ve_9.0-1.dmg
pkthom@MacBook-Pro Downloads % diskutil list
/dev/disk0 (internal, physical):
   #:                       TYPE NAME                    SIZE       IDENTIFIER
   0:      GUID_partition_scheme                        *1.0 TB     disk0
   1:                        EFI EFI                     314.6 MB   disk0s1
   2:                 Apple_APFS Container disk1         1.0 TB     disk0s2

/dev/disk1 (synthesized):
   #:                       TYPE NAME                    SIZE       IDENTIFIER
   0:      APFS Container Scheme -                      +1.0 TB     disk1
                                 Physical Store disk0s2
   1:                APFS Volume Macintosh HD - Data     966.1 GB   disk1s1
   2:                APFS Volume Preboot                 273.8 MB   disk1s2
   3:                APFS Volume Recovery                1.1 GB     disk1s3
   4:                APFS Volume VM                      1.1 GB     disk1s4
   5:                APFS Volume Macintosh HD            21.6 GB    disk1s5
   6:              APFS Snapshot com.apple.os.update-... 21.6 GB    disk1s5s1

/dev/disk2 (external, physical):
   #:                       TYPE NAME                    SIZE       IDENTIFIER
   0:     FDisk_partition_scheme                        *15.6 GB    disk2
   1:               Windows_NTFS UNTITLED                15.5 GB    disk2s1

pkthom@MacBook-Pro Downloads % diskutil unmountDisk /dev/disk2
Unmount of all volumes on disk2 was successful
pkthom@MacBook-Pro Downloads % sudo dd if=proxmox-ve_9.0-1.dmg bs=1M of=/dev/rdisk2
Password:
1565+1 records in
1565+1 records out
1641615360 bytes transferred in 102.427183 secs (16027145 bytes/sec)
pkthom@MacBook-Pro Downloads % echo $?
0
pkthom@MacBook-Pro Downloads % sync
pkthom@MacBook-Pro Downloads % diskutil eject /dev/disk2
Disk /dev/disk2 ejected
pkthom@MacBook-Pro Downloads %
```


了解。下の順番で“いまBIOSが開いているところから”ゴール（3つのVM稼働＋共有と自動運用）まで一気に行きます。
（コマンドはそのままコピペでOK。`<>`はあなたの環境で置換してください）

---

# 0) 事前のIP設計（固定IP）

例として以下で進めます。ご家庭のLANに合わせて変えてOK。

* ルーター（ゲートウェイ）：`192.168.50.1/24`
* Proxmoxホスト：`192.168.50.10`
* VM① Windows「開発」：`192.168.50.21`
* VM② Ubuntu「本体/サイト」：`192.168.50.22`
* VM③ Windows「動画編集」：`192.168.50.23`
* DNS：`192.168.50.1`（または 1.1.1.1/8.8.8.8）

---

# 1) BIOS設定（いまここ）

* **Primary Display / Integrated Graphics**：内蔵iGPUを**優先**
* **CSM**：Disabled（UEFIオンリー）
* **SVM (AMD-V)**：Enabled
* **IOMMU**（AMD IOMMU / SVIOMMU）：Enabled
* **Above 4G Decoding**：Enabled
* **Re-Size BAR**：Enabled
* **SATA Mode**：AHCI
* **Boot**：USBインストーラを最優先（UEFIの方）
* メモリはEXPO/XMPを有効（安定しない場合はAutoに戻す）

保存してUSBから起動。

---

# 2) ProxmoxをZFSミラーへインストール（SATA SSD 250GB×2）

1. インストーラで **Install Proxmox VE** → **Target** を **ZFS** 選択

   * **RAID1（Mirror）** を選び、2台のSATA SSDを指定
   * **ashift=12**（4Kセクタ想定）、**Compression lz4**（後でzstdに変更可）
   * ホスト名：`pve.local` など
2. **IP設定**：ここで**静的IP**にしておく（上の設計通り）

   * IP：`192.168.50.10/24`、GW：`192.168.50.1`、DNS：`192.168.50.1`
3. インストール完了→再起動→`https://192.168.50.10:8006/` にログイン

以降はWeb UIかSSHで作業。

---

# 3) Proxmox初期設定 & 更新

```bash
# 最新化
apt update && apt -y full-upgrade

# ZFSチューニング（任意）
zpool set autotrim=on rpool
# rpoolはOSプール、圧縮は既定lz4のままでOK（必要ならzstdへ）
```

---

# 4) NVMeプール作成（VM格納用・編集用）

**デバイス名は“by-id”で指定**（リネーム事故防止）。まず確認：

```bash
ls -l /dev/disk/by-id/
```

## 4-1) VM格納用プール（nvme1）

```bash
zpool create -o ashift=12 -o autotrim=on nvme1 /dev/disk/by-id/<NVME1-ID>
zfs set compression=zstd-3 nvme1
```

## 4-2) 編集素材/キャッシュ用プール（nvme2）

```bash
zpool create -o ashift=12 -o autotrim=on nvme2 /dev/disk/by-id/<NVME2-ID>
zfs set compression=zstd-3 nvme2
```

## 4-3) Proxmoxにストレージ登録（GUI）

`Datacenter > Storage > Add > ZFS`

* ID: `zfs-nvme1` / `zfs-nvme2`
* Pool: `nvme1` / `nvme2`
* Content: **Disk image, ISO, Container, Snippets**（少なくとも Disk image はオン）

---

# 5) HDDミラープール作成（4TB×2：hdds）

```bash
zpool create -o ashift=12 hdds mirror /dev/disk/by-id/<HDD1-ID> /dev/disk/by-id/<HDD2-ID>
zfs set autotrim=on hdds
zfs set compression=zstd-3 hdds
```

**データセット**（大容量向けにatime OFF, 大きめrecordsize）：

```bash
zfs create -o atime=off -o recordsize=1M hdds/pictures
zfs create -o atime=off -o recordsize=1M hdds/documents
```

マウントポイントは `/hdds/pictures`, `/hdds/documents` になります。

Proxmoxに**共有用**としても登録（任意）：
`Datacenter > Storage > Add > Directory`

* ID: `hdds-files`
* Directory: `/hdds`
* Content: **ISO/Backup**など任意（共有自体はSambaでやります）

---

# 6) Samba（SMB）共有設定（editor=RW, viewer=RO）

```bash
apt -y install samba acl
groupadd smbshare
# Linuxユーザー（ログイン不可）
useradd -M -s /usr/sbin/nologin editor
useradd -M -s /usr/sbin/nologin viewer
usermod -aG smbshare editor
# パスワード（SMB用）
smbpasswd -a editor
smbpasswd -a viewer

# 所有権とACL
chown -R root:smbshare /hdds/pictures /hdds/documents
chmod -R 2775 /hdds/pictures /hdds/documents
setfacl -R -m g:smbshare:rwx /hdds/pictures /hdds/documents
setfacl -R -m u:viewer:rx /hdds/pictures /hdds/documents
```

`/etc/samba/smb.conf` 末尾に追記（**share名はURLで使う名前**。Windowsは `\\<IP>\pictures` 形式でアクセスします）：

```ini
[pictures]
   path = /hdds/pictures
   browsable = yes
   read only = no
   valid users = editor viewer
   write list = editor
   force group = smbshare
   create mask = 0664
   directory mask = 2775
   vfs objects = acl_xattr
   inherit permissions = yes

[documents]
   path = /hdds/documents
   browsable = yes
   read only = no
   valid users = editor viewer
   write list = editor
   force group = smbshare
   create mask = 0664
   directory mask = 2775
   vfs objects = acl_xattr
   inherit permissions = yes
```

再起動：

```bash
systemctl restart smbd
```

> 以後、Windows/macOSから：
> `\\192.168.50.10\pictures` / `\\192.168.50.10\documents`（Windows）
> `smb://192.168.50.10/pictures`（Mac）
> でアクセス。**Xドライブは `\\192.168.50.10\pictures` に割り当て**してください。

---

# 7) ネットワーク（Proxmoxブリッジの固定IP）

Proxmoxは標準で `vmbr0` があるはず。確認・修正（**ifupdown2**形式）：

```bash
nano /etc/network/interfaces
```

例：

```ini
auto lo
iface lo inet loopback

auto enp3s0
iface enp3s0 inet manual

auto vmbr0
iface vmbr0 inet static
    address 192.168.50.10/24
    gateway 192.168.50.1
    bridge-ports enp3s0
    bridge-stp off
    bridge-fd 0
    dns-nameservers 192.168.50.1 1.1.1.1
```

適用：

```bash
ifreload -a
```

---

# 8) VM用zvolの作成（OSディスク in nvme1、編集用D/E in nvme2）

（Proxmox GUIでVM作成時に自動作成でもOKですが、名前を揃えたい場合は先に作ると管理しやすい）

```bash
# OSディスク（サイズは必要に応じて）
zfs create -V 200G -o volblocksize=16K nvme1/win-dev-os
zfs create -V 120G -o volblocksize=16K nvme1/ubuntu-site-os
zfs create -V 220G -o volblocksize=16K nvme1/win-edit-os

# 編集用データ／キャッシュ（大きめブロック。64K〜128K）
zfs create -V 700G -o volblocksize=64K nvme2/d_vol
zfs create -V 300G -o volblocksize=64K nvme2/e_vol
```

---

# 9) VM① Windows「開発」作成（RDPで使う）

**ISO**：Windows 11 (x64 24H2 など)、**virtio-win ISO**もアップロード（`local`のISO領域へ）

GUI: `Create VM`

* **General**：Name `win-dev`
* **OS**：Guest OS: Microsoft Windows 11、**UEFI (OVMF)**、**TPM** 有効（Windows 11要件）
* **System**：Machine `q35`、SCSI Controller `VirtIO SCSI`、**QEMU Agent**チェック
* **Disks**：Bus/Device `VirtIO SCSI`, Storage=`zfs-nvme1`, Disk=既存zvolを指定（`nvme1/win-dev-os`）でもOK
* **CPU**：Type `host`、コア例 `4`（後で増減可）
* **Memory**：例 `10GB`（10240MB）
* **Network**：Bridge `vmbr0`、Model `VirtIO (paravirtualized)`
* **CD/DVD**：Windows ISO と **virtio-win ISO** の2枚をアタッチ（CD/DVDドライブ2つ）

インストール中、**ドライバの読み込み**→virtio ISOから

* ストレージ（vioscsi / viostor）、ネットワーク（viogp3/virtio-net）を入れる

**Windows内で固定IPに設定**

* アダプタのIPv4に `192.168.50.21/24`, GW `192.168.50.1`, DNS `192.168.50.1`

**RDP有効化**

* 設定 → システム → リモートデスクトップ → 有効化

---

# 10) VM② Ubuntu「本体/サイト」（Docker & cloudflared）

GUI: `Create VM`

* Name `ubuntu-site`
* Guest OS：Linux、UEFI(OVMF)可
* Disk：`zfs-nvme1` に 既存zvol `nvme1/ubuntu-site-os`
* CPU：コア例 `4`
* Mem：例 `8GB`
* NIC：VirtIO, Bridge `vmbr0`, QEMU Agent ON

Ubuntuインストール後：

```bash
# 静的IP（netplan例）
sudo nano /etc/netplan/01-net.yaml
```

```yaml
network:
  version: 2
  ethernets:
    ens18:
      addresses: [192.168.50.22/24]
      gateway4: 192.168.50.1
      nameservers:
        addresses: [192.168.50.1,1.1.1.1]
```

```bash
sudo netplan apply

# Docker
sudo apt update
sudo apt -y install ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo \
"deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
 https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo $UBUNTU_CODENAME) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt -y install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo usermod -aG docker $USER
```

（cloudflaredトンネルは後で必要になったら手順書きます）

---

# 11) GPUパススルー（RTX 4060 → VM③「動画編集」）

## 11-1) ホスト側設定

```bash
# IOMMUをカーネルパラメータに（AMD）
nano /etc/default/grub
```

`GRUB_CMDLINE_LINUX_DEFAULT` に追加：

```
quiet amd_iommu=on iommu=pt
```

反映：

```bash
update-grub
echo "vfio" > /etc/modules-load.d/vfio.conf
echo "vfio-pci" >> /etc/modules-load.d/vfio.conf
echo "vfio_virqfd" >> /etc/modules-load.d/vfio.conf
update-initramfs -u
reboot
```

再起動後、GPUのPCIアドレス確認：

```bash
lspci -nn | grep -i nvidia
# 例: 01:00.0 VGA 10de:xxxx / 01:00.1 Audio 10de:yyyy
```

（必要なら `vfio-pci.ids=10de:xxxx,10de:yyyy` をGRUBに追記 → `update-grub`）

**重要**：BIOSで**iGPUをメイン表示**にしているため、ホストがdGPUを掴まずに済みます。念のためNVIDIAドライバはホストに入れない／黒曜化不要。

## 11-2) VM③ Windows「動画編集」作成

* Name `win-edit`
* OS：Windows 11、UEFI(OVMF)+TPM、Machine=q35、QEMU Agent ON
* CPU：例 `8` コア（必要に応じて10〜12でもOK）
* Mem：例 `24〜32GB`
* Disk(OS)：`nvme1/win-edit-os`（VirtIO SCSI）
* NIC：VirtIO, Bridge `vmbr0`
* CD/DVD：Windows ISO + virtio-win ISO

**ハードウェアにGPU追加**
GUIで `Add > PCI Device` から **GPU本体(01:00.0)** と **同Audio関数(01:00.1)** を**両方**追加

* `All Functions` ON、`Primary GPU` は**OFF**（VM内表示に使う訳ではない）
* `PCI-Express` ON
* `Resizable BAR` はWindows側でも有効可

インストール後、WindowsでNVIDIAドライバを通常通りインストール。

## 11-3) 編集用D/EドライブをVMに接続

GUIで `Hardware > Add > Hard Disk`

* **Bus/Device: VirtIO SCSI**
* Storage：`zfs-nvme2` の **Existing Volume** から

  * `nvme2/d_vol` → ドライブレター **D:**
  * `nvme2/e_vol` → ドライブレター **E:**（Adobeのキャッシュ先）
    Windowsのディスクの管理で**GPT/NTFS**でフォーマット。

**Defender除外**（速度優先）：

```powershell
Add-MpPreference -ExclusionPath 'D:\','E:\'
```

**Parsec** をインストール（Macから低遅延で操作）。

---

# 12) Xドライブ（SMB共有）を動画編集VMに割り当て

**X:** を `\\192.168.50.10\pictures` に固定

```powershell
# 管理者PowerShell
$pass = Read-Host -AsSecureString "Enter SMB password for editor"
$cred = New-Object System.Management.Automation.PSCredential("editor",$pass)
cmdkey /add:192.168.50.10 /user:editor /pass:$( [Runtime.InteropServices.Marshal]::PtrToStringAuto([Runtime.InteropServices.Marshal]::SecureStringToBSTR($cred.Password)) )
New-PSDrive -Name X -PSProvider FileSystem -Root "\\192.168.50.10\pictures" -Persist
```

---

# 13) “1クリックで整理＆転送” PowerShellスクリプト（D → X：YYYYMM）

要件：

* D内の各ファイルの**最終更新日時**で `YYYYMM` フォルダに仕分け
* **コピー成功をハッシュで検証後**、元ファイル削除（安全）

**前提**：Windowsに rclone をインストールし、`rclone.exe` がPATHにあること。

`C:\Users\Public\move_to_X.ps1` として保存：

```powershell
param(
  [string]$SourceRoot = "D:\",
  [string]$DestRoot = "X:\"
)

$Log = "$env:USERPROFILE\Desktop\move_to_X.log"
"=== Run $(Get-Date) ===" | Out-File -Append $Log -Encoding utf8

# 対象拡張子は全て（必要なら -Include で制限）
Get-ChildItem -Path $SourceRoot -File -Recurse | ForEach-Object {
  $src = $_.FullName
  $dt  = $_.LastWriteTime
  $ym  = "{0:yyyyMM}" -f $dt
  $destDir = Join-Path $DestRoot $ym
  if (!(Test-Path $destDir)) { New-Item -ItemType Directory -Path $destDir | Out-Null }

  $dest = Join-Path $destDir $_.Name

  # rcloneでコピー（ローカル→SMBもOK）
  $copyCmd = "rclone copyto `"$src`" `"$dest`" --progress --fsync"
  cmd /c $copyCmd

  # ハッシュ検証（SHA256）
  try {
    $h1 = (Get-FileHash -Algorithm SHA256 -Path $src).Hash
    $h2 = (Get-FileHash -Algorithm SHA256 -Path $dest).Hash
    if ($h1 -eq $h2) {
      Remove-Item -Path $src -Force
      "OK: $src -> $dest (deleted source)" | Out-File -Append $Log -Encoding utf8
    } else {
      "HASH MISMATCH: $src" | Out-File -Append $Log -Encoding utf8
    }
  } catch {
    "ERROR: $src $_" | Out-File -Append $Log -Encoding utf8
  }
}
```

**デスクトップにショートカット**を作成して、ターゲットに
`powershell.exe -ExecutionPolicy Bypass -File "C:\Users\Public\move_to_X.ps1"`
を指定すれば**1クリック**で動きます。

---

# 14) Mac →（初回のみ）直接 X（SMB：pictures）へ1TB一括投入

1. Finder → 移動 → サーバへ接続 → `smb://192.168.50.10/pictures`
   （ユーザー：**editor**）
2. 写真.app：**ファイル > 書き出す > ○項目の書き出し**（オリジナル推奨）
3. “年月フォルダ”に整えるため、**Mac側スクリプト**で `YYYYMM` に振り分け
   例：`organize_yyyymm.py`（標準のPython 3で動きます）

   ```python
   # 保存場所例: ~/Desktop/organize_yyyymm.py
   # 使い方: python3 organize_yyyymm.py "/Volumes/pictures" "/path/to/exported_photos"
   import sys, shutil
   from pathlib import Path
   from datetime import datetime

   dest_root = Path(sys.argv[1])  # 例: /Volumes/pictures
   src_root  = Path(sys.argv[2])  # 写真を書き出したローカルフォルダ

   for p in src_root.rglob('*'):
       if p.is_file():
           dt = datetime.fromtimestamp(p.stat().st_mtime)
           ym = dt.strftime('%Y%m')
           dstdir = dest_root / ym
           dstdir.mkdir(parents=True, exist_ok=True)
           dst = dstdir / p.name
           i = 1
           while dst.exists():
               dst = dstdir / f"{p.stem}_{i}{p.suffix}"
               i += 1
           shutil.copy2(p, dst)
   print("Done")
   ```

   実行例：

   ```bash
   python3 ~/Desktop/organize_yyyymm.py /Volumes/pictures /Users/<you>/Pictures/Exported
   ```

   ※ `/Volumes/pictures` はSMBでマウントした共有

---

# 15) VM③の運用：Adobeのキャッシュ先を **E:** に設定

* Premiere/After Effects の **メディアキャッシュ**/スクラッチディスクを **E:** に
* 作業素材は **D:** に置く（編集後はショートカットで **X:** へ退避）

---

# 16) VM①（開発）からVM②（Ubuntu）へSSH（Cursor等）

* Ubuntu側でOpenSSHサーバ：`sudo apt -y install openssh-server`
* 開発VM（Windows）から `ssh ubuntu@192.168.50.22`
* Dockerのデプロイ等はComposeで

---

# 17) SMART監視 & 週1 scrub 自動化

**SMART（HDD）**

```bash
apt -y install smartmontools
# 自動監視・通報はsmartdで有効（必要なら /etc/smartd.conf で /dev/disk/by-id/<HDD> を監視）
systemctl enable --now smartd
```

**scrub（週1）**：ZFSにはtimerが用意されています

```bash
systemctl enable --now zfs-scrub@hdds.timer
systemctl enable --now zfs-scrub@rpool.timer
# nvme1/nvme2も回すなら:
systemctl enable --now zfs-scrub@nvme1.timer
systemctl enable --now zfs-scrub@nvme2.timer
systemctl list-timers | grep scrub
```

**手動で実行**：

```bash
zpool scrub hdds
zpool status hdds
```

破損は**ミラーの健全コピーで自動修復**されます。

---

# 18) Parsec / RDP 接続の整理

* **編集VM**：Parsecで低遅延（Macから）
* **開発VM**：RDP（低スペPCから）
* **Ubuntu**：SSH/VSCode/LLM補助など好みで

---

# 19) リソース配分の目安（同時起動を想定）

Ryzen 9 7900 = 12C/24T, メモリ 128GB なので余裕あり

* **編集VM**：vCPU 8〜12、RAM 24〜32GB、GPU=RTX4060、D/E接続
* **開発VM**：vCPU 4、RAM 8〜12GB
* **Ubuntu**：vCPU 4、RAM 8〜16GB
  足りなければ後から調整してください。`Ballooning`は**OFF**推奨（予測可能な性能のため）。

---

# 20) 仕上げチェック（必ず実施）

* `zpool status`（全プールがONLINE）
* `pvecm status`（シングルノードOK）
* 各VMから互いにping（`192.168.50.x`）
* Windowsの**Defender除外**が効いているか再確認（D:, E:）
* **X: ドライブ**がログイン後自動でマウントされるか
* Parsec/RDPの描画と音声遅延

---

必要なら、cloudflared（サイト公開）や、バックアップ（Proxmoxバックアップ→`hdds`や外付け、さらに外部クラウド）も後追いで整えましょう。ここまで完了すれば、**OSはZFSミラーで堅牢、VM群はNVMeで高速、編集素材はD/E直付け、納品/保管はX（SMBミラー）へ整理**という狙い通りの構成で運用できます。

