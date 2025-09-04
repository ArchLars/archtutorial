# Quick Arch Linux Tutorial (KDE + NVIDIA + Wayland w/ Automounting)

This is a beautifully formatted HTML guide that can be opened in your browser for installing an automounting Arch Linux. Mostly written for myself but maybe someone else could get some use out of it too!

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

systemd automatically creates mount units based on these partition type GUIDs. Each hex code corresponds to a specific 128-bit GUID that tells the system exactly what that partition is for. The system recognizes these GUIDs and then mounts accordingly, just like a modern system should. This approach is similar to how partitioning works on other systems.
For extra storage you can use the generic Linux filesystem code:
- `8300`

systemd won't auto-mount these, giving you control over when and where they mount which again to me is ideal, if need be you can mount them on boot with a systemd service. This allows you to avoid `fstab` issues forever. No more random issues where it's suddenly overwritten for some reason or anything else, mounting is seperate and automated.

---

# - CONS: -

Same Disk Only: Auto-mounting only works for partitions on the same physical disk as your root partition.

Boot Loader Dependency: The boot loader must set the `LoaderDevicePartUUID` EFI variable for root partition detection to work. systemd-boot (used in this guide) supports this. Check if the bootloader you wish to use does.

First Partition Rule: systemd mounts the first partition of each type it finds. If you have multiple 8302 partitions on the same disk, **then only the first one gets auto-mounted.**

No Multi-Disk Support: This won't work on systems where the root filesystem is distributed across multiple disks (like BTRFS RAID).

# - PROS: -

Portability: Your disk image can boot on different hardware without `fstab` changes

Self-Describing: The partition table contains all mounting information

Container-Friendly: Tools like systemd-nspawn can automatically set up filesystems from GPT images

Reduced Maintenance: No broken boots from typos in `/etc/fstab` or random updates doing weird stuff messing with it.

---

## What we will be using/setting:

