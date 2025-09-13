# Complete Arch Linux Tutorial (KDE Plasma + Wayland + LUKS + SecureBoot + TPM2 w/ Automounting Partitions)

This is an Arch installation guide for novices **WITH LUKS encryption, TPM2 and SecureBoot** (More complex) who want a working secure system to game on that's straight forward & encrypted. I am assuming you have already read and understood `arch_kde_tutorial.md` before tackling this one. For this we are going to be building a system that's as secure as it is functional, but without the usual hassle that comes with hardcore security setups. Think of it like having bank-level security that actually gets out of your way instead of constantly asking for passwords.

### The Trio: LUKS + SecureBoot + TPM2

**LUKS encryption** protects your data at rest 

**TPM2** (Trusted Platform Module 2.0) a security chip on your motherboard that can securely store encryption keys and only release them when your system is in a "trusted state." The TPM will automatically unlock your drive as long as your boot chain hasn't been tampered with.

**SecureBoot** ensures that only signed, trusted code can run during the boot process. We'll be signing our own bootloader and kernel images, creating what's called a "chain of trust." If Secure Boot is disabled or its key databases are tampered with, the TPM will not release the key to unlock the encrypted partition.

We will be using systemd-gpt-auto-generator in this tutorial too, and it comes with an added benefit for LUKS setups. When systemd-gpt-auto-generator detects a partition with type 8304 that contains LUKS, it automatically:

* Creates /dev/gpt-auto-root-luks symlink pointing to the encrypted partition
* Attempts to unlock it using systemd-cryptsetup
* Creates /dev/gpt-auto-root symlink pointing to the unlocked volume
* Mounts the unlocked volume as root

This is much less tedious than a manual setup. For more on how GPT Auto-Mounting works in general, read the intro to `arch_kde_tutorial.md`

---

## What I will personally be using/setting:

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

---

*Protip:* This tutorial uses Norwegian keymaps and locale/timezone settings. Simply replace those with your own (e.g. keymap, `LANG`, `TZ`).
If you use an English lang keyboard you can ignore all of it, but it's worth knowing if you are new and use a different keyboard like say `de-latin1` for German keyboards.

**NOTE:** **This tutorial assumes you have a NVME SSD,** which are named `/dev/nvme0n1`. If you don't have that, it's something else. If you don't know, check with `lsblk -l` to see your scheme. It could be `sda` or something else. If it is something else replace all instances of `nvme0n1` and remove the `p` from  `${d}p1` in the formatting.

*Sidenote:* Unless you like the name, replace my hostname (basically the name of your rig) of `BigBlue` with yours. Though note, its generally frowned upon to use "Arch" or "Linux" identifiers in the hostname, use something that doesn't advertise your computer and especially choice of distro for security reasons. That goes same for my user name `lars`, change it if your name ain't Lars. Though if it is, cool. Hi! I thought about doing placeholders but I feel those are more distracting usually, I prefer to see how something would actually work in a guide, maybe you do as well?


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

Create a GPT partition table with two partitions AFTER checking your drive name with `lsblk -l` :

```bash
lsblk -l  # To confirm, replace if it says anything else

sgdisk --zap-all /dev/nvme0n1

# When using 4096-byte sectors in LUKS, the partition size must be a multiple of 4096 bytes (8 sectors of 512 bytes).
# The device mapper will fail if the partition isn't properly aligned. This makes resizing particularily challenging.
# -I makes sgdisk align partition ends, and -a 1M is explicit MiB alignment.
# Using +1GiB avoids decimal vs binary confusion.
#
sgdisk -I -a 1M -n1:0:+1GiB -t1:EF00 -c1:"EFI system" /dev/nvme0n1
sgdisk -I -a 1M -n2:0:0 -t2:8304 -c2:"Linux root (x86-64)" /dev/nvme0n1

# This test should afterwards report zero
test $(( $(blockdev --getsize64 /dev/nvme0n1p2) % 4096 )) -eq 0 && echo OK || echo NOT_ALIGNED
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

* Here you will set up LUKS2 encryption on the root partition.
* You'll be prompted to enter YES (in capitals) and create a passphrase.

```bash
# 4K encryption sectors are set at format time.
# LUKS2 with 4 KiB encryption sectors is valid and often faster,
# it is the optimal sector size according to the Arch Wiki
#
# N.B: 1) that changing this later requires a *FULL* re-encrypt
#
# 2) that the partitions have been aligened correctly before proceeding.
#
cryptsetup luksFormat --type luks2 --sector-size 4096 /dev/nvme0n1p2
```
Open it and persist the useful runtime flags in the LUKS2 header:

```bash
# These will apply automatically at every boot, including gpt-auto unlock.
# This creates /dev/mapper/root which we'll format in the next step
cryptsetup open \
  --allow-discards \  # n.b - This can leak usage patterns. 
  --perf-no_read_workqueue \  
  --perf-no_write_workqueue \
  --perf-submit_from_crypt_cpus \
  --persistent \
  /dev/nvme0n1p2 root
