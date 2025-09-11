# Complete Arch Linux Tutorial (KDE Plasma + Wayland + LUKS + SecureBoot + TPM2 w/ Automounting Partitions)

This is a Arch installation guide for novices **WITH LUKS encryption, TPM2 and SecureBoot** (More complex) who want a working secure system to game on that's straight forward, encrypted, with a DE that is most like Windows
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
For GRUB to set the `LoaderDevicePartUUID` UEFI variable load the bli module in grub.cfg:
```ini
if [ "$grub_platform" = "efi" ]; then
  insmod bli
fi
```

First Partition Rule: systemd mounts the first partition of each type it finds. If you have multiple 8302 partitions on the same disk, **then only the first one gets auto-mounted.**

No Multi-Disk Support: This won't work on systems where the root filesystem is distributed across multiple disks (like BTRFS RAID).

# - PROS: -

Portability: Your disk image can boot on different hardware without `fstab` changes

Self-Describing: The partition table contains all mounting information

Container-Friendly: Tools like systemd-nspawn can automatically set up filesystems from GPT images

Reduced Maintenance: No broken boots from typos in `/etc/fstab` or random updates doing weird stuff messing with it.

## What I will mainly be using/setting:

- systemd-automount for GPT partitions 
- KDE Plasma on Wayland
- NVME SSD
- `linux-zen` default kernel which is a kernel optimized for desktop use.
- `linux-lts` for a fallback and debug kernel
- zsh default shell
- systemd-boot with UKIs
- zswap with a 16 GiB swap file
- Encrypted with LUKS and SecureBoot + TPM2 enabled
- EXT4 for `/`
- AMD CPU + NVIDIA GPU w/ `nvidia-open-dkms` 
**NOTE:** This tutorial assumes you have a Turing (NV160/TUXXX) and newer	card for current driver. Check your card first.

I included some stuff for AMDGPUs too, but my system is NVIDIA so I may have missed some things.

NVIDIA modeset is set by default, and according to the wiki setting fbdev manually is now unnecessary so I will not set those. PLEASE check the wiki before install for anything. **POST-INSTALL GUIDE IS SUPER OPINIONATED, FOLLOW BY OWN VOLITION.**

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

**GPT Auto-Mount + KDE Plasma (Wayland) + LUKS + SecureBoot + TPM2**


## Step 0: Boot from ISO with SecureBoot off

* Turn SecureBoot off, Arch’s installer does not boot with SecureBoot. We will set it up only after the install is done at 4.8a.
* Set up your keyboard layout if you're not on an US keyboard, and verify UEFI boot:

```bash
# Set your keyboard layout, you can skip this is u use a normal keyboard (US)
# each line in these code blocks is a separate line in the terminal FYI

# # List all keymaps (scrollable):
localectl list-keymaps | less

# or filter by country code by writing:
localectl list-keymaps | grep -i -E '^no($|[-0-9])|jis'    # Norway example
                                                           # "no" is our ISO-639 code. Find yours by googling first
                                                           # Then replace 'no' with your country code                 

# For Norway it's "no-latin1". On Arch it's usually "*-latin1" and not just the country code.
# Test out your keyboard after this, if it is wrong try another on the list.
#
# To write "-" on US keyboard which you will need to do to be able to write this command,
# it's usually the first key left of backspace. For Norwegian/Nordic keyboard that's: \.
loadkeys no-latin1

# Verify UEFI firmware, write it all out including && and echo.
# It's just going to be a bunch of random variables that's confusing, however...
#
# If it says the quote at the end there then you are good.
ls /sys/firmware/efi/efivars && echo "UEFI firmware detected"

# Sync system clock
timedatectl set-ntp true
```

## Step 1: Partition the NVMe Drive

Create a GPT partition table with three partitions AFTER checking your drive name with `lsblk -l` :

```bash
lsblk -l  # To confirm, replace if it says anything else

sgdisk --zap-all /dev/nvme0n1

sgdisk -n1:0:+1G -t1:EF00 -c1:"EFI system" /dev/nvme0n1
sgdisk -n2:0:0 -t2:8304 -c2:"Linux root" /dev/nvme0n1
```

**Partition Layout:**

* `/dev/nvme0n1p1` — 1GB EFI System Partition
* `/dev/nvme0n1p2` — Root partition

I won't do a swap partition, don't need hibernation personally. If you do you will need one. I don't do home partitions because I don't distrohop because Arch is a complete distribution of Linux.

