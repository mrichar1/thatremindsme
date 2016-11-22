---
layout: post
title: Alarms, Timeouts and Parallel Processes in Bash
categories: ['computing']
tags: ['bash', 'forking', 'parallel']
---

Yes, this probably doesn't count as a good idea... but sometimes you have to use bash to do these kind of things - so I might as well explain how I did it. Most of this work was based on someone else's hints and tips, but I'm afraid I can't find the references any more...  
  
The concept is quite simple. Launch a series of jobs in parallel, in the background, recording the PID of each one. Also run a timer function, which when it times out sends SIGALRM to the parent. the parent has a trap, listening for the signal, and if it receives it, it actively kills all its children. Otherwise, the children exit cleanly, and the parent tidies up.  
  
{% highlight bash %}  
  
#!/bin/bash  
  
#Time to wait for stuck processes before killing them  
export ALARMTIME=30  
  
PARENTPID=$$  
  
exit_timeout() {  
echo "Alarm signal received : killing all children"  
for pid in ${CHILDPIDS[@]}; do  
kill $pid &gt;/dev/null 2&gt;&amp;1  
done  
exit  
}  
  
CHILDCOUNT=0  
  
for server in alice bob charlie; do  
CHILDCOUNT=$CHILDCOUNT+1  
echo "Running command on $server:"  
ssh $server "uname -a" &amp;  
CHILDPIDS[$CHILDCOUNT]=$!  
done  
  
#Prepare to catch SIGALRM, call exit_timeout  
trap exit_timeout SIGALRM  
  
#Sleep in a subprocess, then signal parent with ALRM  
(sleep $ALARMTIME; kill -ALRM $PARENTPID) &amp;  
#Record PID of subprocess  
ALARMPID=$!  
  
#Wait for child processes to complete normally  
wait ${CHILDPIDS[*]}  
  
echo "Alarm never reached, children exited cleanly."  
#Tidy up the Alarm subprocess  
kill $ALARMPID  
{% endhighlight %}
