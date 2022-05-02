# Arch Linux on Macbook Air 7,1

These are my install notes and configs for my early 2015 Macbook Air running Arch Linux.

In the [conf](conf/) directory, the `*.conf` files are structured as if they were under `/`.

```
$ tree conf
conf
├── boot
│   └── loader
│       ├── entries
│       │   ├── arch.conf
│       │   └── arch-macbook.conf
│       └── loader.conf
└── etc
    ├── fstab
    ├── iwd
    │   └── main.conf
    ├── makepkg.conf
    ├── mkinitcpio.conf
    └── ykfde.conf

5 directories, 8 files
```

## What doesn't work

- Suspend to RAM. Even with the linux-macbook kernel in the AUR I cannot get this to work...

## Requirements

- An existing Arch Linux system (can be a VM or docker container)
- 2 External USB Flash Drives
- (Optional) Yubikey

## Preparation

### Resize macOS partition

#### Boot into Recovery
Shut off the machine. Hold the <kbd>cmd+R</kbd> keys and press the power button. Continue to hold it until you see the Apple logo.

Use Disk Utility to resize the Mac partition and add another one, which will be the Linux root.

#### Preparing an offline install

On an existing Arch Linux installation with internet connection:

```
# Create a custom repo
pacman -Syw --cachedir . --dbpath /tmp/blankdb base base-devel linux linux-firmware linux-headers systemd mkinitcpio broadcom-wl-dkms vim iwd lvm2
repo-add ./custom.db.tar.gz ./*
```

Then copy the above files into another usb.

## Installation

### Partition

```
cgdisk /dev/nvme0n1
```

Then select `New`  and input `+128M`, press enter.

```
Part. #     Size        Partition Type            Partition Name
----------------------------------------------------------------
            3.0 KiB     free space
   1        200.0 MiB   EFI system partition      EFI System Partition
   2        93.3 GiB    Apple APFS
            128.0 MiB   free space
   3        800.6 GiB   Linux filesystem          Arch Linux
```

### Encrypted Root Partition

Create a passphrase to encrypt the partition:
```shell
cryptsetup -v --cipher aes-xts-plain64 --key-size 256 -y luksFormat /dev/nvme0n1p3
```

Then set up this new partition:

```shell
# This will prompt you for the passphrase
cryptsetup luksOpen /dev/nvme0n1p3 lvm

# Create volume
pvcreate /dev/mapper/lvm
vgcreate vgcrypt /dev/mapper/lvm
lvcreate --extents +100%FREE -n root vgcrypt

# Format
mkfs.ext4 /dev/mapper/vgcrypt-root
```

### Mount Partitions

```
mount /dev/mapper/vgcrypt-root /mnt
mkdir /mnt/boot
mount /dev/nvme0n1p1 /mnt/boot
```

### Setup Offline Pacman

Insert the usb containing the offline installation files and figure out what its device name is to mount:

```
lsblk
mkdir /mnt/repo
mount /dev/sdb1 /mnt/repo
```

Comment out the `community,core,extra` and append this to `/etc/pacman.conf`:

```
[custom]
SigLevel = Optional TrustAll
Server = file:///mnt/repo/Packages
```

### Base Installation

```shell
pacstrap /mnt base base-devel linux linux-firmware linux-headers systemd mkinitcpio broadcom-wl-dkms vim iwd lvm2
```

Generate an `fstab`:
```shell
genfstab -L -p /mnt >> /mnt/etc/fstab
```

### Chroot

#### Enter chroot

```shell
arch-chroot /mnt
```

#### Configure password

```
passwd
```

#### Configure Hostname

```shell
echo yourhostname > /etc/hostname
```

#### Set date and time

```
ln -sf /usr/share/zoneinfo/America/New_York /etc/localtime
```

#### Locale

```shell
vim /etc/locale.gen
locale-gen

echo LANG=en_US.UTF-8 > /etc/locale.conf
```

#### Consolefont

Edit `/etc/vconsole.conf`:

