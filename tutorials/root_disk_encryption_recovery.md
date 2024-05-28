---
title: Recovering from a Misconfigured Encrypted Root Filesystem
---
# Tutorial: Recovering from a Misconfigured Encrypted Root Filesystem
## Introduction
So, you were trying out a cool encrypted root filesystem configuration you found online, and you fucked up. Not only does your installation not boot properly, but it's hitting an error during the unlock of its root filesystem. Even if you boot it up with an LiveCD, the incorrect configuration is somewhere inside of the root filesystem, which is still encrypted.

Well, don't panic. We've all been there-- some of us more than others. :P

## Instructions

**Prerequisites**
You'll need a Linux distribution on a LiveCD that at least has `cryptsetup` installed. My recommendation would be the Archlinux installer. Also, this tutorial assumes that even though your root filesystem is encrypted, you still have the means to unlock it-- either you know the the passphrase or you have a copy of the keyfile.

**Identifying the LUKS Partition**
Once you boot your LiveCD, your first step should be to issue `lsblk`. This will show the disks and partitions on the system. 

If you're not sure which partition or disk you were using for your root filesystem, looking through the output of `lsblk` should give you some clues. Right now the only thing mounted will be partitions of the LiveCD itself, so you can ignore those. You will be looking for an unmounted partition with a name like `sdX` or `nvme0nX`, e.g.:

```
NAME                         MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINTS
...
sda                            8:0    0 931.5G  0 disk
`-sda1                         8:1    0 931.5G  0 part
```

Once you have an idea of which one might contain your LUKS partition, you can confirm with the command `cryptsetup luksDump /dev/sdX`. If this was the right partition, this command will print out a whole bunch of information about the key slots of that LUKS partition, which may be helpful depending exactly what kind of tinkering your were doing.

It's difficult to anticipate every single possible configuration that a reader of this guide might be faced, so for now I will provide the solutions to two more likely scenarios.

**Manually Unlocking with Passphrase**
If you have to unlock using a passphrase, this process is fairly straightforwards. Simply issue:
```
cryptsetup open /dev/sdX crypt0
```

You will then be prompted for your passphrase. Afterwards, the unlocked disk will appear as block device at `/dev/mapper/crypt0`.

**Manually Unlocking with a Key File**
This situation, on the other hand, is more tricky. If you have a keyfile, you need some way to get it onto the LiveCD that you're currently running.

To do this, first copy the keyfile onto a second external storage medium and plug it into your host. Once this is done, it should appear as a new disk when issuing `lsblk`.

Now you can mount the new disk onto the LiveCD using, for example:
```
mount /dev/sdY /mnt
```

This will allow you to copy the keyfile off of the external device:
```
cp /mnt/root.keyfile /tmp/root.keyfile
umount /mnt
```

Now that we have a copy of the keyfile, we can use it to unlock the LUKS partition:
```
cryptsetup open -d /tmp/root.keyfile /dev/sdX crypt0
```

**Mounting and Chrooting into the Unlocked Root Filesystem**
Now we should have the partition containing the root filesystem unlocked. What's left is to mount it, and then change root into it. This will allow us not only to make edits to the file but to also execute any processes we need to run to change our configuration as if we had into our installation normally.

Here, I am assuming that `/dev/sdXn` is your root partition and `/dev/sdXm` is your boot partition. If you do not have a separate boot partition (very unlikely) or if you have other mountpoints such as `/home` or `/var` mounted from separate partitions, you should change these commands accordingly.
```
mount /dev/sdXn /mnt
mount /dev/sdXm /mnt/boot

mount -t proc /proc /mnt/proc
mount -t sysfs /sys /mnt/sys
mount -i bind /dev /mnt/dev
mount --bind /run /mnt/run

chroot /mnt /bin/bash
```
At this point you will be back in your installation and ready to undo whatever weird change you made to put yourself in this mess and start over again. :)

*References:*
- https://superuser.com/questions/111152/whats-the-proper-way-to-prepare-chroot-to-recover-a-broken-linux-installation
