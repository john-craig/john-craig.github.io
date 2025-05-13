---
title: Setting up Automated Offline Backups on External Hard Drives
---
# Tutorial: Automated Offline Backups
## Introduction
For Black Friday I got my hands on a pair of nice 4TB LaCie hard drives to use for offline backups. These drives are meant to sit somewhere safe and disconnected as a form of cold storage. My plan is to update them once a month. Ideally all I should have to do is plug them in and wait for the notification that the back-ups are complete.

To accomplish this, we're going to make ample use of `udev` rules in Linux, so that we can kick off a series of `systemd` services in response to a target device being plugged. I will be using NixOS for this guide, although all of the techniques outlined here should work just as well on any other distribution that supports `udev` and `systemd`.

## Instructions
**Preparing the Drives**
As usual, our first step is going to be preparing the drives that we want to use for our offline backups. I prefer to have my drives encrypted. If you don't intend to encrypt your drives, you can skip this step.

Assuming your drive is located at `/dev/sdb` and you have a keyfile located at `/root/keyfile.bin`:
```sh
cryptsetup luksFormat /dev/sdb /root/keyfile.bin
```

At this point you will need to obtain the UUID of the LUKS formatted partition. This can be found by issuing `lsblk -f`, which will display a result like:
```
NAME        FSTYPE      FSVER LABEL    UUID                                 FSAVAIL FSUSE% MOUNTPOINTS
sdb         crypto_LUKS 2              77777777-7777-7777-7777-777777777777                
mmcblk1                                                                                    
├─mmcblk1p1 vfat        FAT16 FIRMWARE AAAA-AAAA                                           
└─mmcblk1p2 ext4        1.0   NIXOS_SD 44444444-4444-4444-8888-888888888888   17.2G    37% /nix/store
                                                                                           /
```

In this example `77777777-7777-7777-7777-777777777777` would be our UUID.

Next you will want to unlock and format the partition with a filesystem.
```sh
cryptsetup open -d /root/keyfile.bin /dev/sdc cryptX
mkfs -t btrfs /dev/mapper/cryptX
```

Repeat the preceding steps for each drive you wish to utilize. If you are not encrypting your drives, you still need to format the partition and obtain the UUID of the disk.

**Setting up `/etc/crypttab`**
`/etc/crypttab` is a useful file that generates a set of `systemd` services for automatically unlocking encrypted devices. Rather than trying to write services for this ourselves, we will make use of it. For each of our offline drives, we want to add an entry to `crypttab` that looks like this:
```
cryptX.77777777        UUID=77777777-7777-7777-7777-777777777777    /root/keyfile.bin nofail,noauto
```

Let's break these down a little bit. The first column indicates the name used be the device once it has been decrypted. This can be anything in principle, but for reasons we'll get into later, it's better to make it unique per device, and for simplicity's sake, derived from the device's UUID. In this example I have used the first 8 character of the device UUID.

The next column is the UUID. This is pretty self-explanatory, we collected these in the first step.

Next is the path to the key file we want to use for decrypting the device. If you're a more advanced user with a different method of encryption than a plain keyfile, you might have a different unlock method here.

Finally are the options we use when unlocking the device. The `nofail,noauto` options are critical. These mean that the system won't attempt to automatically unlock the device on startup (`noauto`) and will allow the system to continue coming up even if the unlock fails (`nofail`). 

These options are critical, if you miss one of these you might accidentally bork your system.

In NixOS, rather than manually creating an entry for each of our drives, we could instead define them like this:
```sh
environment.etc."crypttab".text = let
        offlineDevices = [
            "77777777-7777-7777-7777-777777777777"
            "99999999-9999-9999-9999-999999999999"
        ];
    in lib.strings.concatMapStrings
    (deviceUUID:
        let
            shortUUID = builtins.elemAt (lib.strings.splitString "-" deviceUUID) 0;
        in
        ''
        cryptX.${shortUUID}        UUID=${deviceUUID}    /root/cryptsetup/crypt0/keyfile.bin nofail,noauto
        '')
    offlineDevices;
```

**Setting up `udev` Rules**
Now that our `/etc/crypttab` is set up, we will need to create some `udev` rules. For each device we will need two rules. The first rule will trigger when a device with its UUID is plugged in:

