---
layout: post
title: Spice Desktop -  Limiting Access in a Multi-User Environment
categories: ['computing']
tags: ['it', 'virtualisation', 'vnc']
---

We have begun investigating [Spice](http://www.spice-space.org/), an open-source accelerated 'remote desktop' system for connecting to our virtual machines. so far it appears to be extremely powerful and fits our needs well.  
  
We are running a linux computing lab, and want to run a Windows virtual machine, accessed by Spice, only by the person currently logged in on the console, though there may be multiple users logged in remotely.  
  
However, one of the problems with Spice is that it lacks in authentication methods. The only method available to authenticate from the client appears to be to set a password. However, this is set via qemu/libvirt at the time of guest instantiation, meaning that it is difficult to keep a guest running permanently, and allow only a single user (out of many on that system) to access it, since the password would be known to everyone.  
  
We used the following tools to solve this problem - iptables and the console.handlers locking mechanism found in /etc/security.  
  
Firstly, we configure spice to only listen on the localhost interface (127.0.0.1) on port 5930. _(Details are in the Spice documentation on how to do this)._  
  
Next, we use the iptables to set a 'default deny' to that port:  
  
{% highlight text %}  
iptables -A OUTPUT -o lo -p tcp --dport 5930 -j REJECT  
{% endhighlight %}  
  
Next we write 2 scripts - **allow-user** and **deny-user** which add (and remove) a new rule using the **user** module in iptables:  
  
{% highlight text %}  
#allow-user  
iptables -A OUTPUT -o lo -p tcp -m owner --uid-owner $1 --dport 5930 -j ACCEPT  
{% endhighlight %}  
{% highlight text %}  
#deny-user  
iptables -D OUTPUT -o lo -p tcp -m owner --uid-owner $1 --dport 5930 -j ACCEPT  
{% endhighlight %}  
  
These scripts could be triggered by a variety of methods, depending on what action you want to cause access to Spice to change. In our case, we want to trigger access based on console login.  
  
We edit **/etc/security/console.handlers** to execute these 2 scripts:  
  
{% highlight text %}  
/usr/bin/allow-user lock user  
/usr/bin/deny-user unlock user  
{% endhighlight %}  
  
The **lock/unlock** flag signifies on which action to run this script (login=lock, logout=unlock). The **user**setuid flag is also passed. (see _man console.handlers_ for more info).  
  
Now when a user logs in on the console, an ACCEPT rule will be added granting them access to the port on which Spice is listening. Secondary console logins (e.g a second user dropping to a VT and logging in) won't trigger the handler scripts.
