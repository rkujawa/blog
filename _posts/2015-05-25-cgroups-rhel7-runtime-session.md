---
layout: post
title: RHEL/CentOS 7 run-time and session resource management with cgroups
---

In the previous post I explained how to use systemd and cgroups to limit resources available to services. Here I'll further expand on this topic, explaning how to manage resources for processes that are not a part of the service.

<!-- more -->
## Running odd jobs and unrelated processes

As you already know managing cgroups is easy with systemd, as long as you're managing a service... But what to do about some random program that we just want to run, without the complexity of creating a service configuration? It's possible to have a service or scope unit created on-demand (along with transient cgroups configuration). The `systemd-run` command can be used to achieve this:

{% highlight bash %}
# systemd-run --unit=unit_name --scope --slice=slice_name command
{% endhighlight %}

The `--scope` here is optional, if we pass it, the systemd will create a scope (running in the foreground), instead of a service (running in the background). The `--slice` parameter is also optional, if we don't pass it, the newly created unit will end up in the default `system.slice`.

Let's execute some `dd` as a (temporary) service under the care of systemd:

{% highlight bash %}
# systemd-run --unit=xyzzy dd if=/dev/vda of=/dev/null
{% endhighlight %}

Now investigate the newly created systemd unit:

{% highlight bash %}
# systemctl status xyzzy
xyzzy.service - /bin/dd if=/dev/vda of=/dev/null
   Loaded: loaded (/run/systemd/system/xyzzy.service; static)
  Drop-In: /run/systemd/system/xyzzy.service.d
           └─90-Description.conf, 90-ExecStart.conf, 90-RemainAfterExit.conf, 90-SendSIGHUP.conf
   Active: active (running) since Mon 2015-05-25 16:54:43 CEST; 5s ago
 Main PID: 15677 (dd)
   CGroup: /system.slice/xyzzy.service
           └─15677 /bin/dd if=/dev/vda of=/dev/null
(...)
{% endhighlight %}

Note that the unit is marked as `static`, as it does not have permanent unit configuration. 

Now we can modify the cgroups controllers via systemd properities:

{% highlight bash %}
# systemctl set-property --runtime unitname property=value
{% endhighlight %}

The `--runtime` parameter is important here. Since we have created a temporary unit, we'd like to immediately apply changes to cgroups configuration, instead of writing them to file. 

We can use `BlockIOReadBandwidth` property to limit the I/O bandwidth avaialble to the service. The `BlockIOReadBandwidth` property configures `blkio.throttle.read_bps_device` cgroups controller, which will cap the available bandwidth for this particular service. For other cgroups-related properties see the `systemd.resource-control(5)` man page.

In our example the `dd` command is reading from `/dev/vda` device and causing a slight system load by using the whole avaialble bandwidth to this device. Let's set the `BlockIOReadBandwidth` to limit read from `/dev/vda` to 1MiB/sec (1048576 bytes/sec).

{% highlight bash %}
# systemctl set-property --runtime xyzzy.service BlockIOReadBandwidth='/dev/vda 1048576'
{% endhighlight %}

That's it! You can now observe how `dd` is slowing down by sending `SIGUSR1` signal to it and investigating the journal.

{% highlight bash %}
# kill -USR1 15677
# journalctl _SYSTEMD_UNIT=xyzzy.service
(...)
May 25 17:38:25 vsq.home.c0ff33.net dd[15774]: 1881751552 bytes (1.9 GB) copied, 1316.8 s, 1.4 MB/s
(...)
{% endhighlight %}

Note that these statistics are calculated since the time `dd` was launched. That's why it still shows 1.4MB/s, as for some time `dd` was running with higher bandwidth available.

If we stop the `xyzzy.service`, it will terminate the `dd` process and remove all the transient configuration we have just created. So it's a great tool to run temporary processes, or jobs that will be executed only once and then forgotten. 

{% highlight bash %}
# systemctl stop xyzzy.service 
# systemctl status xyzzy.service
xyzzy.service
   Loaded: not-found (Reason: No such file or directory)
   Active: inactive (dead)
(...)
{% endhighlight %}

## Configuring cgroups for user sessions

What's also interesting, is the fact that slices are automagically created for every user upon first login (and all their processes are automatically assigned to thier personal slice). For example let's check my own slice:

{% highlight bash %}
$ systemctl status user-${UID}.slice
user-1000.slice
   Loaded: loaded
   Active: active since Wed 2015-05-06 16:26:29 CEST; 2 weeks 4 days ago
   CGroup: /user.slice/user-1000.slice
           └─session-463.scope
             ├─13753 sshd: rkujawa [priv]
             ├─13756 sshd: rkujawa@pts/0
             ├─13757 -bash
             └─13779 systemctl status user-1000.slice
{% endhighlight %}

If I had a second session open, processes with the other session would belong to a different scope, but all my processes would still belong to this one slice `user-1000.slice`. It means now there's an easy way to limit the resources of a particular user - just apply cgroups properties to his/her personal slice! 

For example, if we want to limit `cpu.shares` of all processes of user with UID 1000:
{% highlight bash %}
# systemctl set-property user-1000.slice CPUShares=100
# systemctl daemon-reload 
{% endhighlight %}

Now, all of this user's processes will have `cpu.shares` limited to `100`.
