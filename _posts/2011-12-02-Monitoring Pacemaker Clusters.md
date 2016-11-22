---
layout: post
title: Monitoring Pacemaker Clusters
categories: ['computing']
tags: ['crm_mon', 'nagios', 'pacemaker', 'snmp']
---

There seem to be lots of questions about how best to monitor pacemaker clusters without having a console with crm_mon running in the background all the time. I've spent some time playing around with the various options, and none of them are ideal. However, I've found a method which fulfils most of my needs - easy to implement, small-scale, detailed but not too many false-positives, and using Nagios, which we already run. I'll go through the various methods that didn't work for me - if you want to know the one that did, jump to the end!  
  


### crm_mon with HTML output

  
{% highlight text %}  
crm_mon --daemonize --as-html /path/to/docroot/filename.html  
{% endhighlight %}  
  
This is simple enough, but there are a couple of problems. Firstly, you need to be writing the output somewhere there is a web server running, which might not be ideal in your cluster environment. Secondly, this requires you to watch the results, so its not really any different to watching crm_mon in a console window.  
  


## Email Notifications

  
  
Email notifications can be set up for state changes in the pool with crm_mon daemonized:  
  
{% highlight text %}  
crm_mon --daemonize --mail-to user@example.com --mail-host mail.example.com  
{% endhighlight %}  
  
The problem here is false-positives. crm_mon will email you when pretty much anything happens - great if you want to put effort into mail filtering, or like being on top of everything, but annoying if you don't (and possibly a threat to your old flakey mail server!  
  


### crm_mon Nagios support (but not the solution!)

  
{% highlight text %}  
crm_mon -s  
{% endhighlight %}  
  
crm_mon can output a nagios-style simple command line with appropriate return codes to be parsed by Nagios. This can then be tied to a nagios ssh-plugin command or used with NRPE (examples are given in the following article: [Monitoring Pacemaker with Nagios and NRPE](http://offandthenonagain.wordpress.com/2010/08/17/monitoring-pacemaker-with-nagios-and%C2%A0nrpe/ "Monitoring Pacemaker with Nagios and NRPE" ))  
  
The problem here is the opposite of emails - the information you get mostly relates to the state of nodes, not resources. Thus, failing resources, or other situations which you might want to know about, get missed.  
  


### SNMP messages

  
{% highlight text %}  
crm_mon --daemonize --snmp-traps snmptrapd.example.com  
{% endhighlight %}  
  
SNMP is probably the 'best' way to monitor a cluster - but for my setup there were a couple of issues - the main one being that SNMP messages are designed to really be computer, and not human-readable. This relies on having some kind of parser to translate SNMP messages to human-readable output, and while snmptt is good at this, it still needs some manual intervention.  
  
This, coupled with the need for a monitoring system to actually handle/present these messages, meant that I'd probably need to implement something like Zabbix or Zenoss - fine if you're not running anything already, but not as good if you havea monitoring system set up, as we did with Nagios.  
  
If you do want to investigate this avenue, then the following example snmptrapd.conf is probably useful to get you started: [Monitoring Pacemaker with SNMP](http://oss.clusterlabs.org/pipermail/pacemaker/2011-July/010932.html "Monitoring Pacemaker with SNMP" )  


### The solution - A custom crm_mon Nagios plugin

  
  
I had almost given up on this problem when I discovered a simple perl Nagios plugin that had been written to parse the output from crm_mon and return more state information and better description text: [Check CRM: Nagios Plugin](http://exchange.nagios.org/directory/Plugins/Clustering-and-High-2DAvailability/Check-CRM/details "Check_CRM" )  
  
This plugin can be used in place of crm_mon -s in the 'Monitoring Pacemaker with Nagios and NRPE' article above - making a call to this script instead of 'crm_mon -s'. The _-w_ option is especially useful, since it will cause situations like nodes in standby to be marked as Warnings.  
  
For reference, my Nagios config (using ssh) is as follows:  
  
{% highlight text %}  
## localhost.cfg  
define host {  
use linux-server  
host_name VM Cluster  
address vmnagios.example.com  
}  
  
define service {  
use local-service  
host_name VM Cluster  
service_description VM Cluster status  
servicegroups vmclusters  
check_command check_vmcluster  
normal_check_interval 15  
}  
  
define servicegroup {  
servicegroup_name vmclusters  
alias VM Clusters  
}  
  
## commands.cfg  
define command {  
command_name check_vmcluster  
command_line $USER1$/check_by_ssh -H $HOSTADDRESS$ -t 30 -l nagios -C "/usr/sbin/check_crm_v0.5 -w"  
}  
  
  
  
  
{% endhighlight %}
