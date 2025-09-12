# Complete Arch Linux Tutorial (KDE Plasma + Wayland w/ Automounting Partitions)

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
First time : Pick `arch_kde_tutorial.md`
Looking for an encrypted setup: Pick `arch_kde_encrypted_tutorial.md`