- systemd-automount for GPT partitions 
- KDE Plasma on Wayland
- NVME SSD
- zsh default shell
- systemd-boot
- zswap
- EXT4 for `/` and `/home`
- AMD CPU + NVIDIA GPU (4070 RTX, check your own card for which driver to use. For me it's `nvidia-open`)

modeset is set by default, and according to the wiki setting fbdev manually is now unnecessary so I will not set those. PLEASE check the wiki before install for anything. POST-INSTALL GUIDE IS WIP, FOLLOW BY OWN VOLITION.

*Protip:* This tutorial uses Norwegian keymaps and locale/timezone settings. Simply replace those with your own (e.g. keymap, `LANG`, `TZ`).  

**NOTE:** **This tutorial assumes you have a NVME SSD,** which are named `/dev/nvme0n1`. If you don't have that, it's something else. If you don't know, check with `lsblk -l` to see your scheme. It could be `sda` or something else. If it is something else replace all instances of `nvme0n1` and remove the `p` from  `${d}p1` in the formatting.

*Sidenote:* Unless you like the name, replace my hostname (basically the name of your rig) of `bigboy` with yours, same as my user name `lars` if your name ain't Lars. Though if it is, cool. Hi!


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

Create a GPT partition table with three partitions AFTER checking your drive name with `lsblk -l` :

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
d=/dev/nvme0n1
mkfs.fat -F32 -n EFI ${d}p1 && \
mkfs.ext4 -L root ${d}p2 && \
mkfs.ext4 -L home ${d}p3

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
reflector \                                      
      --country 'Norway,Sweden,Denmark,Germany,Netherlands' \
      --age 12 \
      --protocol https \
      --sort rate \
      --latest 10 \
      --save /etc/pacman.d/mirrorlist

# Install minimal base system
pacstrap /mnt base linux-zen linux-lts linux-firmware amd-ucode nano sudo zsh
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

```

### 4.4 Set Hostname and Hosts

```bash
# Set hostname
echo "BigBlue" > /etc/hostname

# Configure hosts file
cat << EOF > /etc/hosts
127.0.0.1   localhost
::1         localhost
127.0.1.1   BigBlue.localdomain BigBlue
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
  networkmanager reflector \
  pipewire pipewire-alsa pipewire-pulse pipewire-jack wireplumber \
  plasma-meta dolphin konsole xdg-desktop-portal-gtk kio-admin \
  sddm linux-zen-headers linux-lts-headers kdegraphics-thumbnailers ffmpegthumbs \
  nvidia-open-dkms nvidia-utils \
  pacman-contrib python-pip \
  git wget noto-fonts-cjk noto-fonts-extra ttf-dejavu \
  base-devel
```

### 4.7 Configure NVIDIA in Initramfs

```bash
# Edit mkinitcpio configuration
nano /etc/mkinitcpio.conf
# MODULES=(nvidia nvidia_modeset nvidia_uvm nvidia_drm)
# Remove 'kms' from HOOKS=()
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
console-mode auto
editor no
EOF

# Confirm boot entries
bootctl list

# Create boot entry for main kernel
cat << EOF > /boot/loader/entries/arch.conf
title   Arch Linux
linux   /vmlinuz-linux-zen
initrd  /amd-ucode.img
initrd  /initramfs-linux-zen.img
options rw zswap.max_pool_percent=25
EOF

# Create boot entry for LTS kernel backup
cat << EOF > /boot/loader/entries/arch-lts.conf
title   Arch Linux (LTS Kernel)
linux   /vmlinuz-linux-lts
initrd  /amd-ucode.img
initrd  /initramfs-linux-lts.img
options rw zswap.max_pool_percent=25
EOF
```

### 4.9 Configure Zswap

```bash
# Create a swap file (zswap needs a backing swap device)
dd if=/dev/zero of=/swapfile bs=1G count=16 status=progress
chmod 600 /swapfile
mkswap /swapfile
```
edit:
```bash
sudo nano /etc/systemd/system/swapfile.swap
```
and add:
```ini
[Unit]
Description=Swap file

[Swap]
What=/swapfile
Priority=100

[Install]
WantedBy=swap.target
```
then:
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now swapfile.swap
sudo swapon --show
```

Optimizations for desktop use:

```bash
sudo tee /etc/sysctl.d/99-zswap.conf >/dev/null << 'EOF'
vm.swappiness = 100
vm.page-cluster = 0
vm.watermark_boost_factor = 0
vm.watermark_scale_factor = 125
EOF
sudo sysctl --system
```

### 4.10 Enable Essential Services

```bash
# Enable network, display manager, and timesyncd
systemctl enable NetworkManager sddm systemd-timesyncd systemd-boot-update.service fstrim.timer reflector.timer
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

# Check zswap is active
dmesg | grep -i zswap
# Expect: "zswap: loaded using pool zstd/zsmalloc"
cat /sys/module/zswap/parameters/enabled
cat /sys/module/zswap/parameters/max_pool_percent
cat /sys/module/zswap/parameters/shrinker_enabled
swapon --show
```

> **Expected Results:**
>
> * KDE Plasma (Wayland) login screen should appear
> * Zswap swap active
> * Auto-mounted root and home partitions
> * NVIDIA drivers loaded and functional
> * Network connectivity via NetworkManager

---

# Post-Install Tutorial

## 1 · Update Base System

```bash
# Bring everything to the latest version
sudo pacman -Syu
```

---

## 2 · Install AUR Helper (yay)

### 2.1 Prerequisites
```bash
# Essential build tools
sudo pacman -S --needed base-devel git
```

### 2.2 Build and install yay
```bash
cd /tmp
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si
cd ~ && rm -rf /tmp/yay

yay --version  # quick test
yay -S --needed --noconfirm fastfetch  # then run fastfetch

yay -S --needed --noconfirm firefox thunderbird-esr-bin
```

---

## 3 · System Optimisation

### 3.1 Pacman speed and candy
Edit `/etc/pacman.conf`:
```ini
Color                      # uncomment
ILoveCandy                 # add under Color
```

### 3.2 Shell and terminal bliss
```bash
yay -S kitty zsh oh-my-zsh-git zsh-autocomplete zsh-syntax-highlighting

# Launch Kitty and set it as default terminal in KDE:
# Settings → Default Applications → Terminal Emulator
kitty

# Copy and configure .zshrc
cp /usr/share/oh-my-zsh/zshrc ~/.zshrc
cat >> ~/.zshrc << 'EOF'
source /usr/share/zsh/plugins/zsh-autocomplete/zsh-autocomplete.plugin.zsh
source /usr/share/zsh/plugins/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh
EOF

# Reload
source ~/.zshrc
```

### 3.3 Kernel timer tweaks (systemd-boot)
```bash
# Edit loader entry
sudo nano /boot/loader/entries/arch.conf

# Modify options line, append the following to the end:
# options   ... tsc=reliable clocksource=tsc

# Re-install loader
sudo bootctl update

# Reboot later to activate
```

### 3.4 Raise vm.max_map_count (gaming)
```bash
sudo nano /etc/sysctl.d/80-gaming.conf
vm.max_map_count = 2147483642
```

### 3.6 Install and configure cpupower
```bash
# Package install
yay -S cpupower  # or: sudo pacman -S cpupower

sudo nano /etc/default/cpupower
# Set:
# governor='performance'
# min_freq="1.5GHz"  # optional
# max_freq="3.5GHz"  # optional

# Enable and verify
sudo systemctl enable --now cpupower.service
cpupower frequency-info
```

### 3.7 Apply all sysctl changes
```bash
sudo sysctl --system
```

---

## 4 · Essential security and AUR quality of life

### 4.1 Firewall (UFW)
```bash
sudo pacman -S ufw
sudo ufw default deny
sudo ufw allow from 192.168.0.0/24
sudo ufw limit ssh

# KDE Connect
sudo ufw allow 1714:1764/tcp
sudo ufw allow 1714:1764/udp

sudo systemctl enable --now ufw
sudo ufw enable
```

### 4.2 Enable multilib
Uncomment in `/etc/pacman.conf`, then refresh:
```ini
[multilib]
Include = /etc/pacman.d/mirrorlist
```
```bash
yay -Syu
```

### 4.3 32-bit runtime
```bash
yay -S lib32-nvidia-utils lib32-pipewire
```

### 4.4 Desktop integration
```bash
yay -S xdg-desktop-portal kdeconnect kdegraphics-thumbnailers ffmpegthumbs
```

---

## 5 · Maintenance hooks (fire and forget)
```bash
yay -S \
  pacdiff-pacman-hook-git \
  reflector-pacman-hook-git \
  paccache-hook \
  systemd-boot-pacman-hook
```

---

## 6 · Daily driver apps

- **Text Editor**: `yay -S --needed --noconfirm kate`  
- **Graphics and Media**: `yay -S --needed --noconfirm gimp mpv`  
- **Gaming Stack**: `yay -S --needed --noconfirm steam dxvk-bin protonup-qt-bin`  
- **System Tools**: `yay -S --needed --noconfirm partitionmanager ksystemlog systemdgenie`

MPV hardware acceleration:
```bash
mkdir -p ~/.config/mpv
echo "hwdec=auto" > ~/.config/mpv/mpv.conf
```

Configure Proton GE as the default in Steam:

1. Launch Steam and open **Settings → Steam Play**.  
2. Tick both **Enable Steam Play** boxes.  
3. In the dropdown, choose **Proton GE**.  
4. Click OK and restart Steam.

---

## 7 · Reboot
```bash
# Reboot into new system
reboot
```

---

## 8 · Verification snippets
After rebooting, verify everything is working correctly:

```bash
# Pacman tweaks
grep -E "ParallelDownloads|ILoveCandy" /etc/pacman.conf

# Kernel cmdline (after reboot)
cat /proc/cmdline | grep -E "tsc=reliable|clocksource=tsc"

# Sysctl
sysctl vm.max_map_count vm.swappiness

# cpupower
enable=$(systemctl is-enabled cpupower) && echo "cpupower service $enable"

# Firewall
sudo ufw status verbose

# Hook sanity
sudo pacman -T  # should output nothing
```

---

