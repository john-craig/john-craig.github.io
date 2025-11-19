---
layout: post
title: "Setting up a Binary Cache from an Alternate /nix/store"
---
# Setting up a Binary Cache from a Secondary Nix Store
In this article we will learn how to set up Nix binary cache from a Nix store not located at `/nix/store`. This will include serving packages from this binary cache over HTTP(S) and pushing to the binary cache over SSH.

**Why Set This Up?**
The main reasons you may want to run a binary cache from a secondary Nix store are the following:
 - keep the packages available in the cache isolated from the packages used by the system running the binary cache
 - store the cache on an external storage medium, or some other mount point separate from the one where the regular `/nix` directory is mounted

## Serving the Binary Cache over HTTP
There exist a wide variety of projects aimed at serving binary caches over HTTP. For the purposes of this tutorial, we will be using [nix-serve-ng](https://github.com/aristanetworks/nix-serve-ng).

Our approach will be to use mount namespaces to make it appear to the `nix-serve-ng` process as though the non-standard `/nix/store` we want to use is the same as the system's root. For example, if we wanted to serve a Nix store from `/srv/cache/nix/store`, we would use the following command:
```
exec unshare --mount --propagation private bash -euxc "
    mount --rbind /srv/cache/nix /nix
    mount --make-rslave /nix
    exec nix-serve --listen 0.0.0.0:5001
"
```

This command uses `unshare` to create a mount namespace and then executes a series of commands within that namespace. These commands bind-mount the `/srv/cache/nix` directory to `/nix`. Then, the `nix-serve` process is started. When this process interacts with the `/nix/store` directory to serve packages, it will really be interacting with the `/srv/cache/nix/store` directory outside of the mount namespace.

A full configuration with a SystemD service would look like so:
```
systemd.services.nix-cache = {
    enable = true;
    description = "HTTP Nix binary cache";
    wantedBy = [ "multi-user.target" ];
    after = [ "network.target" ];

    path = [ 
        pkgs.nix-serve-ng
        pkgs.nix
        pkgs.util-linux
        pkgs.coreutils
        pkgs.bash 
    ];

    preStart = ''
        ${pkgs.nix}/bin/nix copy --to /srv/cache ${pkgs.nix-serve-ng}
        ${pkgs.nix}/bin/nix copy --to /srv/cache ${pkgs.util-linux}
        ${pkgs.nix}/bin/nix copy --to /srv/cache ${pkgs.bash}
        ${pkgs.nix}/bin/nix copy --to /srv/cache ${pkgs.coreutils}
    '';
    script = ''
        exec unshare --mount --propagation private bash -euxc "
            mount --rbind /srv/cache/nix /nix
            mount --make-rslave /nix
            exec nix-serve --listen 0.0.0.0:5001
        "
    '';
    environment = {
        NIX_SECRET_KEY_FILE = "${config.sops.secrets."nix/nix-serve/cache-key".path}";
    };
};
```

This example also sets the `NIX_SECRET_KEY_FILE` environment variable to a package signing key populated by `sops-nix`. If you don't care about signing the packages in your cache, you can safely ignore this variable.

## Pushing Packages to the Binary Cache
In order to push the packages to our Nix store from another host, we can utilize the existing `nix copy` command like so:
```
nix copy --substitute-on-destination \
    --to ssh://cacher@$cacheServer?remote-store=local?root=/srv/cache \
    /nix/store/000j6dh3wl59wijhl99np8bph69m0wpz-python3.13-pyqrcode-1.2.1.drv
```

To push all the packages used by the current system, we can use:
```
nix copy --substitute-on-destination \
    --to ssh://cacher@$cacheServer?remote-store=local?root=/srv/cache \
    /run/current-system
```

Ensure that the remote user used for pushing to cache is a trusted user for the Nix daemon of the cache server and has privileges to read and write the `/srv/cache/nix` directory.

## Cleaning Up the Binary Cache
Lastly, we will want to occasionally prune packages from this binary cache. Although we can use the existing `nix-collect-garbage` with the `--store` option, we need to be careful because none of the packages we have pushed to the store have been added to the garbage collector's roots yet, meaning it will treat everything as garbage and remove all the packages.

To add the packages to the roots, all we need to do is created a symlink from the store path of the package we want to preserve to the directory `/srv/cache/nix/var/nix/gcroots`. For example:

```
ln -s /srv/cache/store/000j6dh3wl59wijhl99np8bph69m0wpz-python3.13-pyqrcode-1.2.1.drv /srv/cache/nix/var/nix/gcroots/python3.13-pyqrcode-1.2.1
```

Then running the command `nix-collect-garbage -d --store /srv/cache` will not remove the symlinked package or any of its dependencies.