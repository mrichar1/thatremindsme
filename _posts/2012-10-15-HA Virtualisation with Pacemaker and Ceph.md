---
layout: post
title: HA Virtualisation with Pacemaker and Ceph
categories: ['computing']
tags: ['ceph', 'drbd', 'ha', 'iscsi', 'pacemaker']
---

I've recently begun playing with ceph's rbd pool as a way to provide network block devices for libvirt guests managed through Pacemaker, having had success with [drbd](http://www.linbit.org) and iscsi. This post should be considered notes of my ongoing experiments, and not a hard-and-fast 'howto' for this concept. Nevertheless, it might be useful to someone!  
  


## Concepts

  
  
We're using ceph rbd pool (RADOS block device) to offer up the storage for the VMs. If you've used iscsi before, this is a similar concept, but with a replicated, distributed backend for the data.  
  
We're using pacemaker with the VirtualDomain RA to manage libvirt (kvm) instances.  
  
  


## Hardware

  
  
You'll need a minimum of 2 nodes to run the osd daemons (object store - i.e. the data) and 3 or more nodes (always an odd number for quorum) nodes for the mon daemons (ceph pool monitoring). The ceph documentation gives suggested hardware requirements for these.  
  
You'll also need 2 (or more) nodes for VMs to allow live migration. CPU and memory are important here.  
  
I started using 4 nodes - 2 disk servers, and 2 vm servers. 3 (or more) nodes run pacemaker (to allow quorum) and one (or more) vm server hosts the extra mon daemon. This was mostly due to the hardware I had to hand falling into 2 categories of 'fast disk' and 'fast cpu/mem' - but the pool can expand later as needed.  
  


## Ceph Configuration

  
  
Follow the [ceph documentation](http://ceph.com/docs/master/start/quick-start/) on setting up a pool - using mkcephfs (or ceph-deploy) to get things going.  
  
Make sure that the path you point your osd's at is the mountpoint for an xfs or btrfs filesystem.  
  
I use the following /etc/ceph/ceph.conf:  
  
{% highlight ini %}  
[global]  
auth supported = none # see also cephx  
  
[mon]  
mon data = /srv/ceph/mon-$id  
[mon.0]  
host = disksrv1  
mon addr = 192.168.1.1:6789  
[mon.1]  
host = disksrv2  
mon addr = 192.168.1.2:6789  
[mon.2]  
host = vmsrv1  
mon addr = 192.168.1.3:6789  
  
[osd]  
osd data = /srv/ceph/osd-$id  
osd journal = /srv/ceph/osd-$id/journal #In production, use an SSD or at least it's own partition  
osd journal size = 2000 # Only needed if using a file for the journal (2GB is a good start for GBe)  
  
[osd.0]  
host = disksrv1  
[osd.1]  
host = disksrv2  
{% endhighlight %}  
  
When you start ceph, run {% highlight text %}ceph status{% endhighlight %} on one of the mons and wait for it to settle - look out for {% highlight text %}HEALTH_OK{% endhighlight %} and check that the appropriate number of mons and osds are running.  
  


## Guest disk Creation

  
  
You now need to create a disk in the pool for the guest to use. This is done from any of the mon nodes with the following command - replacing  with the disk size, and  and  with the name of your rbd pool and guest vm. If --pool is omitted, it will default to the {% highlight text %}rbd{% endhighlight %} pool:  
  
{% highlight text %}  
rbd create [guestname] --size [megabytes] --pool [poolname]  
{% endhighlight %}  
  
Note that rbd images are thinly provisioned - that is no space will be used unless files are written to the image, and the size is only an upper limit. you can later change the size of a disk with:  
  
{% highlight text %}  
rbd resize [guestname] --size [megabytes] --pool [poolname]  
{% endhighlight %}  
  
Full documentation on the rbd commands is available on the [Ceph wiki](http://ceph.com/wiki/Rbd)  
  
By default, ceph pools have replication set at 2 - i.e 2 copies of all data. If you are paranoid, you can up this number, but be aware that this requires a corresponding increase in the number of osds, and also will incur a performance hit.  
  


## Libvirt configuration

  
  
RBD support in qemu has been around for a while - definitely in the 0.15.1 releases. Some distibutions don't compile with it enabled - in which case you need to compile it yourself with the {% highlight text %}--enable-rbd{% endhighlight %} configure option.  
  
To see if your version has rbd support, then try the following command (after having created the rbd image above):  
  
{% highlight text %}  
qemu-img info -f rbd rbd:[poolname]/[guestname]  
{% endhighlight %}  
  
you should see something like:  
  
{% highlight text %}  
image: rbd:rbd/guest1  
file format: rbd  
virtual size: 1.0G (1073741824 bytes)  
disk size: unavailable  
cluster_size: 4194304  
{% endhighlight %}  
  
You need to ensure that your various libvirt daemons can communicate for migration. You can use TLS (recommended, using certificates for auth), ssh, or tcp with no auth (good for testing, but insecure). See the libvirt [Remote documentation](http://libvirt.org/remote.html) for information. Note that many distros also need you to tweak the libvirtd startup options to include {% highlight text %}--listen{% endhighlight %} in {% highlight text %}/etc/sysconfig/libvirtd{% endhighlight %} or {% highlight text %}/etc/defaults/libvirtd{% endhighlight %}.  
  
Once communication is established, you need to create a guest libvirt XML configuration for each guest, and deploy this to all the VMs. The important addition is the inclusion of a disk of type 'network' with source protocol 'rbd' and the name set appropriately to [poolname]/[guestname] from the rbd commands above.  
  
{% highlight xml %}  
  
  
  
  
  
  
{% endhighlight %}  


## Pacemaker

  
  
I am assuming here that you are familiar with the workings of pacemaker. At present, this guide only covers using pacemaker to manage libvirt - though there are resource agents for monitoring ceph's init daemons, and for 'mounting' rbd images available written by the excellent people at [](http://www.hastexo.com/blogs/florian/2012/03/08/ceph-tickling-my-geek-genes)Hastexo. You should also ensure that your OS is automatically starting libvirtd, but **not** automatically starting any guests (e.g. libvirt-guests init.d scripts, libvirt autostart etc).  
  
Be aware that if you are running pacemaker to monitor VirtualDomain guests AND ceph, you may need to put in place location rules to prevent ceph running on hosts with 'VM hardware' and libvirt running on hosts with 'disk hardware'.  
  
The relevant crm configuration snippet for each guest will look something like this:  
  
{% highlight bash %}  
primitive guest1 ocf:heartbeat:VirtualDomain \  
params config=/etc/libvirtcfg/guest1.xml \  
hypervisor="qemu:///system" migration_transport="tls" meta allow-migrate="true" \  
op start timeout="300" op stop timeout="300" \  
op monitor depth="0" timeout="30" interval="10" \  
op migrate_from timeout="300" \  
op migrate_to timeout="300"  
{% endhighlight %}  
  
If you are used to running with DRBD and iscsi, this might seem quite short - however since libvirt is handling all of the rbd access, much of the complexity disappears.
