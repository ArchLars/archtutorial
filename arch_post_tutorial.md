# OPTIONAL: Post-Install Tutorial

## 1 · Update Base System

```bash
# Bring everything to the latest version
sudo pacman -Syu
```

---

## 2 · Install AUR Helper (yay)

### 2.1 Prerequisites

### Set Kitty as default terminal in KDE:

* Go to System Settings
* Then to Default Applications
* locate "Terminal Emulator"
* Set it to `kitty`

### Clear Konsole Global Shortcut and set it to Kitty instead

* System Settings → Shortcuts → Global Shortcuts → search for `konsole`
* Unbind Konsole from Ctrl+Alt+T
* After you clear its shortcut, hit Apply. 

### Bind Kitty to Ctrl+Alt+T
* While still in Global Shortcuts, click “Add Application,” pick `kitty`, set the shortcut to `Ctrl+Alt+T`, then Apply.
* If a conflict dialog appears, choose to reassign.
* Test it by pressing the shortcut. `kitty` should now launch with it instead of `konsole`

### Launch Kitty

* Either open it with your shortcut, or click the application launcher located on the bottom left of the panel.
* Navigate to the "System" submenu, then locate & launch the program entitled: `kitty`

### Essential build tools, you already installed these during install but just to be sure
```bash
sudo pacman -S --needed base-devel git  # when you run pacman with the --needed flag it will skip
                                        # any package that is already on the system. Try it.
```

### 2.2 Build and install yay
```bash
cd /tmp                                      # go to the temporary directory
git clone https://aur.archlinux.org/yay.git  # clone the yay pkgbuild from the aur
cd yay                                       # enter the cloned folder
makepkg -si                                  # build the package, then install it and deps
cd ~ && rm -rf /tmp/yay                      # go home, remove the temporary build folder

yay --version  # quick test | NOTE: Whenever you run any 'yay' command, do not use 'sudo' before it.
yay -S --needed --noconfirm fastfetch   # The --noconfirm flag makes it auto confirms the endless
                                        # questions if you want to install something or not.
```

### 2.5 Shell and terminal bliss
```bash
# Oh-my-zsh makes your terminal nicer, zsh-autosuggestions and the other are plugins
# More on them later.
yay -S --needed --noconfirm oh-my-zsh-git zsh-autosuggestions zsh-syntax-highlighting
```

### Copy .zshrc default template config

```bash
# This makes it so you don't have to write out a buncha crap
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

# Also optionally add any aliases here
#
# Here is one for installing packages:
# alias pacin='yay -S --needed --noconfirm'
#
# with this you can just write 'pacin' and then package to install anything
# Example: pacin firefox
# Reason why this is optional is because some might consider it risky
```

### Reload & Guide
```bash
# Then reload zshrc like so:
source ~/.zshrc

#   GUIDE:
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
### Install Basic packages:

```bash
# essential stuff to have.
yay -S --needed --noconfirm informant \
gst-libav gst-plugins-bad gst-plugins-base gst-plugins-good gst-plugins-ugly \
noto-fonts-cjk noto-fonts-extra systemd-timer-notify rebuild-detector \
python-pip kdeconnect journalctl-desktop-notification

# browser
yay -S --needed --noconfirm firefox

# or anything else
yay -S --needed --noconfirm chromium   # example of "anything else"
```

### Configuring Firefox:

#### (Optional) - Force Firefox to use Dolphin
```bash
# Optional if needed. GDK_DEBUG=portals set earlier should have done it.
# If not, force Firefox to do it, open about:config and set:
widget.use-xdg-desktop-portal.file-picker → 1 (always)
```

#### (Optional) - Add all buttons to Firefox
```bash
# Sometimes Firefox does not have the minimize and maximize buttons
# The only remedy is to install:
yay -S --needed --noconfirm xdg-desktop-portal-gtk
```

#### Make Firefox follow your KDE default apps via mimeapps.list on Arch.
```bash
# create if not already created
mkdir -p ~/.local/share/applications

# backup if a real file already exists
[ -f ~/.local/share/applications/mimeapps.list ] && \
  mv ~/.local/share/applications/mimeapps.list ~/.local/share/applications/mimeapps.list.bak

# symlink
ln -sf ~/.config/mimeapps.list ~/.local/share/applications/mimeapps.list
```
#### Add VA-API to Firefox (GPU accelerated video)
```bash
# Confirm VA-API support
vainfo

# Open up about:config and set:
media.ffmpeg.vaapi.enabled to true
```

#### Ensure Firefox media keys dont conflict with Plasma
```bash
# open about:config and set
media.hardwaremediakeys.enabled → false
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

### 4.2 Enable multilib for 32-bit support (pre-Steam)
Uncomment in `/etc/pacman.conf`:
```ini
[multilib]
Include = /etc/pacman.d/mirrorlist
```
Update your system to include multilib:
```bash
# Tip/Fun Fact: You can update your system by just writing 'yay'.
# This is actually ideal, as pacman -Syu does not update your AUR packages.
# Try it:
#

yay

# This is a good time to teach you the habit of running `checkrebuild` after updates.
# 'checkrebuild' checks if you need to rebuild any packages towards new dependencies.
#
# If you don't do that when needed, it can lead to instability.
#

checkrebuild

# usually it doesn't list anything, that means you're good, but if it does you need to run
# yay  -S <pkg> --rebuild
#
```

### 4.2.5 Steam
```bash
# then after enabling multilib DL Steam
yay -S --needed --noconfirm steam

# Run Steam in terminal to install it:
steam
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

### Configure Proton GE as the default in Steam after installing Proton GE from ProtonUp-Qt:

0. Open up ProtonUp-Qt and install the latest version of Proton GE
1. Launch Steam and open **Settings → Compatibility**.  
2. In the dropdown, choose **Proton GE**.  
3. Click OK and restart Steam.

ProtonGE is a good default for a lot of games, works just as well as regular Proton for most games and for other games include 
propietary codecs and such that Valve cannot package themselves, this helps with video files and music with odd formats.


## Final Reboot

#### Reboot again into new system and you can finally sit back, relax, and use arch btw

```bash
# reboot
reboot

# after reboot open kitty (CTRL + ALT + T)
fastfetch

# press prt scr to take a desktop photo
# save it
```
