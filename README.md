# Quick Arch Linux Tutorial (KDE Plasma + NVIDIA + Wayland w/ Automounting)

This is a quick Arch installation guide for noobs that just want a working system to game on that's straight forward with a DE that is most like Windows
and usually the one most people want to use because of that, at least for their first DE. I've used every DE and WM that is both trendy and some obscure,
I started with KDE Plasma and Arch Linux. I always come back to both eventually. It's fun to try out new things, but KDE Plasma is OP at the moment I am writing 
this. It's fully featured, they finally have a good process in eliminating bugs which plagued the DE before, and it's very easy to customize. Most DEs and WMs have
some caveat, KDE Plasma does not. That is why I use it.


## NOTE (ACTUALLY READ THIS): 

So, I like to use something called `systemd-gpt-auto-generator`. I acknowledge that this is a super opinionated decision for a noob tutorial, and I debated whether or not to use it in this tutorial, but I feel it's so cromulent and underrated that I decided to make a big decision to teach you how to use it as well. If you follow this guide correctly and use it you'll see why it's very convenient.
It is not usually done on Linux and it is kind of new(?), at least relative to `fstab`, however it is a modern way of mounting partitions that are also used by other operating systems you may already be familiar with. 

---

# INTRODUCTION - How GPT Auto-Mounting Works

Modern systemd uses `systemd-gpt-auto-generator` to automatically discover and mount partitions based on specific 128-bit **UUIDs,** eliminating the need for manual `/etc/fstab` entries. This system is useful for centralizing file system configuration in the partition table and making configuration in `/etc/fstab` or on the kernel command line unnecessary. This is similar to the OS you probably switched away from and are more familiar with; Windows. - Windows identifies volumes by what they call "GUIDs" (Volume{GUID} paths). Now for your sake all you need to know is that a GUID is functionally the same thing as the specific 128-bit UUIDs that we will use on Linux, but instead of mounting to `boot` or `root` they mount their "GUIDs" to set drives defined by a letter, so `C:` drives and `D:` drives. That is why some letters are reserved for largely depreciated functions, as mounting on Windows is identified by a set identifier just like your system's UUIDs will do.

Your drive partitions like `boot` and `root` will not be mounted by `fstab`, instead they will automount entirely by using UUIDs by using `systemd-gpt-auto-generator`. This is preferable in my opinion to `fstab` which feels like a hack and places too much control of system reliance upon a single text based config. This is anecdotal, but I have heard of what happens when some package or update randomly decides to destroy your `fstab` and it is **NOT** fun to troubleshoot if it happens. It's often difficult to know what is going wrong and many hours will be wasted until you realize your fstab for whatever reason is empty or has some typos.

Now this is still unconventional which is part of the fun of using this as it justifies the manual install, but since it is unique it's worth familiarizing yourself with how this works before following my guide. I will add a small tutorial on how you would go about adding a new SSD later on with this, it's a *tiny* bit different but still very easy to do. -- **PLEASE NOTE:** that there are extra steps to subvolumes if you choose to use this with **BTRFS,** since subvolumes like snapshots usually require `fstab`. I might write a small tutorial on what you need to do with BTRFS for this type of system if I ever decide to use that filesystem, but essentially instead of `fstab` you just use systemd service for each instead which is also what you will do for new drives. 

## The UUIDs

When you use the partition type codes in this guide:
- `EF00` (EFI System Partition)
- `8304` (Linux x86-64 root)  

systemd automatically creates mount units based on these partition type UUIDs. Each hex code corresponds to a specific 128-bit UUID that tells the system exactly what that partition is for. The system recognizes these GUIDs and then mounts accordingly, just like a modern system should. This approach is similar to how partitioning works on other systems.
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


### TUTORIAL PROPER

**GPT Auto-Mount + KDE Plasma (Wayland) + NVIDIA**