```
ACTION=="add", KERNEL=="sd*", ENV{ID_FS_UUID}=="77777777-7777-7777-7777-777777777777", ENV{SYSTEMD_WANTS}+="systemd-cryptsetup@cryptX.77777777.service"
```

Notice is matches on the UUID we collected earlier. It then triggers the `systemd-cryptsetup@cryptX.77777777.service` service generated from our `/etc/crypttab` entry, which will unlock the device.

This is a good first step, but we still want to mount the device and trigger the backup. For that we will need a rule which triggers when the `/dev/mapper/cryptX.77777777` device becomes available:
```
ACTION=="add", KERNEL=="dm*", ATTR{dm/name}=="cryptX.77777777", ENV{SYSTEMD_WANTS}+="offline-backup.service"
```

Once again, these `udev` rules can be defined easily in NixOS:
```
services.udev.extraRules = let
        offlineDevices = [
            "77777777-7777-7777-7777-777777777777"
            "99999999-9999-9999-9999-999999999999"
        ];
    in lib.strings.concatMapStrings
    (deviceUUID:
        let
        shortUUID = builtins.elemAt (lib.strings.splitString "-" deviceUUID) 0;
        in
        ''
        ACTION=="add", KERNEL=="sd*", ENV{ID_FS_UUID}=="${deviceUUID}", ENV{SYSTEMD_WANTS}+="systemd-cryptsetup@cryptX.${shortUUID}.service"
        ACTION=="add", KERNEL=="dm*", ATTR{dm/name}=="cryptX.${shortUUID}", ENV{SYSTEMD_WANTS}+="offline-backup.service"
        '')
    offlineDevices;
```

**Writing the Offline Backup Service**
Finally we want to define our backup service:

```
systemd.services."offline-backup" = {
    enable = true;
    script = let
            rsyncCmd = "${pkgs.rsync}/bin/rsync -ravPH";
        in ''
        ${pkgs.mount}/bin/mount /dev/mapper/cryptX.* /mnt
        ${rsyncCmd} /srv/ /mnt
        ${pkgs.umount}/bin/umount /mnt
        '';

    serviceConfig = {
        Type = "oneshot";
        User = "root";
    };

    postStop = ''
        ${pkgs.systemd}/bin/systemctl stop systemd-cryptsetup@cryptX.*.service
    '';
};
```

Let's break down this definition slightly. First we define ourselves a shorthand for the `rsync` command. `rsync` will speed up backups by only copying files which have been changes between the two directories. The `rsync` command used here is quite simple, but your use-case may require a more complicated version if you are syncing between directories over a network, etc.

In the main script, we mount the mapped decrypted device onto `/mnt`, then perform our `rsync` command to synchronize the files, and then unmount the device. This takes care of the three main steps. Once this service completes, it would probably be safe to simply yank the device. 

However, as a best practice, we want to also re-lock the device, so we use the `postStop` option to invoke `systemctl` and stop our `systemd.cryptsetup@cryptX.*.service`. This will match on any device which is a variation of the `cryptX.${shortUUID}.service` format, which means we don't need to have separate invocations for each device.

**Security Considerations**
You may have noticed that, in total, this set of `udev` rules will operate on *any* device which happens to have a matching UUID. An attacker could, hypothetically, spoof the UUID in order to substitute their own device, and recieve a copy of all your data. Keep in mind, however, that even if they can spoof the UUID, the attempt to unlock the bad drive will fail because the LUKS partition it was not encrypted with the same key in the first place. 

Additionally, I would argue that if any attacker already has physical access to a device containing backups, those backups should already be treated as compromised to begin with.

Having said that, if you're like me and these "What if's" keep you up at night regardless of their plausibility, my approach to tackling this problem would be this: at the end of each backup, the backup process would create a signature of the contents of the backup using a private key and store it on another partition or subvolume on the drive. This signature would then have to be validated the next time the drive is unlocked.

This would add two very long and expensive steps to the backup process each time, and it would require an extra step to bootstrap the first signature on the empty drive manually, but it would guarantee that the drive is trustworthy each time a backup is performed.

## Conclusion
Offline backups are a useful last line of defense against data loss. Whether it be mission critical data for a business or irreplacable family photos, using offline backups with an external hard drive, or another storage medium which can be stored disconnected from the internet is a precaution worth investing time into.