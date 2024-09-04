# Installing Arch Linux Alongside a Pre-Installed OS [TL;DR]

**#manuall** **#dualboot** **#gnome+gdm** **#encrypted** **#lvm** **#nvidia-friendly**

This guide will help you you manually install Arch Linux. Consider reading the [Full Guide](README.md) for more details. 

## Installing Arch Linux

I will assume that you already in the installation environment.

### 1. Identify the disk & partition it

```bash
lsblk
```

### 2. Create partitions

```bash
fdisk /dev/{disk_id}
```

Use `m` to get help on creating partitions.

Create at least 3 partitions:

1. Swap partition (At least half of your RAM) (optional) `{swap_partition_id}`
2. EFI partition (1GB) `{efi_partition_id}`
3. Boot partition (2GB) `{boot_partition_id}`
4. Root partition (rest of the disk) `{main_partition_id}`

Change the partition type to `Linux LVM` for the root partition using `t` command. (Type `44` for LVM).

Write the changes to the disk using `w` command. This will apply the changes, be careful and fourple check the changes before writing.

Enable swap partition if you created it:

```bash
mkswap -L "Linux Swap" /dev/{swap_partition_id}
swapon /dev/{swap_partition_id}
```

### 3. Format the partitions

```bash
mkfs.fat -F32 /dev/{efi_partition_id}
mkfs.ext4 /dev/{boot_partition_id}
```

### 4. Encrypt the root partition & create LVM

```bash
cryptsetup luksFormat /dev/{main_partition_id}
cryptsetup open --type luks /dev/{main_partition_id} lvm

pvcreate /dev/mapper/lvm
vgcreate vg0 /dev/mapper/lvm

lvcreate -L 30G vg0 -n lv_root
lvcreate -l 100%FREE vg0 -n lv_home
```

List volume groups: `vgdisplay`
List logical volumes: `lvdisplay`

Insert `dm_mod` kernel module

```bash
modprobe dm_mod
```

Scan for available volume groups & activate them

```bash
vgscan
vgchange -ay
```

### 5. Format Logical Volumes

```bash
mkfs.ext4 /dev/vg0/lv_root
mkfs.ext4 /dev/vg0/lv_home
```

### 6. Mount the partitions

```bash
mount /dev/vg0/lv_root /mnt
mkdir /mnt/boot
mount /dev/{boot_partition_id} /mnt/boot
mkdir /mnt/home
mount /dev/vg0/lv_home /mnt/home
```

### 7. Install the base system

```bash
pacstrap -i /mnt base
```

### 8. Generate fstab

```bash
genfstab -U /mnt >> /mnt/etc/fstab
```

## Setting up In-progress installation

### 1. Chroot into the installation

```bash
arch-chroot /mnt
```

### 2. Set up users

```bash
# Set root password
passwd
# Add a user
useradd -m -g users -G wheel -s /bin/bash {username}
# Set user password
passwd {username}
```

### 3. Install necessary packages

```bash
pacman -S base-devel dosfstools grub efibootmgr networkmanager gnome gnome-tweaks lvm2 mtools nano openssh os-prober sudo
```

### 4. Install linux kernel

```bash
pacman -S linux linux-headers linux-firmware
# Or if you want to install LTS kernel as well (may be useful if main kernel will not boot)
pacman -S linux linux-headers linux-lts linux-lts-headers linux-firmware
```

### 5. Installing Video Drivers

#### NVIDIA

If you installed the LTS kernel, you will also need to install the `nvidia-lts` package.

```bash
pacman -S nvidia nvidia-utils nvidia-lts
```

Update the `mkinitcpio` configuration:

```bash
nano /etc/mkinitcpio.conf
```

Find the `MODULES` line and add `nvidia` to it. It should look like this:

```bash
MODULES=(nvidia nvidia_modeset nvidia_uvm nvidia_drm)
```

Update the `grub` configuration:

```bash
nano /etc/default/grub
```

