*Introduction*
> Today, I am going to be going walking through the installation process of Archlinux. This is my Archlinux installation tutorial. There are many Archlinux installation tutorials like it, but this one is mine.
> This tutorial will be aimed at producing an Archlinux installation with a high degree of security, including features such as full disk encryption, a hardened kernel, and a detached boot partition. We will be using UEFI with GRUB for our boot setup, LVM on LUKS for encryption and logical volume management, and BTRFS for our filesystems. In a follow up video, I will also be demonstrating how to configure secure boot on installed operating system. It is left as an exercise for the viewer to decide whether these security features are necessary for their own use-case and threat model.

*Prerequisites*
> To follow this tutorial, you will need, naturally, a computer on which to install Archlinux, as well as at least two USB drives, or some other type of external drive. One of these drives will be used for your detached boot partition, while the other will be used for the installer. There are numerous instructions available for how to create an installer, and it will depend somewhat on your starting platform, so I won't be covering it here.

*Drive Preparation*
> Once we've booted up the installer, our first step is to prepare our drives for installation. You can use the `lsblk` to view disk and partition information on the system. Since I am on a virtual machine, your disk names will probably look slightly different than mine. The important thing is to identify which disk is your device's main drive, which is the installer drive, and which is the drive for the detached boot partition. It should be obvious based on the size of the respective drives which is the system's main hard drive. As for the installer, you can tell based on whether it has any mount points listed under it.
> In my case, my main drive is `/dev/vda`, while my detached boot drive is `/dev/vdb`. The next thing we are going to do is run the `shred` command to overwrite each disk with random data:
```sh
shred --random-source=/dev/urandom -n1 --zero /dev/vda
shred --random-source=/dev/urandom -n1 --zero /dev/vdb
```
> This step is optional, and can possibly take a few hours if your drive is upwards of a terabyte in size, so I would recommend starting this command and letting it run overnight. There are two reasons for doing this. The first is to prevent any leftover data on the drives from being easily recovered. The second is to help obfuscate the drive's contents once we complete our installation. You see, even though full disk encryption can protect data written to a disk, it does nothing to hide where that data is being written. This means that if we were to, for example, install Archlinux on a completely fresh disk, there would be a sort of "silhouette" created in the disk sectors, allowing an attacker to at least identify the presence of an encrypted filesystem stored on the disk. It may even be possible to identify what kind of filesystem and even what operating system distribution is installed, based upon file offsets. By filling the disk with random data befpre, it becomes almost impossible for an attacker to distinguish encrypted filesystem contents from noise.

*Partitioning the Disks*
> Next we'll be partitioning the disks, using the `fdisk` utility. First up is the detached boot disk. We'll press 'g' create a new GPT partition table, and then press `n` to create a new partition. This will be our EFI System Partition, also called an ESP. We'll keep the default number and starting sector, and respond with `+256M` for the end sector, to give ourselves just a bit of wiggle room for the future. Now that we've defined the partition, we want to press `t` to set the partition type to EFI System, which is partition type 1. Once that's done we'll press `n` again to make a second partition, again using the default partition number, and keeping the defaults for the start and end sectors so that the partition takes up all the remaining space on the drive. 
> Now, in this tutorial I will be referring to the first partition as the EFI system partition or the ESP, and to the second partition as the boot partition. Some tutorials will not have a separate boot partition, and therefore refer to the EFI system partition and the boot partition interchangeably. But, in this tutorial, they are separate and distinct.
> Now that we have these two partitions created, we'll finally press `w` to save our changes to the disk.
> The partition for the main disk is not strictly necessary, since we're using LUKS, but I will go through it anyways for consistency. This time we run `fdisk` against the main disk. Again we use `g` to generate a GPT partition table, `n` to create a partition, and then accept all the defaults to have it fill up the entirety of the disk. There is no specific partition type for LUKS, so leaving it as a Linux filesystem partition is fine. Lastly we use `w` to save our changes.