> **Prerequisites:** This guide assumes you have an AMD processor with NVIDIA graphics. For Intel CPUs, replace `amd-ucode` with `intel-ucode` throughout the installation.
For AMDGPU or Intel GPU you should look either up at the Arch Wiki and replace the corresponding packages with those. I'd rather not clutter up the guide with a bunch of different setups, especially if I've never used those. It just confuses new users, like placeholders.



## Step 0: Boot from ISO

Set up Norwegian keyboard layout and verify UEFI boot:

```bash
# Set Norwegian keyboard layout, each line in these code blocks is a separate line in the terminal FYI
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

## Step 3: Base System Install

First update mirrorlist for optimal download speeds, obv replace Norway and Germany:

```bash
# Update mirrorlist before install so you install with fastest mirrors
reflector \                                      
      --country 'Norway,Sweden,Denmark,Germany,Netherlands' \
      --age 12 \
      --protocol https \
      --sort rate \
      --latest 10 \
      --save /etc/pacman.d/mirrorlist
```

and then install the base of Arch Linux! :

```bash
#
# IMPORTANT Note: When it asks you for what font to use type the number for ttf-liberation.
# The reason why is that it provides free, open-source font files that are metrically compatible with
# proprietary Microsoft fonts. This ensures that documents will render with consistent layout and
# text appearance across different systems and applications.
#
pacstrap /mnt base linux-zen linux-lts linux-firmware amd-ucode nano sudo zsh
```

## Step 4: System Configuration

### 4.1 Enter the New System

```bash
# Howeever before you can say you've installed arch you need to configure the system
# This is how you chroot into your newly installed system:
#
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

## 4.5.5 Package Choice

### Info:
I have taken the liberty to make some decisions for a few packages you will install, some of them are technically "optional" but
all of them are in my opinion essential to the well functioning of a KDE Plasma desktop except for kitty and pkgstats. 

Here's why I included those:


### pkgstats 
pkstats is a super harmless way to help out the Arch developers that work hard and mostly for free to make our wonderful distro.
It basically just advertises a list of your core and extra packages that you use to them  so they can know what packages to 
prioritize and other things. 

### kitty 
kitty is a terminal that I think is the best sort of default terminal on Linux. It's easy to use, GPU accelerated, fast enough and hassle free.
It allows you to zoom in by pressing `CTRL + SHIFT and +` and zoom out by `CTRL + SHIFT and -` It doesn't look terrible like some terminals do.
konsole is included as a backup.

---

## **NOT INCLUDED IN THE STEP BUT YOU MAY WANT TO INCLUDE:**

### wireless-regdb
If you use wireless then an **essential package** is also `wireless-regdb`. It installs regulatory.db, a machine-readable table of Wi-Fi rules per country  that allows you to connect properly. If regulatory.db is missing or cannot be read, Linux falls back to the “world” regdomain 00. That profile is **intentionally conservative,** which means fewer channels and more restrictions. For example, world 00 marks many 5 GHz channels as passive-scan only and limits parts of 2.4 GHz (12–13 passive, 14 effectively off).

### audiocd-kio
This adds the audiocd:/ KIO worker so Dolphin and other KDE apps can read and rip audio CDs. Not needed on non-KDE Plasma systems, but KDE has their own thing with this for some reason. If you are on a laptop with a CD player then you are going to want this.

### libdvdread, libdvdnav, and libdvdcss
This is the same as above but for DVD playback. This is needed on any DE.

### libbluray, libaacs
Same for Blu-Rays. After you have installed the system and configured an AUR helper you may also wish to install **libbdplus** from the AUR if you want for BD+ playback. From there you will have to set it up with KEYS which is shown on the Arch Wiki about Blu-Ray.

---

# 4.6 Install the System

```bash
# Update package database
pacman -Syu
```

```bash
pacman -S --needed \
  networkmanager reflector \
  pipewire pipewire-alsa pipewire-pulse pipewire-jack wireplumber \
  plasma-meta dolphin konsole kitty xdg-desktop-portal-gtk kio-admin \
  sddm sddm-kcm linux-zen-headers linux-lts-headers kdegraphics-thumbnailers ffmpegthumbs \
  nvidia-open-dkms nvidia-utils terminus-font pkgstats hunspell hunspell-en_us  \
  pacman-contrib git wget ttf-dejavu \
  base-devel
  
```