## Step 1.5: Set up LUKS Encryption (NEW ENCRYPTION SECTION)
*Important:* This maintains compatibility with `systemd-gpt-auto-generator`. 
The partition type must remain `8304` (Linux x86-64 root) - do **NOT** use a LUKS-specific type.

When systemd-gpt-auto-generator detects a partition with type 8304 that contains LUKS, it automatically:

* Creates /dev/gpt-auto-root-luks symlink pointing to the encrypted partition
* Attempts to unlock it using systemd-cryptsetup
* Creates /dev/gpt-auto-root symlink pointing to the unlocked volume
* Mounts the unlocked volume as root

#### Create the LUKS container on the root partition:
```bash
# 1) Set up LUKS2 encryption on the root partition
# You'll be prompted to enter YES (in capitals) and create a passphrase

# 4K encryption sectors are set at format time and cannot be changed later.
cryptsetup luksFormat --type luks2 --sector-size 4096 /dev/nvme0n1p2

# Open it and persist the useful runtime flags in the LUKS2 header
# (these will apply automatically at every boot, including gpt-auto unlock)
# This creates /dev/mapper/root which we'll format in the next step
cryptsetup open \
  --allow-discards \
  --perf-no_read_workqueue \
  --perf-no_write_workqueue \
  --perf-submit_from_crypt_cpus \
  --persistent \
  /dev/nvme0n1p2 root

# 2) Verify the LUKS header was created successfully
cryptsetup luksDump /dev/nvme0n1p2

# Optional 'Flags' sanity check specifically
cryptsetup luksDump /dev/nvme0n1p2 | grep -i '^Flags'

# Show keys check
dmsetup table /dev/mapper/root --showkeys
```

## Step 2: Format and Mount Partitions

Create filesystems and mount them in the correct order.

We will be mounting EFI at /efi instead of /boot so the kernel
can also be encrypted after we set up our UKIs with SecureBoot 
and TPM2 is enabled.

In the original tutorial we mounted it on /boot which is fine 
for a non-encrypted setup but for a LUKS + SecureBoot setup it 
leaves the kernel vulnerable to be replaced by a malicious actor 
when outside encrypted space:

```bash
# Format partitions
d=/dev/nvme0n1
mkfs.fat -F32 -n EFI ${d}p1
mkfs.ext4 -L root /dev/mapper/root

# Mount the decrypted root partition
mount /dev/mapper/root /mnt

# Create and mount EFI directory at /efi (NOT /boot)
mkdir -p /mnt/efi
mount /dev/disk/by-label/EFI /mnt/efi
```

## Step 3: Base System Install

First update mirrorlist for optimal download speeds, obv replace Norway and Germany.
A good rule of thumb here is doing your country + closest neighbours and then a few larger neighbours after that.
So for me it's Norway,Sweden,Denmark then Germany,Netherlands:

```bash
# Update mirrorlist before install so you install with fastest mirrors
#
# PROTIP: "\" is a pipe, it basically is a fancy way to add a space to a bash command.
# So essentially just write each line until there isnt a "\" and it will run it all as one command.
# This is good for keeping large commands digestible during install.
#
reflector \ # this is a line, press enter                                      
      --country 'Norway,Sweden,Denmark,Germany,Netherlands' \  # and it goes to the 2nd line, do same as first
      --age 12 \ # same here & etc under  
      --protocol https \ 
      --sort rate \
      --latest 10 \
      --save /etc/pacman.d/mirrorlist  # then when pressing enter here w/o "\" it will run all the lines
```

and then **Install the base of Arch Linux!** :

```bash
# IMPORTANT Note: When it asks you for what font to use type the number for ttf-liberation.
#
# The reason why is that it provides free, open-source font files that are metrically compatible with
# proprietary Microsoft fonts. This ensures that documents will render with consistent layout and
# text appearance across different systems and applications.

# For AMD CPUs:
pacstrap /mnt base linux-zen linux-lts linux-firmware amd-ucode nano sudo zsh

# For Intel CPUs:
pacstrap /mnt base linux-zen linux-lts linux-firmware intel-ucode nano sudo zsh
```

## Step 4: System Configuration

### 4.1 Enter the New System

```bash
# However before you can say you've installed arch you need to configure the system
# This is how you chroot into your newly installed system:
#
arch-chroot /mnt
```

### 4.2 Set Timezone

