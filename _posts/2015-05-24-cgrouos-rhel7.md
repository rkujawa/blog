---
layout: post
title: RHEL/CentOS 7 service resource management with cgroups
---

The Linux control groups (cgroups) is one of my favourite tools when I need to set up how system resources are shared between running applications. With cgroups we can limit and prioritize access to CPU, memory and I/O devices (and even more).

Version 7 of RHEL and CentOS brought us the new `/sbin/init` replacement - systemd, with all its err... inevitable changes. One area, where these changes can be confusing at first sight, is cgroups configuration. The old way of configuring it, by means of editing several `/etc/cg*.conf` files is still available but no longer recommended.

So how to handle `cgroups` on RHEL/CentOS 7? Read up.

<!-- more -->
Imagine we have a service called `foo.service` that eats up a lot of CPU time. We'd like to limit its access to the CPU, so that other services executed on the same machine can be more responsive. We also have an important `bar.service`, where we want to have most CPU time assigned.

Lets take a look at the configuration of our imaginary `foo.service`:

{% highlight bash %}
# systemctl status foo.service
foo.service - The foo service that does nothing useful
   Loaded: loaded (/etc/systemd/system/foo.service; disabled)
   Active: inactive (dead)
{% endhighlight %}

We can see the service configuration file is `/etc/systemd/system/foo.service`. Now let's take a peek inside to see what this service really does:

{% highlight bash %}
# cat /etc/systemd/system/foo.service
[Unit]
Description=The foo service that does nothing useful
After=remote-fs.target nss-lookup.target

[Service]
ExecStart=/usr/bin/sha1sum /dev/zero 
ExecStop=/bin/kill -WINCH ${MAINPID}

[Install]
WantedBy=multi-user.target
{% endhighlight %}

Seeing how this service just runs `sha1sum`, we can expect it will cause a slight load on the system. Let's run the service and confirm this:

{% highlight bash %}
# systemctl start foo.service
# systemctl enable foo.service 
ln -s '/etc/systemd/system/foo.service' '/etc/systemd/system/multi-user.target.wants/foo.service'
# systemctl status foo.service
foo.service - The foo service that does nothing useful
   Loaded: loaded (/etc/systemd/system/foo.service; enabled)
   Active: active (running) since Sun 2015-05-24 19:49:32 CEST; 11s ago
 Main PID: 14306 (sha1sum)
   CGroup: /system.slice/foo.service
           └─14306 /usr/bin/sha1sum /dev/zero
(...)
# ps -p 14306 -o pid,comm,cputime,%cpu
  PID COMMAND             TIME %CPU
14306 sha1sum         00:02:10 99.6
{% endhighlight %}

Indeed, since this command will eat up all the available CPU time, the other important processes and services might starve for CPU time. Let's make sure this never happens (with cgroups).

At this point, in RHEL/CentOS 6, we'd have to set up a new cgroup and assign processes of a given service to it. With systemd it no longer necessary to create the cgroup manually. Instead, systemd introduces a new concept of `slice`, which is used to manage resources of a group of processes. Most services are placed in `system.slice`. Note that `systemctl status` command already informed us, that `foo.service` belongs to `system.slice`.

Several other slices are present on a default system installation:
{% highlight bash %}
# systemctl list-units --type=slice
UNIT               LOAD   ACTIVE SUB    DESCRIPTION
-.slice            loaded active active Root Slice
system-getty.slice loaded active active system-getty.slice
system.slice       loaded active active System Slice
user-0.slice       loaded active active user-0.slice
user-1000.slice    loaded active active user-1000.slice
user.slice         loaded active active User and Session Slice
{% endhighlight %}

If we wanted to just know the slice name of `foo.service` we could use:

{% highlight bash %}
# systemctl show -p Slice foo.service
Slice=system.slice
{% endhighlight %}

So, configuring cgroups with systemd is possible either through modifying slice parameters, of service parameters. In most circumstances, where service is simple and consists of one systemd unit, it is enough to modify the properties of a given service unit.

{% highlight bash %}
# systemctl set-property unit-name property=value
{% endhighlight %}

For example:

{% highlight bash %}
# systemctl set-property foo.service CPUShares=250
{% endhighlight %}

The `CPUShares` property modifies the value of cgroups `cpu.shares` controller, which is used to limit (or rather prioritise) the CPU usage. The default "neutral" value of `cpu.shares` is 1024. Setting high value increases the "priority" (i.e. causes more CPU time to be assigned to a given process).

Of course, CPUShares is not the only property we could modify. The `systemd.resource-control(5)` man page has more information about cgroups-related parameters that you can pass to `systemctl`.

Note that this change is persistent and the value of `cpu.shares` will also be set to 250 upon the next `foo.service` start. If we wanted this change to be temporary, we could use `--runtime` parameter to `systemctl set-property`.

Setting the property caused additional file `90-CPUShares.conf` to be created under `/etc/systemd/system/foo.service.d`. This file contains parameters to be appended to the configuration of `foo.service` unit. Actually applying the change requires reloading `systemd` and restarting `foo.service`:

{% highlight bash %}
# systemctl daemon-reload
# systemctl restart foo.service
{% endhighlight %}

Now let's see if the `sha1sum` process really has `cpu.shares` controller value assigned properly.

{% highlight bash %}
# cat /sys/fs/cgroup/cpu/system.slice/foo.service/cpu.shares 
250
# systemctl show -p MainPID foo.service
MainPID=14457
4
# cat /proc/14457/cgroup | grep foo
3:cpuacct,cpu:/system.slice/foo.service
1:name=systemd:/system.slice/foo.service
{% endhighlight %}

Now, let's increase the `cpu.shares` controller value for our other service, the `bar.service`.

{% highlight bash %}
# systemctl set-property bar.service CPUShares=2000
# systemctl daemon-reload
# systemctl restart bar.service
# systemctl status bar.service
bar.service - The bar service that does nothing useful
   Loaded: loaded (/etc/systemd/system/bar.service; enabled)
  Drop-In: /etc/systemd/system/bar.service.d
           └─90-CPUShares.conf
   Active: active (running) since Sun 2015-05-24 20:33:21 CEST; 2min 53s ago
 Main PID: 15057 (md5sum)
   CGroup: /system.slice/bar.service
           └─15057 /usr/bin/md5sum /dev/zero
(...)
{% endhighlight %}

And... let's compare how much CPU time are processes belonging to our two services eating:

{% highlight bash %}
# ps -p 15057,14457 -o pid,comm,cputime,%cpu
  PID COMMAND             TIME %CPU
15057 md5sum          00:00:24 94.6
14457 sha1sum         00:00:01 5.2
{% endhighlight %}

As you can see, the `md5sum` process, as started by `bar.service` is happily executing, taking 95% of our CPU time, while `sha1sum`, as executed by `foo.service` slowly eats away CPU taking only 5%.

By the way, Red Hat provides an excellent documentation on this topic: [Red Hat Enterprise Linux 7 Resource Management Guide](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/7/html/Resource_Management_Guide/index.html).

In the next article in this series, I will explain how to use systemd and cgroups to manage resource for processes that are not a part of a service. Stay tuned.
