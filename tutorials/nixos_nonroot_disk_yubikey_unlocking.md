---
title: Non-Root Disk Encryption with Yubikey FIDO2 on NixOS
---
# Tutorial: Non-Root Disk Encryption with Yubikey FIDO2 on NixOS

## Introduction
In this article I will take you through the process of creating an encrypted partition on a NixOS device which can be unlocked using a Yubikey as a FIDO2 token. This will be particularly useful if your encrypted partition is located on a USB or other external storage media, as it permits a configuration where the internal storage of the host can be kept free of any sensitive materials.

**Note:** At time of writing, a device configured in this way will still require *user presence* in order for it to be unlocked. Truly unattended unlocking is currently not possible using FIDO2 on the Yubikey.

## Instructions
**Prerequisites**  
You will need `libfido2`, `cryptsetup`, and `yubikey-manager` to be installed on the device on which you intend to set up the secure partition.

**Formatting the LUKS partition**  
To begin with, you will need to LUKS format the partition which you intend to encrypt.

```
crypsetup luksFormat /dev/sdX
```

At this point you should enter a strong passphrase. You will want to write this down somewhere, as it will be the fallback in case your Yubikey ever gets lost or damaged.

**Configuring the Yubikey**  
Now, we want to perform some configuration of the Yubikey itself. Personally, I chose to deactivate all the features of the Yubikey which I wasn't making use of:

```
ykman config nfc -D
ykman config usb -d OTP
ykman config usb -d U2F
ykman config usb -d OATH
ykman config usb -d PIV
ykman config usb -d OPENPGP
ykman config usb -d HSMAUTH
```

At this point, we want to confirm that `systemd-cryptenroll` utility sees the Yubikey as an available FIDO token. We can do this by issuing the command `systemd-cryptenroll --fido2-device=list`, which should display something like the following:
```
PATH         MANUFACTURER PRODUCT
/dev/hidraw0 Yubico       YubiKey FIDO
```

If you don't see your Yubikey here, it is likely because you haven't installed the `libfido2` package yet. Or you haven't installed the `yubikey-manager` itself, but then how'd you even get to this step?

**Enrolling the Yubikey as a FIDO2 Token**
Now we're ready to enroll. When you run this command it will prompt you for both the passphrase of the LUKS partition and the PIN number of the Yubikey.

```
systemd-cryptenroll --fido2-device=auto --fido2-with-client-pin=no /dev/sdX
```

**Manually Unlocking the Encrypted Partition**
To make sure everything is working correctly, we now want to try manually unlocking the encrypted partition.

First, re-lock the partition if it has already been unlocked:
```
cryptsetup close crypt0
```

Next try unlocking using the `systemd-cryptsetup` command:
```
systemd-cryptsetup attach crypt0 /dev/sdX - fido2-device=auto
```

This should prompt you to confirm your presence on the security token in order to unlock it.

**Setting Up `crypttab` for the Encrypted Partition**
Once the enrollment is complete and you have tested it out manually, we'll need to add an entry in `/etc/crypttab` to use this FIDO2 token for unlocking the encrypted partition automatically in the future:
```
crypt0            UUID=$MY-UUID - nofail,fido2-device=auto,token-timeout=0,headless=true
```
Here, `$MY-UUID` should be equal to the UUID of your partition, which can be displayed with `lsblk -f`.

Let's look at these options a little bit:
* `nofail` this means the rest of the operating system will continue coming up independent of the unlocking of this device. Again, useful if you plan for the partition to be located on a removable storage medium. Any filesystems mounted from this partition in `fstab` should also specify the `nofail` option.
* `fido-device=auto` this is the same parameter we gave to `systemd-cryptenroll`. It automatically chooses the first available FIDO device on the system.
* `token-timeout=0` this timeout tells `crypttab` how long to wait for the FIDO token to become available before prompting for a password. From the perspective of `crypttab`, *the Yubikey is not available until after it has verified user presence*. Setting this option to zero means that it never times out; the default value is 30 seconds. Note that unless `nofail` is set, the operating system will wait until this timeout is complete to finish coming online.
* `headless=true` this prevents `systemd-cryptenroll` from falling back to manual password entry if the token unlock fails.

Of these options, the most important as `nofail`, as without it you may get yourself stuck in an error state upon your next reboot and have to do some very messy recovery.