```

Verify the LUKS header was created successfully:

```bash
cryptsetup luksDump /dev/nvme0n1p2

# Optional 'Flags' sanity check specifically
cryptsetup luksDump /dev/nvme0n1p2 | grep -i '^Flags'
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
# Example for MODULES if you use amdgpu:
MODULES=(amdgpu)

# Example for MODULES if you use nvidia:
MODULES=(nvidia nvidia_modeset nvidia_uvm nvidia_drm)
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
# Arch’s kernels do not enable lockdown by default.
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

# Set SecureBoot to Setup Mode
# This is different for every BIOS/Firmware. Look up how to do it for yours

# Reboot
# Reboot back into the ArchISO (USB), NOT your current install
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

# Install efitools to read existing keys
pacman -S --needed efitools

# Create a backup directory for existing keys
mkdir -p /root/existing_keys_backup
cd /root/existing_keys_backup
```

#### Read and backup all existing keys in ESL format
```bash
# write out all of this and press enter after each 'do' and 'done' to run.
for var in PK KEK db dbx; do  # Press enter here to go down to 2nd line
    efi-readvar -v $var -o old_${var}.esl 2>/dev/null  # Press Tab for code indentation
done
```

#### Check if any keys were successfully backed up
```bash
ls -la *.esl 2>/dev/null
```

```bash
# Convert ESL files to human-readable certificates to inspect them
# This helps identify if there are OEM-specific certificates
pacman -S --needed sbsigntools
```

#### Convert and inspect the certificates (if they exist)
```bash
# write out all of this and press enter after 'done'
for esl in *.esl; do
    [ -f "$esl" ] || continue
    base=$(basename "$esl" .esl)
    sig-list-to-certs "$esl" "$base" 2>/dev/null
done

# Check for vendor-specific certificates
find . -name "*.der" -type f 2>/dev/null | while read cert; do
    openssl x509 -in "$cert" -inform DER -text -noout | grep -E "Subject:|Issuer:" || true
done

# Alternative: use sbctl to check what vendor keys are available
sbctl list-enrolled-keys 2>/dev/null
```

```bash
# Return to previous directory
cd /mnt
```

### Step 6 Enroll keys including Microsoft’s

Enroll your keys and add Microsoft’s as well. The Arch Wiki recommends -m when you need Microsoft’s certs. -f additionally keeps OEM certificates, which can help on some laptops. Some device firmware and Windows boot components are validated with Microsoft’s CAs. Excluding them can break boot paths or firmware flashes when Secure Boot is on.
```bash
# Check what keys will be enrolled before committing
# You can export/preview what will be enrolled without actually enrolling
sbctl enroll-keys -m --export esl
ls -la *.esl  # Review what would be enrolled
rm *.esl      # Clean up the preview files

---

# OPTION 1: Standard setup (most common)
# Use this for most desktop systems and custom-built PCs
sbctl enroll-keys -m

# OPTION 2: Laptop/OEM system with vendor firmware
# Use this for laptops (especially Framework, Dell, HP, Lenovo)
# or if you found OEM certificates above
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

# optional only for AMDGPU (No shim)
# also sign the fallback BOOTX64.EFI if present:
#
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

## Step 6.6 ONLY FOR NVIDIA: Export sbctl's db certificate for MOK enrollment

If you are using NVIDIA you will need to to make shim trust sbctl's signatures:
```bash
# Export sbctl's db certificate to DER format for MOK enrollment
openssl x509 -in /usr/share/secureboot/keys/db/db.pem \
  -outform DER -out /root/secureboot/mok/sbctl_db.cer

# Verify the certificate was exported correctly
openssl x509 -in /root/secureboot/mok/sbctl_db.cer -inform DER -text -noout | grep Subject
```

## Step 7 Package Choice

