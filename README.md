# Arch Linux Tutorial

A HTML guide that can be opened in your browser to installing Arch Linux with:

- systemd-automount for GPT partitions (no  `fstab` edits!)
- KDE Plasma on Wayland
- NVME SSD
- systemd-boot
- EXT4 for `/` and `/home`
- AMD CPU + NVIDIA GPU

> **Note:** This tutorial uses Norwegian keymaps and locale/timezone settings. It assumes **NVME SSD,** if you don't have that check with `lsblk -l` to see your scheme, sda or something else.  
> Simply replace those with your own (e.g. keymap, `LANG`, `TZ`). Replace my hostname of `bigboy` with yours, same as my user name `lars`.

---

## Prerequisites

- A bootable Arch Linux USB (written with `dd` or similar)
- Internet connection
- UEFI system

---
