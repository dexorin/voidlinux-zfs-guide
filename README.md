This guide explains how to install Void Linux on ZFS with ZFSBootMenu support.

    Reference: Adapted from the ZFSBootMenu UEFI guide

    Live Media: Recommended to use hrmpf LiveCD (includes ZFS tools)

    Tested on: x86_64 (Virtual Machine)


## Configure Live Environment

Confirm EFI support

```bash
dmesg | grep -i efivars
```
Output should be: `Registered efivars operations`

### Optional: Enable GPM (mouse in TTY)

GPM enables mouse support in TTY, useful for selecting and copying text.
It is usually enabled by default in Void Linux Live environment. If not, enable it manually:

```bash
gpm -m /dev/input/mice -t exps2
```

### Define variables

```bash
# list available disks
fdisk -l  

export ID="void"                         # name for zfs boot env
export DISK="/dev/your_disk"             # your disk
export EFI_PART="/dev/your_disk_part"    # EFI partition
export ZFS_PART="/dev/your_disk_part"    # ZFS partition
```

### Generate `/etc/hostid`
```bash
zgenhostid
```

## Disk preparation

```bash
zpool labelclear -f "${DISK}"

wipefs -af "${DISK}"
sgdisk --zap-all "${DISK}"
```

### Create EFI boot partition

```bash
sgdisk -n 1:1m:+512m -t 1:ef00 "${DISK}"
```

### Create zpool partition

```bash
sgdisk -n 2:0:-10m -t 2:bf00 "${DISK}"
```

## ZFS pool creation and configuration

### For the ZFS pool, use /dev/disk/by-id

```bash
ls -la /dev/disk/by-id
```

Find the disk entry starting with (nvme-eui.-XXX-part2) and enter it in the next command.

### Create the pool (named zroot)

```bash
zpool create -f -o ashift=12 \
    -O compression=lz4 \
    -O acltype=posixacl \
    -O xattr=sa \
    -O relatime=on \
    -O dnodesize=auto \
    -O normalization=formD \
    -o autotrim=on \
    -o compatibility=openzfs-2.4-linux \
    -m none zroot /dev/disk/by-id/<enter_your_id>
```

### Create the minimal datasets

```bash
zfs create -o mountpoint=none zroot/ROOT
zfs create -o mountpoint=/ -o canmount=noauto zroot/ROOT/"${ID}"
zfs create -o mountpoint=/home -o compression=zstd zroot/home
```

### Create additional datasets

```bash
zfs create -o mountpoint=none -o compression=zstd zroot/data
zfs create -o mountpoint=/var/log -o recordsize=16k zroot/data/logs
zfs create -o mountpoint=/var/cache zroot/data/cache

zpool set bootfs=zroot/ROOT/"${ID}" zroot
```

### Export, then re-import with a temporary mountpoint of `/mnt`

```bash
zpool export zroot
zpool import -N -R /mnt zroot
zfs mount zroot/ROOT/"${ID}"
zfs mount -a
```

### Verify that everything is mounted correctly

```plaintext
mount | grep mnt

# Example output (where "${ID}" is "void")
zroot/ROOT/void on /mnt type zfs (rw,relatime,xattr,posixacl)
zroot/home on /mnt/home type zfs (rw,relatime,xattr,posixacl)
zroot/data/logs on /mnt/var/log type zfs (rw,relatime,xattr,posixacl)
zroot/data/cache on /mnt/var/cache type zfs (rw,relatime,xattr,posixacl)

# Sometimes there is also a casesensitive attribute, you can safely ignore it.

zfs list

zroot             1.20M  76.5G    96K  none
zroot/ROOT         208K  76.5G    96K  none
zroot/ROOT/void    112K  76.5G   112K  /mnt
zroot/data         288K  76.5G    96K  none
zroot/data/cache    96K  76.5G    96K  /mnt/var/cache
zroot/data/logs     96K  76.5G    96K  /mnt/var/log
zroot/home          96K  76.5G    96K  /mnt/home
```

### Update device symlinks

```bash
udevadm trigger
```

## Installing basic system