```
FONT=sun12x22
```

#### Swapfile

```
dd if=/dev/zero of=/swapfile bs=1G count=20 status=progress
chmod 0600 /swapfile
mkswap -U clear /swapfile
swapon /swapfile

# Add to fstab
echo "/swapfile none swap defaults 0 0"|tee -a /etc/fstab
```

Get the `resume_offset` with:

```
[root@macbookarch ~]# filefrag -v /swapfile
Filesystem type is: ef53
File size of /swapfile is 21474836480 (5242880 blocks of 4096 bytes)
 ext:     logical_offset:        physical_offset: length:   expected: flags:
   0:        0..   28671:     266240..    294911:  28672:            
   1:    28672..   32767:     372736..    376831:   4096:     294912:

```

Look for the value in the first row, 3rd column (physical offset). Here it's `266240` .  You'll need this to configure the kernel options in the bootloader.

#### Mkinitcpio hooks

Modify `/etc/mkinitcpio.conf`:

```
HOOKS=(base udev autodetect modconf block consolefont keyboard encrypt lvm2 filesystems resume fsck)
```

Regenerate:

```
mkinitcpio -P
```

#### Bootloader

`/boot/loader/loader.conf`:

```
default arch
timeout 4
```

`/boot/loader/entries/arch.conf` with `resume_offset=<value you got from filefrag>`:

```
title	Arch Linux
linux	/vmlinuz-linux
initrd	/initramfs-linux.img
options	cryptdevice=UUID=ce4f0ca7-9df6-417d-904d-6246896e704c:vgcrypt:allow-discards root=/dev/mapper/vgcrypt-root resume=/dev/mapper/vgcrypt-root resume_offset=266240 rw
```

Install bootloader:

```
bootctl install
```

#### Exit

Exit the chroot

```
exit
```

### Reboot into new system

```
umount -R /mnt
reboot
```

## Post-Installation

### WiFi

`/etc/iwd/main.conf`:

```
[General]
EnableNetworkConfiguration=true

[Network]
NameResolvingService=systemd

```

Enable:

```
systemctl enable --now systemd-resolved
systemctl enable --now iwd
```

Connect to the internet using `iwctl`

```
[bitjockey@macbookarch ~]$ iwctl
NetworkConfigurationEnabled: enabled
StateDirectory: /var/lib/iwd
Version: 1.27
[iwd]# station wlan0 connect "MySSID"
[iwd]# station list
                            Devices in Station Mode                           *
--------------------------------------------------------------------------------
  Name                State          Scanning
--------------------------------------------------------------------------------
  wlan0               connected     
```

### Unlocking encrypted volume with Yubikey

