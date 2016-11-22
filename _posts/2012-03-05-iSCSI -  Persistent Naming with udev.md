---
layout: post
title: iSCSI -  Persistent Naming with udev
categories: ['computing']
tags: []
---

One of the downsides to using iSCSI for network file store is that it can be difficult to keep track of what device name should be used to reference each lun.  
  
By default, they appear in {% highlight text %}/dev/disk/by-path/{% endhighlight %} as follows:  
  
{% highlight text %}  
ip-192.168.140.10:3260-iscsi-iqn.2010-05.com.example:alice-lun-2 -&gt; ../../sdb  
{% endhighlight %}  
  
This is of course entirely accurate, but is perhaps not what we want - especially since any references to this resource will include the ip address, which in some environments might change.  
  
There is however a useful feature of the {% highlight text %}scsi_id{% endhighlight %} command which is that it can read the extended values of iSCSI resources from the exported disk.  
  
On our iSCSI server we set one of the following page values: _Device Identification Vital Product Data (page 0x83)_ or _Unit Serial Number (page 0x80)_ to a name for this lun - in this case _alice_  
  
We then add the following udev configuration to {% highlight text %}/etc/udev/rules.d/50-iscsi-persistent-naming{% endhighlight %}  
{% highlight text %}  
BUS=="scsi", SYSFS{vendor}=="IET", SYSFS{model}=="VIRTUAL-DISK", KERNEL=="sd?", PROGRAM="scsi_id --whitelisted -p0x80 -d $tempnode", RESULT=="?*", SYMLINK+="disk/by-id/iscsi-%c{3}"  
{% endhighlight %}  
  
Setting the page queried to either 80 or 83 as appropriate. Note that as I'm using the ietd iSCSI server, the SYSFS vendor and model fields are set as above - you may either exclude these, or use the scsi_id command to find out what yours appear as on the client.  
  
This command will map the iSCSI device to {% highlight text %}/dev/disk/by-id/iscsi-X{% endhighlight %} where X is replaced by the value given in the page specified.  
  
  

