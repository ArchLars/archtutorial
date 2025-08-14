# Quick Arch Linux Tutorial

---

## Prerequisites

- A bootable Arch Linux USB (written with `dd` or similar)
- Internet connection
- UEFI system


---


### TUTORIAL PROPER - Non-HTML Version


## Step 0: Boot from ISO

Set up Norwegian keyboard layout and verify UEFI boot:

```bash
# Set Norwegian keyboard layout
loadkeys no-latin1

# Verify UEFI firmware
ls /sys/firmware/efi/efivars && echo "UEFI firmware detected"

# Ensure your network interface is listed and enabled, for example with ip-link:
ip link

# Wireless
iwctl
[iwd]# device list
[iwd]# station *name* scan
[iwd]# station name connect *SSID*
To exit the interactive prompt, send EOF by pressing Ctrl+d

# Sync system clock
timedatectl set-ntp true
```

## Step 1: Partition the Drive

Create a GPT partition table with three partitions AFTER checking your drive name with `lsblk -l` :

```bash
sgdisk --zap-all \
    -n1:0:+1G  -t1:EF00 -c1:"EFI system" \
    -n2:0:+40G -t2:8304 -c2:"Linux root (x86-64)" \
    -n3:0:0    -t3:8302 -c3:"Linux home" \
    /dev/sda
```

**Partition Layout:**

* `/dev/sda1` — 1GB EFI System Partition
* `/dev/sda2` — 40GB Root partition
* `/dev/sda3` — Remaining space for Home partition


## Step 2: Format and Mount Partitions

Create filesystems and mount them in the correct order:

```bash
# Format partitions
d=/dev/sda
mkfs.fat -F32 -n EFI ${d}1 && \
mkfs.ext4 -L root ${d}2 && \
mkfs.ext4 -L home ${d}3

# Mount root partition first
mount /dev/disk/by-label/root /mnt

# Create and mount EFI directory
mkdir -p /mnt/boot
mount /dev/disk/by-label/EFI /mnt/boot

# Create and mount home directory
mkdir /mnt/home
mount /dev/disk/by-label/home /mnt/home
```

## Step 3: Install Base System

```bash
# Update mirrorlist with fastest mirrors
reflector --country Norway --country Germany --age 12 --protocol https --sort age --save /etc/pacman.d/mirrorlist

# Install minimal base system
pacstrap /mnt base linux-lts linux-firmware amd-ucode nano sudo zsh
```

## Step 4: System Configuration

### 4.1 Enter the New System

```bash
arch-chroot /mnt
```

### 4.2 Set Timezone

```bash
# Set timezone to Oslo (Norway)
ln -sf /usr/share/zoneinfo/Europe/Oslo /etc/localtime
# Set hardware clock
hwclock --systohc
```

### 4.3 Configure Locale

```bash
# Edit locale generation file
nano /etc/locale.gen
Uncomment: en_US.UTF-8 UTF-8
Uncomment: nb_NO.UTF-8 UTF-8 # Optional if you need second language

# Generate locales
locale-gen

# Set system locale
cat << EOF > /etc/locale.conf
LANG=en_US.UTF-8
LC_TIME=nb_NO.UTF-8 # Optional if you want to set the date & time to a specific LANG default
EOF

# Set console keymap. This even U.S keyboards has to set!
echo "KEYMAP=no-latin1" > /etc/vconsole.conf 

# Persist configure X11 keymap for non U.S keyboards
cat << EOF > /etc/X11/xorg.conf.d/00-keyboard.conf
Section "InputClass"
    Identifier "system-keyboard"
    MatchIsKeyboard "on"
    Option "XkbLayout" "no"
    Option "XkbModel" "pc105"
EndSection
EOF
```

### 4.4 Set Hostname and Hosts

```bash
# Set hostname
echo "BluePoint" > /etc/hostname

# Configure hosts file
cat << EOF > /etc/hosts
127.0.0.1   localhost
::1         localhost
127.0.1.1   BluePoint.localdomain BluePoint
EOF
```

### 4.5 Create User Account

```bash
# Set root password
passwd

# Create user with necessary groups
useradd -m -G wheel,audio,video,input lars
passwd lars

# Set zsh as default shell for user and root
chsh -s /usr/bin/zsh lars
chsh -s /usr/bin/zsh

# Enable sudo for wheel group
EDITOR=nano visudo
# Uncomment: %wheel ALL=(ALL:ALL) ALL
```

### 4.6 Install Desktop Environment and Drivers

```bash
# Update package database
pacman -Syu

# Install essential packages
pacman -S --needed \
  networkmanager \
  pipewire pipewire-alsa pipewire-pulse pipewire-jack wireplumber \
  plasma-meta dolphin konsole \
  sddm linux-headers \
  mesa vulkan-radeon \
  zram-generator pacman-contrib \
  git wget \
  base-devel
```

### 4.7 Configure Initramfs

```bash
# Edit mkinitcpio configuration
nano /etc/mkinitcpio.conf
# MODULES=(amdgpu radeon)
# Remove 'base' and 'udev' from HOOKS=() and add 'systemd'
(!IMPORTANT! - Otherwise your system won't boot!)

# Regenerate initramfs
mkinitcpio -P
```

### 4.8 Install and Configure Bootloader

```bash
# Install systemd-boot
bootctl install

# Configure bootloader
cat << EOF > /boot/loader/loader.conf
default arch
timeout 10
console-mode max
editor no
EOF

# Create boot entry
cat << EOF > /boot/loader/entries/arch.conf
title   Arch Linux
linux   /vmlinuz-linux-lts
initrd  /amd-ucode.img
initrd  /initramfs-linux-lts.img
options rw amdgpu.si_support=1 amdgpu.cik_support=1 radeon.si_support=0 radeon.cik_support=0
EOF
```

### 4.9 Configure Zram (Compressed Swap)

```bash
# Configure zram
cat << EOF > /etc/systemd/zram-generator.conf
[zram0]
zram-size = min(ram / 2, 4096)
compression-algorithm = zstd
EOF
```

### 4.10 Enable Essential Services

```bash
# Enable network, display manager, and timesyncd
systemctl enable NetworkManager sddm systemd-timesyncd systemd-boot-update.service
```

## Step 5: Complete Installation

```bash
# Exit chroot environment
exit

# Unmount all partitions
umount -R /mnt

# Reboot into new system
reboot
```


---