### 4.7 Configure NVIDIA in Initramfs

```bash
# Edit mkinitcpio configuration
nano /etc/mkinitcpio.conf
# MODULES=(nvidia nvidia_modeset nvidia_uvm nvidia_drm) if you use nvidia
# Remove 'kms' from HOOKS=() also if you use nvidia

# IMPORTANT: Remove 'udev' from HOOKS=() and add 'systemd' E.g. : HOOKS=(base systemd ... )
# Add microcode to HOOKS=()
# Replace keymap and consolefont with sd-vconsole in HOOKS=()
# If you do not remove udev and add systemd your system will not boot.
# Regenerate initramfs
mkinitcpio -P
```

### 4.8 Install and Configure Bootloader

```bash
# Install systemd-boot
bootctl install --variables=yes

# Configure bootloader
cat << EOF > /boot/loader/loader.conf
default arch.conf
timeout 10
console-mode auto
editor no
EOF

# Confirm boot entries
bootctl list

# Create boot entry for main kernel. noatime is a standard optimization for EXT4 partitions.
cat << EOF > /boot/loader/entries/arch.conf
title   Arch Linux
linux   /vmlinuz-linux-zen
initrd  /initramfs-linux-zen.img
options rw rootflags=noatime
EOF

# Create boot entry for LTS kernel backup
cat << EOF > /boot/loader/entries/arch-lts.conf
title   Arch Linux (LTS)
linux   /vmlinuz-linux-lts
initrd  /initramfs-linux-lts.img
options rw rootflags=noatime
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
nano /etc/systemd/system/swapfile.swap
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
systemctl enable swapfile.swap
```

Optimizations for swap use:

```bash
tee /etc/sysctl.d/99-zswap.conf >/dev/null << 'EOF'
vm.swappiness = 100
vm.page-cluster = 0
vm.watermark_boost_factor = 0
vm.watermark_scale_factor = 125
EOF
sysctl --system
```

### 4.10 Enable Essential Services

```bash
# Enable network, display manager, and timesyncd
# Include systemd-boot-update.service if you aren't using the hook
systemctl enable NetworkManager sddm systemd-timesyncd fstrim.timer reflector.timer pkgstats.timer
```

## Step 5: Complete Installation

```bash
# Exit chroot environment
exit

# Unmount all partitions
umount -R /mnt

# Reboot into new system
reboot

# If you see a very generic type of screen, dont worry. That happens to me on every install.
#
# To fix, log in to the system and launch "System Settings" from the Start Menu (Application launcher)
# 
# Navigate to Colors & Themes -> Login Screen (SDDM) -> then select "Breeze" and hit Apply
# It will then have applied it and on next reboot and others after it will persist
# 
# This is also the way to fix if the taskbar (panel) appears on the wrong monitor, simply go to Global Theme
# Press Breeze or Breeze-Dark, select BOTH checkboxes and hit apply. Wait and then it will correctly apply
# This will also persist on reboots as well. Two odd bugs I've ran into but not something that persists afterwards.
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

# OPTIONAL: Post-Install Tutorial

## 1 · Update Base System

```bash
# Bring everything to the latest version
sudo pacman -Syu
```

---

## 2 · Install AUR Helper (yay)

### 2.1 Prerequisites

### Launch Kitty

* Open the application launcher located on the bottom left of the panel.
* Navigate to the "System" submenu, then locate & launch the program entitled: `kitty`

### Set Kitty as default terminal in KDE:

* Go to System Settings
* Then to Default Applications
* locate "Terminal Emulator"
* Set it to `kitty`


### Essential build tools, you already installed these during install but just to be sure
```bash
sudo pacman -S --needed base-devel git  # when you run pacman -S with the --needed flag it will skip
                                        # installing any package that is already on the system.
