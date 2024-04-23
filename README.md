<h1 align="center">Arch Linux Installation Guide</h1>

<p align="center">
  <img src="https://raw.githubusercontent.com/archlinux/.github/6b33d3e9e2a522f73c8a65dc68d504e5deff6ad2/profile/archlinux-logo-dark-scalable.svg" alt="Arch Linux logo">
</p>

<p align="justify">
Welcome to the Arch Linux Installation Guide! This guide will walk you through the process of installing Arch Linux on your system. Arch Linux is a lightweight and flexible Linux distribution, allowing you to customize your system according to your preferences.
<p align="justify">
This guide assumes you have basic knowledge of partitioning, file systems, and the command line. If you're new to Linux or unsure about any step, don't worry, we'll provide explanations along the way.
</p>
<p align="justify">
Let's dive into the installation process!
</p>
</br>

<div align="center">

[![Arch Linux Installation Guide](https://img.shields.io/badge/Official%20Guide-gray?style=flatsquare&logo=arch-linux)](https://wiki.archlinux.org/title/installation_guide)
[![Arch Linux General Recommendations](https://img.shields.io/badge/Post--installation-gray?style=flatsquare&logo=arch-linux)](https://wiki.archlinux.org/title/General_recommendations)
[![Arch Linux ISO Download](https://img.shields.io/badge/ISO%20Download-gray?style=flatsquare&logo=arch-linux)](https://archlinux.org/download/)

</div>

## Table of Contents

1. [Preliminary Checks](#preliminary-checks)
2. [Partitioning](#partitioning)
3. [Formatting Partitions and Mounting Drives](#formatting-partitions-and-mounting-drives)
4. [Create Partition Table](#create-partition-table)
5. [Install Kernel and Necessary Packages](#install-kernel-and-necessary-packages)
6. [Access Mount with chroot](#access-mount-with-chroot)
7. [Set Timezone](#set-timezone)
8. [Language and Keyboard Configuration](#language-and-keyboard-configuration)
9. [Check Kernel Installation](#check-kernel-installation)
10. [User Assignment](#user-assignment)
11. [Grub Configuration](#grub-configuration)
12. [Host Configuration](#host-configuration)
13. [Add Multilib Repo for x32 Libraries](#add-multilib-repo-for-x32-libraries)
14. [Reboot](#reboot)
15. [Enabling Important Services](#enabling-important-services)
16. [Add AUR Repository for yay](#add-aur-repository-for-yay)
17. [Installation of open-source AMD graphic drivers](#installation-of-open-source-amd-graphic-drivers)
18. [Installation of a desktop environment](#installation-of-a-desktop-environment)
19. [Other Interesting Packages](#other-interesting-packages)
20. [How to take snapshots in a btrfs system](#how-to-take-snapshots-in-a-btrfs-system)

</br>

## Preliminary Checks

Change keyboard layout to Spanish:

```shell
loadkeys es
```

Check network status:

```shell
ping -c 1 google.es
```

Check hardware clock status:

```shell
timedatectl status
```

If incorrect, update it:

```shell
timedatectl set-ntp true
```

## Partitioning

List available disks:

```shell
lsblk
```

Open the partitioning program for GPT disks:

```shell
cgdisk /dev/nvme0n1
```

Replace `nvme0n1` with the disk you are partitioning.

Partitioning scheme I use:

| #   | Partition           | Description     | Hex Code |
| --- | ------------------- | --------------- | -------- |
| 1   | 512MB EFI partition | /boot partition | ef00     |
| 2   | 100% size partition | /root partition | 8300     |

Then write with `write` and exit cgdisk with `quit`.

## Formatting Partitions and Mounting Drives

Run `lsblk -l` to verify everything is defined correctly.

<details>
<summary><big>ext4 Format:</big></summary>

First, format the partitions:

```shell
mkfs.vfat -F 32 /dev/nvme0n1p5
mkfs.ext4 /dev/nvme0n1p6
```

Replace `nvme0n1p5` with your EFI partition.
Replace `nvme0n1p6` with your root partition.

Then mount the drives:

```shell
mount /dev/nvme0n1p6 /mnt
mkdir /mnt/boot
mount /dev/nvme0n1p5 /mnt/boot
```

Run `lsblk` to verify everything is mounted correctly.

</details>
</br>
<details>
<summary><big>btrfs Format:</big></summary>

First, format the partitions:

```shell
mkfs.vfat -F 32 /dev/nvme0n1p5
mkfs.btrfs /dev/nvme0n1p6
```

Replace `nvme0n1p5` with your EFI partition.
Replace `nvme0n1p6` with your root partition.

Then mount the drives:

```shell
mount /dev/nvme0n1p6 /mnt/
btrfs su cr /mnt/@
mount -o compress=xstd,subvol=@ /dev/nvme0n1p6 /mnt
btrfs su cr /mnt/@home
btrfs su cr /mnt/@pkg
btrfs su cr /mnt/@log
btrfs su cr /mnt/@snapshots
umount /mnt
mkdir /mnt
mkdir -p /mnt/home
mkdir -p /mnt/var/cache/pacman/pkg
mkdir -p /mnt/var/log
mkdir -p /mnt/.snapshots
mount -o compress=xstd,subvol=@home /dev/nvme0n1p6 /mnt/home
mount -o compress=xstd,subvol=@pkg /dev/nvme0n1p6 /mnt/var/cache/pacman/pkg
mount -o compress=xstd,subvol=@log /dev/nvme0n1p6 /mnt/var/log
mount -o compress=xstd,subvol=@snapshots /dev/nvme0n1p6 /mnt/.snapshots
lsblk
mkdir -p /mnt/boot
mount /dev/nvme0n1p5 /mnt/boot
lsblk
```

</details>
</br>

Now mount the EFI partition to /boot:

```shell
mkdir -p /mnt/boot
mount /dev/nvme0n1p5 /mnt/boot
```

Run `lsblk` to verify everything is mounted correctly.

## Create Partition Table

```shell
genfstab -U /mnt
genfstab -U /mnt > /mnt/etc/fstab
```

## Install Kernel and Necessary Packages

```shell
pacstrap /mnt base base-devel linux=6.7.9.arch1-1 linux-firmware networkmanager grub efibootmgr os-prober pulseaudio man git nano vim neofetch
```

## Access Mount with chroot

```shell
arch-chroot /mnt
```

## Set Timezone

```shell
ln -sf /usr/share/zoneinfo/Europe/Madrid /etc/localtime
```

Sync system time with hardware clock in UTC:

```shell
hwclock --systohc --utc
```

## Language and Keyboard Configuration

```shell
nano /etc/locale.gen
```

- `ctrl + w` to filter by `es_ES`.
- Uncomment `es_ES.UTF-8 UTF-8` and save.

```shell
locale-gen
```

```shell
echo LANG=es_ES.UTF-8 > /etc/locale.conf
export LANG=es_ES.UTF-8
```

Now write and save `KEYMAP=es` to:

```shell
nano /etc/vconsole.conf
```

## Check Kernel Installation

```shell
mkinitcpio -p linux
```

## User Assignment

```shell
passwd
useradd -m nacho
passwd nacho
usermod -aG wheel nacho
```

Now execute the following command and you should see `wheel nacho` in the output:

```shell
groups nacho
```

Install sudo if not already installed:

```shell
pacman -S sudo
```

Now modify the sudoers file and uncomment `%wheel ALL=(ALL:ALL) ALL`:

```shell
nano /etc/sudoers
```

### Checks:

Check if we switch to the user nacho:

```shell
su nacho
```

Check if we belong to the wheel group:

```shell
id
```

Try if `sudo su` turns you into root after entering your password.

## Grub Configuration

For a 64-bit, UEFI, and GPT installation, it is necessary to run this command for your firmware to detect Grub:

```shell
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=Grub --recheck
```

The location of `--efi-directory=/boot` will be where our ESP is mounted. In this case, it was mounted at `/boot`.

To detect these changes and configure Grub, we need to run:

```shell
grub-mkconfig -o /boot/grub/grub.cfg
```

If we want Grub to detect another operating system like Windows, we have to do the following.

First, we need to modify the grub file:

```shell
sudo nano /etc/default/grub
```

Set and uncomment the following parameters:

- GRUB_TIMEOUT=8
- GRUB_DISABLE_OS_PROBER=false

Now we will create a mount point for the Windows EFI System Partition and once mounted there, we will copy the contents of Microsoft into our Grub EFI:

```shell
mkdir /mnt/win
mount /dev/nvme0n1p1 /mnt/win/
cp -r /mnt/win/EFI/Microsoft /boot/EFI
```

Run grub-mkconfig again to apply the changes:

```shell
grub-mkconfig -o /boot/grub/grub.cfg
```

## Host Configuration

Assign a name to the machine.

```shell
echo ArchLinux > /etc/hostname
```

Modify our hosts file with the name of the machine we assigned.

```shell
sudo nano /etc/hosts
```

Add:

```
127.0.0.1	localhost
::1			localhost
127.0.0.1	ArchLinux.localhost ArchLinux
```

## Add Multilib Repo for x32 Libraries

Uncomment `multilib` and NOT `multilib-testing`.

```shell
nano /etc/pacman.conf
```

Update pacman:

```shell
pacman -Syy
```

## Reboot

```shell
exit
umount -R /mnt
reboot now
```

## Checks

- Check if Grub mounted correctly and if we can access our system without a graphical interface.
- Type your username "nacho" and the password created earlier.
- Check if our keyboard language is correct by typing a dash `-`.
- Check Internet connection with `ping -c 1 google.es`.

If you don't have Internet, resolve it in the next step.

## Enabling Important Services

Enable the NetworkManager service to have Internet connection.

```shell
sudo su
systemctl start NetworkManager.service
systemctl enable NetworkManager
```

Check if you have Internet with `ping -c 1 google.es`.

Enable audio service:

```shell
systemctl start pulseaudio
systemctl --user enable pulseaudio
```

## Add AUR Repository for yay

First, make sure you are logged in as the user nacho, not as root.

Create a directory repos where you can clone the yay repo. From there, download the yay official repo or paru as you prefer and makepkg.

```shell
mkdir repos
cd !$
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si
yay
```

## Installation of open source AMD graphic drivers

AMDGPU DRIVER:

This is the main driver we should install.

```shell
sudo pacman -S xf86-video-amdgpu mesa
```

VULKAN:

```shell
sudo pacman -S vulkan-radeon lib32-vulkan-radeon
```

OPENCL:

```shell
sudo pacman -S opencl-mesa
```

VDPU:

```shell
sudo pacman -S libva-mesa-driver lib32-libva-mesa-driver mesa-vdpau lib32-mesa-vdpau
```

## Installation of a desktop environment

In this case, I will install Gnome with Wayland and the default display manager gdm, but you can choose any other like Hyprland and sdmm for example.

```shell
sudo pacman -S gnome gdm
sudo systemctl enable gdm.service
```

## Other Interesting Packages

Here are some interesting packages you can install if you want.

```shell
sudo pacman -S kitty dolphin dunst unrar unzip firefox htop feh gnome-screenshot gimp
```

## How to take snapshots in a btrfs system

### Create a snapshot:

We will create it for the entire root system / in our snapshots directory.

```shell
sudo btrfs subvolume snapshot / /.snapshots/<nombre del snapshot>
```

Check if it is created.

### Restore from a snapshot:

Mount our root partition again, in my case it is on the partition `nvme0n1p6` in /mnt.

```shell
mount /dev/nvme0n1p6 /mnt
```

`nvme0n1p6` is where my btrfs root system is located.

Check if it is mounted correctly:

```shell
ls -l /mnt/
```

Move our root volume @ to a volume @latest_root. Then move our created snapshot to the root volume @. Finally, reboot.

```shell
mv /mnt/@ /mnt/@latest_root
mv /mnt/@.snapshots/Base /mnt/@
reboot
```

### Clean snapshots:

This step is important to delete the old root and not occupy space.

Mount again our root system in my case it is on the partition `nvme0n1p6` in /mnt.

```shell
mount /dev/nvme0n1p6 /mnt/
```

Now we have two options to delete the old volume with all the old root content that we will no longer use:

1. With btrfs:

   ```shell
   btrfs subvolume delete /mnt/@latest_root
   ```

2. With rm:

   ```shell
   sudo rm -r /mnt/@latest_root
   ```

Finally, unmount /mnt.

```shell
umount /mnt
```