```sh
fdisk /dev/vdb # Set up partitions of the detached boot USB
	g # New disk label
	n # New partition
	1 # Set partition number
	(default) # start sector
	+256M # last sector
	
	t # Change partition type
	1 # Set to EFI system
	
	n # New partition
	2 # Set partition number
	(default) # Start sector
	(default) # Last sector
	
	w

fdisk /dev/vda # Set up partitions of main disk
	g # New disk label
	n # New partition
	1 # Set partition number
	(defualt) # Start sector
	(default) # Last sector
	w # Write
```

*Boot Drive Setup*
> Now we'll set up the filesystems on the detached boot drive. The filesystem for the EFI system partition must be FAT32. We can create it with the command, `mkfs -t fat -F 32`, followed by the path to the partition, in my case `/dev/vdb1`. The filesystem type of the boot is not as critical, but I typically use `ext4` since it is broadly supported. For this the command is `mkfs -t ext4 /dev/vdb2`.

```sh
mkfs -t fat -F 32 /dev/vdb1
mkfs -t ext4 /dev/vdb2
```

*Disk Encryption Setup*
> Now that we've created the filesystems for our detached boot drive, we can begin setting up disk encryption on our main drive. The first thing we want to do is mount the boot partition. Again, this is the boot partition, not the EFI system partition. In my case, it is `/dev/vdb2`. So I will mount it with the command `mount /dev/vdb2 /mnt`. Once this partition has been mounted, I will use the following command to create a 16-megabyte file inside the boot partition.

```sh
mount /dev/vdb2 /mnt0
```

> This file will be used the LUKS header for our main disk. Without going into too much detail about the design of LUKS-- which stands for Linux Unified Key Setup-- the header file is a critical part of full disk encryption. If the header file is lost, it is impossible to decrypt the drive; all the data is as good as lost. Additionally, the header file is also the only part of a LUKS-encrypted disk that is distinguishable from random data.
> Because of this, we are going to keep the header file stored on our detached boot drive from the moment it is created. This way even in the event of unexpected power failure during the installation process, it won't be lost. I do, however, strongly recommend creating a backup of the entire boot drive when the process is complete, just in case that drive is ever lost or fails.

```sh
dd if=/dev/zero of=/mnt/header.img bs=16M count=1
```

> Now, in order to actually set up LUKS, we will use this command: `cryptsetup luksformat --header /mnt/header.img --cipher serpent-xts-plain64 --key-size 512 --hash whirlpool /dev/vda1`. Let's break this down slightly. Here we are specifying two non-default algorithms, `serpent-xts-plain64` and `whirlpool`, for our disk encryption and hashing respectively. Both of these algorithms are cryptographically more resilient than their standardized counterparts, Rijndael and SHA-256, but they are notably slower in terms of performance. Again, I would urge the viewer to consider for themselves whether this is the right choice.
> Once we run this command we will be prompted for a password. For the purposes of this tutorial, I am going to use the password, 'password'. I'll leave it up to you to come up with your own strong password and store it safely. Personally, my method is to get a dictionary and roll some dice to select a page and then a word on page, and repeat that until I have at least five or six words, and I write them down on at least two different index cards.
```
cryptsetup luksFormat --header /mnt/header.img --cipher serpent-xts-plain64 --key-size 512 --hash whirlpool /dev/vda1
```
> Once the `luksFormat` command is complete, we now have to unlock the encrypted disk. To do this we will use the command `cryptsetup open --header /mnt/header.img /dev/vda1 crypt0`. Note that we have to specify the header here; if you don't use the header from the previous step, this command will not work. Here, `crypt0` is the name of the unlocked device. That choice is just a convention, in principle you could use anything.
```sh
cryptsetup open --header /mnt/header.img /dev/vda1 crypt0
```

> If run the `lsblk` command again, we will see a very different picture from the first time.
```sh
lsblk
```

*LVM Setup*
> Now we can start setting up LVM. LVM, or Logical Volume Manager, is a device mapper framework that enables the creation of logical volumes. Logical volumes are useful because easily be resized, and they can span over multiple physical devices. Generally, they're a good choice for future-proofing your installation, and they can save you a lot time and headaches down the line.
> To set up LVM, we'll start by running the `pvcreate` command against our unlocked device, which available at `/dev/mapper/crypt0`. This marks the device as a physical volume. Next we'll use `vgcreate` to create a volume group on that physical volume named `volgroup0`. Again, this named is just a convention and in practice you can call it whatever you'd like.

