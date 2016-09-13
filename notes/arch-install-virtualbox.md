# Install Arch Linux

This guide is for installing Arch Linux as a VirtualBox guest using UEFI, grub bootloader, and Plasma (KDE 5) desktop

These instructions are collated from the following sources:
* [Arch Wiki Install Guide](https://wiki.archlinux.org/index.php/Installation_guide)
* [Muktware Arch Install Guide](http://www.muktware.io/arch-linux-guide-the-always-up-to-date-arch-linux-tutorial/)
* [Muktware Plasma Install Guide](http://www.muktware.io/how-to-install-kdes-plasma-5-on-arch-linux/)
* [Linux Scoop's YouTube video](https://www.youtube.com/watch?v=3TB6KYsUyj4) (mainly for the `reflector` stuff)

Other usefull guides:
* [Lifehacker Article](http://lifehacker.com/5680453/build-a-killer-customized-arch-linux-installation-and-learn-all-about-linux-in-the-process)
* [mattiaslundberg's Minimal Install Gist](https://gist.github.com/mattiaslundberg/8620837) (includes encryption)

Unless othewise noted, text sections enclosed in tags (e.g. `<hostname>`) should be replaced

## Pre-Install

* Configure VirtualBox to use UEFI:
    * Go into motherboard settings and check `Enable EFI`
* Start VM

## Check internet connection
```
dhcpcd
ping -c 3 www.google.com
```

## Setup Disk

#### Partition Disk
```
lsblk
cfdisk /dev/sda
```
* select `gpt`
* create 512M `EFI System` type partion as `sda1`
    * Can this be smaller if needed, 200M maybe?
* create larger `Linux Filesystem` type partion as `sda2`
* write and quit

#### Disk Usage After Basic Install

This is an example based on a 16GB Virtual HDD

| Partition | Used | Size | %age |
|-----------|------|------|------|
| sda1      | 42M  | 512M | 9%   |
| sda2      | 3.2G | 16G  | 22%  |

#### Format partitions
```
mkfs.fat -F32 /dev/sda1
mkfs.ext4 /dev/sda2
```

#### Mount main and EFI partitions
```
mount /dev/sda2 /mnt
mkdir /mnt/boot
mount /dev/sda1 /mnt/boot
```

## Prepare mirror list

#### Sync package databases
```
pacman -Sy
```

#### Sort mirror list
```
pacman -S reflector
reflector --verbose --sort rate --save /etc/pacman.d/mirrorlist
```
This sorts the mirrorlist in order of download rate (fastest first)

## Configure Arch

#### Install Arch base packages
```
pacstrap /mnt base base-devel
```

#### Configure FSTAB
```
genfstab -U /mnt >> /mnt/etc/fstab
cat /mnt/etc/fstab
```
`-U` means use UUIDs instead of partition labels

#### Change Arch Root
```
arch-chroot /mnt /bin/bash
```
Use the bash shell
Exit back to previous session using `exit` command

#### Configure Localizations
```
nano /etc/locale.gem
    # Uncomment 'en_US.UTF-8 UTF-8' line
locale-gen
echo LANG=en_US.UTF-8 > /etc/locale.conf
export LANG=en_US.UTF-8
```

#### Set System Time
Use `tzselect` to find correct timezone if necessary
```
ln -s /usr/share/zoneinfo/<region>/<zone> /etc/localtime
hwclock --systohc --utc
    # Sets hardware clock to UTC
```

#### Configure Network
```
echo <hostname> > /etc/hostname
systemctl enable dhcpcd
```

#### Advance Networking (not required)
* List interface devices `ip link`
* Make note of ethernet interface e.g. `enp0s3`
* Copy `netctl` profile
    `cp /etc/netctl/examples/ethernet-dhcp /etc/netctl/enp0s3-dhcp`
* Set internet interface above in profile
    `vim /etc/netctl/enp0s3-dhcp`

#### Set Root Password
```
passwd
```

## Setup Bootloader

#### Install Packages and Add GRUB Entry
```
packman -S grub efibootmgr os-prober
    # os-prober is for multi-boot
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=arch_grub
grub-mkconfig -o /boot/grub/grub.cfg
```

Note that this only works if the ESP (EFI System Partition) is mounted at `/boot` as per instructions above

#### Configure Bootloader for VirtualBox
```
mkdir /boot/EFI/boot
cp /boot/EFI/arch_grub/grubx64.efi /boot/EFI/boot/bootx64.efi
```

#### Reboot Into New Arch Install
```
exit
umount -R /mnt
shutdown now
```
* Remove ISO from virtual CD drive 
* Start VM

## Post Install

#### Enable 64 bit Pacman Repo (`multilib`)
```
nano /etc/pacman.conf
    # Uncomment the '[multilib]' line and the 'Include' line below it
pacman -Sy
```

#### Install Packages
```
pacman -S sudo bash-completion vim
```

#### Create New User
```
useradd -m -g users -G wheel -s /bin/bash <username>
passwd <username>
EDITOR=nano visudo
    # Uncomment the '%wheel ALL=(ALL) NOPASSWD: ALL' line
```

#### Install VirtualBox Guest Additions
```
pacman -S virtualbox-guest-utils
```
When prompted choose:
    * virtualbox-guest-modules-arch
    * mesa-libgl
    * default option

## Install Plasma

#### Install display Server 
```
pacman -S xorg-server xorg-server-utils
```

#### Install Plasma Packages
```
pacman -S plasma sddm sddm-kcm
```
* sddm = Simple Desktop Display Manager
* `sddm-kcm` enables management of SDDM through System Settings

#### Configure SDDM
```
nano /etc/sddm.conf
```
Ensure the `Theme` section has the following settings
```
[Theme]
Current=breeze
CursorTheme=breeze_cursors
FacesDir=/usr/share/sddm/faces
ThemeDir=/usr/share/sddm/themes
```
```
systemctl enable sddm
```

#### Install Basic Apps
```
pacman -S dolphin konsole ttf-{dejavu,liberation} 
```

#### Reboot
```
reboot
```
