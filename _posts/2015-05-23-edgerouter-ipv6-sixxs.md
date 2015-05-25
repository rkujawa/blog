---
layout: post
title: IPv6 routing with EdgeRouter and SixXS
---

Recently I have migrated my home router from OpenBSD to Ubiquiti EdgeRouter. In this post I shall explain how to configure IPv6 tunnel provided by [SixXS](http://sixxs.net) and how to route an IPv6 subnet into your internal network.

<!-- more -->
## Prerequisites

- Ubiquiti EdgeRouter (any model) updated to software 1.6.0 or newer.
- Public, static IPv4 address.
- SixXS account, with already created tunnel and subnet.
- Basic knowledge of EdgeOS (how to use command line, edit configuration, commit and save).

This tutorial assumes you already have your EdgeRouter configured for IPv4 operation and that you have static, public IPv4 address. It is more complicated to configure the tunnel if you don't have static IPv4 address on an external interface, since EdgeRouter does not provide support for SixXS dynamic tunnel configuration protocol known as [TIC](https://www.sixxs.net/tools/tic/). Operation with a dynamic IPv4 address is still possible, as one of the users [created a script](http://community.ubnt.com/t5/EdgeMAX/SIXXS-connectivity-without-AICCU-with-minimum-system/td-p/550538) to support the heartbeat protocol.

Update 2015-05-24: Florian G. Pflug prepared a package that adds support for AICCU (TIC protocol tool) to Vyatta. I didn't test it but it looks promising. See [here](https://github.com/fgp/vyatta-aiccu).

## Configuring the tunnel interface

Let's suppose SixXS provided you with a static tunnel:

<table>
	<tr>
		<td>PoP IPv4</td><td>200.200.100.100</td>
	</tr>
	<tr>
		<td>Your IPv4</td><td>Static, currently 100.11.12.13</td>
	</tr>
	<tr>
		<td>IPv6 Prefix</td><td>2001:1:2:3::1/64</td>
	</tr>
	<tr>
		<td>PoP IPv6</td><td>2001:1:2:3::1</td>
	</tr>
	<tr>
		<td>Your IPv6</td><td>2001:1:2:3::2</td>
	</tr>
</table>

	set interfaces tunnel tun0 address '2001:1:2:3::2/64'
	set interfaces tunnel tun0 encapsulation sit
	set interfaces tunnel tun0 local-ip 100.11.12.13
	set interfaces tunnel tun0 remote-ip 200.200.100.100

In the above example `address` is your IPv6 end of the tunnel, `local-ip` is your external IPv4 address, `remote-ip` is the IPv4 address of PoP assigned to you by SixXS.

Note there's no need to manually set the IPv6 address of the PoP at this point.

After committing these changes, it should now be possible to `ping6` the PoP's IPv6 address from the router.

## Configuring the default route

Just add a static route to `::/0` (IPv6 equivalent of 0.0.0.0) through your PoP's IPv6 address. The `next-hop` parameter should always point to IPv6 address of the PoP.

	set protocols static route6 '::/0' next-hop '2001:1:2:3::1'

After committing this change, you should be able to successfully `ping6` addresses on the internet from the router.

## Routing the IPv6 subnet into your internal network

Fist thing to do is to check your subnet configuration at SixXS site. These days SixXS is by assigning a separate /64 network for every tunnel. However, you may still want to request a separate /48 network. Let's say a network with prefix `2001:5:6::/64` was assigned to you. To be able to use addresses with this prefix, you just need to configure internal ethernet interface (in this example `eth2`).

	set interfaces ethernet eth2 address '2001:5:6::1/64'

Now, machines on the internal network can have addresses assigned from this prefix. They will use the address configured above as their gateway.

## Enabling stateless IPv6 addressing for internal network 

The easiest way of configuring IPv6 addresses on internal network is stateless autoconfiguration, as described in [RFC 4862](https://tools.ietf.org/html/rfc4862). Alternatively, you could use DHCPv6, which is also supported by EdgeOS. However, since stateless configuration is so easy to achieve, we'll do that here.

	set interfaces ethernet eth2 ipv6 router-advert prefix '2001:5:6::/64' 
	set interfaces ethernet eth2 ipv6 router-advert send-advert true

Don't forget to commit your changes.

All major modern operating systems support stateless IPv6 configuration out of the box. At least in OS X, NetBSD and recent CentOS/RHEL versions it is enabled by default.

On Linux you can quickly check for public IPv6 addresses with the following command:
{% highlight bash %}
$ ip a s | grep inet6 | grep global
    inet6 2001:5:6:(...)/64 scope global dynamic 
{% endhighlight %}

Congratulations, machines on your internal network now have public IPv6 addresses. You should be able to `ping6` the internet addresses, visit IPv6-enabled web sites from machines on the internal network, and so on.

## Firewall considerations

Configuring the firewall is out of scope of this article. However, here are a few helpful tips:

- Be sure to let IP protocol 41 (also known as 6in4 or sit protocol) through your public IPv4 interface.
- It is also necessary to let the PoP ping your tunnel's public IPv6 address, as SixXS.net is using ICMPv6 to check if your end of the tunnel is alive.
- To secure the traffic going into your network, you will most likely need a completely separate set of rules for IPv6, since the IPv6 internet traffic is coming from `tun0` interface, not from the `ethX` where your external IPv4 address is configured. Also note that NAT for IPv6 is neither necessary nor recommended (thank God/IETF) and in the above example all machines on internal network will get public IPv6 addresses. This requires careful firewall planning, in a much different way from typical IPv4 setup.