* I removed `pkgstats` from this install considering the nature of it.
* Everything else is the same as in `arch_kde_tutorial.md`

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
  plasma-meta dolphin konsole kitty kio-admin wireless-regdb efibootmgr \
  sddm sddm-kcm linux-zen-headers linux-lts-headers kdegraphics-thumbnailers ffmpegthumbs \
  nvidia-open-dkms nvidia-utils libva-nvidia-driver cuda terminus-font hunspell hunspell-en_us  \
  pacman-contrib git wget ttf-dejavu libva-utils \
  base-devel
```

- or AMDGPU:
```bash
sudo pacman -S --needed \
  networkmanager reflector \
  pipewire pipewire-alsa pipewire-pulse pipewire-jack wireplumber \
  plasma-meta dolphin konsole kitty kio-admin wireless-regdb efibootmgr \
  sddm sddm-kcm linux-zen-headers linux-lts-headers kdegraphics-thumbnailers ffmpegthumbs \
  amd-ucode linux-firmware \
  mesa \
  vulkan-icd-loader vulkan-radeon \
  libva libvdpau libva-utils \
  terminus-font hunspell hunspell-en_us \
  pacman-contrib git wget ttf-dejavu \
  base-devel
```

- or Intel GPUs:
```bash
sudo pacman -S --needed \
  networkmanager reflector \
  pipewire pipewire-alsa pipewire-pulse pipewire-jack wireplumber \
  plasma-meta dolphin konsole kitty kio-admin wireless-regdb efibootmgr \
  sddm sddm-kcm linux-zen-headers linux-lts-headers kdegraphics-thumbnailers ffmpegthumbs \
  linux-firmware \
  mesa \
  vulkan-icd-loader vulkan-intel \
  libva libva-utils intel-media-driver libvdpau-va-gl \
  terminus-font hunspell hunspell-en_us \
  pacman-contrib git wget ttf-dejavu \
  base-devel
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
# This is all that is neeeded for zswap to be enabled on Arch.
# Officially supported kernels like linux-lts and linux-zen have zswap enabled by default.
#
# Be aware that any unofficial kernel may not, however. To enable zswap on those kernels
# add zswap.enabled=1 to the kernel parameters.
#
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

### Step 9.5 Enable Essential Services

```bash
# Enable network, display manager, and timesyncd
systemctl enable NetworkManager sddm systemd-timesyncd fstrim.timer \
reflector.timer systemd-boot-update.service
```


## Step 10 ONLY FOR NVIDIA: DKMS signing for NVIDIA on SecureBoot
**IMPORTANT EXTRA STEP FOR NVIDIA USERS:** when `nvidia-open-dkms` builds its kernel modules, DKMS signs them with your key, and the running kernel accepts them when module signature enforcement is on. The kernel will only load signed modules once you turned on `module.sig_enforce=1`. In-tree modules are already signed and fine. Out-of-tree modules like NVIDIA need to be signed with a key the kernel trusts. 

We will be using DKMS’s Native signing method by setting mok_signing_key and mok_certificate so that nvidia-open-dkms builds come out pre-signed. Then we enroll that cert with mokutil. This works on stock kernels with signature enforcement turned on (module.sig_enforce=1, which you already added in Step 4.7). shim is a Microsoft-signed first-stage bootloader. When you boot through shim and enroll your Machine Owner Key (MOK), the kernel can trust modules signed with that key. DKMS can sign every module it builds if you point it at your private key and certificate. That way nvidia-open-dkms is always signed, so with module.sig_enforce=1 the driver loads cleanly. We will: install shim and tools, generate a MOK keypair, tell DKMS to use it, rebuild the NVIDIA modules, enroll the cert, and verify. 

### 10.5.1 Install shim and tools

* mokutil manages the MOK database that shim uses.
* sbsigntools provides sbsign which you may use later to sign EFI binaries if needed.
* DKMS is the framework that builds the NVIDIA open modules and will sign them for us.
  
```bash
# You already have base-devel and git from Step 8.
# Build shim from AUR (no AUR helper needed):
cd /root
git clone https://aur.archlinux.org/shim-signed.git
cd shim-signed
makepkg -si

# Tools to sign and enroll, plus dkms if it is not present:
pacman -S --needed sbsigntools mokutil dkms
```

### 10.5.2 Install the certs-local DKMS helpers