```bash
# list all timezones
timedatectl list-timezones

# Filter by continent
# '_' is used for space, eg. America/New_York
timedatectl list-timezones | grep '^America/'

# Or filter by closest likely big city.
timedatectl list-timezones | grep '/Oslo$'

# Set timezone to your own continent and city
timedatectl set-timezone Europe/Oslo

# Set hardware clock
hwclock --systohc
```

### 4.3 Configure Locale

```bash
# Now we are going to configure our system language.
# I am going to have my system be in English,
# but my time and date will be set as it is in Norway.
# So an English system with a DD/MM/YYYY and 00:00 "military clock".
#
nano /etc/locale.gen

# Go down the list and uncomment both:
Uncomment: en_US.UTF-8 UTF-8 # English
Uncomment: nb_NO.UTF-8 UTF-8 # Bokmål Norwegian (replace with your own or leave out)

# Then generate locales
locale-gen

# Set system locale
nano /etc/locale.conf

# add
LANG=en_US.UTF-8    # LANG for system language
LC_TIME=nb_NO.UTF-8 # LC_TIME for date & time to my specific LANG default


# Set console keymap & font
nano /etc/vconsole.conf

# add
KEYMAP=no-latin1 # Skip this if US keyboard
FONT=ter-118n  # But add this.
               # This is a console font which makes it larger,
               # and more easily readable on boot

```

### 4.4 Set Hostname and Hosts

```bash
# Set hostname, echo lets you do it quickly w/o using nano
# good for one line stuff
#
echo "BigBlue" > /etc/hostname

# Configure hosts file
nano /etc/hosts

## add to /etc/hosts:
127.0.0.1 localhost
::1 localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
127.0.1.1 BigBlue
```

### 4.5 Create User Account

```bash
# Set root password
passwd

# Create user with necessary groups
useradd -m -G wheel lars
passwd lars

# Set zsh as default shell for user and root
chsh -s /usr/bin/zsh lars
chsh -s /usr/bin/zsh

# Enable sudo for wheel group
EDITOR=nano visudo
# Uncomment: %wheel ALL=(ALL:ALL) ALL
```



### 4.6 Configure Initramfs

```bash
# Edit mkinitcpio configuration
nano /etc/mkinitcpio.conf
# Example for MODULES:
MODULES=(nvidia nvidia_modeset nvidia_uvm nvidia_drm) if you use nvidia
#
# Example for HOOKS (with LUKS encryption)
HOOKS=(base systemd autodetect microcode modconf keyboard sd-vconsole block sd-encrypt filesystems fsck)

# Key changes:
# - MUST use 'systemd' instead of 'udev' 
# - Add 'sd-encrypt' hook after 'block' to handle LUKS decryption
# - Use 'sd-vconsole' instead of 'keymap' and 'consolefont'
# - Remove 'kms' from HOOKS=() also if you use nvidia, AMDGPU can ignore this however
# - Ensure microcode is in HOOKS=()
#
# NOTE: IF you do not remove udev and if you do not replace it with systemd,
# THEN YOUR SYSTEM WILL NOT BOOT.
# This is the only pitfall with systemd-gpt-auto-generator,
#
# It's worth doublechecking.
# Check this again if your system isn't booting post-install.

```

### 4.7 Install UKIs and Configure Bootloader

```bash
# Install systemd-boot
#
# NOTE: Remember to include `--variables=yes` flag. - Here's why:
# Starting with systemd version 257, bootctl began detecting
# environments like arch-chroot as containers...
#
# This is an intended change and without it, it silently skips
# the step of writing EFI variables to NVRAM...
#
# For non-nerds: This prevents issues where the boot entry
# might not appear in the firmware's boot menu...
#
# Install systemd-boot to /efi
bootctl install --esp-path=/efi --variables=yes

# Minimal cmdline with kernel option(s)
nano /etc/kernel/cmdline

# These are the only kernel flags needed for this setup
# With GPT Autoloader you do not need to specify UUIDs here
#
# rootflags add options to the root filesystem, like noatime
# noatime is a typical optimization for EXT4 systems.
#
## /etc/kernel/cmdline
rw rootflags=noatime

# FOR NVIDIA: Arch’s kernels do not enable lockdown by default.
# Signature enforcement is opt-in with module.sig_enforce=1
# Lockdown can be added via lockdown=integrity if you want that policy as well.
## /etc/kernel/cmdline
rw rootflags=noatime module.sig_enforce=1 lockdown=integrity
```

#### Step 4.7.3 Make the ESP directory
```bash
# Make ESP directory
mkdir -p /efi/EFI/Linux
```

