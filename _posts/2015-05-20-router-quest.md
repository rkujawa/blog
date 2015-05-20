---
layout: post
title: On a quest for a reasonable home router 
---

For years I've been routing between my home network and the <del>world</del> internet using Mac Mini (PowerPC, 2005 model), equipped with additional ethernet interface, running OpenBSD. Unfortunately, some time ago the hardware started to show signs of breakage. Random lockups and ethernet interface hangs forced me to consider replacing it.

<!-- more -->
I felt deep resentment towards plasticky home routers you can find on the shop shelves. They have never worked for me. That's why in the first place I used a Mac Mini as my router.

I started to search for a compromise between a full-fledged computer and a low-end router. After doing a bit of research, I stumbled upon Ubiquiti EdgeRouter series of routers. Googling revelaed that they are based on a fork of [Vyatta](http://en.wikipedia.org/wiki/Vyatta), which seemed like a good thing. I decided to order one of the cheaper models, [EdgeRouter Lite](https://www.ubnt.com/edgemax/edgerouter-lite/).

![placeholder]({{ site.url }}/public/images/edgerouter.jpg "EdgeRouter Lite")

It arrived a few days later from one of the polish distributors. Immediately after unpacking, one can notice a few promising things. 

- The metal case looks strudy and well made, unlike plasticky routers from D-Link, Linksys etc. 
- It's very small, the size is completely satisfying for a home solution.
- There's a familiar serial console port, compatible with popular Cisco cross-over cables.

First thing I did after connecting it, I checked the firmware version. It turned out, a few days earlier Ubiquiti released a new (1.6.0) version. So I downloaded it and proceeded to update. This brought a new kernel version, a dozen of bug fixes and a few new features.

The web interface is very limited. But that's okay, I didn't buy this for a fancy web clicky thing. What matters for me is the command line interface and advanced features it provides. The provided command line interface is much like Cisco. If you ever configured Cisco router, you'll feel at home. By default it is accessible both by serial port and by SSH protocol (so you can configure it even if you don't have the serial cable).

Configuration process was fairly painless. I only had to search for the documentation when working with more advanced features like IPv6 tunneling. Unfortunately, there's not so much of a command line documentation on Ubiquiti web site. One has to be prepared to dig deeper and acquire the Vyatta documentation (which on the other hand is excellent).

Worth noting is the fact, that you can also access the bash interpreter command line. In case you ever have to debug anything or install additional software (yes, it is possible).

The router might be physically small, but performance-wise it's a beast. Based on a dual-core 64-bit Cavium Octeon processor and equipped with 512MB of RAM, it's more powerful than enough to handle most home workloads. I have an 80Mbps download link at home, running a few torrents at the same time and saturating it only caused CPU utilization of 5-8%. Ubiquiti is advertising this model as capable of processing 1 million packets per second.

At this point, after owning the router for 3 weeks or so, I have migrated all the old services from Mac Mini onto EdgeRouter. In my network currently it is handling:

- IPv4 routing with source NAT and multiple port forwards.
- DHCP server.
- IPv6 tunnel with /48 class routed into internal network
- IPv6 router advertisement and address configuration for internal machines.
- Firewall for both IPv4 and IPv6.
- OpenVPN server.
- UPnP server to support dynamic port forwarding for things like PS3, Torrents.

If you don't like Vyatta, you can also run [NetBSD](http://blog.netbsd.org/tnf/entry/hands_on_experience_with_edgerouter) or [OpenBSD](http://www.openbsd.org/octeon.html) on it. 

So far I am completely satisfied with EdgeRouter.

