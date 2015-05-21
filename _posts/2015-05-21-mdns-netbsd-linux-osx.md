---
layout: post
title: Using Multicast DNS in small networks
---

Maintaining static DNS zones in an environment where new machines are often connected/disconnected/installed/removed is rather troublesome. Most home networks can be qualified as such environments. How to manage resolving of host names in a less annoying way? One easy solution is implementing [Multicast DNS](http://tools.ietf.org/html/rfc6762) (also known as mDNS).

<!-- more -->
My network consists of a mixture of Linux, OS X, iOS and NetBSD devices.

Typical UNIX implementation of mDNS consists of two components:
- Responder - which is used to reply the requests sent by other machines and expose information about host name and services to the network. 
- Resolver - which is used to answer queries made by the applications running on a local machine (akin to normal DNS resolver).

In some operating systems these two are completely separate and can work independent (i.e. one can have a responder but no resolver configured, or the other way).

Multicast DNS uses a special domain called "local". Host names are automatically registered in this domain.

## Configuring mDNS in OS X and iOS

No action is required. Multicast DNS is supported out of the box. On OS X the host name registered in mDNS is always the name set in `System Preferences -> Sharing -> Computer Name`.

You can confirm it is working by pinging this name in `local` domain:

{% highlight bash %}
$ ping osxbox.local
{% endhighlight %}

If the host name contains spaces (which is never a good thing), substitute them with `-` character.

## Configuring mDNS in Linux

The responder is implemented by Avahi service, which is already a part of all recent distributions. The resolver is implemented as additional nsswitch module `mdns`, which unfortunately is not included in RHEL/CentOS. Without this module only applications that use Avahi API are able to resolve mDNS host names. However, `mdns` module can be installed from [EPEL](https://fedoraproject.org/wiki/EPEL) repository (so be sure to enable it first).

This example is specific to RHEL/CentOS 7, but most Linux distributions use the same components.

Before starting be sure that your machine has locally configured host name:

{% highlight bash %}
# hostnamectl status | grep Static
{% endhighlight %}

If it returns `n/a`, set the hostname to your FQDN:

{% highlight bash %}
# hostnamectl set-hostname foobar.example.com 
{% endhighlight %}

Next, install the necessary packages:
 
{% highlight bash %}
# yum -y install avahi nss-mdns
# systemctl start avahi-service
# systemctl enable avahi-service
{% endhighlight %}

Next, enable the `mdns` module in `/etc/nsswitch.conf` by modifying the `hosts:` line:

`hosts:		files mdns_minimal [NOTFOUND=return] dns mdns`

If you're using IPv4 only, then use `mdns4_minimal` and `mdns4` instead.

That's it! You should be able to resolve your own name in `local` domain now:

%{ highlight bash %}
# getent hosts foobar.local
10.0.0.100	foobar.local
{% endhighlight %}

## Configuring mDNS in NetBSD

Check if your host name is correctly configured:
{% highlight bash %}
# hostname
xyzzy.example.com
{% endhighlight %}

If not, then be sure to set it:

{% highlight bash %}
# echo 'hostname=xyzzy.example.com' >> /etc/rc.conf
# /etc/rc.d/network restart
{% endhighlight %}

All the necessary components are included in base system installation since NetBSD 6.0. The mdnsd service is the same mDNSResponder implementation as used on OS X and iOS.  

{% highlight bash %}
# echo 'mdnsd=YES' >> /etc/rc.conf
# /etc/rc.d/mdnsd start
{% endhighlight %}

Additional nsswitch module also needs to be enabled, to support resolving host names using mDNS. Edit the `hosts:` line in `/etc/nsswitch.conf` to include mdnsd module:

`hosts: files dns mdnsd`

You're done. Testing can be done with the `getent` command just as on Linux:

%{ highlight bash %}
# getent hosts xyzzy.local
10.0.0.110	xyzzy.local
{% endhighlight %}

## Firewall considerations

Your firewall configuration must allow incoming traffic on UDP port 5353. Remember that if you use both IPv4 and IPv6, this port needs to be enabled for both IP protocols. For IPv4, mDNS uses multicast address `224.0.0.251`, for IPv6 `[FF02::FB]`.