```sh
pvcreate /dev/mapper/crypt0
vgcreate volgroup0 /dev/mapper/crypt0
```

> Once we've created our volume group, we'll create two logical volumes inside of it with the `lvcreate` command. The first one will be our swap volume, which I'll name `lv_swap`. The size of this is going to depend on the amount of free space on your hard drive and the amount of RAM on your system. I tend to create my swap volumes of about equal size to my RAM, but this tutorial I will be keeping it for 4GB.
> Next we'll create our root volume. This is where almost our entire installation will live. For this command I'll use the `-l 100%FREE` option to take up all the remaining space in the volume group. Note that this is different from the `-L` flag used in the previous command.

```
lvcreate -L 4G volgroup0 -n lv_swap
lvcreate -l 100%FREE volgroup0 -n lv_root
```

*Main Disk Filesystem Setup*
>Now we'll be setting up the filesystems on our logical volumes. For the swap volume, this is as simple as running `mkswap` on the volume, which is available at `/dev/volgroup0/lv_swap`.
>For our root volume, we will be using a BTRFS filesystem, meaning the setup is slightly more involved. BTRFS, also pronounced "better FS" and "butter FS", is a copy-on-write filesystem. One if its main features is the ability to define different subvolumes and take snapshots of them separately. Unlike LVM logical volumes, these subvolumes don't need to have a defined size, allowing them to grow organically.

```sh
mkswap /dev/volgroup0/lv_swap
mkfs -t btrfs /dev/volgroup0/lv_root
```

>Once we use the `mkfs` command to create the BTRFS filesystem on our root volume, we can start setting up our subvolumes. The first step is to actually mount the filesystem. Since we're done with the boot partition for now, we can unmount that with `umount /mnt`, and then use that as the mount point for the root volume.
```sh
umount /mnt
mount /dev/volgroup0/lv_root /mnt
```

>After the root volume is mounted, we'll create our subvolumes with the `btrfs subvolume create` command. Chosing which directories should be subvolumes is going to be somewhat dependent upon your use-case, but I usually have one subvolume for my root directory, and another subvolume for my home directory. Because I plan to install the Nix daemon on this system in a later video in this series, I will also create a `/nix` subvolume.
```sh

btrfs subvolume create /mnt/root
btrfs subvolume create /mnt/nix
btrfs subvolume create /mnt/home
```
>Once all the subvolumes are created, we now have to unmount the filesystem. Next we can mount the root subvolume by passing the `subvol=root` option to the mount command. We'll also add the `compress=zstd` option to use real-time compression.

```sh
umount /mnt
mount -o compress=zstd,subvol=root /dev/volgroup0/lv_root /mnt
```

>This marks the first point where we are able to interact with what will become our new Archlinux installation. It is completely empty at a moment, however, so we'll need to create some directories with the `mkdir` command. An easy way to create multiple directories at once is with this syntax.
```sh
mkdir /mnt/{boot,home,nix}
```

>Now we can mount the `/home` and `/nix` subvolumes onto their mount points, using the same `subvol` option as before. We can also mount our boot partition, and swap on our swap volume with the `swapon` command.
```sh
mount -o compress=zstd,subvol=home /dev/volgroup0/lv_root /mnt/home
mount -o compress=zstd,subvol=nix /dev/volgroup0/lv_root /mnt/nix
mount /dev/vdb2 /mnt/boot
swapon /dev/volgroup0/lv_swap
```
>For the EFI system partition, we'll create another mount point under `/mnt/boot`, and mount it there. 
```sh
mkdir /mnt/boot/esp
mount /dev/vdb1 /mnt/boot/esp
```
>Now let's run `lsblk` once more and see the full disk layout. This is the skeleton of our Archlinux installation. Now, it's time to start putting some meat on those bones.

