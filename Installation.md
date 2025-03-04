# System76 Laptop Arch Linux Install Notes

## Motivation

The goal is to have thoroughly encrypted system with btrfs snapshots and backups. Primarily, I use
Arch Linux. However, since System 76 develops PopOS and it is useful to have a "known good"
configuration when diagnosing suspected hardware issues and the like, I add a smaller PopOS
installation to the mix.

## I. Disk Preparation

1. Partition main storage device using fdisk utility. You can find storage device name using lsblk command.

    ```
    # lsblk
    NAME                    MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINTS
    ...
    nvme0n1                 259:0    0  1.8T  0 disk
    ├─nvme0n1p1             259:1    0    2G  0 part
    └─nvme0n1p2             259:2    0  1.8T  0 part
    # fdisk /dev/nvme0n1
    ```
    Delete existing partitions, repeat as needed until all existing partitions are removed.
    ```
    Command (m for help): d
    Command (m for help): d
    ```
    Create EFI partition first:
    ```
    Command (m for help): n
    Partition number (1-128, default 1): Enter ↵
    First sector (..., default 2048): Enter ↵
    Last sector ...: +2G
    ```
    Then create the LUKS container (partition):
    ```
    Command (m for help): n
    Partition number (2-128, default 2): Enter ↵
    First sector (..., default ...): Enter ↵
    Last sector ...: Enter ↵
    ```
    and change the partition types:
    ```
    Command (m for help): t
    Partition number (1-3, default 1): 1
    Partion type or alias (type L to list all): uefi
    Command (m for help): t
    Partition number (1-3, default 2): 2
    Partion type or alias (type L to list all): linux
    ```

    Finally, write the partitioning to disk:
    ```
    Command (m for help): w
    ```

2. Create the encrypted LUKS2 container

    ```
    # cryptsetup luksFormat /dev/nvme0n1p2
    ```

3. Open the LUKS2 container and create a physical volume on top of it:

    ```
    # sudo cryptsetup luksOpen /dev/nvme0n1p2 cryptdata
    Enter passphrase for /dev/nvme0n1p2:
    # pvcreate /dev/mapper/cryptdata
    ```

4. Create a volume group (in this example, it is named SSDVolGroup) and add the previously created physical volume to it:

    ```
    # vgcreate SSDVolGroup /dev/mapper/cryptdata
    ```

5. Create all your logical volumes on the volume group (384G for Arch Linux, 128G for PopOS, 72G
   swap at the end, and the rest for /home):

    ```
     # lvcreate -L 384G SSDVolGroup -n Arch
     # lvcreate -L 128G SSDVolGroup -n PopOS
     # lvcreate -l 100%FREE SSDVolGroup -n home
     # lvreduce -L -72G SSDVolGroup/home
     # lvcreate -l 100%FREE SSDVolGroup -n swap
    ```

6. Format your file systems on each logical volume and the EFI partition:

    ```
     # mkfs.fat -F 32 -n ESP /dev/nvme0n1p1 
     # mkfs.btrfs -L Arch /dev/SSDVolGroup/Arch
     # mkfs.btrfs -L Home /dev/SSDVolGroup/home
     # mkswap /dev/SSDVolGroup/swap
    ```
    (Note: do not format the volume for PopOS as the PopOS installer expects an empty
    partition/volume.)

6. Mount the Arch volume to create **subvolumes**:

    ```
    # btrfs subvolume create /mnt/@
    # btrfs subvolume create /mnt/@snapshots
    # btrfs subvolume create /mnt/@cache
    # btrfs subvolume create /mnt/@libvirt
    # btrfs subvolume create /mnt/@log
    # btrfs subvolume create /mnt/@tmp
    ```
    (Note: the Btrfs file system must be mounted to create the subvolumes, but we will remount it the
    way we want in the next step.)

8. Unmount the Arch volume and then mount the subvolumes (as well as the "home" and swap file
   systems) as appropriate:

    ```
     # umount /mnt
     # mount -o subvol=/@ /dev/SSDVolGroup/Arch /mnt
     # mount -o subvol=/@snapshots /dev/SSDVolGroup/Arch /mnt/.snapshots
     # mount -o subvol=/@cache /dev/SSDVolGroup/Arch /mnt/var/cache
     # mount -o subvol=/@libvirt /dev/SSDVolGroup/Arch /mnt/var/lib/libvirt
     # mount -o subvol=/@log /dev/SSDVolGroup/Arch /mnt/var/log
     # mount -o subvol=/@tmp /dev/SSDVolGroup/Arch /mnt/var/tmp
     # mount --mkdir -o /dev/SSDVolGroup/home /mnt/home
     # swapon /dev/SSDVolGroup/swap
    ```
