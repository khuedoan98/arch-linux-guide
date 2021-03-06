# Arch Linux Installation Guide

Installation guide and basic configurations for Arch Linux

## Verify the boot mode

Check if the directory exists:

`ls /sys/firmware/efi/efivars`

## Connect to the internet

Connect to Wi-Fi network:

`wifi-menu`

Check if internet connectivity is available:

`ping -c 3 archlinux.org`

## Update the system clock

Ensure the system clock is accurate:

`timedatectl set-ntp true`

Check the service status:

`timedatectl status`

## Partition the disks

Identify disks:

`lsblk`

Disks are assigned to a *block device* such as `/dev/nvme0n1`.

Clean the entire disk (**do not** do this if you want to keep your data):

* `# gdisk /dev/nvme0n1`
* `x` for extra functionality
* `z` to *zap* (destroy) GPT data structures and exit
* `y` to proceed
* `y` to blank out MBR

Create boot partition and root partition:

* `# cfdisk /dev/nvme0n1`
* Select `gpt`
* Hit `[   New   ]` to create a new patition
* Give the boot partition `1G` and let the rest for the root partition
* Select the boot partition and hit `[   Type   ]` to choose `EFI System`
* Hit `[   Write   ]` then type `yes` to save, then hit `[   Quit   ]`

## Format the partitions

Format the boot partition to FAT32:

`mkfs.fat -F32 /dev/nvme0n1p1`

Format the root partition to ext4:

`mkfs.ext4 /dev/nvme0n1p2`

## Mount the file systems

Mount root partition first:

`mount /dev/nvme0n1p2 /mnt`

Then create mount point for boot partition and mount it accordingly:

`mkdir /mnt/boot`

`mount /dev/nvme0n1p1 /mnt/boot`

## Select the mirrors

Make a list of mirrors sorted by their speed then remove those from the list that are out of sync according to their [status](https://www.archlinux.org/mirrors/status/).

Backup the existing mirrorlist:

`cp /etc/pacman.d/mirrorlist /etc/pacman.d/mirrorlist.backup`

Edit the mirror list, bring the fastest mirrors to the top (you can use `nano` instead of `vim` or `nvim`).
For example this is my top 3 mirrors:

`vim /etc/pacman.d/mirrorlist`

```
##
## Arch Linux repository mirrorlist
## Filtered by mirror score from mirror status page
## Generated on 2019-03-01
##

## Singapore
Server = http://mirror.0x.sg/archlinux/$repo/os/$arch
## Vietnam
Server = http://f.archlinuxvn.org/archlinux/$repo/os/$arch
## Netherlands
Server = http://archlinux.mirror.pcextreme.nl/$repo/os/$arch
```

## Install the base and base-devel packages

Use the **pacstrap** script:

`pacstrap /mnt base linux linux-firmware base-devel`

## Generate an fstab file

Use `-U` or `-L` to define by UUID or labels:

`genfstab -U /mnt >> /mnt/etc/fstab`

## Chroot

Change root to the new system:

`arch-chroot /mnt`

## Install optional packages

`pacman -S efibootmgr intel-ucode`

`pacman -S networkmanager`

`pacman -S git neovim zsh`

## Create swap file

As an alternative to creating an entire swap partition, a swap file offers the ability to vary its size on-the-fly, and is more easily removed altogether.

Create a 32 GB (depend on your RAM) swap file:

`fallocate -l 32G /swapfile`

Set the right permissions:

`chmod 600 /swapfile`

format it to swap:

`mkswap /swapfile`

Activate the swap file:

`swapon /swapfile`

Edit fstab at `/etc/fstab` to add an entry for the swap file:

`nvim /etc/fstab`

```
/swapfile none swap defaults 0 0
```

## Configure time zone

Set your time zone by region:

`ln -sf /usr/share/zoneinfo/Asia/Ho_Chi_Minh /etc/localtime`

Generate `/etc/adjtime`:

`hwclock --systohc`

## Configure locale

Uncomment `en_US.UTF-8 UTF-8` in `/etc/locale.gen`, then generate it:

`sed -i 's/^#en_US.UTF-8 UTF-8/en_US.UTF-8 UTF-8/g' /etc/locale.gen && locale-gen`

Set LANG variable in `/etc/locale.conf`:

`echo 'LANG=en_US.UTF-8' > /etc/locale.conf`

## Change host name

Create hostname file at `/etc/hostname` contain the host name, for example:

`echo 'Precision' > /etc/hostname`

## Set your root password

`passwd`

Enter your password then confirm it.

## Install boot loader

There are many ways to boot but I prefer these method

### Boot with EFISTUB:

Kernel can be booted directly by a UEFI motherboard

`efibootmgr -d /dev/nvme0n1 -p 1 -c -L "Arch Linux" -l /vmlinuz-linux -u 'initrd=/intel-ucode.img initrd=/initramfs-linux.img root=/dev/nvme0n1p2 rw quiet' -v`

### Boot with systemd-boot

Install `systemd-boot` to the `/boot` partition:

`bootctl --path=/boot install`

Edit `systemd-boot` options:

`nvim /boot/loader/loader.conf`

```
default arch
timeout 0
editor  0
```

Add Arch boot entry:

`nvim /boot/loader/entries/arch.conf`

```
title   Arch Linux
linux   /vmlinuz-linux
initrd  /intel-ucode.img
initrd  /initramfs-linux.img
options root=/dev/nvme0n1p2 rw quiet
```

## Enable network services

`systemctl enable NetworkManager`

## Add new user

Add a new user named `khuedoan`:

`useradd -m -G wheel -s /bin/zsh -c "Khue Doan" khuedoan`

Protect the newly created user `khuedoan` with a password:

`passwd khuedoan`

Establish `nvim` as the **visudo** editor:

`EDITOR=nvim visudo`

Then uncomment `%wheel ALL=(ALL) ALL` to allow members of group `wheel` sudo access, uncomment `Defaults targetpw` and change it to `Defaults rootpw` to ask for the root password instead of the user password (then change the comment beside it accordingly).

## Reboot

Exit the chroot environment by typing:

`exit`

Optionally manually unmount all the partitions with:

`umount -R /mnt`

Restart the machine:

`reboot`

## Login

Login with your user account after the machine has rebooted.

## Install dotfiles

Check out my [dotfiles](https://github.com/khuedoan98/dotfiles) repo for more details
