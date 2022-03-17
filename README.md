## ArchLinux installation (Full Disk Encryption)

### Create LUKS-1 encryption and BTRFS volume

```python

CRYPT="/dev/nvme0n1p2"
CLEAR="luksRoot"

# LUKS encryption
cryptsetup luksFormat --type luks1 $CRYPT
cryptsetup luksOpen $CRYPT $CLEAR

# BTRFS formatting
mkfs.btrfs /dev/mapper/$CLEAR
```