*System Installation*
> To actually perform the system installation, we'll use the `pacstrap` command. This command sets up the bulk of our installation's directory structure, configuration files, and system packages. We invoke it by specifying the directory where our system is mounted, in our case `/mnt`, and then a list of the packages we want installed; `pacstrap` does the rest.
> For this tutorial, we'll be using `linux-hardened`, which is a package containing the Linux kernel compiled with a number of security-focused patches applied. To go along with it, we'll also be installing the `linux-hardened-headers` package, and the `linux-firmware` which contains firmware for a number of common hardware devices. I will also add the `base-devel` package, which contains the C compiler collection and a number of other libraries used for software development, and the `btrfs-progs` and `lvm2` packages, which contain utilities for interacting with BTRFS and LVM respectively, and the `git` and `nano` packages, which we will need later in this tutorial. Finally, we'll also add the `grub` and `efibootmgr` package, as these will be needed for our boot process.
```sh
pacstrap -K /mnt base linux-hardened linux-hardened-headers linux-firmware base-devel btrfs-progs lvm2 git nano grub efibootmgr
```
> Once this command completes, our next step is to use the `genfstab` command to generate the filesystem table for our installation. This is required to recreate the mounting layout that we have set up manually during this installation during the next boot.
```sh
genfstab -U > /mnt/etc/fstab
```

*Chrooting into the System*
> Now that the bootstrapping process is completed and we've generated the FS table, we can actually enter our newly-installed system using the `arch-chroot` command. This command, which is based on `chroot`, or "change root", changes the apparent root directory of a process. Here we are changing our root from the installer's root directory to our installation's root directory, giving us an environment similar to if we have booted from the installation itself.
```sh
arch-chroot /mnt
```
> Once we've changed root, we can begin making changes to the installed system directly. The first change we'll want to make is setting up the time and locale. We can set the local time zone of the installation by symlinking the appropriate local time file to the path `/etc/localtime`. In my case, I'll be symlinking the `America/New_York` file, since that's closest to my local time zone.
```sh
ln -sf /usr/share/zoneinfo/America/New_York /etc/localtime
```
> Afterwards we want to synchronize the system's clock to the hardware clock. We can do this with the command `hwclock --systohc`.
```sh
hwclock --systohc
```
> Next we want to generate the locale files with the `locale-gen` command. We can also set our language by populating the `/etc/locale.conf`. In my case I want the system language to be `en_US.UTF-8`, so I will simply echo that into the `local.conf` file.
```sh
locale-gen
echo 'LANG=en_US.UTF-8' > /etc/locale.conf
```
> Finally, we want to configure networking. First we'll set the system's hostname by echoing the desired name into the `/etc/hostname` file. I'll use `virtual-machine` for mine. Next we'll enable two SystemD services, `systemd-networkd` for network management and `systemd-resolved` for DNS resolution management. The last step is symlinking one of two files to the into the `/etc/systemd/network` directory depending on type of network connection. For ethernet, we'll symlink from `/usr/lib/systemd/network/89-ethernet.network.example` to the path `/etc/systemd/network/89-ethernet.network`. For wireless, we'll symlink `/usr/lib/systemd/network/80-wifi-station.network.example` to the path `/etc/systemd/network/80-wifi-station.network`.
```sh
echo 'virtual-machine' > /etc/hostname
systemctl enable systemd-networkd
systemctl enable systemd-resolved

# For ethernet
ln -s /usr/lib/systemd/network/89-ethernet.network.example /etc/systemd/network/89-ethernet.network
# For wifi
ln -s /usr/lib/systemd/network/80-wifi-station.network.example /etc/systemd/network/80-wifi-station.network
```

