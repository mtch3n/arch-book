# Arch Install

Arch fresh setup using `btrfs` and `systemd-boot`.

## TODO

- [ ] Use Ansible to automate the installation.

## Pre-install

Before installing, make sure to:

- Read the official wiki. It is advisable to read that instead.
- Boot Arch install media

### Set keyboard layout

The default console keymap is US. Available layouts can be listed with:

```
# localectl list-keymaps
```

To set the keyboard layout, pass its name to loadkeys. For example, to set a German keyboard layout:

```
# loadkeys de-latin1
```

### Verify Boot Mode

Check if UEFI is enabled

```
# ls /sys/firmware/efi/efivars
```
If the command shows the directory without error, then the system is booted in UEFI mode. If the directory does not exist, the system may be booted in **BIOS** (or **CSM**) mode.

### Connect to the Internet


```
# ip link
```

```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: eno1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP mode DEFAULT group default qlen 1000
    link/ether d4:5d:64:00:24:e9 brd ff:ff:ff:ff:ff:ff
    altname enp0s31f6
```

- `eno1` is the wired network interface

Check connectivity

```
# ping archlinux.org
```

### Update the system clock

Use `timedatectl` to ensure the system clock is accurate:

```
timedatectl
```

## Install

### Prepare Disk

Layout

| Mount Point | Partition      | Partition Type       | Size      | remarks    |
| ----------- | -------------- | -------------------- | --------- | ---------- |
| /mnt/boot   | /dev/nvme0n1p1 | EFI System Partition | 1G        |            |
| /mnt/swap   | /dev/nvme0n1p2 | btrfs                | 4G        | -          |
| /mnt        | /dev/nvme0n1p2 | btrfs                | remainder | -          |
| /btrfs      | /dev/nvme0n1p2 | btrfs                | -         | maintaince |

Btrfs Subvolumes

| Subvolume    | Mount Point           |
| ------------ | --------------------- |
| @active/root | /                     |
| @active/home | /home                 |
| @swap        | /swap                 |
| @snapshots   | /.snapshots           |
| @active/pkg  | /var/cache/pacman/pkg |

### Format NVMe

Optimize NVMe for Linux and set the disk to use 4k sectors

```
DISK=/dev/nvme1n1

# nvme id-ns -H /dev/$DISK |  grep "Relative Performance"
LBA Format  0 : Metadata Size: 0   bytes - Data Size: 512 bytes - Relative Performance: 0x1 Better (in use)
LBA Format  1 : Metadata Size: 0   bytes - Data Size: 4096 bytes - Relative Performance: 0 Best

# nvme format --lbaf=1 /dev/$DISK

# nvme id-ns -H /dev/$DISK |  grep "Relative Performance"
LBA Format  0 : Metadata Size: 0   bytes - Data Size: 512 bytes - Relative Performance: 0x1 Better 
LBA Format  1 : Metadata Size: 0   bytes - Data Size: 4096 bytes - Relative Performance: 0 Best (in use)
```

### Partitioning


- Install `gptfdisk` to partition the disk

    ```
    DISK=/dev/nvme1n1
    # pacman -S  gptfdisk 
    # gdisk /dev/$DISK
    ```

