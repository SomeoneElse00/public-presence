# Arch Linux Experimentation

For this installation, I followed the guide available on the [Arch Organization's Wiki (2024-10-14)](https://wiki.archlinux.org/title/Installation_guide). I will highlight any struggles I had and variances to their steps here.  

## 1 - Pre-Installation

### 1.1 Acquire an Installation Image

I started by using an iso of Arch Linux pulled by [netboot.xyz](https://netboot.xyz), but the process of pulling an image this way took too long. I switched to an iso pulled from the [uwaterloo cs club mirror](https://mirror.csclub.uwaterloo.ca/archlinux/iso/2024.10.01/), pulled on 2024-10-14.  

### 1.2 Verify Signature

Verified sha256 instead.  

### 1.3 Prepare an Installation Medium

For testing purposes, I am installing this OS as a VM in VirtualBox.  

### 1.4 Boot the Live Environment

I allocated a 50gb VHD for this experiment, with 4096MB of RAM, and 4 CPU cores.  

![Arch CMD reached.jpg](<images/Arch CMD reached.jpg>)  

### 1.5 Set KBD Layout

Used `us` (default).  

```bash
loadkeys list-keymaps
loadkeys us
```

### 1.6 Verify Boot Mode

When running `cat /sys/firmware/efi/fw_platform_size`, I discovered that my current version is not running in UEFI 64 bit. I found a toggle in VirtualBox Settings to enable this.  

![Arch EFI.jpg](<images/Arch EFI.jpg>)  

I restarted fom step 1.4, and the command here returned `64` now.  

### 1.7 Internet

`ping archlinux.org` is successful. VM treats the connection as ethernet. Adapter is named `enp0s3`.  

### 1.8 Clock Sync

`timedatectl` showed the clock is synced.  

### 1.9 Partitioning the disks

Here we go, the big confusing step. The table below shows the intended configuration:  

| Disk  | Partition   | Mount Point | type                 | Size |
|-------|-------------|-------------|----------------------|------|
| `sda` | `/dev/sda1` | `/boot`     | EFI System Partition | 1GiB |
| `sda` | `/dev/sda2` | `[swap]`    | Linux Swap           | 4GiB |
| `sda` | `/dev/sda3` | `/`         | Linux x86-64 root    | Rem. |

I am using `GPT` formatting for the disk. This is the sequence of inputs was given to partition the disks using `fdisk`:  

```bash
fdisk -l
# Options of /dev/sda (50GiB), and /dev/loop0 (794.42MiB). Using sda
fdisk /dev/sda
	g # sets disk formatting as GPT
	n # new partition
	 # default:1 for partition number
	 # default:2048 for first sector location
	+1G # allocates 1GiB to the partition (last sector location)
	n
	 # default:2
	 # default:2099200
	+4G
	n
	 # default:3
	 # default:10487808
	 # default:104855551 as end of disk
	t # Partition Type Set
	1 # Partition 1 select
	L # List types
	q # leave partition type inspection
	EFI System # Set partition type
	t
	2
	Linux swap
	t
	3
	23 # Linux root (x86-64)
	p # verify changes (see below image)
	w # write changes
```  

![Arch fdisk.jpg](<images/Arch fdisk.jpg>)  

### 1.10 Format the Partitions

```bash
mkfs.ext4 /dev/sda3
mkswap /dev/sda2
mkfs.fat -F 32 /dev/sda1
```  

### 1.11 Mount the File Systems

```bash
mount /dev/sda3 /mnt
mount --mkdir /dev/sda1 /mnt/boot # mount the boot partition for UEFI
swapon /dev/sda2
```  

## 2 - Installation

### 2.1 Mirror Selection

checked the mirror list, and decided to make no changes.  

```bash
nano /etc/pacman.d/mirrorlist
```  

### 2.2 Install essential packages

I skipped installing CPU microcode since I'm using a VM. I also skipped using the `linux-firmware` package since I am installing in a VM.  

```
pacstrap -K /mnt base linux man-db man-pages texinfo grub efibootmgr nano networkmanager virtualbox-guest-utils
```  

| package                | rationale             |
|------------------------|-----------------------|
| base                   | required for arch     |
| linux                  | kernel                |
| man-db                 | man command           |
| man-pages              | man commands          |
| texinfo                | info commands         |
| grub                   | bootloader            |
| efibootmgr             | bootloader dependency |
| nano                   | text editor           |
| networkmanager         | <--                   |
| virtualbox-guest-utils | VM Utilities          |

## 3 - Configure the System

### 3.1 Fstab

No changes, no errors  

```bash
genfstab -U /mnt >> /mnt/etc/fstab
```  

### 3.2 chroot

No issues  

```bash
arch-chroot /mnt
```  

### 3.3 Time

```bash
ln -sf /usr/share/zoneinfo/America/Toronto /etc/localtime
hwclock --systohc
```  

### 3.4 Localization

```bash
nano /etc/locale.gen
	# Uncommented en_CA.UTF-8 UTF-8
	# ^x Y <enter>
nano /etc/locale.conf
	# new file
	# LANG=en_CA.UTF-8
	# ^x Y <enter>
nano /etc/vconsole.conf
	# new file
	# KEYMAP=us
	# ^x Y <enter>
```  

### 3.5 Network Config

Set hostname as `arch1vm` in `/etc/hostname`  

### 3.6 Initamfs

LVM, encryption, nor RAID were used, so this step was skipped.  

### 3.7 Root Password

```bash
passwd
arch_toor
# root:arch_toor
```  

### 3.8 Bootloader

To keep things familiar, I am using GRUB.  

```bash
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB
# no error reported
```  

For a first install, I am skipping secure boot. Next time I perform an Arch install, I will do this.  

#### 3.8.1 Error Correction

To correct the error earlier, I ran the following CORRECTED command:  

```bash
grub-install /dev/sda --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB
# no error reported
```  

I also generated the GRUB config file:  

```bash
grub-mkconfig -o /boot/grub/grub.cfg
```  

## 4 - Reboot

```bash
exit # leave chroot mode
umount -R /mnt
reboot
```  

![Arch bad grub.jpg](<images/Arch bad grub.jpg>)  

Once I rebooted, the GRUB interface did not appear (a grub terminal did instead), so I assumed I skipped a step in GRUB setup. I realized I did not specify the disk for grub to boot from, so I needed to use grub shell to get back in. 

First, I quickly ran `set pager=1` to clean up the output in the terminal. Next, I ran `ls` to see the drives. I had three familiar drives: `(hd0,gpt3)`, `(hd0,gpt2)`, and `(hd0,gpt1)`. After running `ls (hd0,gpt1)`, and seeing my boot files, I knew that gpt1 was my boot files, gpt2 was my swap partition, and gpt3 was my root directory. I ran `linux (hd0,gpt1)/vmlinuz-linux root=/dev/sda3` to set the kernel, and `initrd (hd0,gpt1)/initramfs-linux.img` to set the boot image. I finally ran `boot` to load the system.  

Luckily, it loaded the login prompt!!  

See 3.8.1 for corrective actions.  

## 5 - Post-Installation

### 5.1 [General Recommendations](https://wiki.archlinux.org/title/General_recommendations)

#### 5.1.1 System Administration

First, to start with package installations, I needed to enable `networkmanager`. Next, I installed the `nano` text editor, and `sudo`.

```bash
systemctl start NetworkManager.service
ping archlinux.org #successful
pacman -S nano sudo
```  

When editing the sudoers file, I noticed that `visudo` was not cooperating or allowing edits to the file. I installed `vim` and `vi` to allow edits, but was having constant crashes. I wanted to switch the editor to `nano` to make it easier. I found the option to set the default editor via environment variable, so I ran the below commands. I will change the visual editor once I have a visual environment set up.  

```bash
export EDITOR="nano"
export VISUAL="$EDITOR"
```

Following this, I used `visudo` to allow members of the `wheel` group become sudo (uncomment the appropriate line).<!--, and then added my user to the wheel group with `gpasswd -a [user] wheel`. -->  

```bash
visudo
# nano opens 
	# uncommented line enabling wheel members to become sudo:
	# 		%wheel ALL=(ALL:ALL) ALL
	# ^x Y <enter>
```  

I created my user, and allowed this user to be a sudoer by adding it to the wheel group.  

```bash
useradd -m -G wheel briar
passwd briar
# briar:bhaal
```  

I now logged out of the root user, and switched to my user.  

##### Security

Looked at [Security-ArchWiki](https://wiki.archlinux.org/title/Security).  

<!--wayland-->

##### Service Management

I added autolaunching for network manager  

```bash
sudo systemctl enable NetworkManager.service
```  

##### System Maintenance

Commands for maintenance:  

```bash
systemctl --failed
journalctl -b
pacman -Syu
pacman -Qtd
pacman -Qm
```  

#### 5.1.2 Package Management

Using `pacman`. Mirrors and repos were already checked during install. No new options were added.  

#### 5.1.3 Booting

[Numlock enable on boot via `systemd`](https://wiki.archlinux.org/title/Activating_numlock_on_bootup#:~:text=With%20systemd%20service):  

```bash
nano /usr/local/bin/numlock

	#!/bin/bash

	for tty in /dev/tty{1..6}
	do
		/usr/bin/setleds -D +num < "$tty";
	done

sudo chmod +x /usr/local/bin/numlock
```  

```bash
nano /etc/systemd/system/numlock.service

	[Unit]
	Description=numlock

	[Service]
	ExecStart=/usr/local/bin.numlock
	StandardInput=tty
	RemainAfterExit=yes

	[Install]
	WantedBy=multi-user.target

sudo systemctl enable numlock.service
```  

This process can also be found in settings of the appropriate DE, or in BIOS/UEFI (for an installation on hardware, instead of virtualized).  

<!--This process can also be done via KDE, Sway, or Hyprland -->

#### 5.1.4 GUI

My next steps for continuing with this experiment will be to get a gui working. After trying to make `hyprland` work for a bit and having minimal success, I will try another DE soon.  

<!--
I've wanted to try a tiling window manager for a bit, so I installed `hyprland`. 

```bash
sudo pacman -S hyprland dunst xdg-desktop-portal-hyprland pipewire wireplumber polkit-kde-agent foot qt5-wayland qt6-wayland wofi dolphin gtk3 gtk2

nano ~/.config/hypr/hyprland.conf
	# change:
	$terminal = kitty -> $terminal = foot
	# add:
	exec-once = /usr/lib/polkit-kde-authentication-agent-1
```

I enabled 3D acceleration in VirtualBox settings, then ran `hyprland`

I was able to get hyprland working, but it was extremeley slow. 
-->