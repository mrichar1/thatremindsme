---
layout: post
title: Disabling Shutdown but not Logout on Gnome
categories: ['computing']
tags: []
---

While this at the start seems like a simple challenge, it ended up being much harder than I anticipated.  
  
Firstly, I had naively assumed that there would be a gconf key for this - and there is, though it has the annoying problem of removing shutdown **and** logout from the Settings Menu.  
  
So my workaround was to firstly enable this gconf option, and then add back a logout icon onto the system tray.  
  
The gconf option is in {% highlight text %}/apps/panel/global/disable_log_out{% endhighlight %} on some versions of Gnome, or in {% highlight text %}/desktop/gnome/lockdown/disable_log_out{% endhighlight %} in other versions.  
  
Once this is set, your System menu will no longer have the Log Out or Shutdown options shown.  
  
Now, you need to add a system tray applet to give users a logout button.  
  
{% highlight python %}  
#!/usr/bin/python  
import subprocess  
import gtk  
def clicked(self):  
subprocess.Popen(["/usr/bin/gnome-session-save","--logout-dialog"])  
icon = gtk.StatusIcon()  
icon.set_from_file("/usr/share/icons/oxygen/22x22/actions/system-shutdown.png")  
icon.set_tooltip("Logout from Linux")  
icon.connect('activate',clicked)  
gtk.main()  
{% endhighlight %}  
  
Note that the syntax for gnome-session-save and the icon chosen might differ on your system.  
  
Next, we need to ensure that this applet is run at login, by putting a desktop link into /etc/xdg/autostart/logout-applet.desktop:  
  
{% highlight text %}  
[Desktop Entry]  
Type=Application  
Encoding=UTF-8  
Name=Logout Applet  
Comment=Notification Applet to allow Logout  
Exec=/usr/bin/logout-applet  
Terminal=false  
{% endhighlight %}  
  
You should now see an icon on the start menu when you log in, which when clicked shows you the logout dialog.  
  
Of course, there are still ways around this if a user really wants to shut down a computer. Without more restrictions on the execution of the shutdown/reboot commands, or polkit restrictions, users can still call a shutdown.  
  
However, there is one way which they might try to switch off a machine 'accidentally' - and that is by pressing the power button. There are 2 places that handle ACPI power button events - the ACPI stack, and gnome-power-manager.  
  
To handle ACPI:  
  
Replace /etc/acpi/events/power.conf with:  
  
{% highlight text %}  
event=button/power.*  
action=/bin/true &amp;  
{% endhighlight %}  
  
Then run  
{% highlight text %}  
killall -SIGHUP acpid  
{% endhighlight %}  
  
Of course you can always replace /bin/true with a script of your choice, if you want to for example log out the user, or display a warning message about pushing the power button!  
  
Finally, you need to tweak gnome to ignore power button events, or it will throw up the shutdown dialog.  
  
Back to gconf - set the following key to **nothing**:  
  
{% highlight text %}/apps/gnome-power-manager/buttons/power{% endhighlight %}  
  
You might also want to investigate many of the other ACPI events that live here and deal with them accordingly.  
  

