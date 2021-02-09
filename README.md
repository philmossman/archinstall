# archinstall

---
title: "Arch Install (btrfs & snapper)"
tags: ""
---

Assume booted from install media.
CHange keyboard layout

```zsh
loadkeys uk
```

If you need to find keymap

```zsh
localectl list-keymaps | grep uk
```

Check internet connection

```zsh
ip a
```

Wifi

    placeholder

synchronise ntp

```zsh
timedatectl set-ntp true
```

create mirrorlist

```zsh
reflector -c 'United Kingdom' -a 6 --sort rate --save /etc/pacman.d/mirrorlist
```

synchronise mirrors

```zsh
pacman -Syyy
```

list disks

```zsh
lsblk
```

start partitioning assume disk is `sda` but could also be `vda` or `nvme01n1`:

```zsh
gdisk /dev/sda
```

Creat partitions there shoud be a boot, swap and main partition. The boot is between 300 MB and 512 MB and shoudl be efi which is hex code `ef00`. the swap partition should be equal to RAM installed and has hex code `8200`. Finally the main partition can be default for the remainder of the disk.

Format the boot partition as FAT32

```zsh
mkfs.fat -F32 /dev/sda1
```

Make the swap patition

```zsh
mkswap /dev/sda2
```

Activate it

```zsh
swapon /dev/sda2
```

format the main parition as btrfs

```zsh
mkfs.btrfs /dev/sda3
```

Mount main partition to create sub-volumes

```zsh
mount /dev/sda3 /mnt
```

Create sub-volume for `/`

```zsh
btrfs su cr /mnt/@
```

Create sub-volume for `/home`

```zsh
btrfs su cr /mnt/@home
```

Create sub-volume for `.snapshots`

```zsh
btrfs su cr /mnt/@snapshots
```

Create sub-volume for `var_log`

```zsh
btrfs su cr /mnt/@var_log
```

Unmount btrfs volume

```zsh
umount /mnt
```

Mount sub-volume

```zsh
mount -o noatime,compress=lzo,space_cache=v2,subvol=@ /mnt/sda3 /mnt
```

Make the relevant directories for the sub-volumes

```zsh
mkdir -p /mnt/{boot,home,.snapshots,var_log}
```

Mount the sub-volumes

```zsh
mount -o noatime,compress=lzo,space_cache=v2,subvol=@home /mnt/sda3 /mnt/home
mount -o noatime,compress=lzo,space_cache=v2,subvol=@snapshots /mnt/sda3 /mnt/.snapshots
mount -o noatime,compress=lzo,space_cache=v2,subvol=@var_log /mnt/sda3 /mnt/var_log
```

Mount boot partiton

```zsh
mount /dev/sda1 /mnt/boot
```

Install base system. Use `intel-ucode` for intel `amd-ucode` for amd

```zsh
pacstrap /mnt base linux linux-firmware vim reflector rsync intel-ucode
```

Generate `fstab`

```zsh
genfstab -U /mnt >> /mnt/etc/fstab
```

Check this worked with:

```zsh
cat /mnt/etc/fstab
```

chroot into the system:

```zsh
arch-chroot /mnt
```

Set up local time

```zsh
ln -sf /usr/share/zoneinfo/Europe/London /etc/localtime
```

If you need to find timezone use `timedatectl list-timezones` e.g.

```zsh
timedatectl list-timezones | grep London
```

Sync hardware clock

```zsh
hwclock --systohc
```

generate locales by editing `locale.gen` and un-commenting the required locales. then running 

```zsh
vim /etc/locale.gen
```

un-comment the required locales then run

```zsh
locale-gen
```

Finally create `locale.conf` with

```zsh
vim /etc/locale.conf
```

and enter

```bash
LANG=en_GB.UTF-8
```

and `vconsole.conf` for keymap

```zsh
vim /etc/vconsole.conf
```

and add

```bash
KEYMAP=uk
```

Now time to set the hostname

```zsh
vim /etc/hostname
```

and enter whatever hostname you want in this case `archbtrfs-vm`
then edit the `hosts` file;

```zsh
vim /etc/hosts
```

with

```bash
# Static table lookup for hostnames.
# see hosts(5) for details.

127.0.0.1	localhost
::1			localhost
127.0.1.1	archbtrfs-vm.localdomain	archbtrfs-vm
```

Set root password with:

```zsh
passwd
```

Now time to install the required packages for the base system. Optionally run `reflctor` again to ensure mirrorlist is up to date:

```zsh
reflector -c 'United Kingdom' -a 6 --sort rate --save /etc/pacman.d/mirrorlist
```

now install the base system `wpa_supplicant` `bluez` `bluez-utils` `pulseaudio-bluetooth` are only necessary on wifi and bluetooth systems. Also optionally for VMs add `virtualbox-guest-utils`

```zsh
pacman -S grub efibootmgr networkmanager network-manager-applet dialog wpa_supplicant mtools dosfstools git snapper bluez bluez-utils cups hplip xdg-utils xdg-user-dirs alsa-utils pulseaudio pulseaudio-bluetooth inetutils bash-completion base-devel linux-headers
```

Edit `mkinitcpio.conf` to add `btrfs` module:

```zsh
vim /etc/mkinitcpio.conf
```

and add `btrfs` to the modules section then build the image with 

```zsh
mkinitcpio -p linux
```

Install GRUB

```zsh
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB
```

Generate the GRUB config file with

```zsh
grub-mkconfig -o /boot/grub/grub.cfg
```

Enable services required on startup

```zsh
systemctl enable NetworkManager
```

```zsh
systemctl enable bluetooth
```

```zsh
systemctl enable cups
```

Add a user in the `wheel` group

```zsh
useradd -mG wheel phil
```

Change password for user

```zsh
passwd phil
```

Edit sudoers file and uncomment the line that allows members of `wheel` 

```zsh
EDITOR=vim visudo
```

```bash
%wheel ALL=(ALL) ALL
```

Exit out of chroot

```zsh
exit
```

then unmount the filesystem

```zsh
umount -a
```