- Create Partitions

    Example output:

    ```
    GPT fdisk (gdisk) version 1.0.1

    Partition table scan:
    MBR: protective
    BSD: not present
    APM: not present
    GPT: present

    Found valid GPT with protective MBR; using GPT.

    Command (? for help): o
    This option deletes all partitions and creates a new protective MBR.
    Proceed? (Y/N): Y

    Command (? for help): n
    Partition number (1-128, default 1): 
    First sector (34-244190463, default = 2048) or {+-}size{KMGTP}: 
    Last sector (2048-244190463, default = 244190463) or {+-}size{KMGTP}: +1G
    Current type is 'Linux filesystem'
    Hex code or GUID (L to show codes, Enter = 8300): EF00
    Changed type of partition to 'EFI System'

    Command (? for help): n
    Partition number (2-128, default 2): 
    First sector (34-244190463, default = 262400) or {+-}size{KMGTP}: 
    Last sector (262400-244190463, default = 244190463) or {+-}size{KMGTP}: 
    Current type is 'Linux filesystem'
    Hex code or GUID (L to show codes, Enter = 8300): 
    Changed type of partition to 'Linux filesystem'

    Command (? for help): p
    Disk /dev/nvme1n1: 244190646 sectors, 931.5 GiB
    Model: CT1000P3SSD8                            
    Sector size (logical/physical): 4096/4096 bytes
    Disk identifier (GUID): E3D8711C-B78D-4E6E-9D5E-558718F166ED
    Partition table holds up to 128 entries
    Main partition table begins at sector 2 and ends at sector 5
    First usable sector is 256, last usable sector is 244190640
    Partitions will be aligned on 256-sector boundaries
    Total free space is 177 sectors (708.0 KiB)

    Number  Start (sector)    End (sector)  Size       Code  Name
    1             256          262399   1024.0 MiB  EF00  
    2          262400       244190463   930.5 GiB   8300  

    Command (? for help): w
    ```

    > Arch Wiki: [GPT fdisk](https://wiki.archlinux.org/title/GPT_fdisk)


- Format Partitions

    ```
    DISK=/dev/nvme1n1
    EFI=/dev/${DISK}p1
    BTRFS=/dev/${DISK}p2

    # mkfs.vfat -F32 -n "EFI" $EFI
    # mkfs.btrfs -L Arch -f $BTRFS
    ```

- Create Btrfs Subvolumes and mount

    ```
    DISK=/dev/nvme1n1
    BTRFS=/dev/${DISK}p2

    # mount $BTRFS /mnt
    # btrfs subvolume create /mnt/@active
    # btrfs subvolume create /mnt/@active/root
    # btrfs subvolume create /mnt/@active/home
    # btrfs subvolume create /mnt/@active/pkg
    # btrfs subvolume create /mnt/@swap
    # btrfs subvolume create /mnt/@snapshots

    # umount /mnt
    ```

    Mount Partitions

    ```
    DISK=/dev/nvme1n1
    EFI=/dev/${DISK}p1
    BTRFS=/dev/${DISK}p2

    # mount -o subvol=@active/root $BTRFS /mnt
    # mkdir -p /mnt/{boot,home,var/cache/pacman/pkg,.snapshots,btrfs}

    # mount -o noatime,compress=zstd:3,subvol=@active/home $BTRFS /mnt/home
    # mount -o noatime,compress=zstd:3,subvol=@active/pkg $BTRFS /mnt/var/cache/pacman/pkg
    # mount -o noatime,compress=zstd:3,subvol=@snapshots $BTRFS /mnt/.snapshots
    # mount -o noatime,compress=no,subvol=@swap $BTRFS /mnt/swap
    # mount $EFI /mnt/boot
    ```

    > about compression issue https://fedoraproject.org/wiki/Changes/BtrfsTransparentCompression

### Config swap

    ```
    # btrfs filesystem mkswapfile --size 4G --uuid clear /mnt/swap/swapfile
    # swapon /mnt/swap/swapfile

    # echo vm.swappiness=10 | sudo tee /etc/sysctl.d/99-swappiness.conf
    ```

### Generate Fstab

Generate fstab using `genfstab`

```
# genfstab -U /mnt >> /mnt/etc/fstab
```

Check Fstab

```
# findmnt --verify --verbose
```

### Install essential packages

```
# pacstrap -K /mnt base linux linux-firmware base-devel btrfs-progs intel-ucode networkmanager sbctl sudo zsh vim nvidia nvidia-settings nvidia-utils wireplumber pipewire
```

### Chroot

Change root into /mnt (system)

```
# arch-chroot /mnt
```

### Time

Set timezone and sync hardware clock with system clock

```
TZ=America/Toronto

# ln -sf /usr/share/zoneinfo/$TZ /etc/localtime
# hwclock --systohc
```

> Windows dual-boot: [FIX](#dual-boot-windows-clock-fix)

### Localization

```
sed -i '/en_US.UTF-8/s/^#//g' /etc/locale.gen
sed -i '/en_CA.UTF-8/s/^#//g' /etc/locale.gen

locale-gen

LANG=en_CA.UTF-8 > /etc/locale.conf
```

### Hostname

```
HOSTNAME=home-dev
echo "$HOSTNAME" >> /etc/hostname

cat <<EOF > /etc/hosts
127.0.0.1 localhost
::1 localhost
127.0.0.1 $HOSTNAME.localdomain $HOSTNAME
EOF
```

### mkinitcpio

Edit `mkinitcpio.conf`

```
MODULES=(btrfs nvidia nvidia_modeset nvidia_uvm nvidia_drm)
BINARIES=(btrfs)
HOOKS=(base udev autodetect microcode modconf keyboard keymap consolefont block filesystems)
```

### Finalize

#### Generate initramfs

```
mkinitcpio -P
```

#### Set root password

```
passwd
```

#### Bootloader

```
bootctl install
```

Create boot entry

```
BOOT=/dev/nvme0n1p2

cat <<EOF > /boot/loader/entries/arch.conf
title Arch Linux
linux /vmlinuz-linux
initrd /intel-ucode.img
initrd /initramfs-linux.img
options root=UUID=$(blkid -s UUID -o value $BOOT) rw
```

## Post-install

### Pacman

edit `/etc/pacman.conf`

```
Color
ILoveCandy
ParallelDownloads = 20
```

### Config Sudo

```
# EDITOR=vim visudo
```

Uncomment the line to allow `wheel` sudo access

```
%wheel      ALL=(ALL:ALL) ALL
```

Create `10-custom` config

/etc/sudoers.d/10-custom

```
# prevent wait timeout
Defaults passwd_timeout=0
# avoid re-enter password while switch terminal
Defaults timestamp_type=global
# increse timeout time
Defaults timestamp_timeout=10
```

Other configurations: [Sudo](#sudo-configs)

### User

```
useradd -m -G wheel -s /usr/bin/zsh [username]
```

### Install yay

Switch user and install `yay`

```
# pacman -S --needed git base-devel

$ git clone https://aur.archlinux.org/yay.git
$ cd yay
$ makepkg -si
```

After install do:

Use `yay -Y --gendb` to generate a development package database for `*-git` packages that were installed without yay. This command should only be run once.

`yay -Syu --devel` will then check for development package updates

Use `yay -Y --devel --save` to make development package updates permanently enabled (`yay` and `yay -Syu` will then always check dev packages)

### Sync Time

Disable `systemd-timesyncd` and use `chrony` & Google NTP service

```
$ yay -S chrony
# systemctl disable systemd-timesyncd
# sudo systemctl enable --now chronyd.service
```

Add server to `/etc/chrony.conf`

```
server time1.google.com iburst
server time2.google.com iburst
server time3.google.com iburst
server time4.google.com iburst
```

> Auto timezone update: [Arch Wiki](https://wiki.archlinux.org/title/System_time#Update_timezone_every_time_NetworkManager_connects_to_a_network)

### Enable Services

```
systemctl enable NetworkManager
systemctl --user enable --now pipewire.socket pipewire-pulse.socket wireplumber.service
systemctl --user enable --now pipewire.service
```

### Create User Directories

```
# pacman -S xdg-user-dirs
$ xdg-user-dirs-update
```

## Appendices

### Sudo configs

Refreshing `sudo` timeout using `alias`

```
alias sudo='sudo -v; sudo '
```

### Dual-boot windows clock fix

Edit and save `.reg` file and exec to change windows registry setting

`windows-utc.reg`: 

```
Windows Registry Editor Version 5.00

[HKEY_LOCAL_MACHINE\System\CurrentControlSet\Control\TimeZoneInformation]
"RealTimeIsUniversal"=dword:00000001
```

`windows-localtime.reg`:

```
Windows Registry Editor Version 5.00

[HKEY_LOCAL_MACHINE\System\CurrentControlSet\Control\TimeZoneInformation]
"RealTimeIsUniversal"=-
```

### Btrfs

Trigger compression after change compression settings

```bash
sudo btrfs fi defrag -v -r -czstd /
```