#### Step 4.7.5 Edit the mkinitcpio presets so they write UKIs to the ESP

```bash
nano /etc/mkinitcpio.d/linux-zen.preset

# Content:
ALL_kver="/boot/vmlinuz-linux-zen"
PRESETS=('default' 'fallback')

default_uki="/efi/EFI/Linux/arch-linux-zen.efi"

fallback_uki="/efi/EFI/Linux/arch-linux-zen-fallback.efi"
fallback_options="-S autodetect"
```

#### Step 4.8 Repeat for LTS:

```bash
nano /etc/mkinitcpio.d/linux-lts.preset

# Content:
ALL_kver="/boot/vmlinuz-linux-lts"
PRESETS=('default' 'fallback')

default_uki="/efi/EFI/Linux/arch-linux-lts.efi"

fallback_uki="/efi/EFI/Linux/arch-linux-lts-fallback.efi"
fallback_options="-S autodetect"
```

#### Step 4.8.5 Build the UKIs / This writes both kernel *.efi's into ESP/EFI/Linux/:

```bash
mkinitcpio -P
```

#### Step 4.9 Configure bootloader

```bash
# write the loader
nano /efi/loader/loader.conf

# Content:
timeout 10
console-mode auto
editor no

# Verify boot entries
bootctl --esp-path=/efi list
```

## Step 5 SecureBoot Install
  
```bash

# Exit chroot environment
exit

# Unmount all partitions
umount -R /mnt

# Reboot into UEFI setup. The sbctl workflow expects Setup Mode before enrolling.
systemctl reboot --firmware-setup
```

* Find Secure Boot, choose “Reset to Setup Mode” or “Delete all Secure Boot keys.” Do not enable Secure Boot yet.
* Boot back into the ArchISO USB.

### Step 5.5 Install sbctl and create signing keys

sbctl creates Platform Key, KEK, and db for you. 

```bash
# re-open LUKS, mount and chroot back in to resume installation
cryptsetup open /dev/nvme0n1p2 root
mount /dev/mapper/root /mnt
mount /dev/disk/by-label/EFI /mnt/efi
arch-chroot /mnt

# Install sbctl
pacman -S --needed sbctl

# sanity
sbctl status    # should say: secure boot disabled, setup mode, etc.

# make a PK/KEK/db set
sbctl create-keys
```

### Step 6 Enroll keys including Microsoft’s

Enroll your keys and add Microsoft’s as well. The Arch Wiki recommends -m when you need Microsoft’s certs. -f additionally keeps OEM certificates, which can help on some laptops. Some device firmware and Windows boot components are validated with Microsoft’s CAs. Excluding them can break boot paths or firmware flashes when Secure Boot is on.
```bash
# Enroll your keys, with Microsoft's keys, to the UEFI:
sbctl enroll-keys -m

# For some PCs, for example with a Framework laptop, it is recommended
# to also include the OEM firmware's built-in certificates.
#
# If you want to retain the ability to upgrade the firmware and
# run others boot applications provided by OEM. In this case run instead:
sbctl enroll-keys -m -f
```

## Step 6.5 Sign the whole boot chain (systemd-boot and UKIs)

Your ESP is at /efi and your UKIs live in /efi/EFI/Linux. Sign both the bootloader and the UKIs.

1. Sign the systemd-boot binary in /usr/lib and emit a .efi.signed. On update, bootctl will prefer the signed copy automatically. Signing the /usr/lib copy avoids a gap with systemd-boot-update.service.
*  The reason why is if you use systemd-boot and systemd-boot-update.service, the boot loader is only updated after a reboot, and the sbctl pacman hook will therefore not sign the new file. So as a workaround, it can be useful to sign the boot loader directly in /usr/lib/. A bootctl install and update will automatically recognize and copy .efi.signed files to the ESP if present, instead of the normal .efi file.
```bash
# sign the bootloader in its source location
sbctl sign -s \
  -o /usr/lib/systemd/boot/efi/systemd-bootx64.efi.signed \
  /usr/lib/systemd/boot/efi/systemd-bootx64.efi

# optional: also sign the fallback BOOTX64.EFI if present
if [ -f /efi/EFI/BOOT/BOOTX64.EFI ]; then
  sbctl sign -s /efi/EFI/BOOT/BOOTX64.EFI
fi
```