```

### 2.2 Build and install yay
```bash
cd /tmp
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si
cd ~ && rm -rf /tmp/yay

yay --version  # quick test
yay -S --needed --noconfirm fastfetch   # Important: the --noconfirm flag makes it auto confirms- 
fastfetch # then run fastfetch          # -the endless questions if you want to install something or not.
```

### 2.5 Shell and terminal bliss
```bash
yay -S --needed --noconfirm oh-my-zsh-git zsh-autosuggestions zsh-syntax-highlighting
```

### Copy .zshrc default template config

```bash
cp /usr/share/oh-my-zsh/zshrc ~/.zshrc
```

### Configure ~/.zshrc

```bash
# Tip: You can press F12 to insert the letter ~ into the terminal
# This avoids having to spider-man hand ALT + whatever to write it
#
nano ~/.zshrc

# Add this to () in plugins:
#
(git zsh-syntax-highlighting zsh-autosuggestions)

# You are also going to want to set your name in PROMPT, otherwise it will just be `~`
# The "PROMPT" below will look like this: [ArchLars], with Arch in Arch blue and Lars in white, same with brackets.
# The ~ will be in cyan, which is your working directory.
# This is a fine early profile name, you can make it nicer later.
#
# Replace "Lars" with your own name and add this to the very bottom of ~/.zshrc:
#
PROMPT='%F{white}%B[%F{#1793d1}Arch%F{white}Lars%F{white}] %F{cyan}%~ %f%(!.#.$) '
```

### Reload & Guide
```bash
# Then reload zshrc like so:
source ~/.zshrc

# 
# - Right arrow: accept a suggestion to autocomplete a command you've run before. 
#
# - Up arrow: recall a previous command that starts the same way. 
#   For example, type 'sudo', then press Up, and it fills in the rest. 
#   This is useful when installing packages, like you will in this tutorial.
#   Every time you type 'yay', you can press Up to autofill your usual flags, 
#   then replace the package name with something else.
#
# - Syntax highlighting makes commands easier to read, and helps you spot obvious mistakes.
#
```

## 3 · System Optimisation

### 3.1 Pacman candy
Edit `/etc/pacman.conf`:
```ini
# Color adds color (duh), ILoveCandy is a fun optional easter egg type setting that adds animations to when you update pacman. No spoilers.
Color                      # uncomment
ILoveCandy                 # add under Color
```

### Enable syntax highlighting in nano
```bash
# create your nano config if it does not exist
mkdir -p ~/.config/nano

# package with enhanced rules
yay -S --needed --noconfirm nano-syntax-highlighting

# enable all bundled syntaxes
printf 'include "/usr/share/nano/*.nanorc"\ninclude "/usr/share/nano/extra/*.nanorc"\n' >> ~/.config/nano/nanorc
echo 'include "/usr/share/nano-syntax-highlighting/*.nanorc"' >> ~/.config/nano/nanorc
```
### Turn off that incessant beeping in kitty without doing it system wide.
```bash
# You can turn this off system wide in KDE settings, but that is a bit overkill.
nano ~/.config/kitty/kitty.conf

# Add these lines:
enable_audio_bell no
visual_bell_duration 0
window_alert_on_bell no
bell_on_tab none

# reload the config
CTRL + SHIFT + F5

# Test that the violation of the Geneva Convention is gone. Printing '\a' should send the BEL character which triggers it if not.
printf '%b' '\a'
```

### Show asterisks when typing your sudo password
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
### Persist configure X11 keymap for non U.S keyboards

```bash
# even if you dont use x11 it's good to set this just in case
# ignore if you dont use a weird keyboard (non US one = weird)
cat << EOF > /etc/X11/xorg.conf.d/00-keyboard.conf
Section "InputClass"
    Identifier "system-keyboard"
    MatchIsKeyboard "on"
    Option "XkbLayout" "no"
    Option "XkbModel" "pc105"
