## ArchLinux installation (BTRFS + Full Disk Encryption)

#### Create LUKS-1 encryption and BTRFS volume

The first step is to create the LUKS encrypted partition.
If you want an **encrypted boot**, you have to choose the **luks1** type, as GRUB is not able to decrypt luks2 for now (see luks documentation for more details).

Then, inside this encrypted partition, we create a BTRFS filesystem.

```python
# CHANGE THIS WITH YOUR DATA
DISK_NAME="/dev/sda2"
MAP_NAME="luksRoot"

# LUKS encryption
cryptsetup luksFormat --type luks1 $CRYPT
cryptsetup luksOpen $CRYPT $MAP_NAME

# BTRFS formatting
mkfs.btrfs /dev/mapper/$MAP_NAME
```

#### Create BTRFS subvolumes and correctly mount on /mnt

Then, we have to :
- mount the BTRFS partition on /mnt,
- create the subvolumes,
- and remount it correctly.

Note that the UEFI partition cannot be encrypted, and has to be formatted in FAT.

```python
# CHANGE THIS WITH YOUR DATA
BTRFS_OPTS="rw,noatime,ssd,compress=zstd,commit=120"
EFI_PART="/dev/sda1"

# First mount, to create the subvolumes
mount /dev/mapper/$MAP_NAME /mnt


```