2. Sign the UKIs you generated with mkinitcpio:
```bash
# adjust names if you changed them in your presets
sbctl sign -s /efi/EFI/Linux/arch-linux-zen.efi
sbctl sign -s /efi/EFI/Linux/arch-linux-lts.efi
```

3.  Quick sweep for anything else that needs signing:
```bash
# list anything unsigned that sbctl tracks, then sign it
sbctl verify | sed -E 's|^.* (/.+) is not signed$|sbctl sign -s "\1"|e'
```

## Step 7 Package Choice

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

## Step 8 Install the System

```bash
# Update package database
pacman -Syu

# Run Reflector again since we rebooted for SecureBoot
reflector \                                  
      --country 'Norway,Sweden,Denmark,Germany,Netherlands' \  
      --age 12 \ 
      --protocol https \ 
      --sort rate \
      --latest 10 \
      --save /etc/pacman.d/mirrorlist
```

**EITHER:**

- NVIDIA: 
```bash
# pipe commands, like before type out each pipe line, press enter on each until base-devel
# then when u press enter it installs it all
pacman -S --needed \
  networkmanager reflector \
  pipewire pipewire-alsa pipewire-pulse pipewire-jack wireplumber \
  plasma-meta dolphin konsole kitty kio-admin \
  sddm sddm-kcm linux-zen-headers linux-lts-headers kdegraphics-thumbnailers ffmpegthumbs \
  nvidia-open-dkms nvidia-utils libva-nvidia-driver cuda terminus-font pkgstats hunspell hunspell-en_us  \
  pacman-contrib git wget ttf-dejavu libva-utils \
  base-devel
```

- or AMDGPU:
```bash
sudo pacman -S --needed \
  networkmanager reflector \
  pipewire pipewire-alsa pipewire-pulse pipewire-jack wireplumber \
  plasma-meta dolphin konsole kitty kio-admin \
  sddm sddm-kcm linux-zen-headers linux-lts-headers kdegraphics-thumbnailers ffmpegthumbs \
  amd-ucode linux-firmware \
  mesa \
  vulkan-icd-loader vulkan-radeon \
  libva libvdpau libva-utils \
  terminus-font pkgstats hunspell hunspell-en_us \
  pacman-contrib git wget ttf-dejavu \
  base-devel
```

- or Intel GPUs:
```bash
sudo pacman -S --needed \
  networkmanager reflector \
  pipewire pipewire-alsa pipewire-pulse pipewire-jack wireplumber \
  plasma-meta dolphin konsole kitty kio-admin \
  sddm sddm-kcm linux-zen-headers linux-lts-headers kdegraphics-thumbnailers ffmpegthumbs \
  linux-firmware \
  mesa \
  vulkan-icd-loader vulkan-intel \
  libva libva-utils intel-media-driver libvdpau-va-gl \
  terminus-font pkgstats hunspell hunspell-en_us \
  pacman-contrib git wget ttf-dejavu \
  base-devel
```
---

## Step 8.5, DKMS signing for NVIDIA (certs-local)
when `nvidia-open-dkms` builds its kernel modules, DKMS signs them with your key, and the running kernel accepts them when module signature enforcement is on. The kernel will only load signed modules once you turned on `module.sig_enforce=1`. In-tree modules are already signed and fine. Out-of-tree modules like NVIDIA need to be signed with a key the kernel trusts. 

In this guide we will use the `certs-local` method for this, which assumes the kernel trusts your out-of-tree certificate, and DKMS auto-signs with it. Kernels do not use your UEFI db/PK for module verification. The kernel validates modules against its own keyrings, not the platform keyring, which is why just enrolling keys with sbctl is not enough for out-of-tree modules. 

#### 8.5.1 Install prerequisites
```bash
# Install the helper for zstd-compressed modules
# nvidia-open-dkms already pulled in dkms itself
sudo pacman -S --needed python-zstandard
```

### 8.5.2 Install the certs-local DKMS helpers

* `arch-sign-modules` are signed (In Tree & Out of Tree) Kernel Modules for `linux` `linux-lts` `linux-hardened` `linux-zen` `linux-rt` + AUR kernels.
* We are using the `linux-zen` and `linux-lts` kernels in this tutorial.

```bash
# build from AUR using makepkg (you already have git and base-devel)
git clone https://aur.archlinux.org/arch-sign-modules.git
cd arch-sign-modules
makepkg -si
cd ..
rm -rf arch-sign-modules

# copy the DKMS helper files into /etc/dkms
sudo install -D /usr/src/certs-local/dkms/kernel-sign.conf /etc/dkms/kernel-sign.conf
sudo install -D /usr/src/certs-local/dkms/kernel-sign.sh   /etc/dkms/kernel-sign.sh
sudo chmod 755 /etc/dkms/kernel-sign.sh
```