EndSection
EOF
```
Basic packages:
```bash
# mainly good stuff to have.
yay -S --needed --noconfirm firefox informant \
gst-libav gst-plugins-bad gst-plugins-base gst-plugins-good gst-plugins-ugly \
noto-fonts-cjk noto-fonts-extra systemd-timer-notify rebuild-detector \
python-pip kdeconnect journalctl-desktop-notification

```

### 3.4 Raise vm.max_map_count (gaming)
```bash
# This is what Valve uses for the SteamDeck.
sudo nano /etc/sysctl.d/80-gaming.conf
vm.max_map_count = 2147483642
```

### 3.6 Install and configure cpupower
```bash
# cpupower sets cpu scheduler and is configurable
yay -S --noconfirm --needed cpupower 
```

```bash
# open in editor
sudo nano /etc/default/cpupower
```

```ini
# then set:
governor='performance'
```

```bash
# finally enable & verify
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
# Tip/Fun Fact: You can update your system by just writing 'yay'. Try it
#
yay
```

### 4.2.5 Steam
```bash
# DL Steam
yay -S --needed --noconfirm steam

# Run Steam in terminal to install it:
steam
```

### 4.3 Ensure you have the 32-bit runtimes
```bash
# Some extra lib32 stuff for other games
yay -S --needed --noconfirm \
  lib32-sdl2-compat \
  lib32-sdl2_mixer lib32-sdl2_ttf
```

---

## 5 · Maintenance hooks
```bash
# these hooks are great for system maintenance
#
# pacdiff shows you if any .pacnew is on your system needed to merge
# reflector will run reflector any time mirrorlist updates
# paccache-hook is the GOAT. it cleans your cache after using pacman.
# systemd-boot-pacman-hook needed to update systemd boot for you if you don't got the timer
#
yay -S --needed --noconfirm \
  pacdiff-pacman-hook-git \
  reflector-pacman-hook-git \
  paccache-hook \
  systemd-boot-pacman-hook
```

### Install & Enable Nohang:
```bash
# This is an OOM killer. It's VITAL.
yay -S --needed --noconfirm nohang-git 

# If your system fills up it's swap and RAM then this will terminate offending processes before your system freeze up.
sudo systemctl enable --now nohang-desktop.service
```

### Set Journalctl limit:
```bash
# SUPER important, journal on desktop use fills up very quickly which takes space
# a large one can slow down boot times after a while.
/etc/systemd/journald.conf.d/00-journal-size.conf
```
```ini
[Journal]
SystemMaxUse=50M
```

### MPV hardware acceleration:
```bash
# install mpv (audio / video)
yay -S --needed --noconfirm mpv

# You have to do this if you want GPU acceleration for your wholesome entertainment
mkdir -p ~/.config/mpv
echo "hwdec=auto" > ~/.config/mpv/mpv.conf
```

### ProtonUp-Qt:
```bash
# install protonup qt (ProtonGE)
yay -S --needed --noconfirm protonup-qt
```

## Configure Proton GE as the default in Steam after installing Proton GE from ProtonUp-Qt:

0. Open up ProtonUp-Qt and install the latest version of Proton GE
1. Launch Steam and open **Settings → Compatibility**.  
2. In the dropdown, choose **Proton GE**.  
3. Click OK and restart Steam.

ProtonGE is a good default for a lot of games, works just as well as regular Proton for most games and for other games include 
propietary codecs and such that Valve cannot package themselves, this helps with video files and music with odd formats.

---

## Mini-Tutorial on How to a new Drive/SSD to GPT-Auto Setups:


- Name of drive will be `data`, 
- Replace ALL instances of `data` in this guide if you don't want that.

#### 0) Identify the new disk (double check before you write to it)
```bash
lsblk -e7 -o NAME,SIZE,TYPE,MOUNTPOINT,MODEL,SERIAL
DEV=/dev/nvme1n1    # <-- set this to your new disk
```
#### 1) Create a GPT partition and give it a PARTLABEL
```bash
#    WARNING: the zap step is destructive. Save data on disk first.
sudo sgdisk --zap-all "$DEV"
sudo sgdisk -n1:0:0 -t1:8300 -c1:"data" "$DEV"   # one Linux partition named "data"
```
#### 2) Make a filesystem (example: ext4)
```bash
sudo mkfs.ext4 -L data "${DEV}p1"   # remove p like in install if ur disk is 'sda' and not nvme
```
#### 3) Verify the persistent symlink created by udev, then wait if needed
```bash
ls -l /dev/disk/by-partlabel/ | grep ' data$' || true
sudo udevadm settle
```
#### 4) Create the mount point
```bash
sudo mkdir -p /mnt/data
```
#### 5) Create a native systemd mount unit
```bash
sudo tee /etc/systemd/system/mnt-data.mount >/dev/null <<'EOF'
[Unit]
Description=Data SSD via PARTLABEL

