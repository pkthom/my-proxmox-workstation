# Proxmoxワークステーション構築手順（初心者向け・実運用版）

> **目的**: 1台の自作PCに Proxmox VE を入れ、
> ① Windows 動画編集 VM（RTX 3060 パススルー）と ② Windows 開発 VM（Docker Desktop/Cursor）を **同時にサクサク**動かす。
> さらに **ZFS スナップショットで安全運用**、**Parsec で快適リモート**、**ECC 有効化確認**まで含む。

---

## 目次

* [1. 想定ハード構成](#1-想定ハード構成)
* [2. 配線と初回BIOS設定](#2-配線と初回bios設定)
* [3. Proxmoxのインストール（ZFSミラー）](#3-proxmoxのインストールzfsミラー)
* [4. NVMeプールとZVOLの作成](#4-nvmeプールとzvolの作成)
* [5. ストレージ設計（本書の前提）](#5-ストレージ設計本書の前提)
* [6. Windows開発VMの作成（Docker/Cursor）](#6-windows開発vmの作成dockercursor)
* [7. GPUパススルー：Windows編集VMの作成](#7-gpuパススルーwindows編集vmの作成)
* [8. Parsec と NLE（Premiere/DaVinci）最適化](#8-parsec-と-nlepremieredavinci最適化)
* [9. ECC 有効化の確認](#9-ecc-有効化の確認)
* [10. ZFS スナップショットとロールバック](#10-zfs-スナップショットとロールバック)
* [11. バックアップ（簡易運用）](#11-バックアップ簡易運用)
* [12. リソース割り当てと“カクつき”防止](#12-リソース割り当てとカクつき防止)
* [13. 将来拡張（128GB/K8s/RKE2）](#13-将来拡張128gbk8srke2)
* [14. トラブルシューティング](#14-トラブルシューティング)
* [15. 付録：よく使うコマンド集](#15-付録よく使うコマンド集)

---

## 1. 想定ハード構成

* **CPU**: AMD Ryzen 9 7900（iGPU 付き / 付属Wraith Prism使用）
* **MB**: ASUS TUF GAMING B650-PLUS WIFI
* **RAM**: Kingston Server Premier ECC UDIMM DDR5-4800 32GB ×2 = **64GB**
* **SATA SSD**: Crucial MX500 250GB ×2 → **ZFSミラー（OS用）**
* **NVMe SSD**: Crucial T500 1TB ×2
* **GPU**: MSI GeForce RTX 3060 12GB（編集VMへパススルー）
* **PSU**: Corsair RM850e 850W（将来余力）
* **Case/Fan**: Corsair 3000D（前面2基 + **リア1基追加**推奨）
* **ネットワーク**: 1GbE 有線（クライアントPCも **必ず有線**）

> 7年間運用を想定。将来は **RAM 128GB**、**キャッシュ用ZVOL 1TB**、（必要なら）**40番台GPU**へ。

---

## 2. 配線と初回BIOS設定

1. **配線**

* RTX3060 を **下段PCIe**へ（上段でもOK。iGPUをホスト出力に使う）。
* SATA SSD 2台は同一規格ポートへ。NVMeは M.2\_1 / M.2\_2 に装着。
* 前面ファン→吸気、リア（必要なら天面）→排気。

2. **BIOS 更新**（ASUS EZ Flash）

* 最新BIOSに更新し、初期化（Load Optimized Defaults）。

3. **仮想化・IOMMU**

* `SVM` = Enabled
* `IOMMU`/`AMD IOMMU` = Enabled

4. **GPU関連**

* `Primary Display` = **iGPU**（ホスト出力を内蔵GPUに固定）
* `Above 4G Decoding` = Enabled
* `Re-Size BAR` = Enabled
* `CSM` = Disabled

5. **メモリ**

* 初期は JEDEC 4800MT/s（EXPOやOCは安定運用の後で）

6. **ストレージ**

* SATA Mode = AHCI

7. **ファン**

* CPU/CHA ファンカーブを静音〜標準（温度60–70℃付近で回転上昇）

---

## 3. Proxmoxのインストール（ZFSミラー）

1. **インストールUSB** を作成 → 起動
2. **ターゲット**: MX500 250GB ×2 を選択
3. **ファイルシステム**: `ZFS (RAID1)` を選択

   * オプションで `ashift=12`（SSD最適）
   * オプションで `Compression=zstd`*
4. 管理ネットワーク（vmbr0）/ホスト名/時刻を設定 → インストール完了

※ ProxmoxのZFSミラー構成において、zstd圧縮が推奨されるのは、高速でありながら高い圧縮率を持つためです。\
これにより、ディスク容量を節約しつつ、パフォーマンス向上とSSDの寿命延長という複数のメリットを得られます。

> **初期ログイン**: https\://\<Proxmox管理IP>:8006 へブラウザからアクセス

5. リポジトリ アップデート
```
# 推奨: コミュニティ（no-subscription）リポジトリへ切替
sed -i 's/enterprise/no-subscription/' /etc/apt/sources.list.d/pve-enterprise.list || true
cat >/etc/apt/sources.list.d/pve-no-subscription.list <<'EOF'
deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription
EOF
apt update && apt -y full-upgrade
reboot
```

6. ZFS ARCを控えめに
```
# 例: 上限 10GiB（64GB RAM の場合）
cat >/etc/modprobe.d/zfs-arc.conf <<'EOF'
options zfs zfs_arc_max=10737418240
EOF
update-initramfs -u
reboot
```

---

## 4. NVMeプールとZVOLの作成

**方針**: NVMe 2枚を別々の **単一ディスクZFSプール** にして役割分離。

* NVMe#1 → `nvme_media`（編集素材用 zvol を作る）
* NVMe#2 → `nvme_fast`（キャッシュ用 zvol + 開発VM用のZFSストレージ）

```
[Client PCs over LAN]
├─ (A) 古いLenovo → RDP → 開発VM (Windows)
└─ (B) 古いMacBook Pro → Parsec → 編集VM (Windows, RTX3060)


[Proxmox Host]
├─ rpool (ZFS mirror on SATA SSD 250GB x2) ← OS/boot, snapshots
├─ pool_media (NVMe #1 1TB, single)        ← ZVOL: media_zvol (D:)
└─ pool_fast (NVMe #2 1TB, single)
       ├─ ZVOL: cache_zvol 500G (E:)
       └─ DATASET: pool_fast/vmdata (残り ~500G, Proxmox ストレージ)
```


### 4.1 プール作成（GUI推奨）

* `Node` → 該当ノード → `Disks` → `ZFS` → `Create: ZFS` で個別に作成

  * `name=nvme_media`, `ashift=12`, デバイス=`/dev/disk/by-id/nvme-XXXX`
  * `name=nvme_fast`,  `ashift=12`, デバイス=`/dev/disk/by-id/nvme-YYYY`

> CLI派は下記（**デバイス名は必ず確認**）

```bash
# 例: 実デバイス確認
lsblk -o NAME,MODEL,SIZE,TYPE,MOUNTPOINT
ls -l /dev/disk/by-id/ | grep nvme

# プール作成
zpool create -f -o ashift=12 pool_media /dev/disk/by-id/nvme-XXXX
zpool create -f -o ashift=12 pool_fast  /dev/disk/by-id/nvme-YYYY
```
### ※ ZFSにおける ashift=12 の意義について

現在流通しているほとんどのSSDやHDDは、データの書き込みを効率的に行うため、内部的に**4K（4096バイト）**の物理セクターを採用している。
ZFSでは、プールを作成する際にこの物理セクターサイズを正しく認識させることが、ストレージの性能と寿命を最大限に引き出す上で非常に重要となる。
`ashift=12` というオプションは、ZFSがこの4Kセクターを適切に認識し、4K単位でデータを読み書きするための設定。

この設定により、以下の利点がもたらされます。
- SSDの寿命延長: ストレージの物理的なブロックサイズとZFSの書き込み単位が一致するため、無駄な書き換え作業（ライトアンプリフィケーション）が抑制されます。これにより、SSDへの書き込み回数が減り、製品の寿命を延ばすことが期待できます。
- パフォーマンスの最適化: データの入出力（I/O）が効率的に行われるようになり、ZFS全体のパフォーマンスが向上します。

以下で確認可能
```
zpool get all pool_name
```

### 4.2 ZVOL 作成（素材/キャッシュ）

**素材（メディア）用 ZVOL**（約 900GB）

```bash
zfs create -V 900G -o volblocksize=128K -o compression=lz4 -o atime=off -o logbias=throughput pool_media/media_zvol
```

- `volblocksize=128K` について

理由: 動画編集の素材ファイルは、数MBからGB単位の大きなデータがまとまっており、小さな単位で読み書きすると効率が悪くなります。\
128Kは、一般的なOSのブロックサイズである64Kより大きく、この種の大きなファイルの読み書きに最適化されています。

他の選択肢ではない理由: これより小さなvolblocksizeを選ぶと、大きなファイルの読み書き回数が増え、パフォーマンスが低下する可能性があります。\
逆に大きすぎると、小さなファイルを扱う際に領域の無駄が生じるため、動画編集用途では128Kがバランスの良い値とされています。

- `compression=lz4` について
  
理由: lz4は、ZFSで最も高速な圧縮アルゴリズムです。動画ファイルは、すでにMP4などの形式で高度に圧縮されていることがほとんどで、再圧縮しても容量はほとんど減りません。
そのため、容量削減よりも圧縮・展開の速度を最優先し、編集作業中のタイムラグを最小限に抑えることを目的としています。

他の選択肢ではない理由: zstdは圧縮率が高いですが、lz4よりわずかに遅く、CPUへの負荷も高くなります。\
動画編集では数GB単位のデータを扱うため、このわずかな差が体感的な速度に影響する可能性があります。

**キャッシュ用 ZVOL**（500GB）

```bash
zfs create -V 500G -o volblocksize=64K -o compression=off -o atime=off -o logbias=throughput pool_fast/cache_zvol
```

- `volblocksize=64K` について

理由: キャッシュには、動画の一時的なプレビューデータ、レンダリングデータ、プロジェクトのメタデータなど、様々なサイズのデータが混在します。\
128Kのような大きなブロックサイズに最適化するよりも、64Kというより一般的なサイズにすることで、多様なデータに柔軟に対応し、バランスの取れた性能を確保します。

他の選択肢ではない理由: 32Kなど小さすぎると、大きなデータの読み書きが非効率になります。動画編集のキャッシュは主に大きなデータを扱うため、64Kが適しています。

- `compression=off` について

理由: キャッシュは一時的なデータを高速に読み書きするためのもので、容量を節約することよりも、最高速のパフォーマンスを確保することが最優先されます。\
圧縮処理にはわずかでもCPUリソースと時間が必要なため、この処理を完全に無効化することで、キャッシュへの書き込み速度を最大限に高めます。

他の選択肢ではない理由: lz4やzstdを有効にすると、データ圧縮のためにわずかな処理時間が加算されます。\
これは、キャッシュの目的である「瞬時の読み書き」と相反するため、この用途では圧縮をオフにするのがベストプラクティスです。

**開発VM 用ストレージ**（ZFS データセット）

```bash
zfs create -o compression=lz4 -o atime=off -o mountpoint=/pool_fast/vmdata pool_fast/vmdata
```

lz4の理由：lz4は、zstdに比べて圧縮率は劣りますが、圧倒的に速く、CPUへの負荷も最小限です。これらの特性は、頻繁なI/Oが発生する開発環境のVMと非常に相性が良いです。
開発VMのデータセットには、容量の節約よりも速度とレスポンスが重要です。そのため、最高速のlz4圧縮を選ぶのが最善の選択と言えます。

zstdでない理由：zstdは高い圧縮率を持つ一方で、lz4よりわずかに遅く、CPUリソースを消費します。開発環境のように常にI/Oが発生し、最高速が求められる用途では、このわずかな遅延が体感的なパフォーマンス低下につながる可能性があります。

atime=off: ファイルのアクセス時間を記録する機能をオフにします。これにより、無駄な書き込みが減り、SSDの寿命を延ばす効果があります。



Proxmox GUI → `Datacenter > Storage > Add > ZFS` で `pool_fast`（or `pool_fast/vmdata`）を **Directory ストレージ**として追加。



> **補足**: キャッシュ容量は将来 **1TB**へ拡張推奨（複数プロジェクトでも詰まりにくくなります）。

---

## 5. ストレージ設計（本書の前提）

* **OS**: `rpool`（MX500ミラー）
* **編集素材**: `nvme_media/media_zvol` を **編集VMへ直付け**（D: など）
* **編集キャッシュ**: `nvme_fast/cache_zvol` を **編集VMへ直付け**（E: など）
* **開発VM**: `nvme_fast/vmdata` 上にディスクを作成

> ZVOL は **VirtIO-SCSI Single + IOThread=ON + SSDエミュ + Discard=ON** を基本。Windows側は **NTFS 64K クラスタ**推奨。

---

## 6. Windows開発VMの作成（Docker/Cursor）

1. **ISO登録**: `Datacenter > Storage > local` の `ISO Images` に Windows ISO と VirtIO ドライバ ISO をアップロード
2. **VM作成**（例: VMID=200, Name=dev-win）

   * **OS**: Windows 11 ISO / UEFI (OVMF), Q35
   * **System**: Q35, SCSI Controller=VirtIO SCSI single, **Qemu Agent=ON**, **NUMA=ON**, CPU type=host
   * **Disk**: Bus=SCSI, Storage=`nvme_fast`（または `nvme_fast/vmdata`）, Size=200GB, **IOThread=ON**, SSD, Discard
   * **CPU**: 6 vCPU（3 sockets ×2 cores など）
   * **Memory**: 20–24GB（Ballooning ON）
   * **NIC**: VirtIO (paravirt)
3. 初回起動 → Windows セットアップ → VirtIO ISO をマウントし **ネットワーク/ストレージ/balloon ドライバ**を導入
4. **Docker Desktop**（WSL2バックエンド）導入 → **リソース上限**（CPU/メモリ）を設定
5. **Cursor** を導入

---

## 7. GPUパススルー：Windows編集VMの作成

### 7.1 カーネルパラメータとVFIO

1. **ブート方式の判定（ここが分岐ポイント）**
```
proxmox-boot-tool status
```
- 出力に systemd-boot と出る → A を実施（PVE8 の既定・UEFI）
- GRUB を使っている → B を実施（旧環境/レガシー）

A. systemd-boot（PVE8 既定・UEFI）

既存文字列を消さずに 末尾へ追記 します（重複させない）
```
sed -i 's/$/ amd_iommu=on iommu=pt/' /etc/kernel/cmdline
proxmox-boot-tool refresh
reboot
```

B. GRUB（旧環境/レガシー）

既存値の先頭へ追記します（重複させない）
```
sed -i 's/GRUB_CMDLINE_LINUX_DEFAULT="/&amd_iommu=on iommu=pt /' /etc/default/grub
update-grub # または grub-mkconfig -o /boot/grub/grub.cfg
reboot
```

2. **IOMMU確認**

```bash
dmesg | grep -e DMAR -e IOMMU -e AMD-Vi
or
dmesg | grep -E 'IOMMU|AMD-Vi'
```

3. **vfio モジュール & 3060 の VFIO バインド（共通）**

```bash
# vfio モジュール
cat >>/etc/modules <<'EOF'
vfio
vfio_iommu_type1
vfio_pci
vfio_virqfd
EOF


# 3060 のデバイス ID を確認（例: 10de:2488, 10de:228b）
lspci -nn | grep -E "VGA|Audio"


# VFIO にバインド
cat >/etc/modprobe.d/vfio.conf <<'EOF'
options vfio-pci ids=10de:2488,10de:228b
EOF


# 任意（nouveau を無効化）
cat >/etc/modprobe.d/blacklist-nouveau.conf <<'EOF'
blacklist nouveau
options nouveau modeset=0
EOF


update-initramfs -u
reboot
```

4. **バインド確認(共通)**
```
lspci -k | grep -A3 -E '10de:|NVIDIA|VGA'
# → Kernel driver in use: vfio-pci であれば OK
```

GUI: `Datacenter > Node > PCI Devices` から対象GPU/Audioを **VFIO-PCI** に割当（あるいは `/etc/modprobe.d/vfio.conf` に `options vfio-pci ids=10de:2504,10de:228e`）。

### 7.2 編集VMの作成（例: VMID=100, Name=edit-win）

* **OS**: Windows 11 ISO / UEFI (OVMF), Q35
* **System**: Q35, VirtIO SCSI single, **Qemu Agent=ON**, **NUMA=ON**, CPU type=host
* **Disk**: OS 150GB（nvme\_fast/vmdata）, IOThread=ON, SSD, Discard
* **追加ディスク（直付け）**:

  * **素材**: `nvme_media/media_zvol` を **Add > Hard Disk** で割当（SCSI, IOThread=ON）
  * **キャッシュ**: `nvme_fast/cache_zvol` を **Add > Hard Disk** で割当（SCSI, IOThread=ON）
* **CPU**: 10 vCPU
* **Memory**: 36–40GB（Balloon ON）
* **NIC**: VirtIO
* **PCI Passthrough**:

  * **Add > PCI Device** で GPU本体 `0000:..:..0` と **Audio関数** `0000:..:..1` の **2つ**を追加
  * `All Functions`/`ROM-Bar` 有効、`PCI-Express` 有効

### 7.3 Windows 初回セットアップ

1. VirtIO ドライバ（ネット/balloon/ストレージ）導入
2. **NVIDIAドライバ** を通常通りインストール（再起動）
3. **ディスクの初期化**（素材/キャッシュ用）→ NTFS（**64K クラスタ**）で D: / E: などに割当

> **うまく映らない/Code 43**: BIOSの iGPU/Primary Display、IOMMU/4G Decoding/CSM 設定を再確認。`video=efifb:off` を付与済みか確認。

---

## 8. Parsec と NLE（Premiere/DaVinci）最適化

### 8.1 Parsec（編集VM＝ホスト側）

* **Codec**: H.265、**FPS**: 60、**帯域**: 40–70Mbps から調整
* **VSync**: Off、**先読み**: 中、**デコーダ**: クライアントGPU
* 作業時は可能なら **1080p** に落として操作（最終書き出しはフル解像度）

### 8.2 Premiere/DaVinci

* **メディアキャッシュ/スクラッチ**: \*\*キャッシュ用ZVOL（E:）\*\*に固定
* **プロキシ**: 自動生成（ProRes Proxy / CineForm）
* **再生解像度**: 1/2〜1/4、**レンダラー**: CUDA/GPU
* 大量素材は **フォルダ分割**し、初回プレビュー生成を段階的に

---

## 9. ECC 有効化の確認

```bash
apt update && apt install -y edac-utils
lsmod | grep edac
edac-util -v          # CE/UE のカウンタ表示
sudo dmesg | grep -i -E "ecc|edac"
dmidecode -t memory | grep -i -E "error|ecc"
```

* Proxmox で **EDACがメモリコントローラを認識**していればOK。
* DDR5の **On-Die ECC** は別概念（チップ内）。本構成は **システムECC + On-Die** の両方を満たす想定。

---

## 10. ZFS スナップショットとロールバック

### 10.1 OS（rpool）

```bash
# 変更前にスナップショット
zfs snapshot rpool/ROOT/pve-1@pre-change
zfs list -t snapshot
# ロールバック（レスキュー/メンテナンスモード推奨）
systemctl isolate rescue.target
zfs rollback -r rpool/ROOT/pve-1@pre-change
reboot
```

### 10.2 VM ディスク/データセット

```bash
# 例: 編集素材zvol のスナップショット
zfs snapshot nvme_media/media_zvol@before-import
# 削除
zfs destroy nvme_media/media_zvol@before-import
```

---

## 11. バックアップ（簡易運用）

### 11.1 VM バックアップ（vzdump）

* USB HDD などを `/mnt/backup` にマウント → `Datacenter > Storage > Directory` で登録
* スケジュール: `Datacenter > Backup` で **週1** など
* 手動例:

```bash
vzdump 100 200 \
  --mode stop --compress zstd --storage backup \
  --notes "weekly backup"
```

### 11.2 データ（素材/キャッシュ）

* 素材は別HDD/NASへ **定期rsync**。キャッシュは復元不要なため除外可。

---

## 12. リソース割り当てと“カクつき”防止

* **編集VM**: 10 vCPU / 36–40GB RAM（Balloon ON）
* **開発VM**: 6 vCPU / 20–24GB RAM（Balloon ON）
* **Proxmoxホスト**: 4–6GB 残す
* **CPU Units**: 編集VM=1024、開発VM=512（編集を優先）
* **ディスク**: VirtIO-SCSI Single、**IOThread=ON**、**Write Back**（UPS が望ましい）
* **NIC**: VirtIO、クライアントは **有線1GbE**

---

## 13. 将来拡張（128GB/K8s/RKE2）

* **RAM 128GB**（32GB×4）へ：編集＋K8s×6台の並行運用が安定
* RKE2学習クラスタ（Ubuntu最小6台）

  * VMあたり: 2 vCPU / 4GB / 20GB Disk 目安
  * etcd/コントロールプレーンは別VMに（実運用学習の場合）

---

## 14. トラブルシューティング

**GPUが映らない/Code 43**

* BIOSの iGPU優先 / 4G Decoding / Re-Size BAR / CSM 無効 を再確認
* `amd_iommu=on iommu=pt video=efifb:off` の適用（`proxmox-boot-tool refresh`）
* GPUとAudioの **両方**をパススルー、`All Functions` 有効

**Parsecがカクつく**

* H.265 / 60fps / 40–70Mbps / クライアントデコード / 1080p作業
* ルータ/スイッチ間のQoSや混雑を確認（**全て有線**）

**素材一覧が遅い**

* プロキシ生成、フォルダ分割、キャッシュを専用ZVOLに固定

**ZFSが遅い**

* `atime=off`、`compression=lz4（キャッシュはoff）`、`volblocksize=64K` を確認
* `sync=standard`（UPSがあれば `write back` 運用との整合性に留意）

---

## 15. 付録：よく使うコマンド集

```bash
# ディスク・プール確認
lsblk -o NAME,SIZE,MODEL
zpool status
zfs list -o name,used,avail,compression,volblocksize

# ZVOL作成例（素材900G / キャッシュ500G）
zfs create -V 900G -o volblocksize=64K -o compression=lz4 -o atime=off nvme_media/media_zvol
zfs create -V 500G -o volblocksize=64K -o compression=off -o atime=off nvme_fast/cache_zvol

# スナップショット
zfs snapshot nvme_media/media_zvol@pre
zfs list -t snapshot
zfs destroy nvme_media/media_zvol@pre

# ECC確認
apt install -y edac-utils
edac-util -v
sudo dmesg | grep -i -E "ecc|edac"

# バックアップ（手動）
vzdump 100 --mode stop --compress zstd --storage backup
```

---

### ライセンス

MIT（必要に応じて変更してください）

### メモ

* まずは **現状の64GBで運用開始** → 体感に応じて **キャッシュZVOLの増量**と **RAM 128GB** へ拡張。
* データは**こまめにバックアップ**（NVMe単発プールは冗長性がありません）。
