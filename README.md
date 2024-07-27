# **Setup Guide**

# WiFi

Enter iwctl

```
$ iwctl
```

When inside, check for the name of your wireless devices.

```
device list
```

If your device name is wlan0, connect using the following command

```
station wlan0 connect <SSID>
```

Make sure to enter in your password

exit when complete

```
exit
```

# SSH

Enable sshd (should be done by default)

```
$ systemctl enable sshd
```

set a password for the current user

```
$ passwd
```

# Write random data

List blocks. In my case, my drives are nvme0n1 and nvme1n1. Your's might be the
same, or the might be an sdx drive, such as sda or sdb.

```
$ lsblk
```

Write random data into your drive. 

```
$ dd if=/dev/urandom of=/dev/nvme0n1 status=progress bs=4096
```

# Partitioning Data

Get the names of the blocks

```
$ lsblk
```
## GParted option

If starting with ubuntu based distro you can optionally use Gparted for partitioning as it is very easy to use. 
Start GParted from the Applications menu.


Format the entire disk with a GPT partition table. This will delete all existing data and partitions on the disk.
GParted > Main Menu > Device > Create Partition Table…

EFI System Partition — Create a 512 MB partition formatted as FAT16. 
Right-click and select Manage Flags. Set the esp and boot flags.

Boot Partitions — Create 2-4 GB boot partitions formatted as EXT4 for each distribution that you want to install. It’s better to create 5–10 boot partitions at the beginning, so that you don’t need to create additional partitions in future.  If you need to have many kernels installed, then increase the size of each partition. If you have the habit of uninstalling older kernels periodically then 2 GB is more than enough.

System Partition — Create one big partition to occupy remaining disk space minus any SWAP partition. Leave it as unformatted. We will format it in the next section.

Swap Partition — You can create a swap partition if you wish to. You usually don’t need one if you have atleast 8 GB of RAM. You may see some benefit from having a swap partition if you run applications that use a lot of RAM.
If over 8 GB RAM 1.5x RAM is reccommended. 


## gedit option

```
$ gdisk /dev/nvme0n1
```

Inside of gdisk, you can print the table using the `p` command.

To create a new partition use the `n` command. The below table shows 
the disk setup I have for my primary drive

| partition | first sector | last sector | code | TYPE    |  NVME part      | Partition label    |
|-----------|--------------|-------------|------|---------|-----------------|--------------------|
| 1         | default      | +512M       | ef00 | EFI     |  /dev/nvme0n1p1 | fwork_efi          |
| 2         | default      | +2/4G       | 8300 | boot    |  /dev/nvme0n1p2 | fwork_boot_mint    |
| 3         | default      | +2/4G       | 8300 | boot    |  /dev/nvme0n1p3 | fwork_boot_arch    |
| 4         | default      | +2/4G       | 8300 | boot    |  /dev/nvme0n1p4 | fwork_boot_suse    |
| 5         | default      | +2/4G       | 8300 | boot    |  /dev/nvme0n1p5 | fwork_boot_        |
| 6         | default      | +2/4G       | 8300 | boot    |  /dev/nvme0n1p6 | fwork_boot_rescue  |
| 7         | default      | -24G        | 8309 | Sysytem |  /dev/nvme0n1p7 | fwork_system       |
| 8         | default      | default     | 8200 | SWAP    |  /dev/nvme0n1p8 | SWAP               |



Labels can be changed in gedit with c option

Labelling — Label partitions as shown in screenshot. Set both LABEL and PARTLABEL to same value. This makes it easy to identify the partitions and we will be using it later to create scripts for some common tasks. Keep a one-word lowercase name for each distribution that you wish to install (like “xenial”, “ubuntu”, “mint”, etc) and use it everywhere while labelling partitions. This will make it very easy to manage the systems once we are done installing them.

> Note: PARTLABEL is displayed as Name in GParted


# LUKS Setup

Run the following commands to format system partition. Copy-paste the entire block of commands in a terminal window and hit Enter.

sudo cryptsetup luksFormat /dev/disk/by-partlabel/fwork_system 
sudo cryptsetup luksOpen /dev/disk/by-partlabel/fwork_system fwork_system   //This will unlock the LUKS partition and map it to /dev/mapper/fwork_system


# Filesystems



Format boot partitions
mkfs.ext4 /dev/nvme0n1p2
mkfs.ext4 /dev/nvme0n1p3
mkfs.ext4 /dev/nvme0n1p4
mkfs.ext4 /dev/nvme0n1p5
mkfs.ext4 /dev/nvme0n1p6

FAT32 on EFI partiton