[Mount]
What=/dev/disk/by-partlabel/data
Where=/mnt/data
Type=ext4
Options=noatime

[Install]
WantedBy=multi-user.target
EOF
```
#### create an automount for on-demand mounting
```bash
sudo tee /etc/systemd/system/mnt-data.automount >/dev/null <<'EOF'
[Unit]
Description=Auto-mount /mnt/data

[Automount]
Where=/mnt/data

[Install]
WantedBy=multi-user.target
EOF
```
#### 6) Enable it
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now mnt-data.automount
```
#### 7) Test
```bash
systemctl status mnt-data.automount
df -h /mnt/data
touch /mnt/data/it-works
```


## Final Reboot

#### Reboot again into new system and you can finally sit back, relax, and use arch btw

```bash
reboot
```

---


# OPTIONAL: How to fix those annoying 'missing firmware' warnings in mkinitcpio

#### 0) Find the module names that warn

```bash
# Everytime you run mkinicpio -P it warns you about a bunch of "missing firmware" 
# but none of these are important, they are ancient. You won't need them.
#
# To see them, run this, and you'll see multiple lines ala:
# "Possibly missing firmware for module: qla2xxx"
# You can find a list of them here : https://wiki.archlinux.org/title/Mkinitcpio#Possibly_missing_firmware_for_module_XXXX
#
sudo mkinitcpio -P
```

#### OPTION A) Install all the firmware from the AUR (which will take up space)
```bash
yay -S --needed --noconfirm mkinitcpio-firmware
```

#### OPTION B) Copy a script that silences them by making dummy firmware
```bash
# The Arch Wiki recommends instead writing dummy files manually for them
#
# I wrote a script that creates harmless dummy firmware files to shut it up
# It automatically captures the ones on a run with this script
# then you run mkinitcpio again to build with the dummies to never see them ever again
#
# First open nano like so, which will create a new file:
#
sudo nano /usr/local/sbin/mkinitcpio-silence-missing-fw
```

