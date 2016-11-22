---
layout: post
title: SSL Support in the Logstash tcp input plugin
categories: ['general']
tags: []
---

I recently added ssl support to the logstash tcp input plugin, and it has now been accepted into the main code. I've seen a few people asking for instructions on how to use it, so I thought I'd write up my notes. Hopefully I will be adding ssl support to the output plugin too in the near future, which should 'close the loop'.  
  
Note - this was mostly a braindump straight from my head, without testing all of the configuration options, commands etc listed below. As a result there may be some errors - if so, please do flag them up and I'll try to get it fixed ASAP!  
  


### Traditional TCP input

  
  
Just to make sure everything is working ok, start with the following:  
  
{% highlight text %}  
input {  
tcp {  
port =&gt; 5555  
type =&gt; "tcp"  
}  
}  
output {  
stdout {  
debug =&gt; true  
}  
}  
{% endhighlight %}  
  
You should now be able to connect to port 5555 on your logstash server with e.g. netcat, telnet etc and type something. You should see this coming out in STDOUT. If not, something else more fundamental is broken!  
  


### SSL Support

  
  
Firstly, you will need to generate a CA certificate, a server certificate for the logstash server, and sign the latter with the former. This is outside the scope of this post, but documentation on doing this is available here:  
[OpenSSL Certificates - Ubuntu Community Docs](https://help.ubuntu.com/community/OpenSSL "OpenSSL" )  
  
You should end up with 4 files - the CA certificate and key file, and the logstash server certificate and key file. For the following, I have named these ca.crt, ca.key, logstash.crt and logstash.key (simples!)  
  
Now, change the tcp section of your logstash configuration to look like the following:  
  
{% highlight text %}  
tcp {  
port =&gt; 5555  
type =&gt; "tcpssl"  
ssl_enable =&gt; true  
ssl_cacert =&gt; "/path/to/ca.crt"  
ssl_certificate =&gt; "/path/to/logstash.crt"  
ssl_key =&gt; "/path/to/logstash.key"  
ssl_passphrase =&gt; "helloworld" # Include this line if you set a passphrase on the logstash key!  
}  
}  
{% endhighlight %}  
_Note:_ If you have a more complex certificate setup, involving a CA chain, or wish to use a certificate path, set ssl_cacert to either the certificate path, or the chain file. By default, logstash will add the default CAPath for your OS to it's certificate store - i.e. /etc/pki/tls/certs for Redhat, and /etc/openssl/certs for Debian.  
  
You now need to copy the CA crt file (not the key!) to any clients that you wish to be able to verify that your logstash server is who it says it is.  
  
Since we can no longer use netcat to test this connection, we will instead use openssl's 's_client' command:  
  
{% highlight text %}  
openssl s_client -connect logstash.example.com:5555 -CAfile /path/to/ca.crt  
{% endhighlight %}  
  
As before, type something, and you should see the log lines appearing on stdout on the logstash server. You will also get some useful diagnostic information from s_client as to the state of the connection, server identity etc.  
  
Optionally you can now run wireshark to sniff the traffic between the server and client to check it really is encrypted.  
  
  


### Client Verification

  
  
The above code will encrypt traffic to the logstash server, and also allow the client to verify the server, should the client be handed a CA cert to verify against. However, we can also use ssl to verify the client's identity, to guarantee that the log messages come from where they claim to!  
  
First, you will need to generate a certificate and key for the client, signed by the CA certificate - the same steps you took to generate logstash.crt/logstash.key, but for the client. We'll call the resulting files client.crt and client.key  
  
Add the following line to the tcp input plugin configuration:  
  
{% highlight text %}  
tcp {  
...existing configuration...  
ssl_verify =&gt; true  
}  
{% endhighlight %}  
  
Your s_client line also changes subtly:  
  
{% highlight text %}  
openssl s_client -connect logstash.example.com:5555 -CAfile /path/to/ca.crt -cert /path/to/client.crt -key /path/to/client.key  
{% endhighlight %}  
  
Now, as well as the client verifying the server's identity and signature, the server will try to verify the client's identity.  
  
If the client verifies successfully, logstash will create a new field in the log entry: _@field.sslsubject_ which contains the full subject of the client certificate. This can be used as a more reliable way to verify the source of the log message.  
  
Of course if you run the above without a client certificate signed by the CA, your connection will just be dropped and nothing recorded.  
  


### Secure logging with rsyslog

  
  
A practical way to use ssl support is to enable secure logging in rsyslog.  
  
Assuming you are already logging plain tcp with the following configuration:  
  
{% highlight text %}  
*.* @@logstash.example.com:5555  
{% endhighlight %}  
  
The configuration is simple.  
  
For situations where you're not verifying the client (ssl_verify =&gt; false):  
  
{% highlight text %}  
$DefaultNetstreamDriverCAFile /path/to/ca.crt  
$DefaultNetstreamDriver gtls # use gtls netstream driver  
$ActionSendStreamDriverMode 1 # require TLS for the connection  
$ActionSendStreamDriverAuthMode anon # Don't verify the server  
*.* @@logstash.example.com:5555  
{% endhighlight %}  
  
To ensure that the server's hostname corresponds to the CN name in the certificate, change:  
  
{% highlight text %}  
$ActionSendStreamDriverAuthMode x509/name  
{% endhighlight %}  
  
For situations where the server is verifying the client, you need to use the above configuration, but also add the following lines:  
  
{% highlight text %}  
$DefaultNetstreamDriverCertFile /path/to/client.crt  
$DefaultNetstreamDriverKeyFile /path/to/client.key  
{% endhighlight %}  
  
This will cause the client to send its certificate for verification by the server.  
  
For more information on rsyslog configuration, see:  
  
[Rsyslog TLS/SSL Overview](http://www.rsyslog.com/doc/rsyslog_tls.html "Rsyslog TLS/SSL Overview" )  
[Rsyslog gtls module](http://www.rsyslog.com/doc/ns_gtls.html "Rsyslog gtls module" )  
[Rsyslog Global Configuration Options](http://www.rsyslog.com/doc/rsyslog_conf_global.html "Rsyslog Global Configuration Options" )  

