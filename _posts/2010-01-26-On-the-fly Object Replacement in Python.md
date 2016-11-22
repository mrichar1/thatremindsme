---
layout: post
title: On-the-fly Object Replacement in Python
categories: ['computing']
tags: ['it', 'python']
---

Today I had an interesting problem with a python module I'm writing, and was provided with a nifty solution by a colleague which I thought I'd share.

The module relies on [ConfigParser](http://docs.python.org/library/configparser.html), which reads a configuration file, and returns an object which has methods such as get(), getboolean() etc. My module has a default configuration file path, and a method _parse_config()_ which instantiates a (global) ConfigParser object as _config_. I wanted to allow the author importing my module to override the default config file, but not to have to explicitly opt whether or not to do so. The solution is as follows:

{% highlight python %}
class LazyConfig(object):

    def __getattr__(self, s):
        parse_config()
        return getattr(config, s)

    config = LazyConfig()
{% endhighlight %}

When the module is imported, the LazyConfig class is instantiated as _config_. If the importer of the module does nothing, the first time _config.get()_ is called, the ___getattr___ method of the class will run, and in turn call _parse_config()_, which will read the default file, create a _config()_ object (overwriting the class), then call that object (via _getattr_). If the importer wishes to use a different configuration file, he simply has to call _parse_config("/foo/bar/baz.cfg")_. Job done!
