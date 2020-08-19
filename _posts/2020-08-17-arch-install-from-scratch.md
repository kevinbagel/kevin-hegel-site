---
title: Arch Installation From Scratch
layout: post
tags: [linux, arch, install]
---

Installing Arch linux manually may seem daunting at first but it is very rewarding when completed. Arch is a distribution of Linux that does not come with very many packages included and relies on the user to install and configure what they want. Here are some benefits of manually installing Arch:

1. Gain a better understanding of how Linux and related software works
2. Only install what you need on your system which conserves system resources and creates a system that runs much faster than other distributions that have a lot of unnecessary software
3. Greater security because you will be the one configuring and securing these packages rather than leaving them at their default settings and hard drive encryption will be enabled
4. Very great [Arch Wiki](https://wiki.archlinux.org/)

## Getting Ready Before Install

You will need to obtain the latest iso from [here](https://www.archlinux.org/download/) and prepare this to a bootable USB flash drive. If you need help on this there are some instructions [here](https://wiki.archlinux.org/index.php/Installation_guide#Pre-installation).

## Booting Into The Arch ISO

Once you have prepared the USB drive check these things first to make sure your computer will boot from the drive:

1. Make sure Secure Boot is turned off in your BIOS
2. Turn on UEFI mode in your BIOS
3. Set the BIOS clock to UTC time
4. Make sure USB booting is enabled and the flash drive is listed in the bootable devices
5. If RAID mode is turned on in your BIOS you may need to change it to AHCI mode so that your drive can be recognized. If you change this setting from RAID to AHCI and you have an OS installed already on the computer than you will not be able to boot the current OS after changing this setting and will need to reinstall it

Turn on your computer and boot to the usb drive and you should see a selection to boot to Arch. Select that and you will be brought to the Arch live environment.

## Environment Configuration

Load the United States keys
~~~
loadkeys us
~~~
Make sure the system is booted into UEFI mode by running this and making sure it outputs a list of files
~~~
ls /sys/firmware/efi/efivars
~~~
Set up timezone settings
~~~
timedatectl list-timezones
timedatectl set-timezone America/Los_Angeles
timedatectl status
~~~
Connect to your wireless network (assuming that you have an Intel wireless card)
~~~
iwctl
device list
station wlan0 scan
station wlan0 get-networks
station wlan0 connect ssid
exit
ping google.com
~~~
If pinging google worked than you are connected to the internet now.

Turn on NTP to make sure your system time is correct
~~~
timedatectl set-ntp true
timedatectl status
~~~
## Hard Drive Configuration

Run `parted -l` to list all your hard drives and determine which drive you would like to install Arch on. Once you have found it use that for the following commands. My drive is `/dev/nvme0n1` so if yours is `/dev/sda` then replace `/dev/nvme0n1` with `/dev/sda`
> The following commands will delete all data on the hard drive you choose so be careful that you select the drive you want to install Arch on

These commands will make a boot partition at the beginning of the drive that is formatted in FAT32 and sized 512MiB. It will then set the flags for the partition to be bootable. Next it makes the root partition which will take up the entire rest of the drive and set the partion to be ext4.
~~~
parted /dev/nvme0n1
mklabel gpt
mkpart "EFI system partition" fat32 1MiB 512MiB
set 1 esp on
set 1 boot on
mkpart "root partition" ext4 512MiB 100%
quit
~~~

## Set Up Disk Encryption
To set up disk encryption you will need to format the root partion which is the second partition on your disk. To find the path to both of your partitions run `ls -al /dev/disk/by-id/` and you will get an output like this
~~~
nvme-Micron_2200S_NVMe_512GB__1948251F33DB -> ../../nvme0n1
nvme-Micron_2200S_NVMe_512GB__1948251F33DB-part1 -> ../../nvme0n1p1
nvme-Micron_2200S_NVMe_512GB__1948251F33DB-part2 -> ../../nvme0n1p2
~~~
diskname-part1 -> ../../nvme0n1p1 means that the first partition which is the boot partition is /dev/nvme0n1p1 and diskname-part2 -> ../../nvme0n1p2 is the second root partition at /dev/nvme0n1p2. Replace the below paths with the paths to your two partitions above. The below command creates an encrypted luks device on the root partition which can be accessed at the path `/dev/mapper/cryptroot` when it is decrypted with the key and opened.

In command below just enter your passphrase which will be the password you use when the computer starts to decrypt the hard drive. Leave the other setting at default.
~~~
cryptsetup -y -v luksFormat /dev/nvme0n1p2
~~~
Use the passphrase you entered above to open the drive.
~~~
cryptsetup open /dev/nvme0n1p2 cryptroot
mkfs.ext4 /dev/mapper/cryptroot
mount /dev/mapper/cryptroot /mnt
~~~
We will be mounting our new Arch system under `/mnt` so we can access it and make configuration changes to our new install while still in the live install environment.
This command formats the boot partition and mounts it. Replace nvme0n1p1 with your first partition you found above.
~~~
mkfs.fat -F32 /dev/nvme0n1p1
mkdir /mnt/boot
mount /dev/nvme0n1p1 /mnt/boot
~~~

## Configuring the New Arch Install

We have mounted the new system into /mnt so that the root and boot partitions are mounted and need to install the base system and some useful packages to have, then chroot into the mounted root partition which will mimic being logged in to the new installation.

This command installs the base arch packages plus some additional packages into our new installation
~~~
pacstrap /mnt base linux linux-firmware vim sudo man-db man-pages texinfo iwd rsync curl intel-ucode grub efibootmgr
~~~
Now we need to generate the fstab file
~~~
genfstab -U /mnt >> /mnt/etc/fstab
~~~
Now we change root to enter the new installation and commands. When we are chrooted in to /mnt, all commands will be run on the new installation rather than in the Live ISO install environment
~~~
arch-chroot /mnt
~~~
### Locale, Time, and Language

Set these settings based on the system language, timezone, and keymap that you want. These are the ones I set for myself.
~~~
ln -sf /usr/share/zoneinfo/Region/City /etc/localtime
hwclock --systohc
vim /etc/locale.gen uncomment en_US.UTF-8 UTF-8
locale-gen
vim /etc/locale.conf
    LANG=en_US.UTF-8
vim /etc/vconsole.conf
    KEYMAP=us
~~~
### Setting Hostname

Replace myhostname with what you want the hostame of your computer to be
~~~
vim /etc/hostname
vim /etc/hosts
127.0.0.1	localhost
::1		localhost
127.0.1.1	myhostname.localdomain	myhostname
~~~
### Setup the Bootloader

In the file `/etc/mkinitcpio.conf` the MODULES section should have the modules section just needs to have what you need to load before the kernel loads. If you get a black screen or no graphics you may need to tweak these. See [here](https://wiki.archlinux.org/index.php/Kernel_mode_setting#Early_KMS_start) for more info.

The important part is that we add the HOOKS to allow for encryption. You need to add keyboard, keymap, and encrypt to the hooks section. Here is what mine looks like:
~~~
vim /etc/mkinitcpio.conf
    MODULES(i915 usbhid xhci_hcd)
    HOOKS=(base udev autodetect keyboard keymap consolefont modconf block encrypt filesystems fsck)
~~~
Next we will install grub which is the bootloader.
~~~
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB
~~~
Here we will need to edit the grub configuration to allow it to know that our root partition is encrypted. Replace *device-UUID* with the UUID of the encrypted block device which would be partition 2 as seen by running `ls -al /dev/disk/by-uuid/`

Here is my truncated output:
~~~
ls -al /dev/disk/by-id/
4fb1d9cf-1cdf-48de-dd51-9e73322203e2 -> ../../nvme0n1p2
EC29-F323 -> ../../nvme0n1p1
efbd83ab-9c4f-4883-9279-c59dd3d342fb -> ../../dm-0
~~~
So I would use the UUID of the device /dev/nvme0n1p2 which is the second partion that contains the encrypted root. Don't use the first partition or dm-0 here.
Now put this UUID below replacing *device-UUID* with the UUID:
~~~
vim /etc/default/grub    
    cryptdevice=UUID=device-UUID:cryptroot
    uncomment GRUB_ENABLECRYPTODISK=y
~~~
Now make the grub configuration so that theses changes are saved.
~~~
grub-mkconfig -o /boot/grub/grub.cfg
~~~
### Add Users and Restart

Change the root password
~~~
passwd
~~~
Add a new user named username, give members of the %wheel group sudo acceses, and add the user to the wheel group
~~~
useradd -m username
passwd username
usermod -aG wheel username
visudo
    under ## Uncomment to allow members of group wheel to execute any command
    uncomment %wheel ALL=(ALL) ALL
~~~
All done! Exit, unmount and shutdown.
~~~
exit
unmount -R /mnt
shutdown -h now
~~~

# Powering up your new System and Troubleshooting

Take out your usb drive and power up you system. You should see the grub bootloader screen then be asked for your password to unencrypt the drive. Type in the password and you should see a login prompt! Rojoice! You are now running Arch! Otherwise see below for troubleshooting.

## Troubleshooting

If the system didn't load or you didn't even get to the prompt to enter the password to decrypt your drive, then there is probably a problem with your bootloader configuration. To fix this we need to recreate the environment that we had during setup.

Boot to the usb install ISO that you used for setup

Use the password you set for the encrypted drive
~~~
cryptsetup open /dev/nvme0n1p2 cryptroot
mount /dev/mapper/cryptroot /mnt
~~~
Mount your boot partition again with the first partiton like you did in the setup
~~~
mount /dev/nvme0n1p1 /mnt/boot
~~~
Change root into the installed system again.
~~~
arch-chroot /mnt
~~~

Check that your grub configuration is correct. Make sure that the UUID of the root partition is correct and that you are using the UUID of the unencrypted block device and not the unencrypted luks cryptroot. If you make a change to `/etc/default/grub` make sure you run `grub-mkconfig -o /boot/grub/grub.cfg` to apply the changes when you are done.

If you got a black screen, you need to modify the `/etc/mkinitcpio.conf` file and change around the MODULES for your graphics card. See [here](https://wiki.archlinux.org/index.php/Kernel_mode_setting#Early_KMS_start)


