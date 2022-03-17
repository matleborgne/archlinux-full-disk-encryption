## ArchLinux installation (Full Disk Encryption)

#### Create LUKS-1 encryption and BTRFS volume

The first step is to create the LUKS encrypted partition.
If you want an **encrypted boot**, you have to choose the **luks1** type, as GRUB is not able to decrypt luks2 for now (see luks documentation for more details).

Then, inside this encrypted partition, we create a BTRFS filesystem.

```python
# CHANGE THIS WITH YOUR DATA
CRYPT="/dev/nvme0n1p2"
CLEAR="luksRoot"

# LUKS encryption
cryptsetup luksFormat --type luks1 $CRYPT
cryptsetup luksOpen $CRYPT $CLEAR

# BTRFS formatting
mkfs.btrfs /dev/mapper/$CLEAR
```

#### Create BTRFS subvolumes and correctly mount on /mnt

Then, we have to :
- mount the BTRFS partition on /mnt,
- create the subvolumes,
- and remount it correctly.

```python
# Options de montage BTRFS
BTRFS_OPTS="rw,noatime,ssd,compress=zstd,commit=120"
DISK_NAME="/dev/sda2"
MAP_NAME="luksRoot"
EFI_PART="/dev/sda1"
```
