# Quick Arch Linux Tutorial (KDE + NVIDIA + Wayland w/ Automounting)

##**NOTE: THERE ARE NO `fstab` edits! A populated `fstab` will break this guide!!!** 


Your drives will automount entirely by using GUIDs and `systemd-gpt-auto-generator` which I find much more convenient, stable and secure. - It's worth familiarizing yourself with how this works before following my guide.

---

### INTRODUCTION - How GPT Auto-Mounting Works

Modern systemd uses `systemd-gpt-auto-generator` to automatically discover and mount partitions based on their GPT partition type GUIDs, eliminating the need for manual `/etc/fstab` entries. This system is useful for centralizing file system configuration in the partition table and making configuration in `/etc/fstab` or on the kernel command line unnecessary.

### The GUIDs

When you use the partition type codes in this guide:
- `EF00` (EFI System Partition)
- `8304` (Linux x86-64 root)  

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
- `linux-zen` default kernel which is a kernel optimized for desktop use, `linux-lts` for a fallback and debug kernel
- zsh default shell
- systemd-boot
- zswap
- EXT4 for `/` and `/home`
- AMD CPU + NVIDIA GPU (4070 RTX, check your own card for which driver to use. For me it's `nvidia-open-dkms`)
- Plymouth which is for loading screen, it's optional but I prefer it personally.

modeset is set by default, and according to the wiki setting fbdev manually is now unnecessary so I will not set those. PLEASE check the wiki before install for anything. POST-INSTALL GUIDE IS SUPER OPINIONATED, FOLLOW BY OWN VOLITION.

*Protip:* This tutorial uses Norwegian keymaps and locale/timezone settings. Simply replace those with your own (e.g. keymap, `LANG`, `TZ`).
If you use an English lang keyboard you can ignore all of it, but it's worth knowing if you are new and use a different keyboard like say `de-latin1` for German keyboards.

**NOTE:** **This tutorial assumes you have a NVME SSD,** which are named `/dev/nvme0n1`. If you don't have that, it's something else. If you don't know, check with `lsblk -l` to see your scheme. It could be `sda` or something else. If it is something else replace all instances of `nvme0n1` and remove the `p` from  `${d}p1` in the formatting.

*Sidenote:* Unless you like the name, replace my hostname (basically the name of your rig) of `BigBlue` with yours, same as my user name `lars` if your name ain't Lars. Though if it is, cool. Hi! I thought about doing placeholders but I feel those are more distracting usually, I prefer to see how something would actually work in a guide, maybe you do as well?


## Prerequisites

- A bootable Arch Linux USB (written with `dd` or similar)
- Internet connection
- UEFI system


---


### TUTORIAL PROPER - Non-HTML Version

**GPT Auto-Mount + KDE Plasma (Wayland) + NVIDIA**

> **Prerequisites:** This guide assumes you have an AMD processor with NVIDIA graphics. For Intel CPUs, replace `amd-ucode` with `intel-ucode` throughout the installation.
For AMDGPU or Intel GPU you should look either up at the Arch Wiki and replace the corresponding packages with those. I'd rather not clutter up the guide with a bunch of different setups, especially if I've never used those. It just confuses new users, like placeholders.



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
sgdisk --zap-all /dev/nvme0n1

sgdisk -n1:0:+1G -t1:EF00 -c1:"EFI system" /dev/nvme0n1
sgdisk -n2:0:0 -t2:8304 -c2:"Linux root" /dev/nvme0n1
```

**Partition Layout:**

* `/dev/nvme0n1p1` — 1GB EFI System Partition
* `/dev/nvme0n1p2` — Root partition

I won't do a swap partition, don't need hibernation personally. If you do you will need one. I don't do home partitions because I don't distrohop because Arch is a complete distribution of Linux, and I don't do any encryption because the one time I did I ended up being locked out of my computer because of stupidity. If you need encryption for security sensitive systems then look elsewhere.

## Step 2: Format and Mount Partitions

Create filesystems and mount them in the correct order:

```bash
# Format partitions
d=/dev/nvme0n1
mkfs.fat -F32 -n EFI ${d}p1
mkfs.ext4 -L root ${d}p2

# Mount root partition first
mount /dev/disk/by-label/root /mnt

# Create and mount EFI directory
mkdir -p /mnt/boot
mount /dev/disk/by-label/EFI /mnt/boot
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
cat > /etc/vconsole.conf <<'EOF'
KEYMAP=no-latin1
FONT=ter-118n
EOF

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
```

## Install packages. All are essential to the well function of a KDE Plasma desktop except for kitty and pkgstats.

## pkgstats 
pkstats is a super harmless way to help out the Arch developers that work hard and mostly for free to make our wonderful distro.
It basically just advertises a list of your core and extra packages that you use to them  so they can know what packages to 
prioritize and other things. Most people reveal more about their system voluntarily in ways that help nobody so this is not a problem.
If you are very paranoid you can leave this one out and not enable the timer of it at the end. I included it however because
I use it and I also try to help out wherever I can personally. I send most information to KDE in crash reports as well. I would like to
promote such an attitude to anyone else in FOSS which has become more & more ungrateful and entitled over the years.

## kitty 
kitty is a terminal that I think is the best sort of default terminal on Linux. It's easy to use, fast enough and hassle free.
It allows you to zoom in by pressing CTRL + SHIFT and + and zoom out by CTRL + SHIFT and -
I install konsole as well for backup, but it's mostly not necessary because kitty is very good. 

## Install
```bash
pacman -S --needed \
  networkmanager reflector \
  pipewire pipewire-alsa pipewire-pulse pipewire-jack wireplumber \
  plasma-meta dolphin konsole kitty xdg-desktop-portal-gtk kio-admin \
  sddm sddm-kcm linux-zen-headers linux-lts-headers kdegraphics-thumbnailers ffmpegthumbs \
  nvidia-open-dkms nvidia-utils terminus-font pkgstats \
  pacman-contrib git wget plymouth plymouth-kcm \
  base-devel
  
```
### 4.7 Configure NVIDIA in Initramfs

```bash
# Edit mkinitcpio configuration
nano /etc/mkinitcpio.conf
# MODULES=(nvidia nvidia_modeset nvidia_uvm nvidia_drm)
# Remove 'kms' from HOOKS=()
# Remove 'udev' from HOOKS=() and add 'systemd' E.g. : HOOKS=(base systemd ... )
# Add 'plymouth' at the end of HOOKS=() if you want plymouth. If not don't add it and remove quiet splash from zen kernel options.
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

# Create boot entry for main kernel. noatime is a harmless optimization for filesystems, zswap pool percentage optimizes zswap.
cat << EOF > /boot/loader/entries/arch.conf
title   Arch Linux
linux   /vmlinuz-linux-zen
initrd  /amd-ucode.img
initrd  /initramfs-linux-zen.img
options rw quiet splash rootflags=noatime zswap.max_pool_percent=25
EOF

# Create boot entry for LTS kernel backup / I will disable plymouth for this kernel to use for troubleshooting booting.
cat << EOF > /boot/loader/entries/arch-lts.conf
title   Arch Linux (LTS)
linux   /vmlinuz-linux-lts
initrd  /amd-ucode.img
initrd  /initramfs-linux-lts.img
options rw rootflags=noatime zswap.max_pool_percent=25 plymouth.enable=0 disablehooks=plymouth
EOF

# Update boot entries
bootctl update
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
systemctl enable NetworkManager sddm systemd-timesyncd systemd-boot-update.service fstrim.timer reflector.timer pkgstats.timer
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
# Essential build tools, you already installed these during install but just to be sure
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
yay -S --needed --noconfirm fastfetch
fastfetch # then run fastfetch
```
Basic packages:
```bash
# mainly desktop apps and codecs, stuff good to have. dont install cuda if you dont have NVIDIA obviously.
yay -S --needed --noconfirm firefox thunderbird-esr-bin informant \
gst-libav gst-plugins-bad gst-plugins-base gst-plugins-good gst-plugins-ugly \
noto-fonts-cjk noto-fonts-extra ttf-dejavu cuda systemd-timer-notify \
python-pip kdeconnect journalctl-desktop-notification

```

---

## 3 · System Optimisation

### 3.1 Pacman candy
Edit `/etc/pacman.conf`:
```ini
# Color adds color (duh), ILoveCandy is a fun optional easter egg type setting that adds animations to when you update pacman. No spoilers.
Color                      # uncomment
ILoveCandy                 # add under Color
```

### 3.2 Shell and terminal bliss
```bash
yay -S --needed --noconfirm zsh oh-my-zsh-git zsh-autosuggestions zsh-syntax-highlighting

#  set Kitty as default terminal in KDE:
# Settings → Default Applications → Terminal Emulator → kitty
```

## Enable syntax highlighting in nano
```bash
# create your nano config if it does not exist
mkdir -p ~/.config/nano

# package with enhanced rules
yay -S --needed --noconfirm nano-syntax-highlighting

# enable all bundled syntaxes
printf 'include "/usr/share/nano/*.nanorc"\ninclude "/usr/share/nano/extra/*.nanorc"\n' >> ~/.config/nano/nanorc
echo 'include "/usr/share/nano-syntax-highlighting/*.nanorc"' >> ~/.config/nano/nanorc
```
## Turn off that incessant beeping in kitty without doing it system wide.
```bash
# You can turn this off system wide in KDE settings, but that is a bit overkill.
sudo nano ~/.config/kitty/kitty.conf

# Add these lines:
enable_audio_bell no
visual_bell_duration 0
window_alert_on_bell no
bell_on_tab none

# reload the config
CTRL + SHIFT + F5

# Test that the Geneva violation is gone. Printing '\a' sends the BEL character.
printf '%b' '\a'
```

## Show asterisks when typing your sudo password
Use `visudo` and add the `pwfeedback` default. This is the safe way to edit sudoers.
```bash
# open a drop-in with visudo
sudo visudo -f /etc/sudoers.d/pwfeedback

# add exactly this line, then save and exit
Defaults pwfeedback

# test by forcing a fresh prompt
sudo -k
sudo true
```
## Persist configure X11 keymap for non U.S keyboards

```bash
cat << EOF > /etc/X11/xorg.conf.d/00-keyboard.conf
Section "InputClass"
    Identifier "system-keyboard"
    MatchIsKeyboard "on"
    Option "XkbLayout" "no"
    Option "XkbModel" "pc105"
EndSection
EOF
```

## Copy and configure .zshrc

```bash
cp /usr/share/oh-my-zsh/zshrc ~/.zshrc
```

```bash
nano ~/.zshrc
# Add to plugins
(git zsh-syntax-highlighting zsh-autosuggestions)
# Add to end of file
PROMPT='%F{white}%B[%F{#1793d1}Arch%F{white}Lars%F{white}] %F{cyan}%~ %f%(!.#.$) '
```

# Reload
```bash
source ~/.zshrc
```

### 3.4 Raise vm.max_map_count (gaming)
```bash
# This is what Valve uses for the SteamDeck.
sudo nano /etc/sysctl.d/80-gaming.conf
vm.max_map_count = 2147483642
```

### 3.6 Install and configure cpupower
```bash
# Package install
yay -S --noconfirm --needed cpupower 
```

```bash
sudo nano /etc/default/cpupower
```

```ini
# Set:
governor='performance'
```

### Enable and verify
```bash
sudo systemctl enable --now cpupower.service
cpupower frequency-info
```

### 3.7 Apply all sysctl changes
```bash
sudo sysctl --system
```

---

## 4 · Essential security and AUR quality of life

### 4.1 Firewall
```bash
sudo pacman -S --needed --noconfirm firewalld python-pyqt6
sudo systemctl enable --now firewalld
sudo firewall-cmd --permanent --zone=public --add-service=kdeconnect
sudo firewall-cmd --reload
```

### 4.2 Enable multilib
Uncomment in `/etc/pacman.conf`, then refresh:
```ini
[multilib]
Include = /etc/pacman.d/mirrorlist
```
```bash
yay
```

### 4.3 32-bit runtime
```bash
yay -S --needed --noconfirm lib32-nvidia-utils lib32-pipewire
```

---

## 5 · Maintenance hooks (fire and forget)
```bash
yay -S --needed --noconfirm \
  pacdiff-pacman-hook-git \
  reflector-pacman-hook-git \
  paccache-hook \
  systemd-boot-pacman-hook
```

---

## 6 · Daily driver apps

- **Text Editor**: `yay -S --needed --noconfirm kate`  
- **Graphics and Media**: `yay -S --needed --noconfirm gimp mpv audacity reaper gwenview spotify`  
- **Gaming Stack**: `yay -S --needed --noconfirm steam dxvk-bin lutris protonup-qt-bin`  
- **System Tools**: `yay -S --needed --noconfirm partitionmanager ksystemlog systemdgenie nohang-git mkinitcpio-firmware ark`

## Enable Nohang:
```bash
# This is an OOM killer. If your system fills up it's swap and RAM this will work to terminate processes doing so before your system freeze up.
sudo systemctl enable --now nohang-desktop.service
```

## Set Journalctl limit:
```bash
# SUPER important, journal on desktop use fills up very quickly which takes space and can slow down boot times after a while.
/etc/systemd/journald.conf.d/00-journal-size.conf
```
```ini
[Journal]
SystemMaxUse=50M
```

## MPV hardware acceleration:
```bash
# You have to do this if you want GPU acceleration.
mkdir -p ~/.config/mpv
echo "hwdec=auto" > ~/.config/mpv/mpv.conf
```

Configure Proton GE as the default in Steam after installing Proton GE from ProtonUp-Qt:

0. Open up ProtonUp-Qt and install the latest version of Proton GE
1. Launch Steam and open **Settings → Steam Play**.  
2. Tick both **Enable Steam Play** boxes.  
3. In the dropdown, choose **Proton GE**.  
4. Click OK and restart Steam.

ProtonGE is a good default for a lot of games, works just as well as regular Proton for most games and for other games include 
propietary codecs and such that Valve cannot package themselves, this helps with video files and music with odd formats.

---

## 7 · Reboot
```bash
# Reboot into new system
reboot
```

---

