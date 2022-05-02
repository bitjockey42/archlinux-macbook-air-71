# Arch Linux on Macbook Air 7,1

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

`/etc/makepkg.conf`

```
#!/hint/bash
#
# /etc/makepkg.conf
#

#########################################################################
# SOURCE ACQUISITION
#########################################################################
#
#-- The download utilities that makepkg should use to acquire sources
#  Format: 'protocol::agent'
DLAGENTS=('file::/usr/bin/curl -qgC - -o %o %u'
          'ftp::/usr/bin/curl -qgfC - --ftp-pasv --retry 3 --retry-delay 3 -o %o %u'
          'http::/usr/bin/curl -qgb "" -fLC - --retry 3 --retry-delay 3 -o %o %u'
          'https::/usr/bin/curl -qgb "" -fLC - --retry 3 --retry-delay 3 -o %o %u'
          'rsync::/usr/bin/rsync --no-motd -z %u %o'
          'scp::/usr/bin/scp -C %u %o')

# Other common tools:
# /usr/bin/snarf
# /usr/bin/lftpget -c
# /usr/bin/wget

#-- The package required by makepkg to download VCS sources
#  Format: 'protocol::package'
VCSCLIENTS=('bzr::bzr'
            'fossil::fossil'
            'git::git'
            'hg::mercurial'
            'svn::subversion')

#########################################################################
# ARCHITECTURE, COMPILE FLAGS
#########################################################################
#
CARCH="x86_64"
CHOST="x86_64-pc-linux-gnu"

#-- Compiler and Linker Flags
#CPPFLAGS=""
CFLAGS="-march=native -O2 -pipe -fno-plt -fexceptions \
        -Wp,-D_FORTIFY_SOURCE=2 -Wformat -Werror=format-security \
        -fstack-clash-protection -fcf-protection"
CXXFLAGS="$CFLAGS -Wp,-D_GLIBCXX_ASSERTIONS"
LDFLAGS="-Wl,-O1,--sort-common,--as-needed,-z,relro,-z,now"
LTOFLAGS="-flto=auto"
#RUSTFLAGS="-C opt-level=2"
#-- Make Flags: change this for DistCC/SMP systems
MAKEFLAGS="-j$(nproc)"
#-- Debugging flags
DEBUG_CFLAGS="-g"
DEBUG_CXXFLAGS="$DEBUG_CFLAGS"
#DEBUG_RUSTFLAGS="-C debuginfo=2"

#########################################################################
# BUILD ENVIRONMENT
#########################################################################
#
# Makepkg defaults: BUILDENV=(!distcc !color !ccache check !sign)
#  A negated environment option will do the opposite of the comments below.
#
#-- distcc:   Use the Distributed C/C++/ObjC compiler
#-- color:    Colorize output messages
#-- ccache:   Use ccache to cache compilation
#-- check:    Run the check() function if present in the PKGBUILD
#-- sign:     Generate PGP signature file
#
BUILDENV=(!distcc color !ccache check !sign)
#
#-- If using DistCC, your MAKEFLAGS will also need modification. In addition,
#-- specify a space-delimited list of hosts running in the DistCC cluster.
#DISTCC_HOSTS=""
#
#-- Specify a directory for package building.
#BUILDDIR=/tmp/makepkg

#########################################################################
# GLOBAL PACKAGE OPTIONS
#   These are default values for the options=() settings
#########################################################################
#
# Makepkg defaults: OPTIONS=(!strip docs libtool staticlibs emptydirs !zipman !purge !debug !lto)
#  A negated option will do the opposite of the comments below.
#
#-- strip:      Strip symbols from binaries/libraries
#-- docs:       Save doc directories specified by DOC_DIRS
#-- libtool:    Leave libtool (.la) files in packages
#-- staticlibs: Leave static library (.a) files in packages
#-- emptydirs:  Leave empty directories in packages
#-- zipman:     Compress manual (man and info) pages in MAN_DIRS with gzip
#-- purge:      Remove files specified by PURGE_TARGETS
#-- debug:      Add debugging flags as specified in DEBUG_* variables
#-- lto:        Add compile flags for building with link time optimization
#
OPTIONS=(strip docs !libtool !staticlibs emptydirs zipman purge !debug !lto)

#-- File integrity checks to use. Valid: md5, sha1, sha224, sha256, sha384, sha512, b2
INTEGRITY_CHECK=(sha256)
#-- Options to be used when stripping binaries. See `man strip' for details.
STRIP_BINARIES="--strip-all"
#-- Options to be used when stripping shared libraries. See `man strip' for details.
STRIP_SHARED="--strip-unneeded"
#-- Options to be used when stripping static libraries. See `man strip' for details.
STRIP_STATIC="--strip-debug"
#-- Manual (man and info) directories to compress (if zipman is specified)
MAN_DIRS=({usr{,/local}{,/share},opt/*}/{man,info})
#-- Doc directories to remove (if !docs is specified)
DOC_DIRS=(usr/{,local/}{,share/}{doc,gtk-doc} opt/*/{doc,gtk-doc})
#-- Files to be removed from all packages (if purge is specified)
PURGE_TARGETS=(usr/{,share}/info/dir .packlist *.pod)
#-- Directory to store source code in for debug packages
DBGSRCDIR="/usr/src/debug"

#########################################################################
# PACKAGE OUTPUT
#########################################################################
#
# Default: put built package and cached source in build directory
#
#-- Destination: specify a fixed directory where all packages will be placed
#PKGDEST=/home/packages
#-- Source cache: specify a fixed directory where source files will be cached
#SRCDEST=/home/sources
#-- Source packages: specify a fixed directory where all src packages will be placed
#SRCPKGDEST=/home/srcpackages
#-- Log files: specify a fixed directory where all log files will be placed
#LOGDEST=/home/makepkglogs
#-- Packager: name/email of the person or organization building packages
#PACKAGER="John Doe <john@doe.com>"
#-- Specify a key to use for package signing
#GPGKEY=""

#########################################################################
# COMPRESSION DEFAULTS
#########################################################################
#
COMPRESSGZ=(gzip -c -f -n)
COMPRESSBZ2=(bzip2 -c -f)
COMPRESSXZ=(xz -c -z --threads=0 -)
COMPRESSZST=(zstd -c -z -q -)
COMPRESSLRZ=(lrzip -q)
COMPRESSLZO=(lzop -q)
COMPRESSZ=(compress -c -f)
COMPRESSLZ4=(lz4 -q)
COMPRESSLZ=(lzip -c -f)

#########################################################################
# EXTENSION DEFAULTS
#########################################################################
#
PKGEXT='.pkg.tar.zst'
SRCEXT='.src.tar.gz'

#########################################################################
# OTHER
#########################################################################
#
#-- Command used to run pacman as root, instead of trying sudo and su
#PACMAN_AUTH=()

```


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