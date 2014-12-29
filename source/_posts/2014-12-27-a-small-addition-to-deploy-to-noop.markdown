---
layout: post
title: "A Small Addition to Deploy to Noop"
date: 2014-12-27 20:06:41 -0800
author: nrvale0
comments: true
categories: [ 'puppet', 'noop']
---

Just a quick update to my [previous post about the 'Deploy to Noop' technique for Puppet](/blog/2014/10/16/turning-the-brownfield-green-aka-puppet-and-deploy-to-noop).

In the [original version of the code in the our demo environment](https://github.com/nrvale0/deploy-to-noop-part-1/blob/a001e9a6c0994048141a3bc5e0349e090796e7d8/puppet/environments/production/manifests/site.pp) we had something like this in our manifests/site.pp:

```puppet
filebucket { 'main':
  server => $::servername,
  path   => false,
}

File { backup => 'main' }

$force_noop = hiera('force_noop')
unless false == $force_noop {
  notify { "Puppet noop safety latch is enabled in site.pp!": }
  noop()
}

hiera_include('classes')
```

As a reminder, the key bit is the callout to Hiera to resolve the value of the 'force_noop' variable. If it resolves to anything other than the boolean value of False we will enable the noop safety latch via the if statement. As you probably recall, the noop() function will force all resources in the current scope and all children scopes into noop or tell-me-what-you-would-do-but-don't-actually-do-it mode. Resolving the state of the noop safety latch from Hiera allows us to selectively disable/enable the noop safety depending on whatever layers we have defined in our Hiera router: ex: datacenter, cloud, osfamily, etc.

Some folks asked how we could not only allow twiddling of noop safety from the Puppet Enterprise Console but do so in a way such that the Console was authoritative even over what was embedded in our Hiera data. A common reason for asking for this functionality was to allow a non-Puppet-programmer to very easily halt convergence runs across an environment when Puppet code had been promoted and later determined to have some problem. Yes, you could just promote fixed code up through your automated deploymnet pipeline (Oh, you don't have one of those? ;)) but this is very quick and very easy. So here's the entire manifests/site.pp with the code to allow such a thing:

```puppet
filebucket { 'main':
  server => $::servername,
  path   => false,
}

File { backup => 'main' }

$force_noop_real = str2bool( pick( $::force_noop, hiera('force_noop', 'true') ) )
if $force_noop_real {
  notify { "Puppet noop safety latch is enabled in site.pp!": }
  noop()
}

hiera_include('classes')
```

Line 8 above has most of the interesting action. It's kind of a jumble but the end result is pretty straight-forward. Let's start at the inside and work our way out.

1. First, check Hiera for the value of 'force_noop'. If no value for 'force_noop' is resolved from Hiera data we will default the value to the 2nd parameter of the function call: in this case Boolean 'true'.
1. Next, use the puppetlabs/stdlib function pick() to check the value of the top-scope variable 'force_noop'. If the value has already been set then pick() will return that value. Otherwise it will return the value we generated in step #1.
1. Lastly, whatever the value of the previous two expressions, we cast the results to a boolean value. That is simple because it allows us to replace the 'unless' conditional in Line 9 with a more vanilla, and probably more easiliy understood, if statement.

At the end of the day, the only substantive change we've made to allow the noop safety latch to be twiddled from the Puppet Enterprise Console is the pick() function call. The rest is just housekeeping.

Happy Puppeting!

