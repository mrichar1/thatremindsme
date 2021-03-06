---
layout: post
title: Working with saltclass from a git repository
categories: ['computing']
tags: ['saltstack', 'saltclass', 'reclass', 'git']
---

Having recently becmoe frustrated with the lack of hierarchy and no ability to merge in salt pillars, I decided to look into alternative options.

I briely toyed with `pillarstack` (`gitstack`) and `varstack`, but found that these were really specialist layers over the top of pillars - what I needed was something that completely replaced pillars.

I found this initially in [reclass](http://reclass.pantsfullofunix.net/ "Reclass") which offers all of the above, and then subsequently in [saltclass](https://git.mauras.ch/salt/saltclass "saltclass") which is now bundled with saltstack.

`saltclass` is built on top of reclass, but offers better integration with pillar and grain information, so I opted for this as the tool of choice.

Where it is lacking is that there is no built-in support for reading saltclass configuration from a git repo, as you can do with the `git` ext_pillar.  As a result I set about working out how to implement this using a post-commit hook from a (non-salt managed) git repository. (If your git server is a minion of your master, then a nice alternaitve is given at this blog post: [Salt git integration without gitfs](https://clinta.github.io/salt-git-nogitfs/)).

This blog post is written for saltclass, but the technique is applicable to reclass, or any other git-based config that is required for running jobs on the master. I assume youalready understand the basic concepts of reclass/saltclass, and have suitable configuration ready (nodes/classes etc).

# Overview

We run the salt-api, listening for web events at the `/hook` url path. Certain hooks will trigger a reactor update, which will call `git.latest` to update a local copy of the saltclass state.`

# API

You need the `rest_cherrypy` API up and running to make this work. Since authentication using external auth can be tricky for git hooks, we will disable this, and instead implement our own auth inside the reactor state.

The salt-api package must be installed and the service restarted after updating the config.

Self-signed certs can be generated by running:

```
salt-call tls.create_ca_signed_cert master CN=salt-master.example.com
```

`Master config:`

```
rest_cherrypy:
  port: 5417
  ssl_key: /etc/pki/tls/certs/master.example.com.key
  ssl_crt: /etc/pki/tls/certs/master.example.com.crt
  # disable external auth in saltstack
  webhook_disable_auth: True
```

Note that the webhook has auth disabled - so auth must be handled elsewhere - in our case in the Reactor.


# Reactor

Now that the API is listening, we need to react to certain events. In this case, we want to react to a hook called `saltclass`:

`reactor.conf`

```
reactor:
  - 'salt/netapi/hook/saltclass':
    - salt://reactor/saltclass.sls
```

Now we must deliver the reactor state to the path provided above. Note that this can be a `salt://` path, or a filesystem path (e.g. `/srv/salt/reactor/`). In this case we provide state config from another git repository which is provided by `gitfs_remotes` in the master config.

`salt://reactor/saltclass.sls`

```
{% raw %}
{% set postdata = data.get('post', {}) %}

{% if postdata.secretkey == "secretgoeshere" %}

reactor_git_saltclass:
  local.state.single:
    - tgt: 'salt-master.example.com'
    - arg:
      - git.latest
      - https://user@git.example.com/saltclass.git
      - kwarg:
          target: /srv/saltclass

{% endif %}
{% endraw %}
```

# Trying it Out

Now we can watch for events on the master:

```
salt-run state.event pretty=True
```

Then try 'kicking' the api hook path, passing in the secretkey token (note `-k` due to self-signed certs):

```
curl  -k -d secretkey="secretgoeshere" https://salt-master.example.com:5417/hook/saltclass
```

You should see the hook event, and then the subsequent `git.latest` job:

```
salt/netapi/hook/saltclass	{ <...snip...> }

salt/job/20180830100053150164/new	{
    "_stamp": "2018-08-30T09:00:53.152986",
        "arg": [
	        "git.latest",
		"https://user@git.example.com/saltclass.git"
        ]
	<...snip...>
```

Assuming the above all worked, you should now have your saltclass repo available at `/srv/saltclass`.

To finish off, simply add the above curl call (or suitable web post)

# Using saltclass config

Now you can tell the master to use your saltclass config for pillars and states:

`master config`

```
master_tops:
  saltclass:
    path: /srv/saltclass

ext_pillar:
  - saltclass:
    - path: /srv/saltclass
```

Note: If you are trying to configure the above using the [salt-formula](https://github.com/saltstack-formulas/salt-formula) module then please see [Issue 383](https://github.com/saltstack-formulas/salt-formula/issues/383) for a workaround to limitations in the config format.

Finally you can look for saltclass config in your pillars:

```
salt 'host.example.com' pillar.items

host.example.com:
    ----------
    __saltclass__:
        ----------
        classes:
            - a
            - b
        environment:
            base
        nodename:
            hostname.example.com
        states:
            - c
            - d

```
