# Installing Arch Linux Alongside a Pre-Installed OS

**#manuall** **#dualboot** **#gnome+gdm** **#encrypted** **#lvm** **#nvidia-friendly**

This guide will help you manually install Arch Linux alongside an existing OS (without wiping any drives). In my case, I'm already using Arch on my primary machine and don’t want to reconfigure it, so I will be installing it on my gaming PC alongside a Windows 11 installation.

## Pre-Installation Steps

### 1. Verify Your Existing OS Boot Mode

- Boot into your current operating system.
- Find the system information.
- Ensure that the BIOS mode is set to UEFI.

### 2. Disable Secure Boot

- Reboot your PC and enter the UEFI Firmware settings.
- Depending on your motherboard, navigate to the security section and find the "Secure Boot" option. Disable it.
- It is also recommended to disable Fast Boot for a smoother installation process.

### 3. Create Unallocated Space on Your Disk

- If you have an empty drive that you plan to use for Arch, you can skip this step.
- In your existing OS, open **Disk Management**.
- Select the disk you want to install Arch on.
- Shrink the partition on this disk. It is recommended to allocate at least 200 GB of unallocated space if you intend to use Arch for work or gaming.
- You should now have unallocated space available for the Arch installation.

### 4. Create a Bootable USB

- Download the Arch Linux `.iso` file from a trusted source. [The official website](https://www.archlinux.org/download) is a good option.
- Use a reliable tool to create a bootable USB drive. I recommend **Rufus** for Windows users.
- Plug in a new USB stick and open Rufus.
- Select your USB drive and the downloaded Arch Linux `.iso` file.
- Choose the **GPT** partition scheme and the **FAT32** file system.
- Press **Start** to begin creating the bootable USB.

## Installing Arch Linux

### 1. Boot from the USB

- Plug the bootable USB into your PC.
- [Windows] Hold down the **Shift** key while clicking **Restart**. Then, choose **Troubleshoot** > **Advanced Options** > **UEFI Firmware Settings**, and click **Restart**.
- [Any OS] Alternatively, reboot and press your boot menu key (e.g., **F8** or **F12**, depending on your motherboard) while booting.
- In the Boot menu, select the USB drive and press **Enter**.

### 2. Enter the Arch Linux Installer

- When you see the Arch Linux boot menu, select the default option to start the installation. You can simply press **Enter**.

### 3. Connect to the Network

To begin, you'll need a network connection. Check if your system has an IP address by typing:

```bash
ip addr show
```

If you see an IP address listed, you're good to go. If not, you'll need to connect via Wi-Fi or Ethernet.

### 2. Identify the Disk for Installation

To find the disk you'll be using for the installation, use the following command:

```bash
lsblk
```

This will list all the storage volumes attached to your system. Identify the disk where you want to install Arch Linux. For example, if your disk is labeled `/dev/nvme0n1`, that's the one you'll target.

**Note:** Be sure to choose the correct disk. You can check the size column in the output of `lsblk` to confirm the drive you're targeting.

**In my case:** It will be `/dev/nvme1n1` where I have a 1TB SSD drive, with only 600GB already partitioned as my Steam storage for Windows 11.

### 3. Partition the Disk

Now that you've identified your disk, it's time to partition it using `fdisk`. To start `fdisk`, use this command:

```bash
fdisk /dev/{disk_id}
```

Replace `{disk_id}` with the correct disk identifier for your system. We'll be wiping the existing partitions and creating new ones.

**In my case:**

```bash
fdisk /dev/nvme1n1
```

- Press `p` to print the current partition table (this is just to see what’s currently there).
- Press `m` to print all the available commands (basically a manual).
- Press `g` to create a new GPT partition table. (This will *fucking erase your disk*, I warned you.)

    **WARNING:** If you want to erase your disk and create partitions from scratch - it's your choice, but think wisely. If you accidentally hit `g`, don't panic. It will not apply until you write the changes, so you can safely leave (`^C`).

- Press `n` to create a new partition. You will be prompted for partition numbers, sizes, etc.
  - **Partition 1**: This will be for the **swap** partition. (See note.)
  - **Partition 2**: This will be for the **EFI** partition (1 GB).
  - **Partition 3**: This will be for the **boot** partition (2 GB).
  - **Partition 4**: The remaining space will be used for **LVM** (Logical Volume Manager), which will hold the rest of the system.

    **NOTE:** Swap partition isn't necessary, but recommended, especially if you don't have a lot of RAM. In my case: I have 64GB of RAM, but I'm still going to create a swap of 32GB. Rule of thumb is to create at least 1x of your RAM, but if you have 64GB or more, you don’t need to burn that much disk space. Just use 1/2 of your RAM size.

Once you've finished partitioning, type `t` to change the partition type:

- Set **Partition 4** to type **44** (Linux LVM).

### 4. Write the Changes

**WARNING:** Before writing the changes, I highly recommend you press `p` a bunch of times and *fourple* check that everything is okay. There is no way back after you write your changes, and if you did something wrong, just go and cry. You were warned.

Now that the partitions are created, write the changes to disk: `w`.
This will delete the existing partition table and apply your new partitions.

### 5. Create SWAP Space

- Use `gdisk -l /dev/{disk_id}` to verify the partition table and take note of the partition number assigned to Swap.
- Enable swap by typing:

```bash
mkswap -L "Linux Swap" /dev/{swap_partition_id}
swapon /dev/{swap_partition_id}
```

- Verify status of swap space by typing:

```bash
free -m
```

If the last line starts with "Swap:", we're good.

### Future Disk/Partition namings

- **Disk**: Will be referred as Disk: `{disk_id}`.
- **Partition 1**: Will be referred as Swap Partition: `{swap_partition_id}`.
- **Partition 2**: Will be referred as EFI Partition: `{efi_partition_id}`.
- **Partition 3**: Will be referred as Boot Partition: `{boot_partition_id}`.
- **Partition 4**: Will be referred as Main Partition: `{main_partition_id}`. I don't want to call it `root` so there will be more visual difference between `boot` and `main`. Huh, got you covered.

**In my case:**

- **disk_id** is `/dev/nvme1n1`
- **swap_partition_id** is `/dev/nvme1n1p3`
- **efi_partition_id** is `/dev/nvme1n1p4`
- **boot_partition_id** is `/dev/nvme1n1p5`
- **main_partition_id** is `/dev/nvme1n1p6`

### 6. Format the Partitions

Format the EFI Partition:

```bash
mkfs.fat -F32 /dev/{efi_partition_id}
```

Now format the Boot Partition:

```bash
mkfs.ext4 /dev/{boot_partition_id}
```

### 7. Encrypt and Set Up LVM on Main Partition

Encrypt the third partition:

```bash
cryptsetup luksFormat /dev/{main_partition_id}
```

It will prompt you to type `YES` to confirm the operation.
It will then prompt you to enter and verify a passphrase. Make sure you don’t lose or forget it—if you do, well, just cry, I guess. I don’t know.

Unlock the partition for further usage:

```bash
cryptsetup open --type luks /dev/{main_partition_id} lvm
```

Next, create the LVM physical volume:

```bash
pvcreate /dev/mapper/lvm
```

Then, create a volume group:

```bash
vgcreate vg0 /dev/mapper/lvm
```

Finally, create logical volumes for root (`30G`) and home (`remaining space`):

```bash
lvcreate -L 30G vg0 -n lv_root
lvcreate -l 100%FREE vg0 -n lv_home
```

To verify everything, list the volume groups:

```bash
vgdisplay
```

Then, list the logical volumes:

```bash
lvdisplay
```

Ensure that the `vg0` volume group and the `lv_root` and `lv_home` logical volumes are present.

Insert `dm_mod` kernel module

```bash
modprobe dm_mod
```

Scan for available volume groups

```bash
vgscan
```

Activate available volume groups

```bash
vgchange -ay
```

### 8. Format Logical Volumes

Format the root partition:

```bash
mkfs.ext4 /dev/vg0/lv_root
```

Format the home partition:

```bash
mkfs.ext4 /dev/vg0/lv_home
```

### 9. Mount the Partitions

Mount the partitions:

```bash
mount /dev/vg0/lv_root /mnt
mkdir /mnt/boot
mount /dev/{boot_partition_id} /mnt/boot
mkdir /mnt/home
mount /dev/vg0/lv_home /mnt/home
```

### 10. Install Arch Linux Base System

Install the base system packages:

```bash
pacstrap -i /mnt base
```

### 11. Generate the File System Table (fstab)

Generate an `fstab` file:

```bash
genfstab -U /mnt >> /mnt/etc/fstab
```

Verify your fstab file:

```bash
cat /mnt/etc/fstab
```

You should have 3 partitions listed here. `/`, `/boot`, `/home`.

## Setting up In-progress installation

### 1. Chroot into In-progress installation

Change root into the new installation to configure it further:

```bash
arch-chroot /mnt
```

Your command prompt will change — don’t worry, that’s normal. It may not be colorful or fancy anymore, but we’ll fix that later.

We're now inside your in-progress installation, so technically, this is already your Arch system—it just won’t function on its own yet. Let’s fix that.

### 2. Set up users

#### Set the root password

```bash
passwd
```

Make sure to set a strong password for the root user. It's the most important account on your system.

#### Create a new user

`-m` creates the user's home directory.
`-g` is the primary group
`-G` is the secondary group
`-s` is the shell.

```bash
useradd -m -g users -G wheel -s /bin/bash {username}
```

Replace `{username}` with your desired username.
It is recommended to add the new user to the `wheel` group to allow them to use `sudo`.

#### Set the password for the new user

```bash
passwd {username}
```

Please don't use the same password as the root user. It's not a good practice.

### 3. Install Required Packages

```bash
pacman -S base-devel dosfstools grub efibootmgr networkmanager gnome gnome-tweaks lvm2 mtools nano openssh os-prober sudo
```

- `base-devel` includes the necessary tools for compiling and building packages from the Arch User Repository (AUR).
- `dosfstools` is required for managing FAT filesystems.
- `grub` is the bootloader.
- `efibootmgr` is required for UEFI systems.
- `networkmanager` is a network connection manager.
- `gnome` is the GNOME desktop environment.
- `gnome-tweaks` is a tool for customizing GNOME.
- `lvm2` is the Logical Volume Manager, system needs this to boot properly on LVM.
- `mtools` is required for managing FAT filesystems.
- `nano` is a simple text editor. You can replace it with `vim` if you prefer. You can also go ahead and install `neovim` if you're a cool kid, then spend the next 3 days configuring it. I'll do that later, in the post-installation steps.
- `openssh` is the OpenSSH server.
- `os-prober` is required for detecting other operating systems.
- `sudo` is the superuser command, so users can run commands as root.

Just press `Enter` couple of times to install all the packages. It will take a while.

### 4. Enabling SSH

If you installed the `openssh` package, you probably want to enable the SSH server:

```bash
systemctl enable sshd
```

### 5. Install linux kernel

We are currently missing the kernel, so let's install it:

```bash
pacman -S linux linux-headers
```

You may also want to install the `linux-lts` package for a more stable kernel. In case you have any issues with the latest kernel, you can boot into the LTS kernel.

```bash
pacman -S linux-lts linux-lts-headers
```

You will probably need to install `linux-firmware` package as well:

```bash
pacman -S linux-firmware
```

### 6. Installing Video Drivers

Determining the correct video drivers for your system is crucial. If you don't know which VGA device you have, you can use the `lspci` command to list all PCI devices. `grep VGA` will filter the list to show only VGA devices.

```bash
lspci | grep VGA
```

You will see the VGA device listed. In my case, it's an NVIDIA card.

```bash
08:00.0 VGA compatible controller: NVIDIA Corporation AD103 [GeForce RTX 4080] (rev a1)
```

#### NVIDIA

If you have an NVIDIA card, you will need to install the proprietary drivers. You can do this by installing the `nvidia` & `nvidia-utils` package:

If you installed the LTS kernel, you will also need to install the `nvidia-lts` package.

```bash
pacman -S nvidia nvidia-utils nvidia-lts
```

I will also need to update my `mkinitcpio` to include the `nvidia` module:

```bash
nano /etc/mkinitcpio.conf

# In order to save the file, press `Ctrl + O`, then `Enter`. To exit, press `Ctrl + X`.
```

Find the `MODULES` line and add `nvidia` to it. It should look like this:

```bash
MODULES=(nvidia nvidia_modeset nvidia_uvm nvidia_drm)
```

We will skip the rebuilding of the `initramfs` image for now, as we will need to configure it further and then we can rebuild it.

<!-- Rebuild the `initramfs` image:

```bash
mkinitcpio -p linux
```

If you're using the LTS kernel, you will need to rebuild the `initramfs` image for it as well:

```bash
mkinitcpio -p linux-lts
``` -->

I also need to update my `grub` configuration to include the `nvidia` module:

```bash
nano /etc/default/grub

# In order to save the file, press `Ctrl + O`, then `Enter`. To exit, press `Ctrl + X`.
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

We also need to run this command, or GDM won’t start:

**WAIT:** We need to install and enable gdm first! I’ll cover that in the upcoming steps. Skip for now, it will remain here for troubleshooting purposes.

```bash
ln -s /dev/null /etc/udev/rules.d/61-gdm.rules
```

<!-- **TROUBLE SHOOTING:** If you have an NVIDIA card and after booting all you see is a black screen with a static underscore in the top-left corner, run this magic command: -->

#### AMD / INTEL

You will need to install `mesa` and `intel-media-driver` or `libva-mesa-driver` according to your hardware:

```bash
# For Intel
pacman -S mesa intel-media-driver

# For AMD
pacman -S mesa libva-mesa-driver
```

### 7. Generating Ram Disks for our Kernels

We need to regenerate the `initramfs` image for our kernels, but before that, we need to configure it.

#### Configure `mkinitcpio`

```bash
nano /etc/mkinitcpio.conf

# In order to save the file, press `Ctrl + O`, then `Enter`. To exit, press `Ctrl + X`.
```

Find the `HOOKS` line and add `encrypt` & `lvm2` before `filesystems`. It should look like this:

```bash
HOOKS=(base udev autodetect microcode modconf kms keyboard keymap consolefont block filesystems fsck)
->
HOOKS=(base udev autodetect microcode modconf kms keyboard keymap consolefont block encrypt lvm2 filesystems fsck)
```

Otherwise, you will not be able to boot into your system, because it won't be able to decrypt the encrypted partition. And without `lvm2`, it won't be able to find the logical volumes. So, it's crucial to have them in the correct order.

Rebuild the `initramfs` image for your kernel(s):

```bash
mkinitcpio -p linux
```

If you're using the LTS kernel, you will need to rebuild the `initramfs` image for it as well:

```bash
mkinitcpio -p linux-lts
```

Make sure you see `lvm2` and `encrypt` in the output. If you don't, you probably forgot to add them to the `HOOKS` line or save the file. Go back and check.

Ignore any Warnings, they are just warnings. We don't care about them, right?

### 8. Let's configure time zones and locales

#### Locale

Edit the locale file:

```bash
nano /etc/locale.gen
```

Uncomment the locales you want to use. For example, if you want to use `en_US.UTF-8`, uncomment this line:

```bash
en_US.UTF-8 UTF-8
```

Uncommenting means removing the `#` at the beginning of the line.

Generate the locales:

```bash
locale-gen
```

#### Time Zone

List the available regions:

```bash
ls -l /usr/share/zoneinfo
```

Choose your zone and list the available cities:

```bash
ls -l /usr/share/zoneinfo/{region}
```

Set the time zone:

```bash
ln -sf /usr/share/zoneinfo/{region}/{city} /etc/localtime
```

Sync the hardware clock:

```bash
hwclock --systohc
```

**In my case:**

```bash
ln -sf /usr/share/zoneinfo/Asia/Almaty /etc/localtime
```

### 9. Set the Hostname & Hosts File

Edit the hostname file:

```bash
nano /etc/hostname
```

Set your hostname. For example:

```bash
my-arch-pc-btw
```

Edit the hosts file:

```bash
nano /etc/hosts
```

Add the following lines:

```bash
127.0.0.1   localhost
::1         localhost
127.0.1.1   my-arch-pc-btw.localdomain my-arch-pc-btw
```

Replace `my-arch-pc-btw` with your hostname. In my case, it will be `gingerarch`.

### 10. Configure Grub

Edit the grub configuration file:

```bash
nano /etc/default/grub
```

Find the `GRUB_CMDLINE_LINUX_DEFAULT` line and add `cryptdevice=/dev/{main_partition_id}:vg0` to it, before `quiet`. It should look like this:

```bash
GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 cryptdevice=/dev/{main_partition_id}:vg0 quiet"
```

**In my case:**

```bash
GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 nvidia_drm.modeset=1 cryptdevice=/dev/nvme1n1p6:vg0 quiet"
```

Nvidia users should also add `nvidia-drm.modeset=1` to the line if they haven't already.

Mount the EFI partition:

```bash
mkdir /boot/efi
mount /dev/{efi_partition_id} /boot/efi
```

**In my case:**

```bash
mkdir /boot/efi
mount /dev/nvme1n1p4 /boot/efi
```

Install grub:

```bash
grub-install --target=x86_64-efi --bootloader-id=grub_uefi --recheck --removable /dev/{disk_id}
```

**In my case:**

```bash
grub-install --target=x86_64-efi --bootloader-id=grub_uefi --recheck --removable /dev/nvme1n1
```

Copy grub locale:

```bash
cp /usr/share/locale/en\@quot/LC_MESSAGES/grub.mo /boot/grub/locale/en.mo
```

Make grub configuration:

```bash
grub-mkconfig -o /boot/grub/grub.cfg
```

### 11. Enable GDM

Enable the GDM service:

```bash
systemctl enable gdm
```

IF you ended up not having gdm installed, you can install it by running:

```bash
pacman -S gdm
```

**FOR NVIDIA USERS:** If you have an NVIDIA card and GDM won’t start, you must run this command:

```bash
ln -s /dev/null /etc/udev/rules.d/61-gdm.rules
```

### 12. Enable NetworkManager

Enable the NetworkManager service:

```bash
systemctl enable NetworkManager
```

### 13. Exit Chroot

Exit the chroot environment:

```bash
exit
```

### 14. Unmount Partitions

Unmount the partitions:

```bash
umount -R /mnt
```

Ignores any errors, they are just errors. We don't care about them, right?

### 15. Reboot

Reboot your system:

```bash
reboot
```

Don't forget to remove the USB stick from your PC.
You might also need to open your BOOT menu and select the correct boot device in some cases.

It should ask you for the passphrase you set up for the encrypted partition. If you don't see the passphrase prompt, you probably forgot to add `cryptdevice=/dev/{main_partition_id}:vg0` to the `GRUB_CMDLINE_LINUX_DEFAULT` line in the grub configuration file.

The system should boot into the GDM login screen. You can now log in with your user account and start customizing your Arch Linux installation.

If on the GDM login screen you see a black screen with an underscore, you probably forgot to run the `ln -s /dev/null /etc/udev/rules.d/61-gdm.rules` command. Or any other steps in the NVIDIA section.

## Post-Installation Steps

I will cover the post-installation steps in a separate guide. This guide is already too long, and I don't want to overwhelm you with too much information at once. I will cover the following topics in the post-installation guide that will be available soon.

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

<!-- If you prefer a video guide, you can watch my YouTube video on this topic: -->

<!-- [![Installing Arch Linux Alongside a Pre-Installed OS](https://img.youtube.com/vi/VIDEO-ID/0.jpg)](https://www.youtube.com/watch?v=VIDEO-ID) -->