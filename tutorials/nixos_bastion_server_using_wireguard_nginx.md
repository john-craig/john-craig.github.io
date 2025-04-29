---
title: NixOS Bastion Server using Wireguard
---

## Introduction
Homelabbing is a fun past-time, but sooner or later many labbers end up wanting to access their suite of self-hosted services from outside of their local area network. There are many tutorials for how to accomplish this, but there's also a lot of ways that misconfiguration can expose your network and devices to outside attackers.

For example, the simplest approach a person could take is to open ports 80 and 443 on their router, forward them to whatever host on their network is responsible for proxying the web pages of their services, and then set up a CNAME record in Cloudflare to point to their home IP address this.

This *works*, but it exposes all their services to the open internet. Any person or bot snooping for open ports can now gain access. All it takes is one of these self-hosted services to have an unpatched vulnerability that the attacker can exploit for them to gain remote access to the underlying infrastructure.

In this post we will look at a safer alternative: using a self-hosted virtual private network using the Wireguard VPN and a bastion server.

## Instructions
**Setting up the Bastion**
The purpose of a bastion server is to reduce the exposure of your personal homelab infrastructure to the open internet as much as possible. Like a medieval bastion, it is intended to be exposed to the brunt of the adversary's attacks in place of more fragile infrastructure. Within our virtual private server, the bastion server will serve as the connection point for hosts outside of our local area network.

Because of this the bastion server should be both publicly accessible and, ideally, already located outside of the network we wish to connect. Because of this, the best candidate is a cheap virtual private server, such as a [Linode](https://www.linode.com/) Nanode, which is only about $5 a month. Linode has a quite in-depth [guide](https://www.linode.com/docs/guides/install-nixos-on-linode/) for setting up NixOS on its server, even though it is not an officially-supported distribution.

**Generating Wireguard Keys**
Once we have our bastion server selected and installed with NixOS, we can begin the process of configuring Wireguard as our VPN. The first step of this process is the generation of keys for each host. Wireguard uses public-key cryptography, more specifically elliptic-curve cryptography, for its security, meaning that each host will have a public and private key. Public keys are shared between hosts, while private keys, as their name implies should be kept private to only the host utilizing them.

First, we will need to install Wireguard on the host we are using for setup. I use Archlinux for most of my workstations; on there the package is `wireguard-tools`. Once we have Wireguard installed, we will need to run the following command once for each host, replacing `${host}` with the name of the host we wish to generate the keys for:

```sh
wg genkey | tee ${host}.private.key | wg pubkey > ${host}.public.key
```

At a minimum, you will need one set of keys for your bastion and another set of keys for the server on your local area network where you run your services, or another server on your local area network which is intended to act as an ingress to your services, if you have a more sophisticated setup. Going forwards we will refer to this latter host as the "home server".

**Adding the Keys to our Configurations with `sops-nix`**
Dealing with all these keys can get tricky, but fortunately, because we are on NixOS, we can use [sops-nix](https://github.com/Mic92/sops-nix) for secrets management. There are numerous good tutorials for setting up `sops-nix`, I recommend following the one [here](https://www.youtube.com/watch?v=G5f6GC7SnhU) by Vimjoyer. 

Now we will use `sops` to store the private keys for each host in the repository you utilized for NixOS configurations. Your exact command will depend on how you have your repository laid out, however for mine, I would use:
```sh
sops edit hosts/bastion0/hostSecrets/wireguard.yaml
```
Inside of the editor session, I would create a file with the following contents:
```yaml
wireguard:
    root:
        private.key: AAAAAAAAAAAAAAAAAAAA
```
replacing `AAAAAAAAAAAAAAAAAAAA` with the contents of the `bastion.private.key` file we generated previously.

Then we will create a file at `hosts/bastion0/hostSecrets/default.nix` and add to it the following:
```
{ pkgs, lib, config, ... }: {
  sops.secrets."wireguard/root/private.key" = {
    mode = "600";

    sopsFile = ./wireguard.yaml;
  };
}
```

These steps should be repeated for each other host we intend to use with Wireguard.

**Configuring Wireguard on the Bastion**
Our next step is to configure Wireguard on our bastion host. I like to keep my configurations fairly modular, so I created a Nix module at `hosts/bastion0/hostModules/wireguard.nix`, and placed in it the following:
```
{ config, pkgs, ... }:

let
  wgPort = 51820;
  wgInterface = "wg0";
  wgIp = "10.100.0.1/24";
  
  homeserverPubkey = "BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB";
in {
  networking.firewall.allowedUDPPorts = [ wgPort ];

  networking.wireguard.interfaces.${wgInterface} = {
    ips = [ wgIp ];
    listenPort = wgPort;
    privateKeyFile = config.sops.secrets."wireguard/root/private.key".path;

    peers = [
      {
        publicKey = homeserverPubkey;
        allowedIPs = [ "10.100.0.2/32" ];
      }
    ];
  };

  boot.kernel.sysctl = {
    "net.ipv4.conf.all.forwarding" = true;
  };
}
```