*Bootloader Installation*
> Because of our detached boot drive, and particularly because of our detached LUKS header file, the boot process of our system will be somewhat unorthodox, and even the standard boot process is not trivial, so let's take a brief moment for review.
> When the machine is turned on, the very first program to run in the firmware. In our case, this is UEFI, or the Unified Extensible Firmware Interface. The firmware then starts the system's bootloader. In UEFI, the bootloader is an EFI application found in the EFI system partition. Our configuration will use GRUB, the Grand Unified Bootloader, as the bootloader. Next, the bootloader loads the kernel into memory, followed by the initial RAM-based filesystem, or `initramfs`. The `initramfs` provides any files or kernel modules that required to proceed to the next stage of the boot process. It is at this point, usually, that the main disk of the system is unlocked, if it is encrypted. Once the main disk is decrypted and the root filesystem is mounted, the system's initialization proceeds from there.
> For our system to work need to make sure that our `initramfs` has access to the header file so it can unlock the main disk of our operating system. On Archlinux, the `initramfs` is typically generated with a utility called `mkinitcpio`. This utility can be given user-defined hooks to embed additional packages and functions into the `initramfs`, which is how we'll be adding the ability for it to work with a detached LUKS header.
> To obtain the hook used in this process, we would typically install it from the Archlinux User Repository, or AUR, using an AUR helper like `yay` or `paru`. We don't have those available right now, though, so we'll just clone and build the package ourselves manually.
> When installing packages or software outside of Archlinux's package management, I usually use the `/opt` directory. In practice, however, the location you chose for this process doesn't matter. After changing to our desired directory, we'll use `git` to clone the repository containing the hook's source. Unfortunately, there's no way around just typing out this URL manually, so be mindful of spelling mistakes.
```sh
cd /opt
git clone https://github.com/john-craig/mkinitcpio-encrypt-detached-header
```
> If we try to run the `makepkg` command to build the package right away, we'll encounter an error. This is because running the `makepkg` command as the root user runs the risk of a malicious package gaining arbitrary code execution. To build the package safely, we need to run the command as a non-root user. We have a few non-root users available on the system already, but the simplest one is `nobody`, which represents a user with the least possible privileges. We'll change ownership of this directory to `nobody` with the `chown` command, then we can run the build command with `runuser -unobody`, followed by `makepkg`.
```sh
chown -R nobody .
runuser -unobody makepkg -si
```
> When the build completes, we'll have our package archive built and ready to install. We can install a package directly from a package archive with `pacman -U`.
```sh
pacman -U mkinitcpio-encrypt-detached-header*.pkg.tar.zst
```
> Not that we have our detached header hook package installed, we can edit the configuration file of `mkinitcpio` to utilize it with `nano`. Here we'll edit two lines. First, we'll add the path to the header file to the `FILES` array, so that it gets embedded into the `initramfs`. Then we'll modify the `HOOKS` file. We can get rid of `systemd` and replace it with `udev`. Then add our `encrypt-dh` hook after the `block` device hook. After that, we'll also need to add the `lvm2` hook. We can save our changes with Ctrl+O, and then exit with Ctrl+X.
```sh
nano /etc/mkinitcpio.conf
	FILES=(/boot/header.img)
	...
	HOOKS=(base udev autodetect microcode modconf kms keyboard keymap consolefont block encrypt-dh lvm2 filesystems fsck)
```
> That takes care of the configuration for `mkinitcpio`, but we also have to manipulate the kernel parameters passed to the kernel when it is first loaded into memory by the bootloader. These kernel parameters are then passed to the `encrypt-dh` hook so that it can function correctly. The kernel parameters are controlled by GRUB, so to change them we'll have to modify the `/etc/default/grub` file.
> However, one of the things we need to specify with these kernel parameters is the path to our encrypted block device on the main disk. Up until now, we've been interacting with this block device using the path `/dev/vda1`. However, this path can potentially change between reboots when there are multiple disks on the system. To ensure our path is consistent, we can use the symlinks under `/dev/disk/by-partuuid`, because these are dependent upon the partition UUID of the block device, which does not change. To get the partition UUID, we can use the command `blkid` on the block device path. You will notice, however, that this value is very long, and would be difficult type. To make things easier on ourselves, we can just append the partition UUID to the end of the `/etc/default/grub` file, like so. Notice the double right-carats. That is important. Two carats appends, a single carat overwrites.
```sh
blkid /dev/vda1
blkid -o value /dev/vda1 >> /etc/default/grub
```
> Now that we've appended the partition UUID of our primary disk to the end of the file, we can edit it. Here, we want to change two lines. The first is the `GRUB_CMDLINE_LINUX_DEFAULT` line, which tells GRUB which kernel parameters to pass. 
> We have to add a few things here: first, the `cryptdevice` parameter. This is where our block device path for the primary disk goes. Once we're ready to type the UUID, we can navigate to the bottom of the file, then cut the UUID we appended earlier with Ctrl+K, then go back up to the kernel parameter line and paste it with Ctrl+U. Afterwards we'll write the named that we wanted the unlocked device to have.
> Next we'll define the parameter for the detached header. This is the path of the file inside of the `initramfs`, so we have to specify that by putting `rootfs` first, then the path. Next we'll place the path of the root volume, which for us is `/dev/volgroup0/lv_root`, and the swap volume, which is `/dev/volgroup/lv_swap`. Lastly we want to uncomment this line for enabling the GRUB cryptodisk feature.
```sh
nano /etc/default/grub
	GRUB_CMDLINE_LINUX_DEFAULT="loglevel3 cryptdevice=/dev/disk/by-partuuid/${PARTUUID}:crypt0 cryptheader=rootfs:/boot/header.img root=/dev/volgroup0/lv_root resume=/dev/volgroup0/lv_swap quiet"
	...
	GRUB_ENABLE_CRYPTODISK=y
```
> Now that all the configuration files for GRUB and `mkinitcpio` have been prepared, we can actually set them up. First we run `mkinitcpio -P` to generate the `initramfs`. When this is done, we install grub with `grub-install --target=x86_64-efi --efi-directory=/boot/esp --removable`. Lastly we generate the GRUB config with `grub-mkconfig -o /boot/grub/grub.cfg`.
```sh
mkinitcpio -P
grub-install --target=x86_64-efi --efi-directory=/boot/esp --removable
grub-mkconfig -o /boot/grub/grub.cfg
```

