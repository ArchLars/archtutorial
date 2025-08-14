# Arch Linux Post-Installation Setup Guide
_System Enhancement • AUR Setup • Essential Hooks_

![Arch Linux Logo](https://upload.wikimedia.org/wikipedia/commons/1/13/Arch_Linux_%22Crystal%22_icon.svg)

> **Prerequisites:** You already have a base Arch Linux plus KDE Plasma install that boots with systemd-boot and you have sudo access.

---

## 0 · Legend
Syntax meaning:

- `$` Run as normal user  
- `#` Run as root (or with sudo)  
- grey text Informational notes

---

## 1 · Update Base System

```bash
# Bring everything to the latest version
sudo pacman -Syu

# Confirm core services are alive
systemctl status NetworkManager
systemctl status sddm

# Confirm zram is active (should list zram0)
swapon -s
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
yay -S fastfetch  # then run fastfetch
```

---

## 3 · System Optimisation

### 3.1 Pacman speed and candy
Edit `/etc/pacman.conf`:
```ini
# ParallelDownloads = 5    # uncomment and bump
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
plugins=(git)
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

# Modify options line, append the following:
# options   ... quiet tsc=reliable clocksource=tsc

# Re-install loader
sudo bootctl update

# Reboot later to activate
```

### 3.4 Raise vm.max_map_count (gaming)
```bash
sudo nano /etc/sysctl.d/80-gaming.conf
vm.max_map_count = 2147483642
```

### 3.5 Tune zram swap
```bash
sudo nano /etc/sysctl.d/99-zram.conf
vm.swappiness              = 180
vm.watermark_boost_factor  = 0
vm.watermark_scale_factor  = 125
vm.page-cluster            = 0
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
  check-broken-packages-pacman-hook-git \
  pacdiff-pacman-hook-git \
  reflector-pacman-hook-git \
  nvidia-pacman-hook \
  paccache-hook \
  systemd-boot-pacman-hook \
  pacman-backup-hook
```

---

## 6 · Daily driver apps

- **Text Editor**: `yay -S kate`  
- **Graphics and Media**: `yay -S gimp mpv`  
- **Gaming Stack**: `yay -S --needed arch-gaming-meta dxvk-bin proton-ge-custom-bin`  
- **Productivity**: `yay -S firefox thunderbird`  
- **System Tools**: `yay -S partitionmanager ksystemlog systemdgenie`

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

**Expected results**

- KDE Plasma boot or login appears as expected  
- Zram swap active for better performance  
- Auto mounted root and home partitions  
- NVIDIA drivers loaded and functional  
- Network connectivity via NetworkManager

---

## 9 · Troubleshooting highlights

**cpupower service inactive**
- Run `sudo systemctl start cpupower`, then `journalctl -u cpupower`.

**No game launches**
- Confirm `vm.max_map_count` and 32-bit NVIDIA libraries.

**AUR build fails**
- Install base-devel with `yay -S --needed base-devel`, check free space in `/tmp`.

---

_Arch Linux Post-Installation Setup Guide, System Enhancement • AUR Setup • Essential Hooks_
