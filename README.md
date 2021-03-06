## ArchLinux installation (BTRFS + Full-disk Encryption)

### Preface

This guide provides instructions for an Arch Linux installation with full-disk encryption via BTRFS on LUKS, in order to protect your system against hacking by kernel image or initramfs modification.

For more information, refer to the ArchLinux wiki for :
- [ArchLinux general installation guide](https://wiki.archlinux.org/title/Installation_guide)
- [Archlinux encrypting BTRFS](https://wiki.archlinux.org/title/Dm-crypt/Encrypting_an_entire_system#Btrfs_subvolumes_with_swap)

The final result of `lsblk` will be :
NAME           | MAJ:MIN | RM  |  SIZE  | RO  | TYPE  | MOUNTPOINT  |
---------------|---------|-----|--------|-----|-------|-------------|
sda            |  8:0    |  0  |   1.8T |  0  | disk  |             |
├─sda1         |  8:1    |  0  |   256M |  0  | part  | /efi        |
├─sda2         |  8:2    |  0  |   1.8T |  0  | part  |             |
..└─luksRoot   |  254:0  |  0  |   1.8T |  0  | crypt | /home       |
..             |         |  0  |        |  0  |       | /.snapshots |
..             |         |  0  |        |  0  |       | /           |


### Prepare the installer

First step is connect to the [internet](https://wiki.archlinux.org/index.php/installation_guide#Connect_to_the_internet).  

Then, configure the installer according to your needs
```ini
# Change installer keyboard layout
loadkeys fr

# Update the system clock
timedatectl set-ntp true
```

### Create disk table and partitions

Use `fdisk`, `gdisk` or `cfdisk` to create a GPT partition table and the following partitions.
I will use
```
cfdisk /dev/sda
```
Number |    Size    |        Name         |
-------|------------|---------------------|
   1   | 256.0 MiB  | EFI System          |
   2   |   1.8 TiB  | Linux filesystem    |

Then format the EFI partition
```ini
mkfs.vfat -F32 /dev/sda1
```



### Create LUKS-1 encryption and BTRFS volume

The first step is to create the LUKS encrypted partition.
If you want an **encrypted boot**, you have to choose the **luks1** type, as GRUB is not able to decrypt luks2 for now (see [documentation](https://wiki.archlinux.org/title/Dm-crypt/Encrypting_an_entire_system#Preparing_the_disk_5) for more details).

Then, inside this encrypted partition, we create a BTRFS filesystem.

```ini
# CHANGE THIS WITH YOUR DATA
DISK_NAME="/dev/sda2"
MAP_NAME="luksRoot"

# LUKS encryption
cryptsetup luksFormat --type luks1 $CRYPT
cryptsetup luksOpen $CRYPT $MAP_NAME

# BTRFS formatting
mkfs.btrfs /dev/mapper/$MAP_NAME
```

### Create BTRFS subvolumes and correctly mount on /mnt

Then, we have to :
- mount the BTRFS partition on /mnt,
- create the subvolumes,
- and remount it correctly.

Note that we need to create each subfolder of our root filesystem, like /home, /var, etc.  
Thus we can create as many subvolumes as we need (like /var, /var/log, /var/tmp, etc.)

```ini
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

### Install ArchLinux base system

The next step is to :
- install basic packages for our ArchLinux system,
- generate the fstab,
- chroot in our basic system.  

```ini
# Install basic recommended packages
pacstrap /mnt base linux-lts linux-firmware reflector \
              sudo rsync nano \
              btrfs-progs

# Generate the fstab
genfstab -U /mnt >> /mnt/etc/fstab

# Chroot in our system
arch-chroot /mnt
```

### Configure our system (timezone, locale, hostname, etc.)

```ini
# CHANGE THIS WITH YOUR DATA
TZ="Europe/Paris"
LANG="fr_FR.UTF-8"
KEYMAP="fr"
HOSTNAME="myArch"

# Timezone
ln -sf /usr/share/zoneinfo/$TZ /etc/localtime

# Locales
sed -i "/$LANG/s/^#//g" /etc/locale.gen
locale-gen
echo 'LANG='$LANG >> /etc/locale.conf

# Keyboard
echo "KEYMAP="$KEYMAP >> /etc/vconsole.conf

# Hostname
echo $HOSTNAME >> /etc/hostname
echo "127.0.0.1 localhost
::1       localhost
127.0.1.1 "$HOSTNAME >> /etc/hosts

# Internal clock set to UTC
timedatectl set-ntp true
hwclock --systohc
```

### Install our system packages

We can now install the "system packages" we need for the system to work correctly.  
This part is highly subjective, as you could need some other packages or not need some of them.

In the end, we will have a functional terminal-based system (GRUB appart), without desktop environment.

```ini
# Filesystem packages
pacman -S --noconfirm --needed \
  ntfs-3g dosfstools usbutils sshfs

# Basic tools
pacman -S --noconfirm --needed \
  base-devel arch-install-scripts pacman-contrib \
  git cryptsetup vi less wget flatpak smartmontools sysfsutils \
  htop fish rsync neofetch lsb-release rpm-tools \
  openssh upower archlinux-keyring

# Network, printing, sound, security
pacman -S --noconfirm --needed \
  networkmanager openvpn openssl networkmanager-openvpn \
  cups cups-filters \
  sof-firmware alsa-{firmware,plugins,utils} pipewire-{alsa,pulse,jack,media-session} \
  nftables firewalld tldr

# Video
pacman -S --noconfirm --needed amd-ucode xf86-video-amdgpu    #AMD
#pacman -S --noconfirm --needed intel-ucode xf86-video-intel   #INTEL
#pacman -S --noconfirm --needed nvidia-{lts,settings,utils} libvdpau libva-vdpau-driver   #NVIDIA

# Battery (for laptops)
pacman -S --noconfirm --needed tlp tlp-rdw powertop

# Service activation (systemd)
systemctl enable \
  sshd reflector.timer fstrim.timer \
  NetworkManager cups.service firewalld \
  tlp
```

### Configure our users

Here we will enable root account, and create a user with sudo rights.

```ini
# CHANGE THIS WITH YOUR DATA
USER="mathieu"

# Create the root password
passwd

# Create and configure the user
useradd -m $USER
passwd $USER

# Add the user to sudo users (optional)
echo $USER" ALL=(ALL) ALL" >> /etc/sudoers.d/$USER

```

### Full disk encryption in GRUB configuration

Here is the tricky step : the GRUB configuration for full disk encryption, including /boot

```ini
# Installation of GRUB packages
pacman -S --noconfirm --needed \
  grub efibootmgr efitools os-prober \
  grub-btrfs
```
The first step is to generate the classic configuration files

```ini
# Initramfs generation
mkinitcpio -P

# GRUB installation
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=ArchLinux \
             --recheck $EFI_PART
grub-mkconfig -o /boot/grub/grub.cfg
```

Then, the manual configuration for GRUB.
This step is crucial. When executing lsblk, note :
- UUID of your **encrypted** partition (UUID of /dev/sda2 for me)
- UUID of your **mapped** volume (UUID of luksRoot for me)

```ini
# State of our system
lsblk -o UUID,NAME,LABEL,MOUNTPOINTS

# Activate the ability of GRUB to decrypt LUKS-1 on boot
echo "GRUB_ENABLE_CRYPTODISK=y" /etc/default/grub

# Configure the boot options to tell GRUB how to boot your system
nano /etc/default/grub

# Change GRUB_CMDLINE_LINUX_DEFAULT line with :
GRUB_CMDLINE_LINUX_DEFAULT="\
 cryptdevice=UUID=$ENCRYPTED_PARTITION_UUID:luksRoot \
 root=UUID=$MAPPED_VOLUME_UUID \
 rootflags=subvol=@ \
 cryptkey=rootfs:/etc/keys/keyfile.key"
```

Note that I indicated a "cryptkey" line, to unlock the root filesystem.
This is optional, but if you don't do that, you will have to enter your password twice : the first time for GRUB, the second time for your root filesystem.

If you don't want that optional part, just remove the "cryptkey" line.

Then we need to configure the initramfs :
- adding the btrfs module,
- adding our keyfile to embedded files (if a keyfile is used),
- adding the **encrypt** and **grub-btrfs-overlayfs** hooks.

```ini
nano /etc/mkinitcpio.conf

MODULES=(btrfs)
FILES=("/etc/keys/keyfile.key")
HOOKS=(base udev autodetect modconf block encrypt filesystems keyboard fsck grub-btrfs-overlayfs)
```

Finally, we need to regenerate the files with correct configuration.

```ini
mkinitcpio -P

grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=ArchLinux \
             --recheck $EFI_PART --removable
grub-mkconfig -o /boot/grub/grub.cfg
```

### Optional : Generate a keyfile to unlock the root filesystem

Note that you will still be prompted for your password before GRUB appears, as it will be needed to unlock your "boot partition".  
We will use **openssl** to generate the keyfile (this can also be done with **dd** or other methods).

```ini
# Install openssl
pacman -S openssl

# Create the folder and securize it
mkdir /keys && chmod 700 /keys

# Generate the keyfile
openssl genrsa -out /etc/keys/keyfile.key 2048
chmod 600 /etc/keys/keyfile.key
```

After the keyfile is generated, we will add it to the list of keys granted by our encrypted partition.
  
Note important things here :
- the **password unlocking at boot can be long**, especially if the encrypted partition has many keys. The reason is that keys have a "slot" (they are ordered), and cryptsetup will test them one by one, beginning with slot 0 ;
- the keyboard layout when entering the password will be **US-layout**, so qwerty ;
- the **keyfile unlocking is not long**, even when the keyfile is not in slot 0.

For these points, it is recommended to have the following "slots order" for the encrypted partition :

Key Slot | Type       | Layout               |
---------|------------|----------------------|
   0     | password   | US - qwerty          |
   1     | keyfile    | -                    |
   2     | password   | FR - azerty (for me) |
      
```ini
# Check Key Slots - ENABLED means occupied, DISABLED means free
cryptsetup luksDump /dev/sda2

# Add the keyfile to slot 1
cryptsetup luksAddKey /dev/sda2 /etc/keys/keyfile.key

# Add the password in your custom layout to slot 2 (for me in french azerty layout)
cryptsetup luksAddKey /dev/sda2

# Replace the slot 0 password by a US-keyboard password
cryptsetup luksKillSlot /dev/sda2 0

loadkeys us
cryptsetup luksAddKey /dev/sda2
```