Using [Yubikey full disk encryption](https://github.com/agherzan/yubikey-full-disk-encryption) , you can use your [yubikey](https://www.yubico.com/products/yubikey-5-overview/) to unlock an encrypted partition.

```
sudo pacman -Syu yubikey-full-disk-encryption
```

`/etc/ykfde.conf`:

```
# Set to non-empty value to use 'Automatic mode with stored challenge (1FA)'.
YKFDE_CHALLENGE="1"

YKFDE_CHALLENGE_SLOT="2"
```

Insert yubikey into computer.

```
ykpersonalize -v -2 -ochal-resp -ochal-hmac -ohmac-lt64 -oserial-api-visible -ochal-btn-trig
sudo ykfde-enroll -d /dev/nvme0n1p3 -s 2
```

Enter the passphrase you set up, then touch the key.

Modify `HOOKS` in `/etc/mkinitcpio.conf` to have `ykfde` be before `encrypt`:

```
HOOKS=(base udev autodetect modconf block consolefont keyboard ykfde encrypt lvm2 filesystems resume fsck)
```

Then regenerate:

```
sudo mkinitcpio -P
```

Reboot with `systemctl reboot`.

At the encrypt unlock prompt, insert yubikey and touch it.

If this succeeds, let the system boot fully, then modify the `HOOKS` array in `/etc/mkinitcpio.conf`. 

So now when you boot your computer you don't have to type in your passphrase, just insert the Yubikey.

### linux-macbook kernel

### Configure makepkg to speed up builds

See: [makepkg.conf](conf/etc/makepkg.conf)

#### Build linux-macbook kernel

[Download AUR snapshot](https://aur.archlinux.org/cgit/aur.git/snapshot/linux-macbook.tar.gz)

Extract the files.

In another terminal, clone the Arch fork of the Linux kernel and checkout the `v5.9.9-arch1` tag:

```shell
git clone https://github.com/archlinux/linux
git checkout -b linux-macbook v5.9.9-arch1
```

Modify the `source` array in the `PKGBUILD`:

```
source=(
  "$_srcname::git+file:///home/bitjockey/Packages/linux"
  # other stuff here
)
```

Install `pacman-contrib` and update the package signatures:

```shell
sudo pacman -S pacman-contrib
updpkgsums
```

"Download" the sources:

```shell
makepkg -od
```

Build and install:

```
makepkg -si
```

Create a `/boot/loader/entries/arch-macbook.conf`

```
title	Arch Linux Macbook
linux	/vmlinuz-linux-macbook
initrd	/initramfs-linux-macbook.img
options	cryptdevice=UUID=ce4f0ca7-9df6-417d-904d-6246896e704c:vgcrypt:allow-discards root=/dev/mapper/vgcrypt-root resume=/dev/mapper/vgcrypt-root resume_offset=266240 rw
```

Then:

```
sudo mkinitcpio -P
```

#### Set linux-macbook as default kernel

There seems to be a bug with `systemd-boot` not respecting the default set in `/boot/loader/loader.conf` so this has to be done.

```
$ bootctl list
Boot Loader Entries:
        title: Arch Linux Macbook (default)
           id: arch-macbook.conf
       source: /boot/loader/entries/arch-macbook.conf
        linux: /vmlinuz-linux-macbook
       initrd: /initramfs-linux-macbook.img
      options: cryptdevice=UUID=ce4f0ca7-9df6-417d-904d-6246896e704c:vgcrypt:allow-discards root=/dev/mapper/vgcrypt-root resume=/dev/mapper/vgcrypt-root resume_offset=266240 rw

        title: Arch Linux
           id: arch.conf
       source: /boot/loader/entries/arch.conf
        linux: /vmlinuz-linux
       initrd: /initramfs-linux.img
      options: cryptdevice=UUID=ce4f0ca7-9df6-417d-904d-6246896e704c:vgcrypt:allow-discards root=/dev/mapper/vgcrypt-root resume=/dev/mapper/vgcrypt-root resume_offset=266240 rw

        title: macOS
           id: auto-osx
       source: /sys/firmware/efi/efivars/LoaderEntries-4a67b082-0a4c-41cf-b6c7-440b29bb8c4f
```

```
sudo bootctl set-default "arch-macbook.conf"
```

### Desktop Environment

- [MATE](https://wiki.archlinux.org/title/MATE)
- [Lightdm](https://wiki.archlinux.org/title/LightDM)
- [i3](https://wiki.archlinux.org/title/I3)

 #### MATE

```shell
sudo pacman -S mate mate-extra
```

#### Lightdm

```shell
sudo pacman -S lightdm lightdm-slick-greeter
sudo systemct enable lightdm
```

Edit `/etc/lightdm/lightdm.conf`:

```conf
# ...other stuff

[Seat:*]
greeter-session=lightdm-slick-greeter

# ...otherstuff
```

Reboot the system:

```shell
systemctl reboot
```

### Recommended Applications

#### Qutebrowser
[Arch Wiki - Qutebrowser](https://wiki.archlinux.org/title/Qutebrowser)

```shell
sudo pacman -S qutebrowser python-adblock
```

**Extensions**
- [qute-bitwarden](https://github.com/qutebrowser/qutebrowser/blob/master/misc/userscripts/qute-bitwarden)
