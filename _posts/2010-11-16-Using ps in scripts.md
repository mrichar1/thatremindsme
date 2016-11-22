---
layout: post
title: Using ps in scripts
categories: ['computing']
tags: []
---

Not an amazing revelation, but something I keep using, then forgetting how to do, and then having to look it up again, so this is more of a mental 'post-it note' than a proper blog entry...  
  
{% highlight bash %}ps --no-headers -eo user,comm,pcpu,vsize,nice | sort -k 4 -r -n{% endhighlight %}  
  
There's a whole range of fields that can be requested from ps - see the man pages for details.  