```
$ mkfs.fat -F32 /dev/nvme0n1p1 
```

EXT4 on Boot partiton

```
$ mkfs.ext4 /dev/nvme0n1p2
$ mkfs.ext4 /dev/nvme0n1p3
$ mkfs.ext4 /dev/nvme0n1p4
$ mkfs.ext4 /dev/nvme0n1p5
$ mkfs.ext4 /dev/nvme0n1p6
```

BTRFS on root

```
$ mkfs.btrfs -f /dev/mapper/fwork_system
```


# Installing Linux Mint 22

Boot from the Linux Mint 22 Live USB
Open a terminal and copy-paste the following commands:
```
$ sudo cryptsetup luksOpen /dev/disk/by-partlabel/fwork_system fwork_system  //This will unlock the LUKS partition and map it to /dev/mapper/fwork_system

```

Start the Linux Mint installer and select custom partitioning. 
Select /dev/mapper/fwork_system as the root device (/) 
and /dev/nvme0n1p2 as the boot device (/boot).

> Note: Make sure that the partitions are NOT marked for formatting. Ignore any warnings displayed by installer for formatting the partitions.
  
> Note: If the installer prompts for unmounting partitions, allow it to unmount. This does not lock the LUKS device that we unlocked in previous step.

> Note: The partition selected for mounting /boot will depend on the distribution you are installing. I had labelled the /dev/nvme0n1p2 partition as fwork_boot_mint so I will use the same one while installing Linux Mint.

Select /dev/nvme0n1 for installing the GRUB bootloader.

Once the installation completes, do not reboot. 
The installed system does not know that it is installed on an encrypted partition, and will fail to boot as a result. 
We need to make some changes to the installed system to make it boot successfully.

Mount system partition under /mnt/btrfs

```
$ sudo mkdir -p /mnt/btrfs
$ sudo mount /dev/mapper/fwork_system /mnt/btrfs
$ cd /mnt/btrfs
```
Rename subvolumes — Rename \@ and \@home to \@mint and \@mint_home
```
# mv -vf @     @mint
# mv -vf @home @mint_home
$ cd /mnt/btrfs
```



Mount system devices — Mount the devices for /, /home, /boot and /boot/efi under /mnt/mint.

```
/////////////////
distname=mint && \
sudo mkdir -p /mnt/${distname} && \
sudo mount /dev/mapper/fwork_system -o subvol=@${distname}      /mnt/${distname}  && \
sudo mount /dev/mapper/fwork_system -o subvol=@${distname}_home /mnt/${distname}/home && \
mp=/mnt/${distname}/boot     && sudo mkdir -p ${mp} && sudo mount /dev/disk/by-partlabel/fwork_boot_${distname} ${mp} && \
mp=/mnt/${distname}/boot/efi && sudo mkdir -p ${mp} && sudo mount /dev/disk/by-partlabel/fwork_efi ${mp}
////////////////

# mkdir -p /mnt/mint
# mount /dev/mapper/fwork_system -o subvol=@mint      /mnt/mint 
# mount /dev/mapper/fwork_system -o subvol=@mint_home /mnt/mint/home 
# mkdir -p /mnt/mint/boot
# mount /dev/disk/by-partlabel/fwork_boot_mint /mnt/mint/boot
# mkdir -p /mnt/mint/boot/efi
# mount /dev/disk/by-partlabel/fwork_efi /mnt/mint/boot/efi
```

Chroot to the installed system.

```
//https://github.com/teejee2008/groot/releases/latest
# cd /mnt/mint
# groot 

//CHroot 
#
#
```

Update Initramfs Settings — Run the following block of commands. It will update the initramfs settings for booting from an encrypted root.
****Run in Chrooted terminal

```
# echo 'CRYPTROOT=target=fwork_system,source=/dev/disk/by-partlabel/fwork_system' >> /etc/initramfs-tools/conf.d/cryptroot && \
# echo 'export CRYPTSETUP=y' > /etc/initramfs-tools/conf.d/cryptsetup && \
# echo "CRYPTSETUP=y" >> /usr/share/initramfs-tools/conf-hooks.d/cryptsetup && \
# echo "CRYPTSETUP=y" >> /usr/share/initramfs-tools/conf-hooks.d/forcecryptsetup
```

Update GRUB Settings — Open the grub configuration file and modify it. If you started the session using groot then you will be able to run GUI editors such as gedit, mousepad, etc instead of terminal-based tools like nano, vi, etc.
****Run in Chrooted terminal
```
# nano /etc/default/grub
```
Change the line for GRUB_CMDLINE_LINUX to the following:

