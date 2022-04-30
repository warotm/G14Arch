# Arch Linux on Asus ROG Zephyrus G14

My journey with Arch Linux installation with these options

- dual boot with Windows11
- btrfs with encryption
- systemd-boot
- asus-linux patch/tools
- xfce4
- pipewire
- i3-wm (in-progress)

## Main references

- https://github.com/Unim8trix/G14Arch
- https://raphtlw.medium.com/guide-to-installing-arch-on-zephyrus-g14-21b93d2e9f49
- https://wiki.archlinux.org/title/installation_guide
- https://asus-linux.org/wiki/arch-guide/
- https://wiki.archlinux.org/title/Dual_boot_with_Windows

## Current issues

- Need to mount EFI partition as `/boot` : it might not have enough space to contain many kernels in the future.

## Pre-Installation

### Prepare USB Booting ISO

Use [Ventoy](https://www.ventoy.net) to create bootable USB drive
Copy Arch Linux `.iso` to USB drive prepared by `Ventoy2Disk.exe`

### Disable Windows power management on WIFI adaptor

To avoid missing WIFI adaptor problem while installing Arch Linux

- Open `Device Manager`
- `Properties` of wireless adaptor
- `Power Management` tab
- Uncheck `Allow the computer to turn off this device to save power` option

### Find empty space for new Linux partition

- Open `Computer Management` > `Storage` > `Disk Management`
- In my case, I shrink Drive C for free spaces
- DO NOT DELETE `EFI` AND `RECOVERY` PARTITION (partition labelled `RESTORE` could be removed but `ASUS Recovery` will be broken)

### Disable Fast boot and Secure boot

- Reboot and press `esc` repeatly to access BIOS settings

## Installation

### Boot Arch Linux

### WIFI connection

```bash
iwctl
station wlan0 scan
station wlan0 connect {SSID}
quit
```

If `wlan0` is not found, Windows power management on WIFI adaptor might be enabled

### Create encrypted btrfs partition

#### Create new partition with `fdisk`

#### Create encrypted filesystem

** my new partition named `/dev/nvme0n1p7`, you may have different name

```bash
cryptsetup luksFormat /dev/nvme0n1p7
cryptsetup open /dev/nvme0n1p7 luks
```

Passphase prompt should be shown

### Create and mount btrfs subvolumes

```bash
mkfs.btrfs -f -L ROOT /dev/mapper/luks
mount -t btrfs LABEL=ROOT /mnt
btrfs sub create /mnt/@
btrfs sub create /mnt/@home
btrfs sub create /mnt/@snapshots
btrfs sub create /mnt/@swap

truncate -s 0 /mnt/@swap/swapfile
chattr +C /mnt/@swap/swapfile
btrfs property set /mnt/@swap/swapfile compression none
fallocate -l 16G /mnt/@swap/swapfile
chmod 600 /mnt/@swap/swapfile
mkswap /mnt/@swap/swapfile

umount /mnt/

mount -o noatime,compress=zstd,commit=120,subvol=@ /dev/mapper/luks /mnt
mkdir -p /mnt/boot
mkdir -p /mnt/home
mkdir -p /mnt/.snapshots
mkdir -p /mnt/swap

mount -o noatime,compress=zstd,commit=120,subvol=@home /dev/mapper/luks /mnt/home/
mount -o noatime,compress=zstd,commit=120,subvol=@snapshots /dev/mapper/luks /mnt/.snapshots/
mount -o noatime,commit=120,subvol=@swap /dev/mapper/luks /mnt/swap/

mount /dev/nvme0n1p1 /mnt/boot/
```

### Change mirror / Install packages

Change `"TH"` to your country code

```bash
pacman -Syy
pacman -S reflector
reflector -c "TH" -f 12 -l 10 -n 12 --save /etc/pacman.d/mirrorlist

pacstrap /mnt base base-devel linux linux-firmware amd-ucode btrfs-progs nano networkmanager

genfstab -Lp /mnt >> /mnt/etc/fstab
echo "/swap/swapfile none swap defaults 0 0" >> /mnt/etc/fstab
```

### Chroot set settings

Change `Asia/Bangkok` to your timezone
Use `timedatectl list-timezones` to view all timezones
Uncomment your language in `/etc/locale.gen` by using `nano /etc/locale.gen`

```bash

arch-chroot /mnt
timedatectl set-timezone Asia/Bangkok
locale-gen
echo LANG=en_US.UTF-8 > /etc/locale.conf
echo {HOSTNAME} > /etc/hostname
```

edit `/etc/hosts`

```bash
127.0.0.1       localhost
::1             localhost
127.0.1.1       {HOSTNAME}
```

### Create user

```bash
useradd -m -G wheel -s /bin/bash {USERNAME}
passwd {USERNAME}

pacman -S sudo
EDITOR=nano visudo
```

Uncomment `%wheel ALL=(ALL) ALL`

### Add btrfs and encrypt to Initramfs

`nano /etc/mkinitcpio.conf` and add `encrypt btrfs` to hooks between block/filesystems

`HOOKS="base udev autodetect modconf block encrypt btrfs filesystems keyboard fsck `

Also include `amdgpu` in the MODULES section

create Initramfs using `mkinitcpio -P`

### Install Systemd Bootloader

`bootctl --path=/boot install` installs bootloader

`nano /boot/loader/loader.conf` delete anything and add these few lines and save

```bash
default         arch.conf
timeout         4
console-mode    max
editor          no
```

`nano /boot/loader/entries/arch.conf` with these lines and save.

```bash
title   Arch Linux
linux   /vmlinuz-linux
initrd  /amd-ucode.img
initrd  /initramfs-linux.img
options cryptdevice=/dev/nvme0n1p7:luks root=/dev/mapper/luks rootflags=subvol=@ rw
```

### Set nvidia-nouveau onto blacklist

using `nano /etc/modprobe.d/blacklist-nvidia-nouveau.conf` with these lines

```bash
blacklist nouveau
options nouveau modeset=0
```

### Leave Chroot and Reboot

Type `exit` to exit chroot

`umount -R /mnt/` to unmount all volumes

Now its time to `reboot` into the new system!

## Install Desktop Environment

### Some workarounds

```bash
sudo systemctl enable --now NetworkManager
sudo nmcli device wifi connect "{SSID}" password "{SSIDPASSWORD}"
```

```bash
sudo pacman -Sy acpid dbus
sudo systemctl enable acpid
```

```bash
systemctl enable --now systemd-timesyncd.service
```
```bash
passwd -l root
```

### Get X.Org / Xfce4 / LightDM / Pipewire / Bluetooth

```bash
sudo pacman -Sy xorg xfce4 xfce4-goodies gvfs xf86-input-synaptics ttf-dejavu network-manager-applet firefox git git-lfs curl wget
```

```bash
sudo pacman -Sy lightdm lightdm-webkit2-greeter
sudo systemctl enable lightdm
```

```bash
sudo pacman -Sy pipewire pipewire-pulse wireplumber
systemctl --user enable pipewire-pulse.service
```

```bash
sudo pacman -Sy bluez bluez-utils blueman
sudo systemctl enable --now bluetooth.service
```

## Installation after having GUI

### Oh-My-ZSH

I like to use oh-my-zsh with Powerlevel10K theme

```bash
sudo pacman -Sy zsh zsh-completions
chsh -s /bin/zsh
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
mkdir -p ~/.local/share/fonts
cd ~/.local/share/fonts
wget https://github.com/romkatv/powerlevel10k-media/raw/master/MesloLGS%20NF%20Regular.ttf
wget https://github.com/romkatv/powerlevel10k-media/raw/master/MesloLGS%20NF%20Bold.ttf
wget https://github.com/romkatv/powerlevel10k-media/raw/master/MesloLGS%20NF%20Italic.ttf
wget https://github.com/romkatv/powerlevel10k-media/raw/master/MesloLGS%20NF%20Bold%20Italic.ttf
fc-cache -v
git clone --depth=1 https://github.com/romkatv/powerlevel10k.git ${ZSH_CUSTOM:-$HOME/.oh-my-zsh/custom}/themes/powerlevel10k
```

Set `ZSH_THEME="powerlevel10k/powerlevel10k"` in `~/.zshrc`

### Nvidia Driver

```bash
sudo pacman -Sy nvidia-dkms nvidia-settings nvidia-prime acpi_call linux-headers
```

### Add patched kernel / tools from [Asus-linux](https://asus-linux.org/)

```bash
sudo bash -c "echo -e '\r[g14]\nSigLevel = DatabaseNever Optional TrustAll\nServer = https://arch.asus-linux.org\n' >> /etc/pacman.conf"

sudo pacman -Sy linux-g14 linux-g14-headers
sudo sed -i 's/vmlinuz-linux/vmlinuz-linux-g14/' /boot/loader/entries/arch.conf
sudo sed -i 's/initramfs-linux/initramfs-linux-g14/' /boot/loader/entries/arch.conf

sudo pacman -Sy asusctl supergfxctl
sudo systemctl enable --now power-profiles-daemon.service
sudo systemctl enable --now supergfxd
systemctl --user enable --now asus-notify
```

### Theming
[Linux Scoop channel](https://www.youtube.com/watch?v=X3siZNJN3ec)