#### 1.5) Then paste the script with CTRL + SHIFT + V
```bash
# ----- /usr/local/sbin/mkinitcpio-silence-missing-fw -----
##!/usr/bin/env bash
set -euo pipefail

FWROOT="/usr/lib/firmware"
LIST="/etc/mkinitcpio.local-firmware-ignore"
MARKER="### mkinitcpio-dummy-fw created, remove if you add matching hardware ###"
LOG="/var/tmp/mkinitcpio-warnings.$$.log"
HOST="$(uname -n 2>/dev/null || echo unknown-host)"

usage() {
  cat <<'USAGE'
Usage:
  mkinitcpio-silence-missing-fw         Run mkinitcpio, show output live, detect modules that warn, create dummy firmware only for those modules not in use.
  mkinitcpio-silence-missing-fw --undo  Remove only the dummy firmware files previously created by this tool.
Notes:
  - Future new warnings remain visible, this only touches modules found in this run.
  - Modules currently loaded are skipped.
USAGE
}

undo() {
  if [[ ! -f "$LIST" ]]; then
    echo "Nothing to undo, $LIST not found."
    exit 0
  fi
  while IFS= read -r f; do
    [[ -z "$f" || ! -e "$f" ]] && continue
    if grep -q "$MARKER" "$f" 2>/dev/null; then
      rm -v -- "$f"
    else
      echo "Skip non-dummy file: $f"
    fi
  done < "$LIST"
  : > "$LIST"
  echo "Removed recorded dummy firmware files."
}

# Try modinfo with common dash/underscore variants
get_fw_list() {
  local m="$1"
  # print unique non-empty firmware paths to stdout
  for name in "$m" "${m//-/_}" "${m//_/-}"; do
    if mapfile -t _x < <(modinfo -F firmware "$name" 2>/dev/null); then
      printf '%s\n' "${_x[@]}" | sed '/^$/d' | sort -u
      return 0
    fi
  done
  return 1
}

create_from_warnings() {
  echo "Running mkinitcpio -P (output will be shown and logged to $LOG)..."
  set +e
  mkinitcpio -P 2>&1 | tee "$LOG"
  status=${PIPESTATUS[0]}
  set -e
  if [[ $status -ne 0 ]]; then
    echo "mkinitcpio exited with status $status, continuing (warnings were captured)."
  fi

  # Extract module names from warning lines
  declare -A seen=()
  modules=()
  while IFS= read -r line; do
    case "$line" in
      *"Possibly missing firmware for module"*)
        m="${line##*: }"
        m="${m#"${m%%[![:space:]]*}"}"; m="${m%"${m##*[![:space:]]}"}"
        [[ "$m" == \"*\" && "$m" == *\" ]] && m="${m:1:${#m}-2}"
        [[ "$m" == \'*\' && "$m" == *\' ]] && m="${m:1:${#m}-2}"
        if [[ -n "$m" && -z "${seen[$m]+x}" ]]; then
          seen[$m]=1
          modules+=("$m")
        fi
      ;;
    esac
  done < "$LOG"

  if [[ ${#modules[@]} -eq 0 ]]; then
    echo "No 'Possibly missing firmware' warnings found in this run."
    echo "Log: $LOG"
    return 0
  fi

  echo "Modules with warnings: ${modules[*]}"
  touch "$LIST"

  for module in "${modules[@]}"; do
    # Skip if the module is actually in use
    if lsmod | grep -qw "$module"; then
      echo "SKIP $module (module is loaded, will not create dummies for hardware in use)"
      continue
    fi

    mapfile -t fws < <(get_fw_list "$module" || true)
    if [[ ${#fws[@]} -eq 0 ]]; then
      echo "No firmware filenames reported by $module, nothing to create."
      continue
    fi

    for fw in "${fws[@]}"; do
      target="$FWROOT/$fw"
      mkdir -p "$(dirname "$target")"
      if [[ -e "$target" ]]; then
        echo "Exists: $target, leaving as is."
        continue
      fi
      {
        echo "$MARKER"
        echo "Dummy firmware for module $module on $HOST."
        echo "If you later add matching hardware, delete this file and install the proper firmware package."
      } > "$target"
      chmod 0644 "$target"
      echo "$target" >> "$LIST"
      echo "Created dummy: $target"
    done
  done

  echo "Recorded created files in $LIST"
  echo "Done. You can now rebuild: sudo mkinitcpio -P"
  echo "Full mkinitcpio output is in: $LOG"
}

case "${1:-}" in
  --undo) undo ;;
  -h|--help) usage ;;
  *) create_from_warnings ;;
esac
```
#### 2) Make it executable
```bash
sudo chmod +x /usr/local/sbin/mkinitcpio-silence-missing-fw
```
#### 3) Run & Undo

```bash
## 1) FIRST run this to capture the ancient modules warned on your machine
sudo /usr/local/sbin/mkinitcpio-silence-missing-fw

## 2) THEN run this to confirm, and you are done
sudo mkinitcpio -P

---

## 1) If you ever want to undo
sudo /usr/local/sbin/mkinitcpio-silence-missing-fw --undo

## 2) And if so, run to confirm.
sudo mkinitcpio -P
```
---

