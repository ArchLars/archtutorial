# OPTIONAL: Add LXQt with Openbox + picom for a Lightweight backup X11 session

Since X11 has been depreciated with Plasma (though you can still try to use it I wouldn't recc it) 
I would instead choose to use something like LXQt (which works well with SDDM as well) for your backup
session with X11.

#### Packages
```bash
#    - breeze, breeze-icons ship the Qt style and icon theme.
#    - breeze-gtk provides Breeze and Breeze-Dark GTK themes.
#    - lxqt + openbox gives you the LXQt desktop and Openbox WM for the X11 session.
#    - picom is the X11 compositor.
sudo pacman -S --needed xorg-server lxqt openbox picom breeze breeze-icons breeze-gtk
```

#### Make LXQt use Openbox as its window manager (X11 session).
```bash
#    This is the canonical LXQt way to pick the WM. - create:
mkdir -p ~/.config/lxqt

# create
sudo nano ~/.config/lxqt/session.conf

## add/change to session.conf
[General]
window_manager=openbox
```

#### Set LXQt appearance via config.
```bash
#    - icon_theme uses the icons directory name, "breeze".
#    - style is the Qt widget style "Breeze" (capitalization matters for Qt styles).
#    This is read by LXQt and applied to Qt apps through the lxqt-qtplugin.
#    If lxqt.conf exists already, just ensure these keys exist or are updated.
sudo nano ~/.config/lxqt/lxqt.conf

## add/change to lxqt.conf
icon_theme=breeze
style=Breeze
```

#### Autostart picom only in LXQt, not in Plasma Wayland.
```bash
#    ensure autostart is created
mkdir -p ~/.config/autostart

# then create/edit picom.desktop
sudo nano ~/.config/autostart/picom.desktop

## add/change to picom.desktop
[Desktop Entry]
Type=Application
Name=picom
Comment=X11 compositor
Exec=picom --config ~/.config/picom/picom.conf --vsync     # vsync for tearfree
OnlyShowIn=LXQt;    # This makes it only autostart on LXQt (important)
X-GNOME-Autostart-enabled=true
```

#### Provide a simple picom.conf. Start with Arch defaults then tweak.
```bash
# create
mkdir -p ~/.config/picom

# then create
sudo nano ~/.config/picom/picom.conf

## and finally add to picom.conf
backend = "glx";
vsync = true;
unredir-if-possible = true;

# Subtle shadows
shadow = true;
shadow-radius = 12;
shadow-opacity = 0.25;
shadow-offset-x = -10;
shadow-offset-y = -10;

# Slight transparency for unfocused windows
inactive-opacity = 0.95;

# Respect Openbox stacking
detect-client-leader = true;
detect-transient = true;
```

#### Make sure an X11 LXQt session appears in your greeter.
```bash
#    Nothing to write here; just verify the entry exists:
grep -H . /usr/share/xsessions/*lxqt*.desktop
```

#### Test run: log out of Plasma, pick "LXQt" in SDDM, log in.
```bash
#    Verify in LXQt that you're on X11 and LXQt picked Openbox and Breeze:
#    - echo $XDG_SESSION_TYPE should print "x11"
#    - ps aux | grep -E 'openbox|picom' should show both running
#    - Qt apps and LXQt panel should look like Breeze, GTK apps should use Breeze and breeze icons
```

---