*Finishing Touches*
> At this point, are system is fully functional. The last thing we need to do before we reboot is run the `passwd` command to set the password of our root user, otherwise we won't be able to log in once the system comes back up. Additionally, this would be a good time to use BTRFS to take a snapshot of the initial state of the filesystem. For this we'll use `btrfs subvolume snapshot / /.snapshot-initial`. 
```sh
passwd

btrfs subvolume snapshot / /.snapshot-initial
```
> Finally, we use the `exit` command to exit our chroot, and then `poweroff` to poweroff the system. This gives us time to unplug our installer USB and boot into our new Archlinux installation.
```sh
exit
poweroff
```

*Post-Installation Steps*
> Once we've been able to successfully boot into our newly-installed system for the first time, there are a few final post-installation steps. Namely, we want to create a user for ourselves so that we are not constantly logged in to root. We can do this with the `useradd` command, passing the `-m` flag to create a home directory for the new user. To keep things simple, my user's username will be `operator`.
> Once the user is created we want to set its password so we can login. We can do this with the command `passwd`, followed by the username. I'll be using the password `operator`. Next, we'll want to give our user `sudo` privileges. We can do this with the `visudo` command, which is used to securely edit the `/etc/sudoers` file. I'll also set `EDITOR=nano` when running the command to tell it to use the `nano` editor.
> In this file, we need want to add the following line for our user. This will allow the user to run any command as root after providing their password.

```sh
useradd -m operator
passwd operator      # Set password to 'operator'

EDITOR=nano visudo
	## User privilege specification
	operator ALL=(ALL:ALL) ALL
```

> Now, in theory, this is the last thing we have to do. However, practically speaking, this tutorial isn't complete until we have some kind of Arch User Repository helper installer. I typically use `paru`.
> To install `paru`, we can first switch to our newly-created user with the `su` command. Next, we want to create a directory to clone the `paru` repository into. Like the the `mkinitpcio` hook from earlier, this can technically be anywhere, but I like to use the path `.local/opt` in the user's home directory. First we have to create this directory with `mkdir -p`, then we can change directory to it and clone the `paru` repository with `git`.
> Now all that's left to do is enter the `paru` directory and run `makepkg -si`. This command will build `paru`, and then finally prompt us for our password to complete the installation.

```sh
mkdir -p ~/.local/opt
cd ~/.local/opt
git clone https://aur.archlinux.org/paru.git
cd paru
makepkg -si
```

> And there you have it. Archlinux, fully installed and ready to use.

