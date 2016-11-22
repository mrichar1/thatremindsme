---
layout: post
title: Mouse Cursor Alignment with KVM/QEMU and VNC
categories: ['computing']
tags: ['kvm', 'qemu', 'vnc']
---

After getting repeatedly annoyed with the fact that the mouse pointer in vncviewer on virtual machines never properly follows the local mouse pointer, and is always offset, I spent far too long trying to find a solution. The answer is actually quite easy:  
  
libvirt guest XML:  
  
{% highlight text %}  
<devices>
<input type='tablet' bus='usb'/>
...
</devices>
{% endhighlight %}  
  
Command-line:  
  
{% highlight text %}  
qemu -usbdevice tablet  
{% endhighlight %}  