> [!NOTE]
> Optional step
> 
> By default, `base-system` pulls in the `linux` meta-package, which
> currently points to the LTS kernel (6.12). If you prefer a different
> kernel, you can avoid installing the wrong one from the start.
>
> First, blacklist `linux` and `linux-headers`:
>
> ```bash
> mkdir -p /mnt/etc/xbps.d
>
> cat << EOF > /mnt/etc/xbps.d/kernel-ignore.conf
> ignorepkg=linux
> ignorepkg=linux-headers
> EOF
> ```
>
> Then append your desired kernel to the next install command, for example:
>
> ```plaintext
> base-system linux-base linux6.18 linux6.18-headers
> ```
>
> `linux-base` contains only firmware and shared scripts (no kernel),
> so it is always required regardless of your kernel choice.

### Installing system packages

```bash
XBPS_ARCH=x86_64 xbps-install \
  -S -R https://repo-de.voidlinux.org/current \
  -r /mnt base-system
```

### Copy hostid into the new install

```bash
cp /etc/hostid /mnt/etc
```

### Chroot into the new OS

```bash
xchroot /mnt
```

### Upgrade the system and install the ZFS
```bash
xbps-install -Su
xbps-install -S dkms zfs
```


## Basic Void configuration

### Set the keymap, timezone, and hardware clock

```bash
cat << EOF >> /etc/rc.conf
KEYMAP="us"
HARDWARECLOCK="UTC"
EOF
```

```bash
ln -sf /usr/share/zoneinfo/<timezone>/<city> /etc/localtime
```

### Configure glibc locale
In `/etc/default/libc-locales` uncomment the locales you need.

### Install important setup tools

```bash
xbps-install -S efibootmgr curl
```

### Configure Dracut

```bash
cat << EOF > /etc/dracut.conf.d/zol.conf
nofsck="yes"
add_dracutmodules+=" zfs "
omit_dracutmodules+=" btrfs "
EOF
```

Then reconfigure all packages (this will also rebuild the initramfs)
```bash
xbps-reconfigure -fa
```

### Set a root password

```bash
passwd
```

## Install and configure ZFSBootMenu

### Create the `vfat` filesystem

```bash
mkfs.vfat -F32 "${EFI_PART}"
```

### Set ZFSBootMenu properties for the datasets

```bash
zfs set org.zfsbootmenu:commandline="quiet" zroot/ROOT
```

### Create fstab and mount partition

```bash
blkid -s UUID -o value "${EFI_PART}"
# Replace <UUID> with the output from the previous command
export EFI_UUID="<UUID>"

echo "UUID=$EFI_UUID /boot/efi vfat utf8,dmask=0077 0 2" | column -t | tee /etc/fstab

mkdir -p /boot/efi
mount /boot/efi
```

### Install ZFSBootMenu

Fetch the prebuilt ZFSBootMenu EFI executable and save it to the EFI system partition.

```bash
mkdir -p /boot/efi/EFI/ZBM
curl -o /boot/efi/EFI/ZBM/VMLINUZ.EFI -L https://get.zfsbootmenu.org/efi
cp /boot/efi/EFI/ZBM/VMLINUZ.EFI /boot/efi/EFI/ZBM/VMLINUZ-BACKUP.EFI
```

### Configure EFI boot entries

```bash
efibootmgr -c -d "${DISK}" -p 1 \
  -L "ZFSBootMenu (Backup)" \
  -l '\EFI\ZBM\VMLINUZ-BACKUP.EFI'

efibootmgr -c -d "${DISK}" -p 1 \
  -L "ZFSBootMenu" \
  -l '\EFI\ZBM\VMLINUZ.EFI'
```

### Install network packages

```bash
xbps-install -S iwd openresolv dhcpcd
```

> [!WARNING] 
> Do not touch any services while in chroot. This can cause various issues with `/etc/sv` and `/var/service` after reboot. Enable services after first boot.

## Exit the chroot, unmount everything

```bash
exit
```

```bash
umount -R /mnt
```

### Export the zpool and reboot
```bash
zpool export zroot
reboot
```
