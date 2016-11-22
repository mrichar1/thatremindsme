---
layout: post
title: The Pacemaker ping Resource Agent
categories: ['computing']
tags: []
---

Pacemaker has a resource agent designed to allow you to monitor network connectivity, and allocate other resources based on the connectivity score.  
  
There are 3 resource agents which do the same thing:  
  
ocf:heartbeat:pingd **(Deprecated**)  
ocf:pacemaker:pingd  
ocf:pacemaker:ping  
  
The main difference between the last two is that ping uses the system ping tool, allowing it to run on non-Linux systems. In general it is advisable to use the **ping** RA. In either case the name of the attribute defaults to _pingd_.  
  


### Setup

  
  
To start, you should create one ping primitive, and clone it to all nodes:  
  
{% highlight text %}  
primitive ping-gateway ocf:pacemaker:ping \  
params host_list="192.168.1.1" multiplier="100"  
clone pingclone ping-gateway meta interleave="true"  
{% endhighlight %}  
**host_list** is a space-separated list of one or more hosts to try to ping.  
  
**multiplier** is an amount to multiply successful pings by. This gives the final 'ping score' which is stored as an attribute, named **pingd**. You can then use this score to create location rules, associating other resources with appropriate ping scores. The exact number isn't important at this point, but can be useful when looking at more complicated scoring examples.  
  
Note that when cloning a ping resource, it is best to use the **interleave** meta option. This 'unlinks' the various clones, such that the resource will be considered started when any one of the clones is started, and restarting any individual clone won't affect the other clones.  
  


### Scoring

  
  
Assuming you have one host in your host_list, if it pings successfully, this will count as _1_. This will be multiplied by the **multiplier** value, here giving a score of _100_. Thus, if you have 2 nodes running the ping clone, where _nodeA_ can ping the target, but _nodeB_ can't, then nodeA will score _100_, while _nodeB_ will score _0_.  
  
Similarly, if you had two targets in the host_list, where _nodeA_ can ping both, but _nodeB_ can only ping one, then _nodeA_ would score _200_ and _nodeB 100_.  
  


### Location Rules

  
  
There are several ways to use the ping attribute in deciding location rules. Assuming you had another resource (resourceA) that you wanted to locate with a good ping score:  
  
{% highlight text %}  
location resAwithping resourceA \  
rule -inf: not_defined pingd or pingd lte 0  
{% endhighlight %}  
  
This rule will cause resourceA to not run on any nodes where the pingd attribute is undefined (e.g. ping RA is not running) or the ping score is less than or equal to 0 (e.g. pings are failing).  
  
A problem with this rule is that in the '2 hosts' example above, a score of 100 or 200 would both cause this rule to _not_ 'trigger', meaning either node might be considered suitable for your resource - probably what you want.  
  
A more complicated rule might be:  
  
{% highlight text %}  
location resAwithping resourceA \  
rule pingd: defined pingd  
{% endhighlight %}  
  
This rule is slightly different - it used the pingd attribute 'score' as the score for the location rule for that node. This means that if there are different nodes with different scores then the location rule with the highest score will 'win'. This works for both the '0 and 100', and the '100 and 200' examples given above.  
  
In this example choosing a multiplier number requires more attention, since by varying it you can affect the 'importance' of ping location rules compared to other location rules.  
  


### Testing

  
  
To test out a ping resource, you can temporarily block ping packets via the iptables firewall:  
  
{% highlight text %}  
iptables -A OUTPUT -p icmp -j DROP  
{% endhighlight %}  
  
Once this rule is applied, you can look at the pingd attribute 'score' for that node:  
  
{% highlight text %}  
crm_attribute -G -t status -N  -n pingd  
{% endhighlight %}  


### Example

  
  
We have 2 nodes - nodeA and nodeB. We want resourceA to run on the node which has the best network connectivity, but, all things being equal, we want it to prefer nodeA:  
  
{% highlight text %}  
primitive ping-nodes ocf:pacemaker:ping \  
params host_list="192.168.1.1 192.168.1.2" multiplier="100"  
clone pingclone ping-nodes  
  
location prefernodeA resourceA \  
rule 50: #uname eq nodeA  
  
location resourceAwithping resourceA \  
rule pingd: defined pingd  
{% endhighlight %}  
  
In this example, with both nodes working as expected:  
  
**nodeA** = 50 + 100 + 100 = **250**  
nodeB = 100 + 100 = 200  
  
When nodeA stops being able to ping one of the ping targets:  
  
nodeA = 50 + 100 = 150  
**nodeB** = 100 + 100 = **200**  
**Note** the importance of keeping the location preference below that of the ping multiplier!
