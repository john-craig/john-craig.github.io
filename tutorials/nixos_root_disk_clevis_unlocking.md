---
title: Root Disk Encryption with Clevis and Tang on NixOS
---
# Tutorial: Root Disk Encryption with Clevis and Tang on NixOS
## Introduction
There will always need to be a trade off between security and convenience, but that doesn't mean there can't be compromises. One of these compromises is network-bound disk encryption. Under this scheme, a disk or partition which is encrypted can be automatically unlocked by retrieving the information necessary to decrypt the partition from a server running on the network\*. This means that if the disk or device containing the encrypted partition is taken off the network, it will not longer be able to unlocked; but as long as it is on the network, it can be booted and unlocked automatically.

\*Note: Although this sounds like a key escrow, it is different. In a key escrow, the actual encryption key is transmitted from the server to the client. Under the network-bound disk encryption, only the public key used to derive the encryption key is transmitted over the network.

Two tools used to achieve network bound disk encryption-- `clevis` on the client side and `tang` on the server side-- are available in the NixOS distribution. However, it can be tricky to set up the options correctly. That's what we'll be taking a look at in this tutorial.

## Instructions
**Prerequisites**
On the client device, you will need to have the `clevis` package installed. If you are performing this tutorial on a live installer, the live installer will also need to have access to the `clevis` package.

**Setting up the Tang Server**
This is by far the simplest part. On the host you intend to use for the `tang` server, simply specify these options in the configuration:
```
  services.tang = {
    enable = true;

    ipAddressAllow = [ "192.168.1.0/24" ];
    listenStream = [ "0.0.0.0:7654" ];
  };
```

Here the `192.168.1.0/24` subnet range may be changed to suit your needs. Once this is completed, the `tang` server should be up and running.

**Preparing the Encrypted Root Partition**
The way the NixOS configuration for `clevis` works makes it so that, rather than unlocking the encrypted partition directly using `clevis`, we instead use `clevis` to decrypt an encypted `jwe` file containing the key file that is then used to decrypt the partition.

Therefore, when setting up our partition, we will set it up as we would any other LUKS partition we were encrypting with a key file:
```sh
dd bs=512 count=4 if=/dev/random of=/tmp/keyfile.bin iflag=fullblock

cryptsetup luksFormat /dev/sdX /tmp/keyfile.bin
```

Replacing `/dev/sdX` with path for the disk you wish to format.

**Bind the Key File to the Tang Server**
Now we want to use `clevis` to encrypt the keyfile we created using our `tang` server to create a `jwe`. At boot time, this `jwe` will be decrypted to retrieve the keyfile, which will then be used for unlocking the root partition.

```
cat /tmp/keyfile.bin | clevis encrypt tang '{"url": "192.168.1.255"}' > /root/keyfile.jwe
```

The `/root/keyfile.jwe` file should be stored in a location that is accessible during the building of the NixOS configuration for this host.

**Configuration for the Client**
Here is the configuration to use for the client device:
```
boot.kernelParams = [ "console=tty0" "ip=::::::dhcp" ];

boot.initrd = {
    # Configuration needed for clevis
    network.enable = true;

    clevis = {
        enable = true;
        useTang = true;
        devices = {
            crypt0 = {
                secretFile = /root/keyfile.jwe;
            };
        };
    };

    luks.devices = {
        crypt0 = {
            device = "/dev/disk/by-uuid/$DISK_UUID";
            preLVM = true;
            allowDiscards = true;
        };
    };
};
```

Let's break these down. `clevis` needs to have networking available at boot time in order to reach the `tang` server, hence our specification 