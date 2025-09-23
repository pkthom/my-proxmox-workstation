# 0) ProxmoxインストーラーUSBを作る

[ISO置き場](http://proxmox.com/en/downloads/proxmox-virtual-environment/iso)から、
[最新版](https://enterprise.proxmox.com/iso/proxmox-ve_9.0-1.iso)をダウンロード

<img width="1408" height="411" alt="image" src="https://github.com/user-attachments/assets/33525586-65ef-490a-9525-d0c08fc449c0" />


念のためハッシュを確認
```
pkthom@MacBook-Pro Downloads % shasum -a 256 proxmox-ve_9.0-1.iso
228f948ae696f2448460443f4b619157cab78ee69802acc0d06761ebd4f51c3e  proxmox-ve_9.0-1.iso
```

[https://github.com/balena-io/etcher/releases/download/v2.1.4/balenaEtcher-2.1.4-x64.dmg](こちら)からEtcherをダウンロード

ダウンロードしたProxmoxISO、使用予定のUSBを選択し、「Flash!」をクリック

<img width="818" height="499" alt="Screen Shot 2025-09-20 at 3 36 44 PM" src="https://github.com/user-attachments/assets/99aadfaa-bf02-45c5-99a6-38ab1fb2e6de" />

インストーラー作成完了　自作PCに差しておく

<img width="819" height="497" alt="Screen Shot 2025-09-20 at 3 39 26 PM" src="https://github.com/user-attachments/assets/8342d186-c0b1-4da8-bdbf-25e0cba55aff" />

<details><summary>この方法でもいけるかもしれないが、結局試していない</summary>
   
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

</details>

# 0) パーツを組み立てる

Q-LEDが緑になった

<img width="682" height="872" alt="image" src="https://github.com/user-attachments/assets/b6dcb66d-08e0-466b-9997-baeb9b60a6a6" />

SSDをケース側面に固定するマウンタが手に入らないため、ガムテで固定する

<img width="500" height="579" alt="image" src="https://github.com/user-attachments/assets/8438f41d-78ac-4180-a7bb-98c3925f490a" />


UEFI BIOSが出た

<img width="1121" height="510" alt="image" src="https://github.com/user-attachments/assets/34c99252-a4a7-4d64-b940-6e66ae68c811" />

全てのディスクと、USBが読み込めている

<img width="792" height="592" alt="image" src="https://github.com/user-attachments/assets/a4d3db85-1998-42af-983b-a82b22cf6b68" />

<img width="790" height="589" alt="image" src="https://github.com/user-attachments/assets/5d3a2532-8625-4279-8031-d1e58ac695cb" />

<img width="648" height="579" alt="image" src="https://github.com/user-attachments/assets/922f7c5f-4712-4edb-8a60-3581d8e54f2a" />


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

# 1) BIOS設定

### Primary Display / Integrated Graphics**：内蔵iGPUを優先　

Advanced -> NB Configuration -> Primary Video Device : IGFX Video (デフォルトはPCIE Video)

<img width="1246" height="657" alt="image" src="https://github.com/user-attachments/assets/c1e6111a-4c55-4102-88a7-574487be5e9b" />


### CSM：Disabled（UEFIオンリー）

Boot -> CSM(Compatibility Support Module) -> Launch CSM : Disabled (最初からこう)

<img width="789" height="348" alt="image" src="https://github.com/user-attachments/assets/89455353-55f5-48ba-981e-6c9050bb4a69" />

  
### SVM (AMD-V)：Enabled

Advanced -> CPU Configuration -> SVM Mode : Enabled (最初からこう)

<img width="787" height="336" alt="image" src="https://github.com/user-attachments/assets/d93338bb-c6a5-4099-9cda-336824580cbc" />


### IOMMU（AMD IOMMU / SVIOMMU）：Enabled

Advanced -> AMD CBS -> IOMMU : Enabled (最初はAuto)

<img width="792" height="418" alt="image" src="https://github.com/user-attachments/assets/1ef45450-1858-4b67-9f2b-7f2b3109ecd4" />


### Above 4G Decoding：Enabled

Advanced -> PCI Subsystem Settings -> Above 4G Decoding : Enabled (最初からこう)

<img width="792" height="408" alt="image" src="https://github.com/user-attachments/assets/7ac2a3ca-4b56-47eb-8e61-d57ca0420d42" />


### Re-Size BAR：Enabled

Advanced -> PCI Subsystem Settings -> Resize BAR Support : Enabled (最初からこう)

上画像に写っている

### SATA Mode：AHCI

Advanced -> SATA Configuration -> SATA Mode : AHCI (最初からこう)

<img width="1240" height="590" alt="image" src="https://github.com/user-attachments/assets/de3129c1-dc9a-4611-99fd-e7bb668eed45" />

  
### メモリはEXPO/XMPを有効（安定しない場合はAutoに戻す）-> ⚠️これ無理だったため、今はDisabledにしている

EXPO : Enabled -> これすると、Q ~LEDが黄色(DRAM)で進まなくなった　

以下でBIOS初期化して治った 
```
完全放電手順
・電源長押しでOFF → PSU背面を「0」に → 電源ケーブルを抜く
・PCの電源ボタンを10秒長押し（放電）

CMOSクリア手順（EXPOをリセット）
・マザボ上の CLRTC（Clear CMOS）ピン をドライバー等で 5〜10秒ショート
```

### Legacy USB Support & XHCI Hand-off : Enabled

Advanced -> USB Configuration -> Legacy USB Support & XHCI Hand-off : Enabled (最初からこう)

<img width="791" height="431" alt="image" src="https://github.com/user-attachments/assets/38dbd2af-18a6-4e95-9280-451981dd7b4a" />

### Secure Boot : Disable

Boot -> Secure Boot -> OS Type : Other OS (最初からこう)

写真撮り忘れた

### Fast Boot : Disable

Boot -> Boot Configuration -> Fast Boot : Disabled　(最初はEnabled)

写真撮り忘れた

### 変更を保存してUSBから起動

Boot -> 一番下のUSB（UEFI:BUFFALO USB Flash Disk...）をクリック -> 

<img width="790" height="542" alt="image" src="https://github.com/user-attachments/assets/8684d08d-cdb5-41e0-8635-339b7731eaeb" />

「Save configuration and reset?」と聞かれるので、「OK」-> 再起動される

<img width="794" height="538" alt="image" src="https://github.com/user-attachments/assets/e2b868c2-d615-4740-b78d-98646ad403e9" />

再起動の後なぜか再度BIOSが表示されたため、再度「Save & Exit(F10)」で再起動 -> Proxmoxが起動した

# 1) Proxmox設定

### Proxmox起動

「Install Proxmox VE (Graphical)」を選択

<img width="1060" height="588" alt="image" src="https://github.com/user-attachments/assets/95ab480a-a8dc-40ef-b1e5-a6232182f691" />

### 規約に同意する

I agree をクリック

<img width="792" height="467" alt="image" src="https://github.com/user-attachments/assets/023a744e-9cfb-4253-bb60-83c3649f884e" />

### どこにOSをインストールするか　を選ぶ -> ZFSでSSD SATA 250GB × 2枚でRAID1(ミラー)を組む

<img width="597" height="789" alt="image" src="https://github.com/user-attachments/assets/5bf97661-797c-4ac7-9472-12c0d5a7ebed" />

<img width="596" height="794" alt="image" src="https://github.com/user-attachments/assets/ffb67925-125c-42d9-a6e9-54fc8af65d9a" />

### 国、タイムゾーン、キーボードレイアウトを選ぶ

<img width="496" height="292" alt="image" src="https://github.com/user-attachments/assets/06f1bc06-f7b7-4343-a709-c41b42acf93c" />

### rootパスワード & メールアドレスを入力

### NIC、ホスト名、IP、GW、DNSサーバー　を入力

<img width="328" height="162" alt="image" src="https://github.com/user-attachments/assets/4b489509-d502-437d-a892-5bbfb72eb877" />

### 確認画面

-> 確認して、OK -> インストール始まる

<img width="496" height="643" alt="image" src="https://github.com/user-attachments/assets/a73774a4-f2c4-4cf0-95ac-1cd0849afcec" />


### コンソールにアクセス

インストールが終わると、設定したパスワードでCLIからログインできる

<img width="488" height="307" alt="image" src="https://github.com/user-attachments/assets/92912dca-6354-4ee2-aef9-99106599902c" />

マシンとルーターをLANケーブルで繋いだら、設定したIP(192.168.11.10)にpingが通るようになり、https://192.168.11.10:8006/ でコンソールにアクセスできる

# 3) Proxmox初期設定 & 更新

ディスクの状態
```bash
root@pve:~# lsblk -o NAME,SIZE,MODEL
NAME      SIZE MODEL
sda       3.6T ST4000VN006-3CW104
sdb       3.6T ST4000VN006-3CW104
sdc     232.9G CT250MX500SSD1
├─sdc1   1007K 
├─sdc2      1G 
└─sdc3    231G 
sdd     232.9G CT250MX500SSD1
├─sdd1   1007K 
├─sdd2      1G 
└─sdd3    231G 
nvme0n1 931.5G CT1000T500SSD8
nvme1n1 931.5G CT1000T500SSD8
root@pve:~#
```

OS領域のautotrimをONにする
```
root@pve:~# zpool get autotrim
NAME   PROPERTY  VALUE     SOURCE
rpool  autotrim  off       default
root@pve:~# zpool set autotrim=on rpool
root@pve:~# zpool get autotrim
NAME   PROPERTY  VALUE     SOURCE
rpool  autotrim  on        local
root@pve:~# 
```

有料版のお知らせをdisableする
```
root@pve:~# sed -i 's/^deb/#deb/g' /etc/apt/sources.list.d/pve-enterprise.list
echo 'deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription' \
  > /etc/apt/sources.list.d/pve-no-subscription.list
sed: can't read /etc/apt/sources.list.d/pve-enterprise.list: No such file or directory
root@pve:~# 
```

最新にする
```
root@pve:~# apt update
```
```
root@pve:~# apt full-upgrade -y
```

apt update 時にエラーが出る
```
root@pve:/hdds/pictures/_inbox# apt update
Hit:1 http://deb.debian.org/debian trixie InRelease
Get:2 http://deb.debian.org/debian trixie-updates InRelease [47.3 kB]           
Get:3 http://security.debian.org/debian-security trixie-security InRelease [43.4 kB]                              
Get:4 http://security.debian.org/debian-security trixie-security/main amd64 Packages [44.3 kB]                  
Get:5 http://security.debian.org/debian-security trixie-security/main Translation-en [29.9 kB]                             
Hit:6 http://download.proxmox.com/debian/pve bookworm InRelease                                                 
Err:7 https://enterprise.proxmox.com/debian/ceph-squid trixie InRelease                                      
  401  Unauthorized [IP: 51.79.228.122 443]
Err:8 https://enterprise.proxmox.com/debian/pve trixie InRelease
  401  Unauthorized [IP: 51.79.228.122 443]
Error: Failed to fetch https://enterprise.proxmox.com/debian/ceph-squid/dists/trixie/InRelease  401  Unauthorized [IP: 51.79.228.122 443]
Error: The repository 'https://enterprise.proxmox.com/debian/ceph-squid trixie InRelease' is not signed.
Notice: Updating from such a repository can't be done securely, and is therefore disabled by default.
Notice: See apt-secure(8) manpage for repository creation and user configuration details.
Error: Failed to fetch https://enterprise.proxmox.com/debian/pve/dists/trixie/InRelease  401  Unauthorized [IP: 51.79.228.122 443]
Error: The repository 'https://enterprise.proxmox.com/debian/pve trixie InRelease' is not signed.
Notice: Updating from such a repository can't be done securely, and is therefore disabled by default.
Notice: See apt-secure(8) manpage for repository creation and user configuration details.
```
以下2ファイルを、以下のように編集
```
root@pve:~# cat /etc/apt/sources.list.d/pve-enterprise.sources 
Enabled: no
Types: deb
URIs: https://enterprise.proxmox.com/debian/pve
Suites: trixie
Components: pve-enterprise
Signed-By: /usr/share/keyrings/proxmox-archive-keyring.gpg
root@pve:~# cat /etc/apt/sources.list.d/pve-no-subscription.sources 
Types: deb
URIs: http://download.proxmox.com/debian/pve
Suites: trixie
Components: pve-no-subscription
Signed-By: /usr/share/keyrings/proxmox-archive-keyring.gpg
root@pve:~#
```
cephに関するエラーがあったので、無償版にする
```
root@pve:~# ls -l /etc/apt/sources.list.d | grep -i ceph || true
-rw-r--r-- 1 root root 163 Aug  5 19:42 ceph.sources
root@pve:~# cat /etc/apt/sources.list.d/ceph.sources 
Types: deb
URIs: https://enterprise.proxmox.com/debian/ceph-squid
Suites: trixie
Components: enterprise
Signed-By: /usr/share/keyrings/proxmox-archive-keyring.gpg
root@pve:~# sed -i '1i Enabled: no' /etc/apt/sources.list.d/ceph.sources 
```
解消
```
root@pve:~# apt update
Hit:1 http://deb.debian.org/debian trixie InRelease
Hit:2 http://deb.debian.org/debian trixie-updates InRelease                                      
Hit:3 http://security.debian.org/debian-security trixie-security InRelease                       
Hit:4 http://download.proxmox.com/debian/pve trixie InRelease
40 packages can be upgraded. Run 'apt list --upgradable' to see them.
root@pve:~# 
```
念の為もう一度アップグレード
```
root@pve:~# apt full-upgrade -y
```

再起動
```
reboot
```

再起動後、apt update 
```
root@pve:~# apt update
Hit:1 http://deb.debian.org/debian trixie InRelease
Hit:2 http://deb.debian.org/debian trixie-updates InRelease                                                            
Hit:3 http://security.debian.org/debian-security trixie-security InRelease                                             
Hit:4 http://download.proxmox.com/debian/pve trixie InRelease
All packages are up to date.    
root@pve:~# 
```

# NVMe設定

ゴール
```
・NVMe1枚目 -> VM作る場所
・NVMe2枚目 -> 編集用VM直付け用
     |---> d_vol -> 素材置き場
     |---> e_vol -> キャッシュ領域
```

Poolを作る
```
root@pve:~# zpool create -o ashift=12 nvme1 /dev/nvme1n0
root@pve:~# zpool create -o ashift=12 nvme2 /dev/nvme1n1
root@pve:~# zpool list
NAME    SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
nvme1   928G   420K   928G        -         -     0%     0%  1.00x    ONLINE  -
nvme2   928G   516K   928G        -         -     0%     0%  1.00x    ONLINE  -
rpool   230G  2.05G   228G        -         -     0%     0%  1.00x    ONLINE  -
root@pve:~# 
```
```
root@pve:~# zfs get compression,atime nvme1
NAME   PROPERTY     VALUE           SOURCE
nvme1  compression  on              default
nvme1  atime        on              default
root@pve:~# zfs get compression,atime nvme2
NAME   PROPERTY     VALUE           SOURCE
nvme2  compression  on              default
nvme2  atime        on              default
root@pve:~# 
```
atimeをONにする
```
root@pve:~# zfs set atime=off nvme1
root@pve:~# zfs set atime=off nvme2
```

圧縮形式を、lz4にする
```
root@pve:~# zfs set compression=lz4 nvme1
root@pve:~# zfs set compression=lz4 nvme2
root@pve:~# zfs get compression,atime nvme1
NAME   PROPERTY     VALUE           SOURCE
nvme1  compression  lz4             local
nvme1  atime        off             local
root@pve:~# zfs get compression,atime nvme2
NAME   PROPERTY     VALUE           SOURCE
nvme2  compression  lz4             local
nvme2  atime        off             local
root@pve:~# 
```
autotrimをONにする
```
root@pve:~# zpool get autotrim
NAME   PROPERTY  VALUE     SOURCE
nvme1  autotrim  off       default
nvme2  autotrim  off       default
rpool  autotrim  on        local
root@pve:~# zpool set autotrim=on nvme1
root@pve:~# zpool set autotrim=on nvme2
root@pve:~# zpool get autotrim
NAME   PROPERTY  VALUE     SOURCE
nvme1  autotrim  on        local
nvme2  autotrim  on        local
rpool  autotrim  on        local
root@pve:~# 
```

仮想ディスクを作成
```
root@pve:~# zfs create -V 700G -o volblocksize=64K nvme2/d_vol
root@pve:~# zfs create -V 200G -o volblocksize=64K nvme2/e_vol
cannot create 'nvme2/e_vol': out of space
root@pve:~# zfs create -V 195G -o volblocksize=64K nvme2/e_vol
```
```
root@pve:~# zfs list -o name,volsize,refreservation,used,refer -r nvme2
NAME         VOLSIZE  REFRESERV   USED  REFER
nvme2              -       none   899G    96K
nvme2/d_vol     700G       703G   703G    56K
nvme2/e_vol     195G       196G   196G    56K
root@pve:~# zfs get volsize,volblocksize,compression,atime,refreservation -r nvme2
NAME         PROPERTY        VALUE           SOURCE
nvme2        volsize         -               -
nvme2        volblocksize    -               -
nvme2        compression     lz4             local
nvme2        atime           off             local
nvme2        refreservation  none            default
nvme2/d_vol  volsize         700G            local
nvme2/d_vol  volblocksize    64K             -
nvme2/d_vol  compression     lz4             inherited from nvme2
nvme2/d_vol  atime           -               -
nvme2/d_vol  refreservation  703G            local
nvme2/e_vol  volsize         195G            local
nvme2/e_vol  volblocksize    64K             -
nvme2/e_vol  compression     lz4             inherited from nvme2
nvme2/e_vol  atime           -               -
nvme2/e_vol  refreservation  196G            local
root@pve:~# 
```

Proxmoxにストレージ登録（GUI）

nvme1
<img width="1849" height="1137" alt="image" src="https://github.com/user-attachments/assets/15600789-87e8-4bc6-99a3-597bac0a7720" />

nvme2
<img width="1849" height="1137" alt="image" src="https://github.com/user-attachments/assets/095b429d-8106-4cf7-a131-9f5d612ff786" />

↓

こうなる

<img width="1849" height="1137" alt="image" src="https://github.com/user-attachments/assets/b38877cf-46a1-4ba7-8e17-55084acbe87f" />


# HDD設定

ZFSでRAID1(ミラー)を組む
```
root@pve:~# zpool create -o ashift=12 hdds mirror /dev/sda /dev/sdb
root@pve:~# echo $?
0
```
圧縮はzstd-3にする
```
root@pve:~# zfs set compression=zstd-3 hdds
```
atime=offにする（autotrimはOFFのままで良い）
```
root@pve:~# zfs set atime=off hdds
root@pve:~# zpool get autotrim
NAME   PROPERTY  VALUE     SOURCE
hdds   autotrim  off       default
nvme1  autotrim  on        local
nvme2  autotrim  on        local
rpool  autotrim  on        local
root@pve:~#
root@pve:~# zpool status hdds
  pool: hdds
 state: ONLINE
config:

        NAME        STATE     READ WRITE CKSUM
        hdds        ONLINE       0     0     0
          mirror-0  ONLINE       0     0     0
            sda     ONLINE       0     0     0
            sdb     ONLINE       0     0     0

errors: No known data errors
root@pve:~# zfs list hdds
NAME   USED  AVAIL  REFER  MOUNTPOINT
hdds   800K  3.51T   104K  /hdds
root@pve:~# 
```

<img width="1849" height="1137" alt="image" src="https://github.com/user-attachments/assets/1dab9ecc-252d-4372-9eaf-69399e03f280" />


# 6) Samba（SMB）共有設定（editor=RW, viewer=RO）

```
root@pve:~# apt update
root@pve:~# apt install -y samba samba-vfs-modules
```
既にenabled
```
root@pve:~# systemctl is-enabled smbd
enabled
root@pve:~# systemctl is-enabled nmbd
enabled
```

仮想ディスクを作る
```
root@pve:~# zfs create hdds/pictures
root@pve:~# zfs create hdds/documents
```
できている
```
root@pve:~# zfs list
NAME               USED  AVAIL  REFER  MOUNTPOINT
hdds               800K  3.51T   104K  /hdds
hdds/documents      96K  3.51T    96K  /hdds/documents
hdds/pictures       96K  3.51T    96K  /hdds/pictures
nvme1              588K   899G    96K  /nvme1
nvme2              899G   764M    96K  /nvme2
nvme2/d_vol        703G   703G    56K  -
nvme2/e_vol        196G   197G    56K  -
rpool             2.13G   221G   104K  /rpool
rpool/ROOT        2.12G   221G    96K  /rpool/ROOT
rpool/ROOT/pve-1  2.12G   221G  2.12G  /
rpool/data          96K   221G    96K  /rpool/data
rpool/var-lib-vz    96K   221G    96K  /var/lib/vz
root@pve:~# 
```

xattr=sa, acltype=nfsv4 に設定
```
root@pve:~# zfs set xattr=sa hdds
root@pve:~# zfs set acltype=nfsv4 hdds
root@pve:~# zfs get -r xattr,acltype hdds
NAME            PROPERTY  VALUE     SOURCE
hdds            xattr     on        local
hdds            acltype   nfsv4     local
hdds/documents  xattr     on        inherited from hdds
hdds/documents  acltype   nfsv4     inherited from hdds
hdds/pictures   xattr     on        inherited from hdds
hdds/pictures   acltype   nfsv4     inherited from hdds
root@pve:~# 
```

まず、読み書きできる予定のグループ editors を作る
```
root@pve:~# groupadd -f editors
root@pve:~# getent group
...
editors:x:1000:
```
ユーザー（editor, viewer）を作る

既にあるか念のためチェック
```
root@pve:~# id editor
id: ‘editor’: no such user
root@pve:~# id viewer
id: ‘viewer’: no such user
```
ユーザーを作成
```
root@pve:~# useradd -m -s /bin/bash editor
root@pve:~# useradd -m -s /usr/sbin/nologin viewer
root@pve:~# cat /etc/passwd
...
editor:x:1000:1001::/home/editor:/bin/bash
viewer:x:1001:1002::/home/viewer:/usr/sbin/nologin
```
editorユーザーは、editorsグループに所属させておく
```
root@pve:~# usermod -aG editors editor
```
```
root@pve:~# id editor
uid=1000(editor) gid=1001(editor) groups=1001(editor),1000(editors)
root@pve:~# id viewer
uid=1001(viewer) gid=1002(viewer) groups=1002(viewer)
root@pve:~# 
```
パスワード設定
```
root@pve:~# smbpasswd -a editor
New SMB password:
Retype new SMB password:
Added user editor.
root@pve:~# smbpasswd -a viewer
New SMB password:
Retype new SMB password:
Added user viewer.
root@pve:~# 
```

仮想ディスクの所有グループを、editorsにする(再起的に全部)
```
root@pve:~# chown -R root:editors /hdds/pictures
root@pve:~# chown -R root:editors /hdds/documents
root@pve:~# ls -lah /hdds
total 10K
drwxr-xr-x  4 root root     4 Sep 20 19:36 .
drwxr-xr-x 21 root root    25 Sep 20 19:33 ..
drwxr-xr-x  2 root editors  2 Sep 20 19:36 documents
drwxr-xr-x  2 root editors  2 Sep 20 19:36 pictures
root@pve:~# 
```
※ 変更前はroot:rootだった
```
root@pve:/hdds# ls -lah
total 10K
drwxr-xr-x  4 root root  4 Sep 20 19:36 .
drwxr-xr-x 21 root root 25 Sep 20 19:33 ..
drwxr-xr-x  2 root root  2 Sep 20 19:36 documents
drwxr-xr-x  2 root root  2 Sep 20 19:36 pictures
root@pve:/hdds# 
```

仮想ディスクの中に今後作られる全てのファイル、フォルダの権限を指定する

変更前テスト
```
root@pve:/hdds# touch pictures/test
root@pve:/hdds# touch documents/test

root@pve:/hdds/pictures# mkdir testdir
root@pve:/hdds/documents# mkdir testdir

root@pve:/hdds# ls -lah pictures/
total 2.0K
drwxr-xr-x 3 root editors 4 Sep 21 08:36 .
drwxr-xr-x 4 root root    4 Sep 20 19:36 ..
-rw-r--r-- 1 root root    0 Sep 21 08:34 test
drwxr-xr-x 2 root root    2 Sep 21 08:36 testdir
root@pve:/hdds#
root@pve:/hdds# ls -lah documents/
total 2.0K
drwxr-xr-x 3 root editors 4 Sep 21 08:36 .
drwxr-xr-x 4 root root    4 Sep 20 19:36 ..
-rw-r--r-- 1 root root    0 Sep 21 08:34 test
drwxr-xr-x 2 root root    2 Sep 21 08:36 testdir
root@pve:/hdds# 
```
2775 -> 2:以下に作成されるすべては、親ディレクトリのグループを引き継ぐ + 775:rootとeditorsグループのメンバーはrwx全部OK 部外者はrx(書きはできない)

664 -> 664: rootとeditorsグループのメンバーはrw(読み書きOK)　部外者はrのみ(書きはできない)

-> ⚠️ ファイルが775ではない理由は、ただの写真で実行する必要ないから（スクリプトならx与えてもいいが）　

-> ⚠️ フォルダが775の理由は、cd や ls など、実行(x)がないとできないから
```
root@pve:~# find /hdds/pictures  -type d -exec chmod 2775 {} \;
root@pve:~# find /hdds/pictures  -type f -exec chmod 664  {} \;
root@pve:~# find /hdds/documents -type d -exec chmod 2775 {} \;
root@pve:~# find /hdds/documents -type f -exec chmod 664  {} \;
```
acl をインストール
```
root@pve:~# apt install -y acl
```
```
root@pve:~# setfacl -R -m g:editors:rwx /hdds/pictures /hdds/documents
setfacl: /hdds/pictures: Operation not supported
setfacl: /hdds/documents: Operation not supported
```

```
root@pve:/etc/samba# cp smb.conf smb.conf.bak
root@pve:/etc/samba# ls
gdbcommands  smb.conf  smb.conf.bak
```

⚠️ vi で、deleteキー、矢印キーが使えなかった
```
apt install -y vim 
```
これでvimエディターをインストールすると解決した

smb.conf を編集
```
root@pve:~# cat /etc/samba/smb.conf
[global]
   workgroup = WORKGROUP
   server string = PVE Samba
   interfaces = 192.168.11.0/24 lo
   bind interfaces only = yes

   server min protocol = SMB2
   server max protocol = SMB3

   vfs objects = catia fruit streams_xattr acl_xattr
   fruit:metadata = stream
   fruit:resource = stream
   fruit:delete_empty_adfiles = yes
   fruit:encoding = native
   map acl inherit = yes
   store dos attributes = yes
   ea support = yes
   map to guest = never

[pictures]
   path = /hdds/pictures
   browseable = yes
   read only = yes
   guest ok = no
   valid users = editor, viewer
   write list = editor, @editors
   vfs objects = catia fruit streams_xattr acl_xattr
   fruit:metadata = stream
   fruit:resource = stream
   fruit:delete_empty_adfiles = yes

[documents]
   path = /hdds/documents
   browseable = yes
   read only = yes
   guest ok = no
   valid users = editor, viewer
   write list = editor, @editors
   vfs objects = catia fruit streams_xattr acl_xattr
   fruit:metadata = stream
   fruit:resource = stream
   fruit:delete_empty_adfiles = yes
root@pve:~# 
```
構文チェックOK
```
root@pve:~# testparm -s
Load smb config files from /etc/samba/smb.conf
Loaded services file OK.
```
設定を反映
```
root@pve:~# systemctl restart smbd nmbd
```

windowsのエクスプローラーで、以下で入れるようになった
```
\\192.168.11.10\pictures
\\192.168.11.10\documents
```

MacではFinderで、以下で入れる
```
smb://192.168.11.10/pictures
smb://192.168.11.10/documents
```

<img width="598" height="343" alt="image" src="https://github.com/user-attachments/assets/94eb722a-7b03-4a7e-a186-51a24330eed8" />

<img width="542" height="398" alt="image" src="https://github.com/user-attachments/assets/d1721371-d5e2-436d-a73b-95fa34609a09" />

<img width="1238" height="886" alt="image" src="https://github.com/user-attachments/assets/7dd875f5-aade-4b07-bd20-df2a6c18eb5d" />

# 「RTX 4060 を“ホスト(Proxmox)”では使わず、Windowsの箱(=VM)に丸ごと渡す」**ための準備をする

前提知識
```
PCの内部バス(PCIe)に刺さっている部品には住所があります。
書式は 「バス:デバイス.機能」（bus:device.function）。

01:00.0 … 4060本体（映像を出す機能 = VGA）

01:00.1 … 4060の音（HDMIオーディオ）

1枚のグラボでも「映像」と「音」は別の住所を持っています。だから2つともWindowsに渡す必要があります。
```

目標
```
1.ホストが4060を使わないようにする
→ 4060を「貸出窓口（vfio）」に預ける

2.Windows VMに 4060 を貸し出す
→ ProxmoxのVM設定から「この住所のデバイスを渡す」

このときの合言葉が vfio-pci。
lspci -k で 「Kernel driver in use: vfio-pci」 と出れば、ホストは触っていない＝VMに貸せる状態です。
```
vfio-pci とは？
```
ざっくり言うと、**vfio-pci は「PCIeデバイスの貸出係」**です。
Linux（Proxmox）の通常ドライバの代わりにそのデバイスを“掴んで”、まるごとVMに貸し出すための仕組み（ドライバ）です。

VMにパススルーする時、GPUを予約してくれるドライバ
```

IOMMUが有効か確認（後でGPUをパススルーするため）
```
root@pve:/hdds/pictures# dmesg | grep -e IOMMU -e AMD-Vi
[    0.244511] AMD-Vi: Using global IVHD EFR:0x246577efa2254afa, EFR2:0x0
[    0.502524] pci 0000:00:00.2: AMD-Vi: IOMMU performance counters supported
[    0.505792] AMD-Vi: Extended features (0x246577efa2254afa, 0x0): PPR NX GT [5] IA GA PC GA_vAPIC
[    0.505801] AMD-Vi: Interrupt remapping enabled
[    0.756776] AMD-Vi: Virtual APIC enabled
[    0.757030] perf/amd_iommu: Detected AMD IOMMU #0 (2 banks, 4 counters/bank).
root@pve:/hdds/pictures#
```

vfio-pci の準備
```
root@pve:/etc/modules-load.d# ls
modules.conf
root@pve:/etc/modules-load.d# cat >/etc/modules-load.d/vfio.conf <<'EOF'
vfio
vfio_iommu_type1
vfio_virqfd
vfio_pci
EOF
root@pve:/etc/modules-load.d# cat vfio.conf 
vfio
vfio_iommu_type1
vfio_virqfd
vfio_pci
root@pve:/etc/modules-load.d# 
```

現状 -> 01:00.0 も 01:00.1 も、ホストに掴まれたまま -> これが両方、vfio-pci に掴まれている必要がある
```
root@pve:/etc/modules-load.d# lspci -k -s 01:00.0
01:00.0 VGA compatible controller: NVIDIA Corporation AD107 [GeForce RTX 4060] (rev a1)
        Subsystem: ASUSTeK Computer Inc. Device 8936
        Kernel modules: nvidiafb, nouveau
root@pve:/etc/modules-load.d# lspci -k -s 01:00.1
01:00.1 Audio device: NVIDIA Corporation AD107 High Definition Audio Controller (rev a1)
        Subsystem: ASUSTeK Computer Inc. Device 8936
        Kernel driver in use: snd_hda_intel
        Kernel modules: snd_hda_intel
root@pve:/etc/modules-load.d#
```

GPUのデバイスIDを調べる

vfio-pci に予約してもらうため、GPUのデバイスIDが必要
```
root@pve:/hdds/pictures# lspci -nn | egrep -i 'nvidia|vga|audio'
01:00.0 VGA compatible controller [0300]: NVIDIA Corporation AD107 [GeForce RTX 4060] [10de:2882] (rev a1)
01:00.1 Audio device [0403]: NVIDIA Corporation AD107 High Definition Audio Controller [10de:22be] (rev a1)
0d:00.0 VGA compatible controller [0300]: Advanced Micro Devices, Inc. [AMD/ATI] Raphael [1002:164e] (rev c4)
0d:00.1 Audio device [0403]: Advanced Micro Devices, Inc. [AMD/ATI] Radeon High Definition Audio Controller [Rembrandt/Strix] [1002:1640]
0d:00.6 Audio device [0403]: Advanced Micro Devices, Inc. [AMD] Family 17h/19h/1ah HD Audio Controller [1022:15e3]
root@pve:/hdds/pictures# 
```
```
01:00.0 -> 10de:2882
01:00.1 -> 10de:22be
```
デバイスIDが分かったので、vfio-pciに予約してもらう
```
root@pve:/etc/modprobe.d# ls
amd64-microcode-blacklist.conf  pve-blacklist.conf  zfs.conf
root@pve:/etc/modprobe.d# cat >/etc/modprobe.d/vfio-pci.conf <<'EOF'
options vfio-pci ids=10de:2882,10de:22be disable_vga=1
EOF
root@pve:/etc/modprobe.d# cat vfio-pci.conf 
options vfio-pci ids=10de:2882,10de:22be disable_vga=1
root@pve:/etc/modprobe.d# 
```
反映
```
update-initramfs -u
reboot
```

再起動後、無事vfio-pciによる予約完了
```
root@pve:~# lspci -k -s 01:00.0
01:00.0 VGA compatible controller: NVIDIA Corporation AD107 [GeForce RTX 4060] (rev a1)
        Subsystem: ASUSTeK Computer Inc. Device 8936
        Kernel driver in use: vfio-pci
        Kernel modules: nvidiafb, nouveau
root@pve:~# lspci -k -s 01:00.1
01:00.1 Audio device: NVIDIA Corporation AD107 High Definition Audio Controller (rev a1)
        Subsystem: ASUSTeK Computer Inc. Device 8936
        Kernel driver in use: vfio-pci
        Kernel modules: snd_hda_intel
root@pve:~# 
```
編集用windows VMにパススルーする準備完了


# 9) VM① Windows「開発」作成（RDPで使う）

Windows11 のISOファイルをダウンロード

https://www.microsoft.com/ja-jp/software-download/windows11
<img width="1528" height="1137" alt="image" src="https://github.com/user-attachments/assets/9ce41b52-2957-4e9c-8c07-c20867930ab7" />

localにアップロード

<img width="1849" height="1137" alt="image" src="https://github.com/user-attachments/assets/0a122594-46f0-4a0b-8af1-7a7490e00ea0" />

<img width="1849" height="1137" alt="image" src="https://github.com/user-attachments/assets/4e203e7b-630b-4e6a-8688-d0d793e19b3a" />


ドライバをダウンロード

https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/archive-virtio/virtio-win-0.1.285-1/

<img width="1772" height="1117" alt="image" src="https://github.com/user-attachments/assets/e89fcc2d-1d37-4f00-b52f-17a778039102" />

localにアップロード

<img width="1849" height="1137" alt="image" src="https://github.com/user-attachments/assets/47a7f5c8-dfec-4e31-a42e-3825a9271ba4" />

<img width="1849" height="1137" alt="image" src="https://github.com/user-attachments/assets/7d25cd96-3802-4221-824d-7ce3ff132913" />

<img width="1728" height="396" alt="image" src="https://github.com/user-attachments/assets/50437d91-704d-4f65-a676-d4726206624c" />

VMを作る

<img width="1849" height="1137" alt="image" src="https://github.com/user-attachments/assets/418e3535-43a7-4701-b115-a5dcf02425ca" />

<img width="1849" height="1137" alt="image" src="https://github.com/user-attachments/assets/46bb820c-a880-4614-9432-abcd9ff9c4d1" />

<img width="1849" height="1137" alt="image" src="https://github.com/user-attachments/assets/653b9dd8-cc03-458e-a037-e8a4a194daf0" />

aaa

<img width="1849" height="1137" alt="image" src="https://github.com/user-attachments/assets/16be32bd-0bc7-4c20-96a3-b5f8e36c6ce0" />


<img width="1849" height="1137" alt="image" src="https://github.com/user-attachments/assets/d0a7bbd9-acc0-4cda-a173-0023a7f97f26" />

<img width="1849" height="1137" alt="image" src="https://github.com/user-attachments/assets/43e12a03-30e3-42b6-87cd-40555dbfc4bd" />

<img width="1849" height="1137" alt="image" src="https://github.com/user-attachments/assets/948f7e8e-b994-4eea-9c8d-e27367de3e4c" />

確認

<img width="1849" height="1137" alt="image" src="https://github.com/user-attachments/assets/2c5a181a-30ff-4c9d-a5a9-aa0f2c3f8039" />

<img width="1849" height="1137" alt="image" src="https://github.com/user-attachments/assets/4c4db9d1-6fcc-4640-b952-ec75cc5e478f" />

Finish で作成開始

完成

<img width="1849" height="1137" alt="image" src="https://github.com/user-attachments/assets/7cda9e0f-8f96-4807-9e05-49dd321b62c0" />

ドライバを足す

<img width="1849" height="1137" alt="image" src="https://github.com/user-attachments/assets/144a31e1-ede7-41d1-b8fc-1f2758f30b08" />

<img width="1849" height="1137" alt="image" src="https://github.com/user-attachments/assets/e77b3c47-a41f-4d5f-b013-aebf5ef1c3df" />

SCSIで足すと、windwosセットアップ時に見えなかった

仮想ドライブを、windows VMに直付けする
```
root@pve:~# qm stop 100
root@pve:~# qm set 100 -scsi2 /dev/zvol/nvme2/d_vol,discard=on,iothread=1,cache=writeback
update VM 100: -scsi2 /dev/zvol/nvme2/d_vol,discard=on,iothread=1,cache=writeback
root@pve:~#
root@pve:~# qm set 100 -scsi3 /dev/zvol/nvme2/e_vol,discard=on,iothread=1,cache=writeback
update VM 100: -scsi3 /dev/zvol/nvme2/e_vol,discard=on,iothread=1,cache=writeback
root@pve:~# 

```

直付けできたか確認
```
root@pve:~# qm config 100 | egrep 'scsi[12]|scsihw'
boot: order=scsi0;ide2;net0;scsi1
scsi1: local:iso/virtio-win-0.1.285.iso,media=cdrom,size=771138K
scsi2: /dev/zvol/nvme2/d_vol,cache=writeback,discard=on,iothread=1,size=700G
scsihw: virtio-scsi-single
root@pve:~# 
```
```
root@pve:~# qm start 100
swtpm_setup: Not overwriting existing state file.
root@pve:~# 
```

<img width="1849" height="1137" alt="image" src="https://github.com/user-attachments/assets/497dc44b-c8a0-48d4-9424-2ec3c5911cbc" />

# RTX4060をWindows VMにパススルー

GPU本体( 0000:01:00.0)とオーディオ(0000:01:00.1)をパススルー
```
root@pve:~# qm set 100 -hostpci0 0000:01:00.0,pcie=1
update VM 100: -hostpci0 0000:01:00.0,pcie=1
root@pve:~# qm set 100 -hostpci1 0000:01:00.1,pcie=1
update VM 100: -hostpci1 0000:01:00.1,pcie=1
root@pve:~# qm config 100 | grep hostpci
hostpci0: 0000:01:00.0,pcie=1
hostpci1: 0000:01:00.1,pcie=1
root@pve:~#
```
IDは、以下のIDの頭に、0000 をつけたものらしい
```
root@pve:/hdds/pictures# lspci -nn | egrep -i 'nvidia|vga|audio'
01:00.0 VGA compatible controller [0300]: NVIDIA Corporation AD107 [GeForce RTX 4060] [10de:2882] (rev a1)
01:00.1 Audio device [0403]: NVIDIA Corporation AD107 High Definition Audio Controller [10de:22be] (rev a1)
```



最終の状態
<img width="1849" height="1137" alt="image" src="https://github.com/user-attachments/assets/b399da95-13cd-42d3-b511-15426d7ca7d1" />

GUIでの追加はうまくいかなかった


# WindowsVMを起動する

適当なキーを押す
<img width="1849" height="1137" alt="image" src="https://github.com/user-attachments/assets/de6558ee-8996-4341-95c4-aa687dd21f6f" />

Boot Managerに行く
<img width="1805" height="1093" alt="image" src="https://github.com/user-attachments/assets/653ccc3a-7d1c-4203-b897-c3c466345140" />

「UEFI QEMU DVD-ROM」を選ぶ（windows ISO）
<img width="1849" height="1137" alt="image" src="https://github.com/user-attachments/assets/8b34a5f1-8e70-4b65-afcb-e5250ef947dc" />

「何らかのキーを押して、CDかDVDから起動して」みたいなメッセージがほんの2秒ほど出るので、素早くEnterを押すと、、、

<img width="1849" height="1137" alt="image" src="https://github.com/user-attachments/assets/28670a1e-3a97-4d09-963e-f65984245441" />

<img width="1849" height="1137" alt="image" src="https://github.com/user-attachments/assets/26bd9be1-62ad-47b7-b230-a2af487536d0" />

<img width="1849" height="1137" alt="image" src="https://github.com/user-attachments/assets/968e9140-b018-4440-a72a-c5ab257a59e6" />

<img width="1849" height="1137" alt="image" src="https://github.com/user-attachments/assets/51365265-916a-4555-ae3e-6036295389fc" />

<img width="1849" height="1137" alt="image" src="https://github.com/user-attachments/assets/250882e3-e94f-415a-bbea-079bf1438afa" />

Load Driverをクリック
<img width="1849" height="1137" alt="image" src="https://github.com/user-attachments/assets/a7fb6c7d-2389-4331-9f09-09c553641c59" />

<img width="1849" height="1137" alt="image" src="https://github.com/user-attachments/assets/c1f8a16f-1573-4afc-a63e-7323f89f1eb7" />

これを選択して、Installをクリック
<img width="1805" height="1093" alt="image" src="https://github.com/user-attachments/assets/055fe198-498d-430d-bb09-c6fa4d14d259" />

<img width="1849" height="1137" alt="image" src="https://github.com/user-attachments/assets/32441916-cbd9-4edf-99d3-3e63504f203c" />

<img width="1849" height="1137" alt="image" src="https://github.com/user-attachments/assets/9427a3f6-c126-4bb9-9d5d-2a65891b1c94" />

インストールが始まる
<img width="1849" height="1137" alt="image" src="https://github.com/user-attachments/assets/6e17a817-b1af-401d-ba4c-83f6b2ad842c" />


<img width="1849" height="1137" alt="image" src="https://github.com/user-attachments/assets/cf553f23-2943-4526-acf7-c0a633e7a248" />

<img width="1849" height="1137" alt="image" src="https://github.com/user-attachments/assets/ab318cd4-8855-43ec-b125-6f93cccd7783" />

<img width="1849" height="1137" alt="image" src="https://github.com/user-attachments/assets/23f231a0-be07-48c6-b3eb-fb49c04d6df8" />

Install driver をクリック
<img width="1849" height="1137" alt="image" src="https://github.com/user-attachments/assets/edd571fe-0cae-4b73-92c9-47fd8e39826f" />

NetKVM\w11\amd64　を選択
<img width="1849" height="1137" alt="image" src="https://github.com/user-attachments/assets/804c4415-2cc7-4cb6-b138-473ef5f944f0" />

<img width="1849" height="1137" alt="image" src="https://github.com/user-attachments/assets/b4ab1587-8841-4797-9722-aa4502822701" />

<img width="1849" height="1137" alt="image" src="https://github.com/user-attachments/assets/678cbaa3-1e26-4b73-878e-6a630c30c26b" />

<img width="1849" height="1137" alt="image" src="https://github.com/user-attachments/assets/98d7d580-1236-44dd-b777-c00d8ea26eb7" />

<img width="1849" height="1137" alt="image" src="https://github.com/user-attachments/assets/2d83a1d0-1ea4-4be6-b57f-2614f75ce071" />

私用かビジネスか　私用を選んだ
<img width="2169" height="1226" alt="image" src="https://github.com/user-attachments/assets/e4600482-87d7-43c5-84ad-e4f59de3b0ce" />


<img width="1849" height="1137" alt="image" src="https://github.com/user-attachments/assets/6840262f-9de0-4067-9068-971880efd1ba" />

<img width="1849" height="1137" alt="image" src="https://github.com/user-attachments/assets/2c132ee4-3169-4776-902e-c7c593ad3898" />

<img width="1849" height="1137" alt="image" src="https://github.com/user-attachments/assets/ea234787-70b0-4f20-ac3f-7199eeac431d" />

<img width="1849" height="1137" alt="image" src="https://github.com/user-attachments/assets/a85a38d0-187b-404a-8f29-aef6569d0564" />

<img width="1849" height="1137" alt="image" src="https://github.com/user-attachments/assets/cdd3f7d0-0c7c-43d3-a6f6-76d9656fbaf4" />

<img width="1849" height="1137" alt="image" src="https://github.com/user-attachments/assets/72c36b33-db75-4f50-9079-bd422f1433d9" />

インストール完了
<img width="1849" height="1137" alt="image" src="https://github.com/user-attachments/assets/c03d4607-cbf7-4675-899b-a25864a9b46a" />

# VirtIO 一括インストール

virtio-win-guest-tools.exe（ISOのルート直下）を実行する（ダブルクリックで良い）
<img width="1849" height="1137" alt="image" src="https://github.com/user-attachments/assets/6850b261-6c94-40e4-8b4c-61fc422b1eed" />

<img width="1849" height="1137" alt="image" src="https://github.com/user-attachments/assets/8039157f-d7cc-49de-9656-416b9204cd7b" />

全部そのままNext -> Finish
<img width="1849" height="1137" alt="image" src="https://github.com/user-attachments/assets/c38e46f7-8d1c-43d1-a97e-c50db4291178" />

<img width="1849" height="1137" alt="image" src="https://github.com/user-attachments/assets/bd3eeb53-9895-44fa-8f61-88e3127a5c5f" />

# GPUドライバ（4060）を入れる

GPUに⚠️マークがついている これを無くすことを目指す
<img width="1849" height="1137" alt="image" src="https://github.com/user-attachments/assets/cc328c3d-47c0-4468-88a8-33ac14eb0348" />
<img width="1849" height="1137" alt="image" src="https://github.com/user-attachments/assets/113114ed-49ec-4015-b4dd-5130d4a0270f" />

<img width="1849" height="1137" alt="image" src="https://github.com/user-attachments/assets/9a40c560-5327-4063-8271-b5306a201626" />
<img width="1849" height="1137" alt="image" src="https://github.com/user-attachments/assets/886865cc-e8b3-4bd9-affb-ef9ad1749ee2" />

<img width="1849" height="1137" alt="image" src="https://github.com/user-attachments/assets/0b07c7ef-a378-47da-9d1f-1b36202006e6" />
<img width="1849" height="1137" alt="image" src="https://github.com/user-attachments/assets/298a57ba-b658-4fe2-beb0-60c655e6ac2f" />
<img width="1849" height="1137" alt="image" src="https://github.com/user-attachments/assets/19ace9b0-81fb-4eeb-a2c8-9976f065135a" />
<img width="1849" height="1137" alt="image" src="https://github.com/user-attachments/assets/6a158943-48d0-4524-a007-b7a7d10ad42d" />
<img width="1849" height="1137" alt="image" src="https://github.com/user-attachments/assets/fda4d6d6-9acb-4bc6-937e-ec7d6048c1da" />
<img width="1849" height="1137" alt="image" src="https://github.com/user-attachments/assets/3cdb19d4-53a4-4d3a-9750-fc947609d844" />
<img width="1849" height="1137" alt="image" src="https://github.com/user-attachments/assets/189edff4-32dc-43d5-82ca-e159c4c0090f" />
<img width="1849" height="1137" alt="image" src="https://github.com/user-attachments/assets/4638b98f-1bb4-434d-9424-8c8c3d1d220d" />

インストール完了 -> VM再起動
<img width="1849" height="1137" alt="image" src="https://github.com/user-attachments/assets/b610ba18-4551-43bd-b55a-f68ba79839bb" />

This device is working propery と出ているのでOK
<img width="1849" height="1137" alt="image" src="https://github.com/user-attachments/assets/a5a32511-5a60-4b75-b9c3-16110fefa23d" />

Audioも正常に動いている
<img width="1849" height="1137" alt="image" src="https://github.com/user-attachments/assets/e93fb9ac-4ac2-4597-94e4-0451ef579f1b" />

タスクマネージャーに、GPUが追加されている
<img width="1849" height="1137" alt="image" src="https://github.com/user-attachments/assets/161ab204-bebf-4b4f-a5e2-55db370ceea3" />

NVIDIA Control Panel を入れる
<img width="1849" height="1137" alt="image" src="https://github.com/user-attachments/assets/c314855d-1863-466e-b34a-bf61d7777edb" />
<img width="1849" height="1137" alt="image" src="https://github.com/user-attachments/assets/376953f9-1f6e-4ba9-a144-a25e207aa214" />
<img width="1849" height="1137" alt="image" src="https://github.com/user-attachments/assets/dbcca08b-a87a-4862-be85-668cf2e70f55" />


# d_vol, e_vol を割り当てる

今、Dドライブ、EドライブはCD-ROM（winsows iso とドライバ）になっている

<img width="759" height="539" alt="image" src="https://github.com/user-attachments/assets/07af962f-1ce9-478d-9171-640ae2a2e34f" />

Windows + R -> diskmgmt.msc で、Disk Management を開く
<img width="395" height="239" alt="image" src="https://github.com/user-attachments/assets/9795d1cc-3240-4c4b-909e-b0e5e56fcfa8" />
<img width="1849" height="1137" alt="image" src="https://github.com/user-attachments/assets/41d31a36-21fc-4490-9340-9e7c6f4c92c0" />

まずDドライブ(ドライバ)で右クリック -> Change Drive Letter and Paths... 
<img width="641" height="507" alt="image" src="https://github.com/user-attachments/assets/6239de02-b282-427d-a57a-139c7561dff7" />

Xを選んでOK 
<img width="630" height="496" alt="image" src="https://github.com/user-attachments/assets/9c414a4a-3670-4c26-9764-1e458276045e" />
<img width="773" height="525" alt="image" src="https://github.com/user-attachments/assets/00e4c805-9bdf-488a-baa2-5702a78bfaad" />

EもYに変える
<img width="732" height="486" alt="image" src="https://github.com/user-attachments/assets/07ebbd0b-06d5-4dc9-acfc-c6ec0d0570a5" />

DとEが空いたので、初期化する

d_vol, e_volは、Unallocated の状態で見えてはいる
<img width="637" height="504" alt="image" src="https://github.com/user-attachments/assets/01a321aa-84ee-44be-9ae5-1b05f1a6d1c6" />

d_vol, e_vol のどちらかの上で右クリック -> Initialize Disk
<img width="629" height="499" alt="image" src="https://github.com/user-attachments/assets/dc4fb606-bd13-4e7c-8576-e8b8d8abefce" />

GPTを選んで、initialize
<img width="631" height="500" alt="image" src="https://github.com/user-attachments/assets/dc10706c-7df2-4d98-8f5a-09000a8b3512" />

d_vol, e_vol上の、警告が消える -> e_volの上で右クリック -> New Simple Volume
<img width="625" height="496" alt="image" src="https://github.com/user-attachments/assets/e349e43c-a9e2-40f2-9667-5e4ae1f0c9b8" />

そのままOK
<img width="630" height="503" alt="image" src="https://github.com/user-attachments/assets/1de0a062-79e7-4b0f-a3ee-4bd901acbd8d" />

そのままOK（デフォルトで、割り当てられる最大容量になっている　これ以上上に行けない）
<img width="628" height="497" alt="image" src="https://github.com/user-attachments/assets/c8de0c8d-f1dc-4478-94d7-553b0b9b07a1" />

文字Dを割り当てる
<img width="627" height="498" alt="image" src="https://github.com/user-attachments/assets/b3e83258-bf6f-4bce-8c4b-27e9c7cea54f" />

<img width="628" height="490" alt="image" src="https://github.com/user-attachments/assets/b228ac23-95f8-4d0d-8628-c95bed9a85d3" />

Dドライブにできた
<img width="704" height="525" alt="image" src="https://github.com/user-attachments/assets/16cf801d-1ab9-4d88-ac50-41f8d5cc76f3" />

続いてEドライブも作る
<img width="636" height="545" alt="image" src="https://github.com/user-attachments/assets/a5dccbf8-1099-47ee-aa11-76ae4921429e" />
<img width="626" height="501" alt="image" src="https://github.com/user-attachments/assets/d4f69f2f-5fe3-4425-8519-f17b8fbb209b" />
<img width="624" height="499" alt="image" src="https://github.com/user-attachments/assets/66a75eab-b128-4b4e-9dfb-c17ed3e808ce" />
<img width="626" height="498" alt="image" src="https://github.com/user-attachments/assets/6691f752-abe6-47b1-be80-a711a1394f3b" />
<img width="623" height="494" alt="image" src="https://github.com/user-attachments/assets/9445ccd0-ad33-4d1e-a32f-0dfce6ecd5b8" />
<img width="700" height="543" alt="image" src="https://github.com/user-attachments/assets/9d031af0-1bb8-4543-810d-dd3e77ca03cc" />

# D,Eドライブを、Windows Defenderの管轄外にする（ここは速度重視する）

理由：インデックスは検索を速くしますが、キャッシュ用途では不要で I/O を増やすだけ。

手順
```
設定 → Privacy & Security → Windows Security → Virus & threat protection → Manage settings → Add or remove exclusions → Add an exclusion → Folder → D:\ と E:\ を追加。
```
<img width="1010" height="581" alt="image" src="https://github.com/user-attachments/assets/93b0f589-1653-4394-9817-041e0d1456b1" />

<img width="383" height="420" alt="image" src="https://github.com/user-attachments/assets/816549a7-fc2c-4483-93aa-1fe456382657" />

# Eドライブのインデックスをdisableする（速度重視）

「このドライブ上のファイルに対し、プロパティだけでなく…」のチェックを外す → 適用
<img width="494" height="582" alt="image" src="https://github.com/user-attachments/assets/da30b782-0092-4a79-b983-1fc16d5be4a7" />

再起的にdisabbleしてOK
<img width="528" height="575" alt="image" src="https://github.com/user-attachments/assets/e0301b1e-13da-453c-affc-f964b60c9f53" />

# パフォーマンス重視にする

Win + R -> powercfg.cpl -> High performance を選択
<img width="660" height="498" alt="image" src="https://github.com/user-attachments/assets/9b1a103d-9d58-42a9-977b-3fb98ce6fdee" />

# VMバックアップ

hdds(hdd 4tb ミラー)に、VMバックアップ用の領域(/hdds/backups)を作る 
```
root@pve:~# zfs create hdds/backups
root@pve:~# zfs list
NAME                  USED  AVAIL  REFER  MOUNTPOINT
hdds                 5.13M  3.51T   104K  /hdds
hdds/backups           96K  3.51T    96K  /hdds/backups
hdds/documents         96K  3.51T    96K  /hdds/documents
hdds/pictures        4.05M  3.51T  4.05M  /hdds/pictures
nvme1                 201G   698G    96K  /nvme1
nvme1/vm-100-disk-0     3M   698G   124K  -
nvme1/vm-100-disk-1   201G   864G  35.2G  -
nvme1/vm-100-disk-2     6M   698G    64K  -
nvme2                 899G   763M    96K  /nvme2
nvme2/d_vol           703G   703G   370M  -
nvme2/e_vol           196G   196G   197M  -
rpool                7.79G   215G   104K  /rpool
rpool/ROOT           2.15G   215G    96K  /rpool/ROOT
rpool/ROOT/pve-1     2.15G   215G  2.15G  /
rpool/data             96K   215G    96K  /rpool/data
rpool/var-lib-vz     5.63G   215G  5.63G  /var/lib/vz
root@pve:~# 
```
local（250gb ssd ミラー）でも容量が95%くらい余っているので良かったが、OS死んだ時にVMバックアップも死ぬのが微妙

圧縮形式等を変更
```
root@pve:~# zfs get compression,atime,recordsize hdds/backups
NAME          PROPERTY     VALUE           SOURCE
hdds/backups  compression  zstd-3          inherited from hdds
hdds/backups  atime        off             inherited from hdds
hdds/backups  recordsize   128K            default
root@pve:~# zfs set compression=zstd atime=off recordsize=1M hdds/backups
root@pve:~# zfs get compression,atime,recordsize hdds/backups
NAME          PROPERTY     VALUE           SOURCE
hdds/backups  compression  zstd            local
hdds/backups  atime        off             local
hdds/backups  recordsize   1M              local
root@pve:~# 
```

バックアップ領域(/hdds/backups) を、Proxmoxに登録 -> バックアップ時、Proxmoxから見えるようにする
<img width="1849" height="1137" alt="image" src="https://github.com/user-attachments/assets/0d65ff0f-28bf-4361-980a-0715cf0bdc9d" />

登録完了
<img width="1849" height="1137" alt="image" src="https://github.com/user-attachments/assets/7ed291fa-4003-4025-b910-8073a27e178e" />

バックアップ取得開始
<img width="1805" height="1093" alt="image" src="https://github.com/user-attachments/assets/5d931287-28fb-43aa-81d4-80be8c8bd95d" />

<img width="1849" height="1137" alt="image" src="https://github.com/user-attachments/assets/7806fa6c-835d-4549-b37b-58a2e6971c7d" />

<img width="1849" height="1137" alt="image" src="https://github.com/user-attachments/assets/c8a36aac-98db-4e1e-9b8e-48ef60502d49" />

# プライベートIPを固定する

今のIPは、、、

Settings -> Network & internet -> Ethernet -> IPv4 address:192.168.11.16

192.168.11.100 にしたい(空いてることはpingで確認済み)

<img width="831" height="520" alt="image" src="https://github.com/user-attachments/assets/1e943938-0547-48e7-a5af-06a740c66d01" />

<img width="502" height="631" alt="image" src="https://github.com/user-attachments/assets/374dd846-9de0-4b02-b917-0a957346b36e" />

command prompt で、ipconfigで、確かに変わっていることを確認

# 写真をSambaに移動

```
root@pve:/hdds/pictures# mkdir _inbox
```
権限調整（editorsグループメンバーが書き込めるようにする）
```
root@pve:/hdds/pictures# ls -lah
total 1.5K
drwxrwsr-x 3 root editors 3 Sep 22 09:36 .
drwxr-xr-x 5 root root    5 Sep 22 08:56 ..
drwxr-sr-x 2 root editors 2 Sep 22 09:36 _inbox
root@pve:/hdds/pictures# chmod 2775 _inbox
root@pve:/hdds/pictures# ls -lah
total 1.5K
drwxrwsr-x 3 root editors 3 Sep 22 09:36 .
drwxr-xr-x 5 root root    5 Sep 22 08:56 ..
drwxrwsr-x 2 root editors 2 Sep 22 09:36 _inbox
root@pve:/hdds/pictures# find /hdds/pictures/_inbox -type d -exec chmod 2775 {} \;
root@pve:/hdds/pictures# find /hdds/pictures/_inbox -type f -exec chmod 664 {} \;
root@pve:/hdds/pictures# 
```
```
root@pve:~# apt install -y libimage-exiftool-perl
```
振り分けコマンド
```
exiftool -r -P -d %Y%m \
  '-Directory</hdds/pictures/${DateTimeOriginal}' \
  /hdds/pictures/_inbox
```
残ったら順番に下を試していく
```
exiftool -r -P -d %Y%m   '-Directory</hdds/pictures/${CreateDate}'   /hdds/pictures/_inbox
```
```
exiftool -r -P -d %Y%m   '-Directory</hdds/pictures/${MediaCreateDate}'   /hdds/pictures/_inbox
```
```
exiftool -r -P -d %Y%m   '-Directory</hdds/pictures/${FileModifyDate}'   /hdds/pictures/_inbox
```


# 開発用windows VM作成

CPUだけ6Core あとは全く同じ

# UbuntuVM作成

cloud image をダウンロード

<img width="1530" height="1338" alt="image" src="https://github.com/user-attachments/assets/8d4692f8-2c01-422c-adaa-1f7c44bd6ad4" />

Proxmoxにアップロードしたいが、アップロードできる場所がない

<img width="1593" height="405" alt="image" src="https://github.com/user-attachments/assets/498ae6ff-d4ad-4dcb-bc00-d34ff8ad8a26" />

CLIで local (/var/lib/vz/ 下) にアップロードする

```
root@pve:/var/lib/vz/images# wget https://cloud-images.ubuntu.com/noble/current/noble-server-cloudimg-amd64.img \
     -O ubuntu-24.04-cloudimg-amd64.img
--2025-09-23 11:19:59--  https://cloud-images.ubuntu.com/noble/current/noble-server-cloudimg-amd64.img
Resolving cloud-images.ubuntu.com (cloud-images.ubuntu.com)... 185.125.190.37, 185.125.190.40, 2620:2d:4000:1::17, ...
Connecting to cloud-images.ubuntu.com (cloud-images.ubuntu.com)|185.125.190.37|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 618925568 (590M) [application/octet-stream]
Saving to: ‘ubuntu-24.04-cloudimg-amd64.img’

ubuntu-24.04-cloudimg-amd64.img   100%[=============================================================>] 590.25M  5.79MB/s    in 50s     

2025-09-23 11:20:50 (11.8 MB/s) - ‘ubuntu-24.04-cloudimg-amd64.img’ saved [618925568/618925568]

root@pve:/var/lib/vz/images# ls
ubuntu-24.04-cloudimg-amd64.img
```

テンプレートの箱だけ一旦作る

<img width="1835" height="1131" alt="image" src="https://github.com/user-attachments/assets/9ec901b7-5714-43c1-b1c2-85aa3d933c60" />

<img width="1835" height="1131" alt="image" src="https://github.com/user-attachments/assets/f009a2b9-2f7a-4200-9b72-57d243a8b9d3" />

<img width="1835" height="1131" alt="image" src="https://github.com/user-attachments/assets/19ca79be-95c2-4420-a1ec-75dc5d80aec0" />

左の scsi0 を選んで、右上のゴミ箱をクリック → 既存ディスクを削除

何もディスクがない状態で Next →
<img width="1835" height="1131" alt="image" src="https://github.com/user-attachments/assets/cd6b6782-8016-4257-98e7-b94099eb4685" />

<img width="1835" height="1131" alt="image" src="https://github.com/user-attachments/assets/345104b2-af22-4ea4-ab09-df43651ef917" />
<img width="1835" height="1131" alt="image" src="https://github.com/user-attachments/assets/40491174-7b33-4929-94ce-2d5e79634dbd" />
<img width="1835" height="1131" alt="image" src="https://github.com/user-attachments/assets/189a6176-74ce-41d0-b3cf-d6869cd19fe1" />
<img width="1835" height="1131" alt="image" src="https://github.com/user-attachments/assets/34cd8c77-1d18-4ba9-8be3-f04fe9cb1ed8" />

箱をテンプレートとして充実させていく

仮想マシンID 9000 専用に、ubuntu-24.04-cloudimg-amd64.img というディスクイメージを local というストレージにインポートする
```
root@pve:/var/lib/vz/images# qm importdisk 9000 ubuntu-24.04-cloudimg-amd64.img local
importing disk 'ubuntu-24.04-cloudimg-amd64.img' to VM 9000 ...
Formatting '/var/lib/vz/images/9000/vm-9000-disk-0.raw', fmt=raw size=3758096384 preallocation=off
transferred 0.0 B of 3.5 GiB (0.00%)
transferred 35.8 MiB of 3.5 GiB (1.00%)
transferred 71.7 MiB of 3.5 GiB (2.00%)
transferred 109.0 MiB of 3.5 GiB (3.04%)
transferred 144.8 MiB of 3.5 GiB (4.04%)
...
transferred 3.4 GiB of 3.5 GiB (97.14%)
transferred 3.4 GiB of 3.5 GiB (98.14%)
transferred 3.5 GiB of 3.5 GiB (99.14%)
transferred 3.5 GiB of 3.5 GiB (100.00%)
transferred 3.5 GiB of 3.5 GiB (100.00%)
unused0: successfully imported disk 'local:9000/vm-9000-disk-0.raw'
```
その結果として、9000 というディレクトリが作成され、その中に vm-9000-disk-0.raw という名前でディスクイメージが保存されます。
```
root@pve:/var/lib/vz/images# ls
9000  ubuntu-24.04-cloudimg-amd64.img
root@pve:/var/lib/vz/images# cd 9000/
root@pve:/var/lib/vz/images/9000# ls
vm-9000-disk-0.raw
```
↑でインポートしたイメージを、アタッチする（SCSI0にアタッチ）
```
root@pve:~# pvesm list local | grep 9000
local:9000/vm-9000-disk-0.raw        raw     images    3758096384 9000
root@pve:~# qm set 9000 --scsihw virtio-scsi-single --scsi0 local:9000/vm-9000-disk-0.raw
update VM 9000: -scsi0 local:9000/vm-9000-disk-0.raw -scsihw virtio-scsi-single
```
アタッチできた
<img width="1791" height="1087" alt="image" src="https://github.com/user-attachments/assets/0caf8069-a259-4aa8-aeb3-cc1dd4f8932e" />

Proxmoxの仮想マシンID 9000 に、Cloud-init の設定ディスクを IDE2 デバイスとしてアタッチしています。
```
root@pve:~# qm set 9000 --ide2 local:cloudinit
update VM 9000: -ide2 local:cloudinit
Formatting '/var/lib/vz/images/9000/vm-9000-cloudinit.qcow2', fmt=qcow2 cluster_size=65536 extended_l2=off preallocation=metadata compression_type=zlib size=4194304 lazy_refcounts=off refcount_bits=16
ide2: successfully created disk 'local:9000/vm-9000-cloudinit.qcow2,media=cdrom'
generating cloud-init ISO
```
Cloud-init とは？
```
Cloud-initは、VMが初めて起動する際に、ネットワーク設定、ホスト名、ユーザーアカウント、SSHキー などを自動的に設定するためのオープンソースのパッケージです。

このコマンドを実行することで、手動でIPアドレスやパスワードを設定しなくても、VMが起動時にProxmox WebUIの「Cloud-init」タブで設定した内容を自動で適用できるようになります。これにより、テンプレートからVMをクローンした後の初期設定が不要になり、作業が大幅に効率化されます。
```

Cloud-init の設定
```
root@pve:~# qm set 9000 --ciuser ubuntu
update VM 9000: -ciuser ubuntu
```

<img width="397" height="145" alt="image" src="https://github.com/user-attachments/assets/a8e39c8f-488d-427b-abe5-b4207812caff" />

DHCPでIPを振るようにする（これをしないと、IPが振られなかった）
```
root@pve:~# qm set 9000 --ipconfig0 ip=dhcp
update VM 9000: -ipconfig0 ip=dhcp
```

60GBにする
```
root@pve:~# qm disk resize 9000 scsi0 60G
Image resized.
```

開発用VMの鍵を発行する
<img width="397" height="145" alt="image" src="https://github.com/user-attachments/assets/4be0b9f8-01b8-4ac6-b49a-c6d2d950bb06" />

鍵をcloud init に登録する
<img width="1352" height="757" alt="image" src="https://github.com/user-attachments/assets/dda39cd1-77ae-4b6a-a0e1-50e4f8e24275" />


起動順番をUbuntuディスクイメージ（SCSI0）から　にする
```
root@pve:~# qm set 9000 --boot order=scsi0
update VM 9000: -boot order=scsi0
root@pve:~# 
```

ubuntu ユーザーのパスワードも登録しておく -> 「Regenerate Image」で反映する
<img width="547" height="431" alt="image" src="https://github.com/user-attachments/assets/e8281404-2f58-4d02-9109-e3eb8369c826" />


テンプレート化する
```
root@pve:~# qm template 9000
```

テンプレートのアイコンが変わる

<img width="162" height="34" alt="image" src="https://github.com/user-attachments/assets/52429cdc-02b2-4530-84fb-f17bd0d473ae" />

テンプレートから、VMを作る
```
root@pve:~# qm clone 9000 9010 --name ubuntu-dev-01 --full 0
create linked clone of drive efidisk0 (nvme1:base-9000-disk-0)
create full clone of drive ide2 (local:9000/vm-9000-cloudinit.qcow2)
Formatting '/var/lib/vz/images/9010/vm-9010-cloudinit.qcow2', fmt=qcow2 cluster_size=65536 extended_l2=off preallocation=metadata compression_type=zlib size=4194304 lazy_refcounts=off refcount_bits=16
create linked clone of drive scsi0 (local:9000/base-9000-disk-0.raw)
clone 9000/base-9000-disk-0.raw: images, vm-9010-disk-0.qcow2, 9010 to vm-9010-disk-0.qcow2 (base=../9000/base-9000-disk-0.raw)
Formatting '/var/lib/vz/images/9010/vm-9010-disk-0.qcow2', fmt=qcow2 cluster_size=131072 extended_l2=on preallocation=off compression_type=zlib size=3758096384 backing_file=../9000/base-9000-disk-0.raw backing_fmt=raw lazy_refcounts=off refcount_bits=16
root@pve:~# 
```

スペックが指定通りになっている
<img width="1141" height="511" alt="image" src="https://github.com/user-attachments/assets/547651b9-e7af-4ef4-b68c-36eca5a472c5" />

VMを開始する
```
root@pve:~# qm start 9010
generating cloud-init ISO
root@pve:~# 
```
Ubuntuユーザーで、先ほど指定したパスワードでログインできる

<img width="1835" height="1131" alt="image" src="https://github.com/user-attachments/assets/f1105699-246e-4782-9a61-fe126ae57d9d" />

IPを確認する
<img width="683" height="208" alt="image" src="https://github.com/user-attachments/assets/2a7f8d67-f6b7-417f-8344-1eaadf6fce73" />

開発用windows VMからSSHできた

<img width="654" height="123" alt="image" src="https://github.com/user-attachments/assets/f25daeb3-06ed-4f11-824a-aad44d9ace8d" />

<img width="599" height="143" alt="image" src="https://github.com/user-attachments/assets/988bc65f-ecb2-42d9-acb1-ba76292e0f71" />




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





