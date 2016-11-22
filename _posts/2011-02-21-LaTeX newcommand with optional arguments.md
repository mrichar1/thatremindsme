---
layout: post
title: LaTeX newcommand with optional arguments
categories: ['computing']
tags: ['latex']
---

I was struggling to find a way to pass 1 or more arguments to a latex newcommand, without running into problems. In this case I was trying to build an index where the section title could be placed into one or more categories.  
  
\newcommand allows you to specify a number of arguments, but this number is mandatory, and if you specify a higher number than you actually supply, then it will start to 'eat' into the next part of the document:  
  
{% highlight tex %}  
\newcommand{\entry}[3]{#1 #2 #3}  
  
%In the document:  
\entry{a}{b}  
This is a test.  
{% endhighlight %}  
  
Results in:  
  
{% highlight text %}  
a b T  
his a test.  
{% endhighlight %}  
  
If you supply less arguments than the number specified, you will receive an error.  
  
My solution was to use one of the internal variables to generate a for loop, iterating over a comma-separated list of categories.  
  
{% highlight tex %}  
\makeatletter  
\newcommand{\entry}[2]{  
\@for \xx:=#2 \do{  
\index{\xx!{#1}}  
}  
}  
\makeatother  
  
%In the document:  
\entry{Title}{cat1,cat2,cat3}  
{% endhighlight %}  
  
This results in and index of:  
  
{% highlight text %}  
cat1,  
Title  
cat2,  
Title  
cat3,  
Title  
{% endhighlight %}  
  
The tricky bit is that {% highlight text %}\@{% endhighlight %} is a command in its own right. We need to use the {% highlight text %}\makeatletter{% endhighlight %} and {% highlight text %}\makeatother{% endhighlight %} to prevent the code being read as {% highlight text %}\@{% endhighlight %} followed by {% highlight text %}for{% endhighlight %}.
