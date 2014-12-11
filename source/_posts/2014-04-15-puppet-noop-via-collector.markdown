---
layout: post
title: "puppet noop via collector"
date: 2014-04-15 00:00:00 -0800
author: Nathan Valentine
comments: true
categories: [puppet, noop, devops]
---
# Overview
In a [previous article](https://nrvale0.github.io/posts/2014/04/the-basics-of-puppet-noop) I
wrote about Puppet's noop mode which allows you to compare a node's current
config state against the state specified in Puppet code without actually making
any changes to the node. This is a super useful low-risk way of testing
your Puppet code on production nodes and ensuring you don't break anything
while testing. (Of course it is always useful to have 
infrastructure for automated testing but that is another discussion for another
time.) I also wrote about the three commonly understood ways of invoking noop
mode:

1. via the 'noop = true' setting in the '[agent]' section of puppet.conf
1. via 'puppet agent -t --noop' and 'puppet apply --noop &lt;puppet code&gt;'
1. via 'noop => true' specified as a [metaparameter](http://docs.puppetlabs.com/references/latest/metaparameter.html) of a Puppet [resource](http://docs.puppetlabs.com/puppet/latest/reference/lang_resources.html)

In the first two examples above our noop mode is all-or-nothing; noop will 
be enforced for every resource Puppet is managing on our node. That provides
the ultimate in safety but there are scenarios where we'd like to enforce 
some resources and not others. The third option above provides the abilty 
to specify noop per-resource in our Puppet DSL code and while that is useful
we certainly don't want to have to add that code to every resource we are
testing.

In this article we are going to look at two additional ways to specify noop
for some subset of Puppet-managed resources:

1. noop via [Resource Defaults](http://docs.puppetlabs.com/puppet/latest/reference/lang_defaults.html)
1. noop via [Resource Collectors](http://docs.puppetlabs.com/puppet/latest/reference/lang_collectors.html)

# noop via Resource Defaults
For every [resource type](http://docs.puppetlabs.com/references/latest/type.html) available in our
Puppet code, whether that be via the core resource types or those added to our
codebase via a Puppet [module](http://forge.puppetlabs.com), we can use
Resource Default syntax to specify default attribute values. Let's look at
some example code:

```puppet
file { '/tmp/file1':
  ensure => file,
  owner => 'root',
  group => 'root',
  mode => '0644',
  nope => true,
}

file { '/tmp/file2':
  ensure => file,
  owner => 'root',
  group => 'root',
  mode => '0600',
  noop => true,
}
```

You might think the code above would create the files '/tmp/file1' and
'/tmp/file2' and you would be close to correct. Note the 'noop => true' on
both file resources. Puppet is going to check whether these files exist with
the specified attributes for ownership, group, etc and then if there is a 
misalignment between the state of the files on the system and the state 
specified in our code it is going to *tell you what it would like to, but
cannot, change due to the noop metaparameter values*. Also, it would seem there
is probably an opportunity for some code compression; we have specified 
identical values for attributes like 'owner' in both of the file resources. Let's
see an example of Resource Defaults which knocks out both of those issues:

```puppet
File { owner => 'root', group => 'root', mode => '0644', noop => true, }

file { '/tmp/file1':
  ensure => file,
}

file { '/tmp/file2':
  ensure => file,
  mode => '0600',
  noop => false,
}
```

In the code above the "File {..." line sets a series of defaults for all
file resource types in the current
[scope](http://docs.puppetlabs.com/puppet/latest/reference/lang_scope.html).
For example, unless otherwise specified/overridden, all files created in the 
current scope will have a mode value of '0644'. As you can see, the
'/tmp/file2' file resource above actually overrides both 'mode' and 'noop'
values. Assuming I have saved the above code in /tmp/foo.pp, here's the output
of a 'puppet apply' run:

```console
$ sudo puppet apply /tmp/foo.pp
Notice: Compiled catalog for testnode.private in environment production in 0.12 seconds
Notice: /Stage[main]/Main/File[/tmp/file1]/ensure: current_value absent, should be file (noop)
Notice: Class[Main]: Would have triggered 'refresh' from 1 events
Notice: Stage[main]: Would have triggered 'refresh' from 1 events
Notice: Finished catalog run in 0.31 seconds
$ ls -l /tmp/file* 
-rw------- 1 root root 0 Apr 15 07:17 /tmp/file2
```

Hopefully no great surprises there. :) 

Because the Resource Default syntax is available for all resource types, you
might also do the following kinds of things:

```puppet
User { managehome => true, noop => true, }
Package { provider => 'gem', noop => true, }
Service { ensure => 'running', noop => true, }
```

While there is some value in being able to specify noop for lots of resources
in one fell swoop, there are a couple of limitations with the above approach:

* First, the Resource Defaults only apply in the current
scope and child scopes. Since 90% of the Puppet code you write will live
in a module with its own class-based
[namespace](http://docs.puppetlabs.com/puppet/latest/reference/lang_namespaces.html)
and associated scope the impact of your Resource Defaults will be limited. 
That is a good thing and a bad thing. You *could* go into
[site.pp](http://docs.puppetlabs.com/puppet/latest/reference/dirs_manifest.html)
and specify Resource Defaults at global scope but then you are tweaking 
defaults for all resources on the node and that introduces the possibility
for all kinds of odd interactions which you'd probably rather avoid. Just as 
you would be careful doing things in global scope in other languages you should
be careful doing things at global (site.pp) scope in Puppet.

* Second, if you wanted to enable noop mode for all Types using Resource
Defaults you'd have to explicitly do so for every Type in your Puppet codebase.
Until the Puppet DSL supports some form of introspection (and maybe not even
then) it isn't possible to know every Type available in your codebase. Even 
doing so for the core types, of which there are several dozen, would be a
lot of ugly boilerplate code and it still wouldn't give you coverage for custom
Types which come bundled with a [Forge](http://forge.puppetlabs.com) module.
you'll have to include a Resource Default statement for every resource type

So, at best, it probably only makes sense to use Resource Defaults for
enabling noop mode at the class/module level.

Wouldn't it be nice if we could somehow query the list of managed resources
from within the Puppet DSL and set noop where appropriate? Read on!

# noop via Resource Collectors

Up to this point I've written about "resources managed on the node" and been
very careful to not explore that concept at any greater depth. Time for more
Puppet fundamentals...

It is a common misconception that during a [Puppet agent](http://docs.puppetlabs.com/references/latest/man/agent.html) run the
agent actually downloads the Puppet code from the Puppet master. Other 
configuration management systems often work that way but the design of Puppet
is such that instead the Puppet master 'compiles a catalog' which is 
essentially a compressed JSON document which contains all of the details of
the resources to be managed on the requesting node. That
[catalog](http://docs.puppetlabs.com/puppet/latest/reference/subsystem_catalog_compilation.html)
is downloaded by the agent and passed to a layer in the Puppet agent called the 
[Resource Abstraction Layer](http://docs.puppetlabs.com/learning/ral.html)
which is responsible for converting the catalog into the desired node config
state. There are some constructs in the Puppet DSL that allow us, as one of
the last stages of catalog compilation, to manipulate the resources in the 
catalog.  

In this case I'm referring to [Resource Collectors](https://docs.puppetlabs.com/puppet/latest/reference/lang_collectors.html). Let's take a look at some examples of Resource Collectors.

Here's a simple resource collector to force all Package resources on the system
to noop mode:

```puppet
Package <| |> {
  noop => true,
}
```

This means all Package resources in the node's catalog will be noop'ed 
and thus the Puppet agent will *report* packages it would like to change but
will not actually make those changes. A note for those who might be a little 
more advanced in their Puppet usage, know that Resource Collector operations are
[parse order independent](http://docs.puppetlabs.com/learning/ral.html).

There are scenarios where we might want to mark just specific packages for 
noop as opposed to the entire set of packages in the catalog. We can pick out 
a subset of packages using [Resource Collector search expressions](http://docs.puppetlabs.com/puppet/latest/reference/lang_collectors.html).

Here's an example of marking the 'kernel' package noop:

```puppet
Package <| title == 'kernel' |> {
  ensure => latest,
  noop => true,
}
```
so if the 'kernel' package is in our catalog and a newer version is available
in our repositories we'll get a entry in the agent run report indicating as
such but Puppet will not, perhaps thankfully, take it upon itself to install
the new package. 

Here are some additional examples of Resource Collectors using search
expressions:

```puppet
Package <| title == 'openssl' or title == 'openssh-server' |> {
  ensure => latest,
}

File <| mode == '0777' |> {
  mode => '0644',
}

Service <| title == 'selinux' or title == 'rtools' |> {
  ensure => stopped,
  enable => false,
}
```

though some of these are represent questionable practices in actual production.
;) 

Resource Collectors, especially when paired with search expressions, give us
a lot more flexibility to pick out catalog resources for noop mode but they
suffer a similar limitation to Resource Defaults in that if we were to try
to use them to noop all resources in the Catalog we'd have to include 
a statement for each Resource Type in our catalog and we can never know the set
of all available Resource Types in our codebase without some clever 
introspection capabilities not currently present in the DSL.

## An Important Wrinkle with Resource Collectors

There's a very important, and surprising, wrinkle you should be aware of when
you consider a Resource Collector; if you were to do something like this:

```puppet
User <| |> {
  ensure => present, 
}
```

that seems rather innocuous. "For every user in my catalog ensure that user
is present.' At worst, you might override some code somewhere else that
specified a particular user should be 'ensure => absent', right? Well, there's
a corner case involving [Virtual Resources](http://docs.puppetlabs.com/puppet/latest/reference/lang_virtual.html).
If you collect a Resource Type which has Virtual Resources in the catalog you
may very well instantiate those resources. Please read up on Virtual Resources
before using the Collector.

# Let's Review

We've covered quite a bit of ground in the last two posts so let's step back
for a second to get our bearings before we go on to the next article. We've 
talked about:

* the basic concept of the noop mode in Puppet
* basic ways to invoke noop mode acrossed an entire Puppet enforcement run:
    * puppet agent -t --noop
    * puppet apply --noop /tmp/foo.pp
* a handful of ways to invoke noop mode for individual resources (metaparameter 'noop => true')
* invoking noop acrossed a limited set of resources:
    * Resource Defaults
    * Resource Collectors

The end goal of all of this noop cleverness is to allow us to test our Puppet
code on production nodes and catch any issues which might have snuck through
earlier testing with non-production nodes. We want the ability to not only
do coarse-grained/whole-catalog noop testing but also the ability to pick
and choose which parts of our codebase and managed resources are in full 
enforcement or noop mode.

In the next post we'll tie all of this together and start our investigation
of the techniques I call "deploy to noop" which allows us to start picking
out particular parts of the codebase but parts of our codebase for partciular
sets of nodes in our production network.