```
GRUB_CMDLINE_LINUX="cryptopts=target=fwork_system,source=/dev/disk/by-partlabel/fwork_system rootflags=subvol=@mint"
```
Update fstab file— Edit /etc/fstab file and change the subvolume name.
```
# nano /etc/fstab
```

```
# <file system> <mount point> <type> <options> <dump> <pass>

/dev/mapper/fwork_system			          /		      btrfs	defaults,subvol=@mint,noatime		    0	1
/dev/mapper/fwork_system			          /home		  btrfs	defaults,subvol=@mint_home,noatime	0	2
/dev/disk/by-partlabel/fwork_boot_mint	/boot		  ext4	defaults,noatime			              0	2
/dev/disk/by-partlabel/fwork_efi		    /boot/efi	vfat	defaults                            0 2
```
> Note: I replaced the UUID device names with /dev/disk/by-partlabel/<partlabel> syntax. It’s easier to remember and you are less likely to make mistakes.
  
> Note: Columns in fstab file can be separated by spaces, tabs or a combination of both. Line up the columns so that they are easier to read.

##Create crypttab file—Create a new /etc/crypttab file as you are not likely to have one.
```
# nano /etc/crypttab
```

```
# <target name> <source device> <key file> <options>

fwork_system /dev/disk/by-partlabel/fwork_system none luks,discard
```

##Rebuild Initramfs — 
We need to rebuild the initramfs file for changes we have done above. 
Run the following commands:
```
# nupdate-initramfs -u -k all
# update-grub
# grub-install --target=x86_64-efi --efi-directory=/boot/efi /dev/nvme0n1
```
> Note: This will regenerate the initramfs, update the grub menu, and re-install GRUB to /dev/sda

Exit the chroot session by typing exit in the terminal

Reboot the laptop and boot into the new system. 
You will be prompted to unlock the system partition when it boots. 
This indicates that the previous steps were successful


## Install the Boot Manager

Install the rEFInd boot manager from PPA or by downloading the DEB file.
```
sudo apt-add-repository ppa:rodsmith/refind
sudo apt-get update
sudo apt-get install refind
```
  rEFInd installs itself to folder /boot/EFI/refined/ and updates the EFI firmware settings to make itself the default boot manager.

Generate a grub image file on boot partition. This generates a file named core.efi in /boot. 
This will be detected by rEFInd and used for booting the system. 
Once this file has been created, we no longer need the GRUB version installed on /dev/sda for booting the system.
```
# cd /boot
# grub-mkimage -o core.efi --format=x86_64-efi '--prefix=(hd0,gpt2)/grub' ext2 part_gpt
```
> Note: Change the GPT partition number --prefix=(hd0,gpt2) to match the boot partition number. In this case, the boot partition is /dev/sda2 which is GPT partition 2.
  
> Note: This file must be generated after booting to the installed system using GRUB. If you generate this from a chrooted session or from another system, then you will get an error “error: symbol table not found” while booting the system. This error is harmless and you can continue booting by pressing Enter.

##Configure the Boot Manager
Reboot the system.
You will be greeted by the rEFInd boot manager.
rEFInd displays boot icons for the core.efi files we created above.
It also displays boot icons for every kernel it finds on the boot partitions, and also for the grubx64.efi files that Ubuntu installs to /boot/efi/efi/ubuntu. We need to hide these extra entries. Select the icons one by one and hit Delete. We need to do this every time a new kernel is installed to the boot partition while installing system updates.



# Installing Arch

Create subvolume for arch installation

```
$ sudo mkdir -p /mnt/btrfs
$ sudo mount /dev/mapper/fwork_system /mnt/btrfs
$ btrfs subvolume create /mnt/btrfs/@arch
$ btrfs subvolume create /mnt/btrfs/@arch_home
```



BTRFS on home if exists

```
$ mkfs.btrfs -L home /dev/mapper/arch-home
```

Setup swap device

```
$ mkswap /dev/mapper/arch-swap
```

## Mounting

Mount swap

```
$ swapon /dev/mapper/arch-swap
$ swapon -a
```

Mount root 

```
$ mount /dev/mapper/arch-root /mnt
```

Create home and boot

```
$ mkdir -p /mnt/{home,boot}
```

Mount the boot partiton

```
$ mount /dev/nvme0n1p2 /mnt/boot
```

Mount the home partition if you have one, otherwise skip this

```
$ mount /dev/mapper/arch-home /mnt/home
```

Create the efi directory

```
$ mkdir /mnt/boot/efi
```

Mount the EFI directory

