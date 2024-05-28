---
title: Bastion Server using Tailscale and Nginx
---

## Introduction
Homelabbing is a fun past-time, but sooner or later many labbers end up wanting to access their suite of self-hosted services from outside of their local area network. There are many tutorials for how to accomplish this, but there's also a lot of ways that misconfiguration can expose your network and devices to outside attackers.

For example, the simplest approach a person could take is to open ports 80 and 443 on their router, forward them to whatever host on their network is responsible for proxying the web pages of their services, and then set up a CNAME record in Cloudflare to point to their home IP address this.

This *works*, but it exposes all their services to the open internet. Any person or bot snooping for open ports can now gain access. All it takes is one of these self-hosted services to have an unpatched vulnerability that the attacker can exploit for them to gain remote access to the underlying infrastructure.

A safer alternative, and one that I used for a while myself, was to set-up a single sign-on service such as Authelia, and configure the network's reverse-proxy to authenticate any users coming from IP addresses external to the local area network. 

This is a big improvement, but it still requires having a port exposed to the open internet from the local area network. It would be an improvement to not have any ports exposed whatsoever, but of course this raises the question: how do you actually access the services?