* shim looks for a file named grubx64.efi by default. We keep systemd-boot, we just let shim launch it by using that filename. Your ESP is /efi from earlier that we signed.
* This follows the Arch Wiki’s shim setup: copy shimx64.efi and mmx64.efi, and use grubx64.efi as the next stage so shim finds it reliably. We are using systemd-boot’s binary at that name.
* Tip: leave your existing systemd-boot boot entry in place for recovery. Your firmware will try the new “Shim (MOK) [Arch]” entry first once you move it up in boot order.

```bash
# Make a standard fallback path and copy shim:
mkdir -p /efi/EFI/BOOT
cp /usr/share/shim-signed/shimx64.efi /efi/EFI/BOOT/BOOTX64.EFI
cp /usr/share/shim-signed/mmx64.efi   /efi/EFI/BOOT/

# Let shim chainload systemd-boot entry that we signed way earlier under the name grubx64.efi:
cp /usr/lib/systemd/boot/efi/systemd-bootx64.efi.signed /efi/EFI/BOOT/grubx64.efi

# Add an NVRAM entry that boots shim
efibootmgr --create --disk /dev/nvme0n1 --part 1 \
  --label "Shim (MOK) [Arch]" \
  --loader '\EFI\BOOT\BOOTX64.EFI'
efibootmgr -v  # sanity check
```

### 10.5.3 Make a MOK keypair (PEM for DKMS, DER for enrollment)

* Create a private key and X.509 certificate. 
* DKMS will use the PEM files, shim/MokManager wants DER for enrollment.
* The shim + MOK workflow expects a 2048-bit RSA key.
* MokManager wants the certificate in DER format for import. 

```bash
# Create a directory for your MOK
mkdir -p /root/secureboot/mok
cd /root/secureboot/mok

# Generate 2048-bit RSA MOK (recommended for shim)
openssl req -new -x509 -newkey rsa:2048 -nodes -days 36500 \
  -keyout MOK.key \
  -out    MOK.crt \
  -subj "/CN=Arch DKMS MOK/"

# Convert certificate to DER for mokutil enrollment
openssl x509 -in MOK.crt -outform DER -out MOK.cer

chmod 400 MOK.key
```

### 10.5.4 Tell DKMS to sign with your MOK (Native DKMS method)

* DKMS natively signs what it builds if you point it to your key and cert in /etc/dkms/framework.conf (or a drop-in). 
* Also make sure sign-file comes from the kernel headers. On Arch’s stock kernels, headers provide scripts/sign-file.
* This is exactly the “Native DKMS method” described on the Arch Wiki’s Signed kernel modules page, and DKMS’s upstream README documents these mok_* variables.

```bash
mkdir -p /etc/dkms/framework.conf.d

cat >/etc/dkms/framework.conf.d/10-mok.conf <<'EOF'
mok_signing_key=/root/secureboot/mok/MOK.key
mok_certificate=/root/secureboot/mok/MOK.crt
sign_file=/lib/modules/${kernelver}/build/scripts/sign-file
EOF
```

### 10.5.5 Rebuild NVIDIA modules so they are signed

* Now rebuild nvidia-open-dkms for both kernels you installed (zen and lts). The DKMS hooks will sign as it builds.
* After this, each built .ko should have an embedded signature. We will verify after enrollment. 

```bash
# Make sure headers for both kernels are present (you installed them earlier).
# Reinstall to trigger a DKMS rebuild and signing:
pacman -S --needed nvidia-open-dkms

# Check DKMS status
dkms status

# Optional: force a rebuild for all installed kernels
dkms autoinstall
```

### 10.5.6 Enroll the MOK certificate

* Queue the DER certificate for enrollmen.

```bash
# Queue the cert for shim to enroll
# First, enroll the DKMS MOK for module signing
mokutil --import /root/secureboot/mok/MOK.cer

# Set and remember a password for DKMS MOK #

# Then, enroll sbctl's db certificate so shim trusts systemd-boot
mokutil --import /root/secureboot/mok/sbctl_db.cer

# Set and remember a password for sbctl db cert #
```
---

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

# If on NVIDIA / Enable SecureBoot #
1. Turn off Setup Mode
2. Enable SecureBoot in your BIOS
3. Remove ArchISO
4. Choose your "Shim (MOK) [Arch]" boot entry
5. Boot from it
6. MokManager appears - you'll need to enroll BOTH certificates:
   
   First enrollment (DKMS MOK):
   a. Select "Enroll MOK"
   b. Select MOK.cer
   c. View key to verify it's your DKMS MOK
   d. Confirm enrollment
   e. Enter the password you set for DKMS MOK
   
   Second enrollment (sbctl db):
   f. Select "Enroll MOK" again
   g. Select sbctl_db.cer  
   h. View key to verify it's sbctl's db certificate
   i. Confirm enrollment
   j. Enter the password you set for sbctl db
   
