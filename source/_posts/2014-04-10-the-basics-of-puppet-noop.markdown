---
layout: post
title: "the basics of puppet noop"
date: 2014-04-10 00:00:00 -0800
comments: true
author: Nathan Valentine
categories: [puppet, noop, devops]
---
# Overview
There are a few solutions out there for configuration management and server
orchestration these days but over the next few articles I'm going to talk about
an oft-overlooked feature unique to 
[Puppet](http://puppetlabs.com) called "noop mode". If you are old-hand with 
Puppet please be patient. I'm going to start with the basics of noop mode and 
build to some advanced deployment techniques I collectively refer to as 
"deploy to noop" which greatly reduce guesswork and hand-wringing when doing 
brownfield Puppet and Puppet Enterprise deployments in large, multi-platform,
complicated environments.

# A Quick Intro to Puppet
For those who might not be familar with Puppet, it is a programming language
and toolchain for managing the configuration of the nodes in your computing
infrastructure. Traditionally this has meant UNIX-like operating systems but
as of the last couple of years that has expanded to include MS Windows systems
and even network and storage devices. Instead of logging into these nodes
and hand-configuring nodes we instead choose to manage nodes via the Puppet
Domain Specific Language(DSL) and a server-client ( master-agent in 
Puppet parlance ) architecture which centralizes and automates the enforcement
of our nodes' desired state as specified in our Puppet code. The process of
taking a node from current config state to our desired state is called
"convergence". In the example below I show some Puppet code which lives in
my Puppet master's cached copy of my Puppet codebase and which I've chosen to
have enforced on my Ubuntu workstation:

```puppet
user { 'testuser':
  ensure => present,
}

package { 'vim-haproxy':
  ensure => installed,
}
```

The above code will converge the state of my node such that at the end of 
a Puppet agent run I should have a user called 'testuser' and the package
'vim-haproxy' will be installed. Let's see the output of a manual verbose
[Puppet agent](http://docs.puppetlabs.com/references/latest/man/agent.html) run:

```console
$ sudo puppet agent -t
Notice: Compiled catalog for testnode.private in environment production in 0.75 seconds
Notice: /Stage[main]/Main/Package[vim-haproxy]/ensure: ensure changed 'purged' to 'present'
Notice: /Stage[main]/Main/User[testuser]/ensure: created
Notice: Finished catalog run in 7.10 seconds
```

We can manually check our new state against our desired state with the usual
UNIX command-line tools: 

```console
$ id testuser
uid=1004(testuser) gid=1007(testuser) groups=1007(testuser)
$ dpkg -l vim-haproxy
Desired=Unknown/Install/Remove/Purge/Hold
| Status=Not/Inst/Conf-files/Unpacked/halF-conf/Half-inst/trig-aWait/Trig-pend
|/ Err?=(none)/Reinst-required (Status,Err: uppercase=bad)
||/ Name           Version      Architecture Description
+++-==============-============-============-=================================
ii  vim-haproxy        2:7.4.000-1u amd64        Vi IMproved - enhanced vi editor 
$ id testuser2
id: testuser2: no such user
```

or with Puppet toolchain itself with the
'puppet resource' command where 'puppet resource' takes a type and the 
title of the resource we'd like to verify:

```console
$ puppet resource user testuser
user { 'testuser':
  ensure => 'present',
  gid    => '1007',
  home   => '/home/testuser',
  bash  => '/bin/sh',
  uid    => '1004',
}
$ puppet resource package vim-haproxy
package { 'vim-haproxy':
  ensure => '1.4.24-1',
}
$ puppet resource user testuser2
user { 'testuser2':
  ensure => 'absent',
}
```

Let's consider the output of the 'puppet resource' verification commands above.
If you are familar with the Puppet DSL you'll notice that the 'puppet resource'
command spits out valid Puppet DSL which represents the current config state
of our node. I threw in an extra query for the user 'testuser2' to demonstrate
the output of the 'puppet resource' command when we query for a resource which
is not present on the node. 'testuser2' is not on our node so we'd have to
'ensure => absent' in Puppet code to get the current config state.

Some readers might be saying, "I'm already familar with the 'id' and 'dpkg'
commands on Ubuntu so why bother with the 'puppet resource' queries at all?"
Well, consider that once you become proficient with Puppet you might be writing
Puppet code to manage users and packages on many different types of systems
and 'dpkg' and 'id' may not be present on all of those platforms.
'puppet resource' should work everywhere you have Puppet installed!

We've had a quick fly-by of how Puppet works. There's plenty more to learn
at [http://docs.puppetlabs.com](http://docs.puppetlabs.com). Let's get back 
to our discussion of noop mode.

# What is noop mode?
In the examples I provided above we converged a node from its present state to 
the desired state as captured in our Puppet codebase. That workflow is fine
if we have spun up devtest nodes for testing our Puppet code but it's not always
possible to have a devtest node which perfectly mirrors the config state of a
node in our production environment. Puppet's noop mode allows us to see the 
changes Puppet would *like* to make while preventing Puppet from actually
performing the convergence. Once we start digging into Puppet we are going to
see there are lots of different places we can enable noop mode and where we
choose to enable noop mode has an impact on which and how many changes we
prevent.

In this article we are going to cover only the three most widely known ways
of enabling noop mode:

1. in puppet.conf
1. as a command-line parameter to the 'puppet agent' and 'puppet apply'
commands.
1. as a resource metaparameter

## Enabling noop mode in puppet.conf
If we want absolute assurance that Puppet will only ever report the changes
it would like to make without ever actually making those changes we can do so
via a setting in the node's puppet.conf. 

FLOSS Puppet and Puppet Enterprise have the puppet.conf file
in slightly different locations. The easiest way to find your puppet.conf file
is with the following command:

```console
$ sudo puppet agent --configprint config
/etc/puppet/puppet.conf
```

If you wanted to force noop mode for all Puppet agent runs you simply set
'noop = true' in the '[agent]' section of your puppet.conf as shown below:

```console
# puppet.conf
...
[agent]
  noop = true
```

Let's see how that impacts a Puppet agent run after we unwind the changes we
made in our previous testing:

```console
$ sudo puppet resource user testuser ensure=absent
Notice: /User[testuser]/ensure: removed
user { 'testuser':
  ensure => 'absent',
}
$ sudo puppet resource package vim-haproxy ensure=absent
Notice: /Package[vim-haproxy]/ensure: removed
package { 'vim-haproxy':
  ensure => 'purged',
}
$ sudo puppet agent -t
Notice: Compiled catalog for testnode.private in environment production in 0.69 seconds
Notice: /Stage[main]/Main/Package[vim-haproxy]/ensure: current_value purged, should be present (noop)
Notice: /Stage[main]/Main/User[testuser]/ensure: current_value absent, should be present (noop)
Notice: Class[Main]: Would have triggered 'refresh' from 2 events
Notice: Stage[main]: Would have triggered 'refresh' from 1 events
Notice: Finished catalog run in 0.53 seconds
```

We can see by the '(noop)' at the end of the lines above that Puppet wanted
to converge the user and the package but did not due to our noop setting. 
If you run 'puppet resource' for the user and the package now the results of the
queries would verify that neither testuser nor the vim-haproxy package are
installed.

Congratulations, you have absolute assurance Puppet is never going to converge
your node but, um, that's kind of the point of a configuration management
system so, in the end, perhaps that is not very useful. Go ahead and remove
the 'noop = true' from your puppet.conf and we'll look at a couple more ways
you can enable noop mode.

## Enabling noop mode during manual 'puppet agent' or 'puppet apply' runs

I'm not going to get into the difference between the ['puppet agent'](http://docs.puppetlabs.com/references/latest/man/agent.html)
and ['puppet apply'](http://docs.puppetlabs.com/references/latest/man/apply.html) 
commands in this article but you can click through
the provided links to dig into the details. For now it is sufficient to say
they are two ways you can converge your system to a state specified in Puppet
code. For either command if you instead preferred to see what Puppet would
like to change without actually making any changes you could do so like so:

```console
$ puppet agent -t --noop
```

```console
$ puppet apply --noop /path/to/puppetcode.pp
```

and you will see output appended with '(noop)' and could verify the config 
state of your node had not changed with 'puppet resource' queries just as in 
the case where you had specified noop mode in your puppet.conf.

## Enable noop mode as a Resource Metaparameter
To this point in the article I've held off on some Puppet DSL jargon which is
now necessary. First, please take a look at the following Puppet DSL code:

```puppet
user { 'testuser':    
  ensure => present,    
  noop => true,    
}    
    
package { 'vim-haproxy':    
  ensure => present,    
}
```

In the code shown above we have two 'resources':

1. a user resource with title 'testuser' with attribute ensure value present.
1. a package resource with title 'vim-haproxy' with attribute ensure value
present.

There's something a little special about the user resource though. There's 
an additional attribute 'noop' with a value of 'true'. The 'noop' attribute
is what's called a
[metaparameter](http://docs.puppetlabs.com/references/latest/metaparameter.html).
Let's just say that metaparameters are present for all
[Types](http://docs.puppetlabs.com/references/latest/type.html)
in the Puppet DSL and leave it at that for now. The important take-away here
is that for any resource type you are attempting to manage in the DSL you
can set 'noop => true' and even if you have not enabled noop mode from 
puppet.conf nor from the command-line that particular resource will not be 
converged. Also important to note is that barring a change in the puppet.conf
or a noop passed at the command-line the package the above code *will* install
the vim-haproxy package. The noop metaparameter is a way to enforce noop mode
with a very fine resolution. In this case for just a single resource.

# Where have we been and where are we going?

In this article I provided some very basic examples of Puppet code and we 
talked a little bit about config state convergence. We also stepped through
various ways you can enable Puppet's "noop mode" to get a convergence dry-run.
Noop mode is useful in lots of situations but our particular use case 
is for testing config state changes on production systems while ensuring
we aren't going to make any changes to that production system and 
therefore potentially break production. We saw three places where we 
could easily enable noop mode even as a Puppet beginner.

Upcoming articles will require more advanced understanding of the Puppet DSL
and the Puppet stack. I will introduce more sophisticated ways of utilizing 
noop mode for testing and verification. 

Please stay tuned and thanks for reading.
