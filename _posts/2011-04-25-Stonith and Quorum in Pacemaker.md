---
layout: post
title: Stonith and Quorum in Pacemaker
categories: ['computing']
tags: []
---

One of the key requirements of Pacemaker is that there must be good communication between the nodes at all times. This is so that the cluster is always aware of what is going on, and can ensure that resources are properly managed. A break in communication can mean that nodes might each 'go off and do their own thing', at best a massive headache for a sysadmin, and at worst total data loss and system corruption. When nodes in a cluster begin to operate independently from one another, the situation is known as a **split-brain**, and each node or collection of nodes in this situation is known as a **sub-cluster**.  
  
You can do your best to mitigate this by ensuring you have 2 _independent_ routes of communication (and independent here means ideally physically separate wires, switches etc). However, whatever methods you use to ensure connectivity, there will always be one fateful day when for some reason communication with one or more nodes is lost, or corruption on a node causes it to start behaving erratically. At this point you need to have previously set up fencing.  
  


## Quorum

  
  
Quorum refers to the concept of 'voting' by nodes as to what should happen, where the majority vote wins. If you have 3 or more nodes in a pool, and 1 of them starts behaving erratically, the majority can decide amongst themselves that the other node should be dealt with. Of course, quorum by its very definition requires more than 2 nodes. With 2 nodes, the loss of communication means that no half of the cluster has any 'authority' to deal with the other half.  
  
So, to start with lets assume you have 3 or more nodes. Here, using quorum is easy. Whenever quorum is present, pacemaker will go with the majority vote on important decisions. When quroum is lost (i.e the cluster is separated into groups where no group has a majority of votes) the behaviour of the pool is determined by the **no-quorum-policy** property:  
  


  

  * _ignore_ \- Do nothing when quorum is lost.
  

  * _stop_ (default) - stop all resources in the affected cluster partition.
  

  * _freeze_ \- continue running existing resources, but don't start any stopped ones.
  

  * _suicide_ \- fence all nodes in the affected partition.
  

  
  
There are some explanations of the behaviour under each of these options here: [SLE HA Configuration Basics](http://doc.opensuse.org/products/draft/SLE-HA/SLE-ha_draft/cha.ha.configuration.basics.html)  


## Stonith

  
  
So, lets assume you have quorum set up with enough nodes, and suddenly one node disappears out of the cluster (_killall -9 corosync_ would do it!). What should now happen?  
  
Well, the cluster has no idea what that node is doing with the resources it was running - are they still running? Might they start running in the future? The only safe option is to kill that node as quickly as possible - by 'Shooting The Other Node In the Head' - or **Stonith** or short.  
  
To enable stonith, set the property:  
  
{% highlight text %}stonith-enabled=true{% endhighlight %}  
  
Once this is enabled (the default) your cluster will refuse to run unless at least one stonith resource is in place.  
  
Stonith devices take many forms, but all share one task - to power off of otherwise disable a node as quickly as possible so that it can't do anything that could corrupt data, or otherwise interfere with the proper working of the rest of the cluster.  
  
As to which stonith plugin to use, there are many to choose from, which can be seen by running:  
  
{% highlight text %}  
stonith -L  
{% endhighlight %}  
  
To see the documentation for each plugin, run the following (inserting the name from the above list):  
  
{% highlight text %}  
crm ra meta stonith:&lt;plugin name&gt;  
{% endhighlight %}  
  
Some caveats with stonith to be aware of are:  
  


  

  * Always choose the most 'independent' stonith plugin you can, in the following order:  
  

    1. Use a PDU to remotely turn off the node's power supply.
  

    2. Use an onboard controller like IPMI, which is independent to the node's OS, but still shares access, such as a common network port.
  

    3. Tell the node to turn itself off via a serial port, or even ssh or a similar protocol.
  

    4. Use _meatware_ which will notify a human to go turn it off (useful for testing).
  

    5. Finally, there are suicide and null plugins, but the best advice here is _just don't use them_
  

  

  * Most plugins will offer to _reset_ or _poweroff_ the node. If you use the former, make sure that you don't have pacemaker configured to start on boot, or you may end up with a node that comes up broken and tries to go back into service before you have a chance to diagnose why it was killed.
  

  * If your route of communication to the node is down for pacemaker, you might also have no route to stonith - this is especially true for devices such as IPMI which share a network connection and power source with the node.
  

  * You can setup more than one stonith device, and configure their use order using the _priority_ parameter. I always like to add _meatware_ at the lowest priority, so that if all else fails, at least I'll be notified to go switch off the offending node.
  

  


### 2-Node Clusters  
  
All of the above is fine with 3 or more nodes in your cluster, but what can you do if you run a 2-node cluster?  
  


### Quorum Nodes

  
  
The obvious advice is to turn it into a 3-node cluster for quorum. There is one problem with this of course - ensuring that resources don't run on that third node, so it is 'quorum-only'.  
  
There are 2 ways to achieve this:  
  


  

  1. The simplest way to achieve this is to add **standby=on** as a parameter to that node's configuration.
  

  2. Run a node in 'quorum-only' mode by starting corosync but not pacemaker.
  

  
  
The caveat with the first option is that it must at least be theoretically capable of running the probe/monitor step for all potential resources, or you will end up with a lot of 'not installed' errors when you load your configuration.  
  
The second option can be achieved by removing the _service_ section from your corosync.conf file, so that no pacemaker service is started. However, be aware that now your node won't show up in any of the pacemaker tools, and tracking its existence/performance becomes a lot harder.  
  


### SBD Stonith Device

  
  
This is another variation for 2-node clusters, which also requires a 3rd machine, though not one managed in this pacemaker cluster.  
  
Here, an SBD (Storage-Based Death) is a disk or disks that can be accessed from both (all) nodes. Loss of access to the disks, or noticing a fencing request being written to the disk will cause that node to fence itself.  
  
There is much more information on implementing SBD here: [SBD Fencing](http://linux-ha.org/wiki/SBD_Fencing)  


## Further Reading

  
  
The following are all worth a read to add more information to your understanding of these issues:  
  
[Split Brain and Quorum](http://techthoughts.typepad.com/managing_computers/2007/10/split-brain-quo.html) \- A more in-depth overview of these topics.  
  
[CRM Fencing](http://www.clusterlabs.org/doc/crm_fencing.html) \- Targeted more at specific stonith configurations  
  
[STONITH](http://www.linux-ha.org/wiki/STONITH) \- Information on the concept, and origins of the name.  

