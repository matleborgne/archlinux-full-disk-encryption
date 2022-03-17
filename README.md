## ArchLinux installation (Full Disk Encryption)

#### Create LUKS-1 encryption and BTRFS volume

The first step is to create the LUKS encrypted partition.
If you want an **encrypted boot**, you have to choose the **luks1** type (see luks documentation for more details).

Then, inside this encrypted partition, we create a BTRFS filesystem.

```python

CRYPT="/dev/nvme0n1p2"
CLEAR="luksRoot"

# LUKS encryption
cryptsetup luksFormat --type luks1 $CRYPT
cryptsetup luksOpen $CRYPT $CLEAR

# BTRFS formatting
mkfs.btrfs /dev/mapper/$CLEAR
```

#### Create BTRFS subvolumes and correctly mount on /mnt