7. Select "Continue boot"
8. System will reboot

# If on AMDGPU / Enable SecureBoot #
1. Turn off Setup Mode
2. Enable SecureBoot in your BIOS
3. Remove ArchISO (USB)
4. Reboot
```

* And now you can finally reboot after removing your ArchISO USB into your actual system.
* After hopefully being able to successfully boot. Verify:

```bash
sbctl status   # should show: Secure Boot enabled, and files signed

If NVIDIA:
bootctl status | sed -n '1,25p'   # look for: Secure Boot: enabled (user)

# Check that both MOKs are enrolled
mokutil --list-enrolled | grep -E "CN=Arch DKMS MOK|CN=Platform Key"

# Verify shim can load systemd-boot
sbverify --list /efi/EFI/BOOT/grubx64.efi

# Verify NVIDIA modules are signed and loadable
modinfo nvidia | grep signature
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

### Step 14. ONLY FOR NVIDIA: Pacman hook to harmonize Shim with Systemd-boot on Systemd Update

When systemd-boot updates, bootctl will refresh EFI/systemd/systemd-bootx64.efi and prefer the .efi.signed file
but your EFI/BOOT/grubx64.efi copy will not be touched. For this we will add a tiny pacman hook that, after systemd updates, 
copies the freshly signed /usr/lib/.../systemd-bootx64.efi.signed to EFI/BOOT/grubx64.efi. The “.efi.signed preference” is 
documented in the man page and ArchWiki.

* Add a tiny pacman hook that runs after any systemd update.
* It copies the already-signed systemd-boot from /usr/lib/.../systemd-bootx64.efi.signed
* to the filename shim expects, /efi/EFI/BOOT/grubx64.efi

```bash
# create the helper
sudo install -Dm0755 /dev/stdin /usr/local/bin/copy-sdboot-to-grubx64 <<'EOF'
#!/bin/sh
set -eu

SRC="/usr/lib/systemd/boot/efi/systemd-bootx64.efi.signed"
ALT="/usr/lib/systemd/boot/efi/systemd-bootx64.efi"
DST="/efi/EFI/BOOT/grubx64.efi"

# need ESP mounted at /efi
mountpoint -q /efi || exit 0

# prefer the signed artifact, fallback to unsigned if needed
if [ -f "$SRC" ]; then
  install -Dm0644 "$SRC" "$DST"
elif [ -f "$ALT" ]; then
  install -Dm0644 "$ALT" "$DST"
else
  # nothing to copy
  exit 0
fi

# optional sanity if sbverify is present
if command -v sbverify >/dev/null 2>&1 && [ -f "$SRC" ]; then
  sbverify --list "$DST" >/dev/null 2>&1 || true
fi
EOF
```

```bash
# hook that calls the helper on systemd update
sudo install -d /etc/pacman.d/hooks
sudo tee /etc/pacman.d/hooks/99-sdboot-to-grubx64.hook >/dev/null <<'EOF'
[Trigger]
Operation = Install
Operation = Upgrade
Type = Package
Target = systemd

[Action]
Description = Copy signed systemd-boot to ESP/EFI/BOOT/grubx64.efi
When = PostTransaction
Depends = systemd
Exec = /usr/local/bin/copy-sdboot-to-grubx64
EOF
```


---

# 1) OPTIONAL: Post-Install Tutorial
Head to `arch_post_tutorial.md` to do the post-install tutorial.
(NOTE: I am currently writing the version of adding a new drive that's encrypted for this tutorial, don't do the one that's there now)

---

# 2) OPTIONAL: How to fix those annoying 'missing firmware' warnings in mkinitcpio

* Whenever you write `mkinitcpio -P` you might notice it keeps warning you about firmware that you are supposedly missing.
* If this bothers you, check out my tutorial, `mkinitcpio-fix.md` to fix this.

---

# 3) OPTIONAL: Add LXQt with Openbox + picom for a Lightweight backup X11 session

Since X11 has been depreciated with Plasma (though you can still try to use it I wouldn't recc it) 
I would instead choose to use something like LXQt (which works well with SDDM as well) for your backup
session with X11.

Head to `lxqt-post-install.md` for this tutorial.

---