Find the `GRUB_CMDLINE_LINUX_DEFAULT` line and add `nvidia-drm.modeset=1` to it, before `quit`. It should look like this:

```bash
GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 nvidia_drm.modeset=1 quiet"
```

You may also need to modify `/etc/modprobe.d/nvidia.conf`:

```bash
nano /etc/modprobe.d/nvidia.conf
```

to include the following:

```bash
options nvidia-drm modeset=1 fbdev=1
```

#### AMD / INTEL

```bash
# For Intel
pacman -S mesa intel-media-driver

# For AMD
pacman -S mesa libva-mesa-driver
```

### 6. Generating Ram Disks for our Kernels

#### Configure `mkinitcpio`

```bash
nano /etc/mkinitcpio.conf
```

Find the `HOOKS` line and add `encrypt` & `lvm2` before `filesystems`. It should look like this:

```bash
HOOKS=(base udev autodetect microcode modconf kms keyboard keymap consolefont block encrypt lvm2 filesystems fsck)
```

Rebuild the `initramfs` image for your kernel(s):

```bash
mkinitcpio -p linux
```

If you're using the LTS kernel, you will need to rebuild the `initramfs` image for it as well:

```bash
mkinitcpio -p linux-lts
```

Ignore any Warnings.

### 7. Let's configure time zones and locales

#### Locale

Edit the locale file and uncomment the locales you want to use:

```bash
nano /etc/locale.gen
```

Generate the locales:

```bash
locale-gen
```

Set the locale:

```bash
echo "LANG=en_US.UTF-8" > /etc/locale.conf
```

#### Timezone

```bash
ln -sf /usr/share/zoneinfo/{Region}/{City} /etc/localtime
hwclock --systohc
```

### 8. Set the Hostname & Hosts File

```bash
echo "{hostname}" > /etc/hostname
```

Edit the hosts file:

```bash
nano /etc/hosts
```

Add the following lines:

```bash
127.0.0.1   localhost
::1         localhost
127.0.1.1   {hostname}.localdomain {hostname}
```

### 9. Configure Grub

Edit the grub configuration file:
    
```bash
nano /etc/default/grub
```

Find the `GRUB_CMDLINE_LINUX_DEFAULT` line and add `cryptdevice=/dev/{main_partition_id}:vg0` to it, before `quiet`. It should look like this:

```bash
GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 cryptdevice=/dev/{main_partition_id}:vg0 quiet"
```

Nvidia users should also add `nvidia-drm.modeset=1` to the line if they haven't already.

Mount the EFI partition:

```bash
mkdir /boot/efi
mount /dev/{efi_partition_id} /boot/efi
```

Install grub:

```bash
grub-install --target=x86_64-efi --bootloader-id=grub_uefi --recheck --removable /dev/{disk_id}
```

Copy grub locale:

```bash
cp /usr/share/locale/en\@quot/LC_MESSAGES/grub.mo /boot/grub/locale/en.mo
```

Make grub configuration:

```bash
grub-mkconfig -o /boot/grub/grub.cfg
```

### 11. Enable Services

```bash
systemctl enable gdm
systemctl enable NetworkManager
systemctl enable sshd
# Enable everything else that you need
```

**FOR NVIDIA USERS:** If you have an NVIDIA card and GDM won’t start, you must run this command:

```bash
ln -s /dev/null /etc/udev/rules.d/61-gdm.rules
```

### 12. Exit & Reboot
```bash
exit
```

```bash
umount -R /mnt
reboot
```

## Post-Installation Steps

I will cover the post-installation steps in a separate guide.

You might want to edit the `sudoers` file to allow your user to use `sudo` without entering a password. This is optional, but it can be convenient.

```bash
nano /etc/sudoers
```

Just find the `wheel` group and uncomment that line.

```bash
## Uncomment to allow members of group wheel to execute any command
%wheel ALL=(ALL:ALL) ALL
 ```

## Youtube guide

If you prefer a video guide, It will be available soon.