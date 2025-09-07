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
4. 管理ネットワーク（vmbr0）/ホスト名/時刻を設定 → インストール完了

> **初期ログイン**: https\://\<Proxmox管理IP>:8006 へブラウザからアクセス

---

## 4. NVMeプールとZVOLの作成

**方針**: NVMe 2枚を別々の **単一ディスクZFSプール** にして役割分離。

* NVMe#1 → `nvme_media`（編集素材用 zvol を作る）
* NVMe#2 → `nvme_fast`（キャッシュ用 zvol + 開発VM用のZFSストレージ）

### 4.1 プール作成（GUI推奨）

* `Node` → 該当ノード → `Disks` → `ZFS` → `Create: ZFS` で個別に作成

  * `name=nvme_media`, `ashift=12`, デバイス=`/dev/nvme0n1`
  * `name=nvme_fast`,  `ashift=12`, デバイス=`/dev/nvme1n1`

> CLI派は下記（**デバイス名は必ず確認**）

```bash
# 例: 実デバイス確認
lsblk -o NAME,SIZE,MODEL
# プール作成
zpool create -f -o ashift=12 nvme_media /dev/nvme0n1
zpool create -f -o ashift=12 nvme_fast  /dev/nvme1n1
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
zfs create -V 900G -o volblocksize=64K -o compression=lz4 -o atime=off -o logbias=throughput nvme_media/media_zvol
```

**キャッシュ用 ZVOL**（500GB）

```bash
zfs create -V 500G -o volblocksize=64K -o compression=off -o atime=off -o logbias=throughput nvme_fast/cache_zvol
```

**開発VM 用ストレージ**（ZFS データセット）

```bash
zfs create -o compression=lz4 -o atime=off nvme_fast/vmdata
```

Proxmox GUI → `Datacenter > Storage > Add > ZFS` で `nvme_fast`（or `nvme_fast/vmdata`）を **VMディスク格納先**として追加。

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

1. **カーネルコマンドライン**

```bash
# 既存内容に追記（amd_iommu=on iommu=pt video=efifb:off）
cat /etc/kernel/cmdline
# 例: root=... ro quiet amd_iommu=on iommu=pt video=efifb:off
proxmox-boot-tool refresh
reboot
```

2. **IOMMU確認**

```bash
dmesg | grep -e DMAR -e IOMMU -e AMD-Vi
```

3. **デバイス確認とVFIOバインド**

```bash
lspci -nn | grep -E "VGA|Audio"
# 例: 0000:0a:00.0 VGA [10de:2504] / 0000:0a:00.1 Audio [10de:228e]
# → 上記2つを vfio-pci にバインド
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
