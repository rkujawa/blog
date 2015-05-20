---
layout: post
title: On a quest for a reasonable home router 
---

For years I've been routing between my home network and the <del>world</del> internet using Mac Mini (PowerPC, 2005 model), equipped with additional ethernet interface, running OpenBSD. Unfortunately, some time ago the hardware started to show signs of breakage. Random lockups and ethernet interface hangs forced me to consider replacing it.

I felt deep resentment towards plasticky home routers you can find on the shop shelves. They have never worked for me. That's why in the first place I used a Mac Mini as my router.

I started to search for a compromise between a full-fledged computer and a low-end router. After doing a bit of research, I stumbled upon Ubiquiti EdgeRouter series of routers. Googling revelaed that they are based on a fork of [Vyatta](http://en.wikipedia.org/wiki/Vyatta), which seemed like a good thing. I decided to order one of the cheaper models, [EdgeRouter Lite](https://www.ubnt.com/edgemax/edgerouter-lite/).

![placeholder]({{ site.url }}/public/images/edgerouter.jpg "EdgeRouter Lite")

It arrived a few days later from one of the polish distributors. Immediately after unpacking, one can notice a few promising things. 
- The metal case is small, but looks strudy and well made, unlike plasticky routers from D-Link, Linksys etc. 
- There's a familiar serial console port, compatible with popular Cisco cross-over cables.

The web interface is very limited. But that's okay, I didn't buy this for a fancy web clicky thing.

If you don't like Vyatta, you can also run [NetBSD] or OpenBSD on it. 

