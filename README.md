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

Note that we need to create each subfolder of our root filesystem, like /home, /var, etc.  
Thus we can create as many subvolumes as we need (like /var, /var/log, /var/tmp, etc.)

```python
# CHANGE THIS WITH YOUR DATA
BTRFS_OPTS="rw,noatime,ssd,compress=zstd,commit=120"
EFI_PART="/dev/sda1"

# First mount, to create the subvolumes
mount /dev/mapper/$MAP_NAME /mnt

# Creation of subvolumes, with naming format compatible with Timeshift backup tool for example
btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@home
btrfs subvolume create /mnt/@snapshots
umount /mnt

# Second mount, with correct options
mount -o $BTRFS_OPTS,subvol=@ /dev/mapper/$MAP_NAME /mnt

mkdir -p /mnt/home
mount -o $BTRFS_OPTS,subvol=@home /dev/mapper/$MAP_NAME /mnt/home
mkdir -p /mnt/.snapshots
mount -o $BTRFS_OPTS,subvol=@snapshots /dev/mapper/$MAP_NAME /mnt/.snapshots
mkdir -p /mnt/boot/efi
mount -o rw,noatime $EFI_PART /mnt/boot/efi

# We can check that everything in correctly mounted
lsblk
```

#### Install ArchLinux base system

The next step is to :
- install basic packages for our ArchLinux system,
- generate the fstab,
- chroot in our basic system.  

```python
# Install basic recommended packages
pacstrap /mnt base linux-lts linux-firmware reflector \
              sudo rsync nano \
              btrfs-progs

# Generate the fstab
genfstab -U /mnt >> /mnt/etc/fstab

# Chroot in our system
arch-chroot /mnt
```
