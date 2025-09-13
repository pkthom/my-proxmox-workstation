
http://proxmox.com/en/downloads/proxmox-virtual-environment/iso

https://pve.proxmox.com/wiki/Prepare_Installation_Media

https://pve.proxmox.com/wiki/Prepare_Installation_Media#_instructions_for_macos
```
pkthom@MacBook-Pro ~ % hdiutil convert Downloads/proxmox-ve_9.0-1.iso -format UDRW -o
pkthom@MacBook-Pro ~ % cd Downloads
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