### 8.5.3 Tell DKMS to auto-sign NVIDIA builds

* The certs-local method uses `ln -s kernel-sign.conf module_name.conf`. 
* Create one symlink per DKMS module name so DKMS runs the signer:

```bash
cd /etc/dkms
# NVIDIA’s DKMS module name is "nvidia"
sudo ln -sf kernel-sign.conf nvidia.conf
```

### 8.5.4 Rebuild and sign the NVIDIA modules for all installed kernels

DKMS compiles per installed headers and runs the post-build signer you just set. 

```bash
# build for zen and lts in one go
sudo dkms autoinstall
```

### 8.5.5 Verify signatures

`modinfo` exposes the module’s signer field, and the kernel’s module-signing facility validates on load. 

```bash
# show the signer recorded in the module
modinfo -F signer nvidia

# you can also look for signature messages
dmesg | grep -i 'module.*sign'
```
---

### Step 9 Create swap file & Configure Zswap

```bash
# Create a swap file (zswap needs a backing swap device)
# This swap file will be 16 GiB. Change 'count=16' if you want less.
#
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
# These are optimizations taken from the wiki.
# Generally considered to be optimal.
nano /etc/sysctl.d/99-zswap.conf

# add
vm.swappiness = 100
vm.page-cluster = 0
vm.watermark_boost_factor = 0
vm.watermark_scale_factor = 125

# update the sysctl
sysctl --system
```

#### Force GTK to use Portals
```bash
# This is important for file pickers etc
# Sometimes programs insist on using the wrong one
# instead of Dolphin (Your File Manager)
#
nano ~/.config/environment.d/gtk-portal.conf
```
```ini
# gtk-portal.conf
GTK_USE_PORTAL=1
GDK_DEBUG=portals
```

### Step 10 Enable Essential Services

```bash
# Enable network, display manager, and timesyncd
systemctl enable NetworkManager sddm systemd-timesyncd fstrim.timer \
reflector.timer pkgstats.timer systemd-boot-update.service
```

## Step 11: Complete Pre-SecureBoot Install

```bash
# Exit chroot environment
exit

# Unmount all partitions
umount -R /mnt
```

## Step 12. Enable SecureBoot and verify It

Reboot into firmware, enable Secure Boot

```bash
# Reboot into the firmware
systemctl reboot --firmware-setup

# Enable SecureBoot #
1. Turn off Setup Mode
2. Enable SecureBoot in your BIOS
3. Remove ArchISO (USB)
4. Reboot
```

* And now you can finally reboot after removing your ArchISO USB into your actual system.
* After hopefully being able to successfully boot. Verify:

```bash
sbctl status   # should show: Secure Boot enabled, and files signed
```

### Step 13. Enroll TPM2 for LUKS after Secure Boot is active
* Do this only after Secure Boot is enabled, so the binding to PCR 7 reflects your real Secure Boot state.
* The reason why is that when you bind to PCR 7, the TPM ties the secret to the Secure Boot state and enrolled certs.
* If SB is off during enrollment, the TPM policy will not match once SB is on.
* The Arch Wiki explicitly says to enroll TPM after signing and enabling SB.
* The HOOKS=(... systemd ... sd-encrypt ...) are correct for LUKS with systemd’s decryptor.
* With a same-disk layout and the 8304 “Linux x86-64 root” type, the systemd-gpt-auto-generator approach is valid, so you still do not need cryptdevice kernel args.

```bash
# optional but strongly recommended
systemd-cryptenroll --recovery-key /dev/nvme0n1p2

# verify a TPM is visible
systemd-cryptenroll --tpm2-device=list

# enroll TPM2 bound to PCR 7 (SB state). You can add +15 if you want firmware config sensitivity.
systemd-cryptenroll --tpm2-device=auto --tpm2-pcrs=7 /dev/nvme0n1p2
# or, with an explicit policy including PCR 15 all-zero to reduce brittleness:
# systemd-cryptenroll /dev/nvme0n1p2 --tpm2-device=auto --tpm2-pcrs=7+15:sha256=$(printf '0%.0s' {1..64})
```

---

# OPTIONAL: Post-Install Tutorial
Head to `arch_kde_tutorial.md` to do the post-install tutorial.
