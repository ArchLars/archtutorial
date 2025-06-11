# Quick Arch Linux Tutorial (KDE + NVIDIA + Wayland w/ Automounting)

This is a beautifully formatted HTML guide that can be opened in your browser to installing Arch Linux.

##**NOTE: THERE ARE NO `fstab` edits! A populated `fstab` will break this guide!!!** 


Your drives will automount entirely by using GUIDs and `systemd-gpt-auto-generator` which I find much more convenient, stable and secure. - It's worth familiarizing yourself with how this works before following my guide.

---

### INTRODUCTION - How GPT Auto-Mounting Works

Modern systemd uses `systemd-gpt-auto-generator` to automatically discover and mount partitions based on their GPT partition type GUIDs, eliminating the need for manual `/etc/fstab` entries. This system is useful for centralizing file system configuration in the partition table and making configuration in `/etc/fstab` or on the kernel command line unnecessary.

### The GUIDs

When you use the partition type codes in this guide:
- `EF00` (EFI System Partition)
- `8304` (Linux x86-64 root)  
- `8302` (Linux /home)

systemd automatically creates mount units based on these partition type GUIDs. Each hex code corresponds to a specific 128-bit GUID that tells the system exactly what that partition is for. The system recognizes this GUID and then mounts accordingly like a modern system should, similar to other universal types of partitioning on other systems.
For extra storage you can use the generic Linux filesystem code:
- `8300`

systemd won't auto-mount these, giving you control over when and where they mount which again to me is ideal, if need be you can mount them on boot with a systemd service. This allows you to avoid `fstab` issues forever. No more random issues where it's suddenly overwritten for some reason or anything else, mounting is seperate and automated.

---

Important Limitations & Caveats

Same Disk Only: Auto-mounting only works for partitions on the same physical disk as your root partition.
Boot Loader Dependency: The boot loader must set the LoaderDevicePartUUID EFI variable for root partition detection to work. systemd-boot (used in this guide) supports this.
First Partition Rule: systemd mounts the first partition of each type it finds. If you have multiple 8302 partitions on the same disk, **then only the first one gets auto-mounted.**
No Multi-Disk Support: This won't work on systems where the root filesystem is distributed across multiple disks (like BTRFS RAID).

Pros:

Portability: Your disk image can boot on different hardware without fstab changes
Self-Describing: The partition table contains all mounting information
Container-Friendly: Tools like systemd-nspawn can automatically set up filesystems from GPT images
Reduced Maintenance: No broken boots from typos in `/etc/fstab` or random updates doing weird stuff messing with it.


## Specs:

- systemd-automount for GPT partitions 
- KDE Plasma on Wayland
- NVME SSD
- zsh default shell
- systemd-boot
- EXT4 for `/` and `/home`
- AMD CPU + NVIDIA GPU (4070 RTX, check your own card for which driver to use. For me it's `nvidia-open`)

modeset is set by default, and according to the wiki setting fbdev manually is now unnecessary so I will not set those. PLEASE check the wiki before install for anything. POST-INSTALL GUIDE IS WIP, FOLLOW BY OWN VOLITION.

> **Note:** This tutorial uses Norwegian keymaps and locale/timezone settings. Simply replace those with your own (e.g. keymap, `LANG`, `TZ`).  
> It assumes **NVME SSD,** which uses `/dev/nvme0n1` partitions. If you don't have that, it's something else. If you don't know, check with `lsblk -l` to see your scheme. It could be `sda` or something else. If it is something else replace all `nvme0n1` with that.
>  Replace my hostname of `bigboy` with yours, same as my user name `lars`.


## Prerequisites

- A bootable Arch Linux USB (written with `dd` or similar)
- Internet connection
- UEFI system


---


### TUTORIAL PROPER - Non-HTML Version

**GPT Auto-Mount + KDE Plasma (Wayland) + NVIDIA**

> **Prerequisites:** This guide assumes you have an AMD processor with NVIDIA graphics. For Intel CPUs, replace `amd-ucode` with `intel-ucode` throughout the installation.



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

Update mirrorlist for optimal download speeds and install the base system, obv replace Norway and Germany:

```bash
# Update mirrorlist with fastest mirrors
reflector --country Norway --country Germany --age 12 --protocol https --sort age --save /etc/pacman.d/mirrorlist

# Install minimal base system
pacstrap /mnt base linux linux-firmware amd-ucode nano sudo zsh
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

# Set console keymap. 
echo "KEYMAP=no-latin1" > /etc/vconsole.conf 

# Persist console keymap and configure X11 keymap. Optional unless u have a specific type of non-US keyboard
localectl set-x11-keymap no pc105 latin1
localectl set-keymap --no-convert no-latin1
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
# Update package database
pacman -Syu

# Install essential packages
pacman -S --needed \
  networkmanager \
  pipewire pipewire-alsa pipewire-pulse wireplumber \
  plasma-meta dolphin konsole \
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
# Remove 'kms' from HOOKS=() - Remove 'base' and 'udev' from HOOKS=() and add 'systemd' (IMPORTANT - Otherwise your system won't boot!)

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
> * Zram swap active
> * Auto-mounted root and home partitions
> * NVIDIA drivers loaded and functional
> * Network connectivity via NetworkManager

---
