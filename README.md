# Arch Linux Tutorial

A HTML guide that can be opened in your browser to installing Arch Linux with:

- systemd-automount for GPT partitions (no  `fstab` edits!)
- KDE Plasma on Wayland
- NVME SSD
- zsh default shell
- systemd-boot
- EXT4 for `/` and `/home`
- AMD CPU + NVIDIA GPU

> **Note:** This tutorial uses Norwegian keymaps and locale/timezone settings. Simply replace those with your own (e.g. keymap, `LANG`, `TZ`).  
> It assumes **NVME SSD,** which uses `/dev/nvme0n1` partitions. If you don't have that, it's something else. If you don't know, check with `lsblk -l` to see your scheme. It could be `sda` or something else. If it is something else replace all `nvme0n1` with that.
>  Replace my hostname of `bigboy` with yours, same as my user name `lars`.

---

## Prerequisites

- A bootable Arch Linux USB (written with `dd` or similar)
- Internet connection
- UEFI system


# Quick Version (if you dont want to look at my html ☹️)

**GPT Auto-Mount + KDE Plasma (Wayland) + NVIDIA**

> **Prerequisites:** This guide assumes you have an AMD processor with NVIDIA graphics. For Intel CPUs, replace `amd-ucode` with `intel-ucode` throughout the installation.

---

## Step 0: Boot from ISO

Set up Norwegian keyboard layout and verify UEFI boot:

```bash
# Set Norwegian keyboard layout
loadkeys no-latin1

# Verify UEFI firmware
ls /sys/firmware/efi/efivars && echo "UEFI firmware detected"

# Sync system clock
timedatectl set-ntp true
```

## Step 1: Partition the NVMe Drive

Create a GPT partition table with three partitions:

```bash
sgdisk --zap-all \
    -n1:0:+1G  -t1:EF00 -c1:"EFI system" \
    -n2:0:+40G -t2:8304 -c2:"Linux root (x86-64)" \
    -n3:0:0    -t3:8302 -c3:"Linux home" \
    /dev/nvme0n1
```

**Partition Layout:**

* `/dev/nvme0n1p1` — 1GB EFI System Partition
* `/dev/nvme0n1p2` — 40GB Root partition
* `/dev/nvme0n1p3` — Remaining space for Home partition

## Step 2: Format and Mount Partitions

Create filesystems and mount them in the correct order:

```bash
# Format partitions
d=/dev/nvme0n1; mkfs.fat -F32 -n EFI ${d}p1 && mkfs.ext4 -L root ${d}p2 && mkfs.ext4 -L home ${d}p3

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

Update mirrorlist for optimal download speeds and install the base system:

```bash
# Update mirrorlist with fastest mirrors
reflector --country Norway --country Germany --age 12 --protocol https --sort rate --save /etc/pacman.d/mirrorlist

# Install minimal base system
pacstrap -K /mnt base linux linux-firmware amd-ucode nano sudo zsh
```

**Packages installed:**

* `base`
* `linux`
* `linux-firmware`
* `amd-ucode`
* `nano`

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
# Uncomment: en_US.UTF-8 UTF-8

# Generate locales
locale-gen

# Set system locale
echo "LANG=en_US.UTF-8" > /etc/locale.conf

# Set console keymap
echo "KEYMAP=no-latin1" > /etc/vconsole.conf
```

### 4.4 Set Hostname and Hosts

```bash
# Set hostname
echo "bigboy" > /etc/hostname

# Configure hosts file
cat << EOF > /etc/hosts
127.0.0.1   localhost
::1         localhost
127.0.1.1   bigboy.localdomain bigboy
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
# Update mirrorlist again
reflector --country Norway --country Germany --age 12 --protocol https --sort rate --save /etc/pacman.d/mirrorlist

# Update package database
pacman -Syu

# Install essential packages
pacman -S --needed \
  networkmanager \
  pipewire pipewire-alsa pipewire-pulse wireplumber \
  plasma-meta dolphin konsole plasma-wayland-session \
  sddm linux-headers \
  nvidia-open nvidia-utils \
  zram-generator pacman-contrib \
  git curl wget \
  base-devel
```

### 4.7 Configure NVIDIA in Initramfs

```bash
# Edit mkinitcpio configuration
nano /etc/mkinitcpio.conf
# MODULES=(nvidia nvidia_modeset nvidia_uvm nvidia_drm)
# Remove 'kms' from HOOKS=()

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
timeout 3
console-mode max
editor no
EOF

# Create boot entry
cat << EOF > /boot/loader/entries/arch.conf
title   Arch Linux
linux   /vmlinuz-linux
initrd  /amd-ucode.img
initrd  /initramfs-linux.img
options rw quiet loglevel=3
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

# Enable zram service
systemctl enable systemd-zram-setup@zram0.service
```

### 4.10 Enable Essential Services

```bash
# Enable network, display manager, and timesyncd
systemctl enable NetworkManager sddm systemd-timesyncd
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

## Post-Installation Verification

After rebooting, verify everything is working correctly:

```bash
# Check mounted filesystems
findmnt -o TARGET,SOURCE,FSTYPE | grep -E '/ |/home|/boot'

# Verify NVIDIA drivers are loaded
lsmod | grep nvidia

# Check zram is active
swapon -s
```

> **Expected Results:**
>
> * KDE Plasma (Wayland) login screen should appear
> * Zram swap active for better performance
> * Auto-mounted root and home partitions
> * NVIDIA drivers loaded and functional
> * Network connectivity via NetworkManager

---
