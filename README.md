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

#### Configure our system (timezone, locale, hostname, etc.)

```python
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

# Internal clock
timedatectl set-ntp true
hwclock --systohc
```

#### Install our system packages

We can now install the "system packages" we need for the system to work correctly.  
This part is highly subjective, as you could need some other packages or not need some of them.

In the end, we will have a functional terminal-based system, without desktop environment.

```python
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

#### Configure our users

Here we will enable root account, and create a user with sudo rights.

```python
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
