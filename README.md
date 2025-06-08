# Arch Linux Tutorial

A HTML guide that can be opened in your browser to installing Arch Linux with:

- systemd-automount for GPT partitions (no  `fstab` edits!)
- KDE Plasma on Wayland
- NVME SSD
- systemd-boot
- EXT4 for `/` and `/home`
- AMD CPU + NVIDIA GPU

> **Note:** This tutorial uses Norwegian keymaps and locale/timezone settings. It assumes **NVME SSD,** which uses `/dev/nvme0n1` partitions. If you don't have that, it's something else. If you don't know, check with `lsblk -l` to see your scheme. It could be `sda` or something else. If it is something else replace all `nvme0n1` with that.
> Simply replace those with your own (e.g. keymap, `LANG`, `TZ`). Replace my hostname of `bigboy` with yours, same as my user name `lars`.

---

## Prerequisites

- A bootable Arch Linux USB (written with `dd` or similar)
- Internet connection
- UEFI system

---
