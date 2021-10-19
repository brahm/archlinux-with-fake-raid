![alt text][logo]

# Installation on Fake RAID (Intel Rapid Storage Technology)



- [Introduction](##introduction)
- [Pre-installation](##pre-installation)
- [Installation](##installation)



## Introduction



This guide walks you through the process of installing Arch Linux 2021.10.01 on a PC with Intel RTS "fake RAID 0" habilitated.

Intel Rapid Storage Technology, previously known as Intel Matrix RAID, is a feature included on several modern Intel chipsets. This firmware-based RAID (also known as “fake RAID”) differ from hardware RAID in that the array is ultimately managed by the operating system instead of a dedicated controller. The burden of processing is delegated to the host CPU. Additionally, firmware and software RAID do not benefit from features such as battery-backed write caching that are often included on dedicated controllers. As such, Intel RST is not an ideal option for users with critical reliability requirements — but it is a very reasonable choice for home users looking to achieve additional redundancy or performance.



## Pre-installation



### Acquire an installation image and verify signature

Visit the [Download](https://archlinux.org/download/) page and, depending on how you want to boot, acquire the ISO file or a netboot image, and the respective [GnuPG](https://wiki.archlinux.org/index.php/GnuPG) signature. Then [verify](https://wiki.archlinux.org/index.php/GnuPG#Verify_a_signature) it with:

```
$ gpg --keyserver-options auto-key-retrieve --verify archlinux-version-x86_64.iso.sig
```

Alternatively, from an existing Arch Linux installation run:

```
$ pacman-key -v archlinux-version-x86_64.iso.sig
```

### Prepare an installation medium

The installation image can be supplied to the target machine via a [USB flash drive](https://wiki.archlinux.org/index.php/USB_flash_installation_medium), an [optical disc](https://wiki.archlinux.org/index.php/Optical_disc_drive#Burning) or a network with [PXE](https://wiki.archlinux.org/index.php/PXE): follow the appropriate article to prepare yourself an installation medium from the chosen image.

### Boot the live environment

1. Ethernet—plug in the cable.

2. Point the current boot device to the one which has the Arch Linux installation medium. Typically it is achieved by pressing a key during the POST phase, as indicated on the splash screen. Refer to your motherboard's manual for details.

3. When the installation medium's boot loader menu appears, select Arch Linux install medium and press Enter to enter the installation environment.

4. You will be logged in on the first [virtual console](https://en.wikipedia.org/wiki/Virtual_console) as the root user, and presented with a [Zsh](https://wiki.archlinux.org/index.php/Zsh) shell prompt.

5. Verify the connection with [ping](https://en.wikipedia.org/wiki/ping_(networking_utility)):

   ```
   # ping archlinux.org
   ```

> **Note**: Arch Linux installation images do not support Secure Boot. You will need to disable Secure Boot to boot the installation medium. If desired, Secure Boot can be set up after completing the installation.

### Setting up the environment 

#### Set the keyboard layout

```
# loadkeys us-acentos
```

#### Set the default editor

```
# export EDITOR=nano
```


#### Update the system clock

```
# timedatectl set-ntp true
```

### Drive Preparation

#### Creating the RAID

The following example utilizes mdadm to create a 2-device RAID-0 array named /dev/md/data comprised of /dev/sda and /dev/sdb. You may need to adjust the parameters accordingly to suit the configuration of your system.

Use lsblk to display your disk tree:

```
# lsblk
NAME  MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINTS
sda     8:0    0 238.5G  0 disk  
sdb     8:16   0 238.5G  0 disk

```

Because Intel RST utilizes external metadata, we must first create a container for our array. Create a container named /dev/md/imsm on the target devices using the imsm (Intel Matrix Storage Manager) metadata format.

```
# mdadm -C /dev/md/imsm --raid-devices=2 --metadata=imsm /dev/sd[ab]
Continue creating array? y
mdadm: container /dev/md/imsm prepared.
```

Create the array named /dev/md/data in the /dev/md/imsm container.

```
# mdadm -C /dev/md/data --raid-devices=2 --level=1 /dev/md/imsm
mdadm: array /dev/md/data started.

# lsblk
NAME        MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINTS
sda           8:0    0 238.5G  0 disk  
├─md126       9:126  0 476.9G  0 raid0 
└─md127       9:127  0     0B  0 md    
sdb           8:16   0 238.5G  0 disk  
├─md126       9:126  0 476.9G  0 raid0 
└─md127       9:127  0     0B  0 md 
```

Use cfdisk to partition the disk using the following layout: 

```
# cfdisk /dev/md126
```

| Size | Partition    | Partition Code       | File System |
| ---- | ------------ | -------------------- | ----------- |
| 512M | /dev/md126p1 | EFI system partition | FAT32       |
| *    | /dev/md126p2 | Linux File System    | XFS         |

> (*) use the remaining disk space.

#### Formatting the drives

```
# mkfs.vfat -F32 /dev/md126p1
# mkfs.xfs -f /dev/md126p2 
```

#### Mounting drives for install

```
# mount /dev/md126p2 /mnt
# mkdir /mnt/boot
# mount /dev/md126p1 /mnt/boot
# lsblk
NAME        MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINTS
sda           8:0    0 238.5G  0 disk  
├─md126       9:126  0 476.9G  0 raid0 
│ ├─md126p1 259:0    0   512M  0 part  /boot
│ └─md126p2 259:1    0 476.4G  0 part  /
└─md127       9:127  0     0B  0 md    
sdb           8:16   0 238.5G  0 disk  
├─md126       9:126  0 476.9G  0 raid0 
│ ├─md126p1 259:0    0   512M  0 part  /boot
│ └─md126p2 259:1    0 476.4G  0 part  /
└─md127       9:127  0     0B  0 md 
```

#### Find the best mirror list for downloading Arch Linux

```
# pacman -Sy --noconfirm reflector
# reflector --verbose --latest 100 --protocol https --threads 8 --sort rate --save /etc/pacman.d/mirrorlist
# cat /etc/pacman.d/mirrorlist
```

#### Update the keyring

```
# echo "keyserver hkp://keyserver.ubuntu.com" >> /mnt/etc/pacman.d/gnupg/gpg.conf
# pacman -Syyu
# pacman-key --init
# pacman-key --populate archlinux
# pacman-key --refresh-keys
```

## Installation



### Essential packages

Use the [pacstrap](https://man.archlinux.org/man/pacstrap.8) script to install the base package, Linux kernel and firmware for common hardware:

```
# pacstrap /mnt base linux linux-firmware linux-headers mdadm intel-ucode dosfstools xfsprogs sudo vim nano wget git zsh libnewt --noconfirm --needed
```

#### Fstab

Generating fstab for the drives:

```
# genfstab -U -p /mnt >> /mnt/etc/fstab
```

Creating RAM Disk:

```
echo "tmpfs	/tmp	tmpfs	rw,nodev,nosuid,size=2G	0 0" >> /mnt/etc/fstab
```

Creating the swap file:

```
# dd if=/dev/zero of=/mnt/swapfile bs=1M count=34816 status=progress
# chmod 600 /mnt/swapfile
# mkswap /mnt/swapfile
# sudo swapon /mnt/swapfile
# echo "/swapfile    none    swap    defaults    0 0" >> /mnt/etc/fstab
```

#### Enter the new system

```
# arch-chroot /mnt
```

### Configure the system

#### Setting machine name

```
# echo mchinename > /etc/hostname
# echo "127.0.0.1 machinename" >> /etc/hosts
# echo "::1       machinename" >> /etc/hosts
# echo "127.0.1.1 machinename.localdomain	machinename" >> /etc/hosts
```

#### Setting font for vconsole

```
# echo "KEYMAP=us-acentos" >> /etc/vconsole.conf
```

#### Setting system wide language

```
# echo LANG=en_US.UTF-8 >> /etc/locale.conf
# echo LC_COLLATE=C >> /etc/locale.conf
# sed -i '/#en_US.UTF-8/s/^#//g' /etc/locale.gen
# locale-gen
```

#### Time zone

```
# ls -sf /usr/share/zoneinfo/America/Sao_Paulo /etc/localtime
# hwclock --systohc --utc
```

#### Setting environment variables

```
# echo EDITOR=nano visudo >> /etc/environment
# echo GTK_IM_MODULE=cedilla >> /etc/environment
# echo QT_IM_MODULE=cedilla >> /etc/environment
```

#### Set root password

```
# passwd
```

#### Create a user

```
# useradd  -m -g users -G wheel,sys,log,network,floppy,scanner,power,rfkill,users,video,storage,optical,lp,audio,adm,ftp,mail,git -s /bin/zsh MYUSERNAME
# passwd MYUSERNAME
```

#### Giving user wheel access

```
# sed -i '/%wheel ALL=(ALL) NOPASSWD: ALL'/s/^#//g /etc/sudoers
```

#### Installing the bootloader

```
# pacman -S --noconfig --needed grub efibootmgr
# mkdir /boot/grub
# grub-mkconfig -o /boot/grub/grub.cfg
# grub-install --target=x86_64-efi --efi-directory=/boot --recheck /dev/md126
```


### Extra packages

#### System utilities

```
# pacman -S --needed man-pages man-db bash-completion zsh-completions zsh-syntax-highlighting zsh-autosuggestions nano base-devel git pacman-contrib  usbutils lsof dmidecode dialog mc neofetch fwupd powertop gpm htop zip unzip unrar p7zip lzop lm_sensors imv bat fzf jq
```

#### Some network tools

```
# pacman -S --needed rsync traceroute bind-tools nmap speedtest-cli wavemon net-tools 
```

#### Install AUR package manager (yay)

```
# git clone https://aur.archlinux.org/yay.git
# cd yay
# makepkg -si
```

#### Install the missing modules

```
# yay -Syu aic94xx-firmware wd719x-firmware upd72020x-fw
```

#### Install and enable system services

```
# pacman -S --needed networkmanager openssh cronie xdg-user-dirs haveged samba bluez bluez-libs ntp
# systemctl enable NetworkManager
# systemctl enable NetworkManager-dispatcher.service
# systemctl enable sshd
# systemctl enable cronie
# systemctl enable haveged
# systemctl enable bluetooth
# systemctl enable ntpd
# systemctl enable man-db.timer
# systemctl enable fstrim.timer
# systemctl start fstrim.timer
# systemctl start fstrim.service
# systemctl disable systemd-resolved.service
# systemctl enable avahi-daemon.service
# systemctl start 


# sudo nano /etc/systemd/system/powertop.service
```

Uncomment the profile of choice, then start enable the service

```
# systemctl enable powertop
```

### Last Configs

```
# nano /etc/mkinitcpio.conf
```
Edit this line:
HOOKS="base udev resume autodetect modconf block filesystems keyboard fsck"
run
```
# mkinitcpio -p linux
```
```
# nano /etc/default/grub
```
GRUB_CMDLINE_LINUX_DEFAULT='quiet resume=/swapfile'

```
# nano /etc/default/grub
```
grub-mkconfig -o /boot/grub/grub.cfg

```
# umount -R /mnt
# reboot
```


[logo]: https://archlinux.org/static/logos/archlinux-logo-black-90dpi.0c696e9c0d84.png "Arch Linux"
