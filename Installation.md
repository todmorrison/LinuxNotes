# System76 Laptop Arch Linux Install Notes

## Disk Preparation

1. Partition main storage device using fdisk utility. You can find storage device name using lsblk command.
```
    $ fdisk /dev/nvme0n1
                    [repeat this command until all existing partitions are deleted]
    Command (m for help): d
    Command (m for help): d

                    [create partition 1: efi]
    Command (m for help): n
    Partition number (1-128, default 1): Enter ↵
    First sector (..., default 2048): Enter ↵
    Last sector ...: +2G

                    [create partition 2: luks container]
    Command (m for help): n
    Partition number (2-128, default 2): Enter ↵
    First sector (..., default ...): Enter ↵
    Last sector ...: Enter ↵

                    [change partition types]
    Command (m for help): t
    Partition number (1-3, default 1): 1
    Partion type or alias (type L to list all): uefi
    Command (m for help): t
    Partition number (1-3, default 2): 2
    Partion type or alias (type L to list all): linux

                    [write partitioning to disk]
    Command (m for help): w
```
2. Create the encrypted LUKS2 container

```$ cryptsetup luksFormat /dev/nvme0n1p2```

3. Create 
```
$ sudo cryptsetup luksOpen /dev/nvme0n1p2 cryptdata
# Enter passphrase for /dev/nvme0n1p2:
$ sudo pvs
#  PV                    VG   Fmt  Attr PSize  PFree
#  /dev/mapper/cryptdata data lvm2 a--  47.39g    0
sudo vgs
#  VG   #PV #LV #SN Attr   VSize  VFree
#  data   1   1   0 wz--n- 47.39g    0
sudo lvs
#  LV   VG   Attr       LSize  Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
#  root data -wi-a----- 47.39g
sudo lsblk /dev/mapper/data-root -f
#NAME      FSTYPE FSVER LABEL UUID                                 FSAVAIL FSUSE% MOUNTPOINTS
#data-root ext4   1.0         bcb96b85-03df-45ca-92aa-6a182386631b 
```

Create a physical volume on top of the opened LUKS container:

```$ pvcreate /dev/mapper/cryptdata```

Create a volume group (in this example, it is named SSDVolGroup) and add the previously created physical volume to it:

```$ vgcreate SSDVolGroup /dev/mapper/cryptdata```

Create all your logical volumes on the volume group:

```
$ lvcreate -L 384G SSDVolGroup -n Arch
$ lvcreate -L 128G SSDVolGroup -n PopOS
$ lvcreate -l 100%FREE SSDVolGroup -n home
$ lvreduce -L -72G SSDVolGroup/home
$ lvcreate -l 100%FREE SSDVolGroup -n swap
```

Format your file systems on each logical volume:

```
$ mkfs.btrfs /dev/SSDVolGroup/Arch
$ mkfs.btrfs /dev/SSDVolGroup/home
$ mkswap /dev/SSDVolGroup/swap
```

Create **subvolumes**:
```
...
```

Mount your file systems:

```
$ mount -o subvol=/@ /dev/SSDVolGroup/Arch /mnt
$ mount --mkdir -o /dev/SSDVolGroup/home /mnt/home
$ swapon /dev/SSDVolGroup/swap
```
