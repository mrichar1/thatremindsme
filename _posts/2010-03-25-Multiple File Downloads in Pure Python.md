---
layout: post
title: Multiple File Downloads in Pure Python
categories: ['computing']
tags: ['python']
---

I was playing around with a way to get a number of files from a webserver in Python automatically (ala wget), and came up with the following script. It parses the HTML returned by a web page, looks for all _a href_ tags for files ending with a given extension, then returns these filenames as a list. The rest script then steps through this list, getting each file in turn into a specified target folder.

{% highlight python %}
#!/usr/bin/python

from HTMLParser import HTMLParser
import httplib
import re

#Config options
target = "/tmp"
#DNS name, not a url
webserver = "www.example.com"
webpath = "/path/to/files"
ext = ".txt"

class AnchorParser(HTMLParser):

    def __init__(self):
        HTMLParser.__init__(self)
        self.items = []

    def handle_starttag(self, tag, attrs):
        if tag =='a':
            for key, value in attrs:
                if key == 'href'and re.search(ext + '$', value):
                    self.items.append(value)

    def get_items(self):
        return self.items


#Setup an HTMLParser object
parser = AnchorParser()

#Get the HTML for the web directory
web = httplib.HTTPConnection(webserver)
web.request('GET', webpath)
data = web.getresponse()

#Pass this HTML to the Parser
parser.feed(data.read())

#Get the returned list of filenames
filelist = parser.get_items()

#Get each file in turn
for item in filelist:
    print "Getting file:" + item
    web.request('GET', webpath + '/' + item)
    resp = web.getresponse()

    #Write out the received data to a file in 'target'
    with open(target + '/' + item, 'w') as f:
    f.write(resp.read())

{% endhighlight %}