```
$ mount /dev/nvme0n1p1 /mnt/boot/efi
```

## Install arch

```
$ pacstrap -K /mnt base linux linux-firmware
```

With base-devel

```
$ pacstrap -K /mnt base base-devel linux linux-firmware
```

Load the file table

```
$ genfstab -U -p /mnt > /mnt/etc/fstab
```

chroot into your installation

```
$ arch-chroot /mnt /bin/bash
```

## Configuring

### Text Editor

Install a text editor

```
$ pacman -S neovim
```

```
$ pacman -S nano
```

### Decrypting volumes

Open up mkinitcpio.conf

```
$ nvim /etc/mkinitcpio.conf
```

add `encrypt` and `lvm2` into the hooks

```
HOOKS=(... block encrypt lvm2 filesystems fsck)
```

install lvm2

```
$ pacman -S lvm2
```

### Bootloader

Install grub and efibootmgr

```
$ pacman -S grub efibootmgr
```

Setup grub on efi partition

```
$ grub-install --efi-directory=/boot/efi
```

obtain your lvm partition device UUID

```
blkid /dev/nvme0n1p3
```

Copy this to your clipboard

```
$ nvim /etc/default/grub
```

Add in the following kernel parameters

```
root=/dev/mapper/arch-root cryptdevice=UUID=<uuid>:luks_lvm
```

### Keyfile

```
$ mkdir /secure
```

Root keyfile
```
$ dd if=/dev/random of=/secure/root_keyfile.bin bs=512 count=8
```

Home keyfile if home partition exists

```
$ dd if=/dev/random of=/secure/home_keyfile.bin bs=512 count=8
```

Change permissions on these

```
$ chmod 000 /secure/*
```

Add to partitions

```
$ cryptsetup luksAddKey /dev/nvme0n1p3 /secure/root_keyfile.bin
# skip below if using single disk
$ cryptsetup luksAddKey /dev/nvme1n1p1 /secure/home_keyfile.bin
```

```
$ nvim /etc/mkinitcpio.conf
```

```
FILES=(/secure/root_keyfile.bin)
```

### Home Partition Crypttab (Skip if single disk)

Get uuid of home partition

```
$ blkid /dev/nvme1n1p1
```

Open up the crypt table.
```
$ nvim /etc/crypttab
```

Add in the following line at the bottom of the table
```
arch-home      UUID=<uuid>    /secure/home_keyfile.bin
```

Reload linux

```
$ mkinitcpio -p linux
```

## Grub

Create grub config

```
$ grub-mkconfig -o /boot/grub/grub.cfg
$ grub-mkconfig -o /boot/efi/EFI/arch/grub.cfg
```

## System Configuration

### Timezone

```
ln -sf /usr/share/zoneinfo/America/Chicago /etc/localtime
```

### NTP

```
$ nvim /etc/systemd/timesyncd.conf
```

Add in the NTP servers

```
[Time]
NTP=0.arch.pool.ntp.org 1.arch.pool.ntp.org 2.arch.pool.ntp.org 3.arch.pool.ntp.org 
FallbackNTP=0.pool.ntp.org 1.pool.ntp.org
```

Enable timesyncd

```
# systemctl enable systemd-timesyncd.service
```

### Locale

```
$ nvim /etc/locale.gen
```

uncomment the UTF8 lang you want

```
en_US.UTF-8 UTF-8
```

```
$ locale-gen
```

```
$ nvim /etc/locale.conf
```

```
LANG=en_US.UTF-8
```


### hostname

enter it into your /etc/hostname file

```
$ nvim /etc/hostname
```

or 

```
$ echo "mymachine" > /etc/hostname
```

### Users

First secure the root user by setting a password

```
$ passwd
```

Then install the shell you want

```
$ pacman -S zsh
```

Add a new user as follows

```
$ useradd -m -G wheel -s /bin/zsh user
```

set the password on the user

```
$ passwd user
```

Add the wheel group to sudoers

```
$ EDITOR=nvim visudo
```

```
%wheel ALL=(ALL:ALL) ALL
```

### Network Connectivity

```
$ pacman -S networkmanager
$ systemctl enable NetworkManager
```

### Display Manager

```
$ pacman -S gnome
```

```
$ systemctl enable gdm
```


### Microcode

For AMD

```
$ pacman -S amd-ucode
```

For intel

```
$ pacman -S intel-ucode
```

```
$ grub-mkconfig -o /boot/grub/grub.cfg
$ grub-mkconfig -o /boot/efi/EFI/arch/grub.cfg
```


## Reboot

```
$ exit
$ umount -R /mnt
$ reboot now
```
