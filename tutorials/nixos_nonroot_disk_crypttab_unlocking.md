---
title: Non-Root Disk Encryption on NixOS
---
# Tutorial: Non-Root Disk Encryption on NixOS

Although it is easy to find examples of how to configure a system with an encrypted root filesystem using NixOS, the if you want to encrypt a non-root disk and mount it later on after boot, the details on that are a little bit more sparse.

[This](https://discourse.nixos.org/t/how-to-unlock-some-luks-devices-with-a-keyfile-on-a-first-luks-device/18949/5) forum post finally provided the answer: plain ol' [crypttab](https://www.man7.org/linux/man-pages/man5/crypttab.5.html).

```
environment.etc."crypttab".text = ''
    crypt0            UUID=<My UUID>    /path/to/my.keyfile
  '';
```

Here we are specifying the text contents of a file called `/etc/crypttab` directly. In Linux, the `/etc/crypttab` file is used as an equivalent to `/etc/fstab` for encrypted devices. It contains instructions about how to decrypt non-root disks.

Though in hindsight it certainly feels intuitive to try this approach, one of the issues with a project as ambitious as NixOS is that it can be difficult to tell which configurations are fully supported with the Nix configuration syntax and which must be re-created by-hand with drop-in files, such as the one seen above.

Hopefully, one day NixOS will have a baked-in feature for decrypting non-root encrypted disks and this guide will be obsolete. Until then, I hope it saves you a little bit of time and effort. :)