---
layout: post
title: Automatic Logout with Gnome Screensaver
categories: ['computing']
tags: ['gconf', 'gnome', 'labs', 'linux']
---

There are a few nice features in gnome screensaver which allow you to control logouts - specifically automatic logouts and display of the logout button - ideal for setting up Linux computing labs or other environments where locked computers can cause problems.  
  
These features are documented, but not that easy to find unless you know about them. The secret to getting to them all is to use the Gnome _gconf-editor_ command. For these you will need a fairly recent version of gnome-screensaver.  
  
Once you run gconf-editor you'll find a tree-like structure for configuring a whole host of Gnome applications. However, we're interested in the _/apps/gnome-screensaver_ section.  
  
There are several options in here:  
  


  

  * _lock_enabled_ \- Enabling this option will display a password prompt to unlock the screen. You probably already have this enabled.
  

  * _logout_enabled_ \- Enabling this option places a logout button on the password prompt when the screensaver is locked.
  

  * _logout_delay_ \- Set this to the number of minutes of delay you want from the screen locking to the logout button appearing on the password prompt. Useful if you want to let people lock the screen for a short while, but let someone else kick them off if its been locked for ages. (0 is immediately).
  

  * _enable_auto_logout_  

  * _auto_logout_delay_ \- The number of minutes delay before the user is automatically logged out (0 is immediately).
  

  * _logout_command_ \- Set this to the command to execute when the automatic logout occurs/the logout button is pressed. A safe option is _gnome-session-save --kill --silent_. On earlier versions of gnome, you could use something like _killall gnome-session_.
  
  
There are lots of other options in this part of the gconf-editor, which might also be of interest to you.  
  
In our labs we set the following:  
  
{% highlight text %}  
logout_enabled= yes  
logout_delay = 30  
enable_auto_logout = yes  
auto_logout_delay = 240  
logout_command = gnome-session-save --kill --silent  
{% endhighlight %}  
  
This means that once the screen is locked, the logout button will appear on the password prompt after 30 minutes. After 4 hours of being locked, the user will be logged out automatically.  


