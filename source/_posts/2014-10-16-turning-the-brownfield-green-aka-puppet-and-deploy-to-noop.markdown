---
layout: post
title: "Turning the Brownfield Green - aka Puppet and 'Deploy to Noop'"
date: 2014-10-16 00:00:00 -0800
comments: true
author: nrvale0
categories: [puppet, noop, devops]
---
## This is not my beautiful house!

I'm going to tell you a story. You are the protagonist in this story and you are in TechOps or perhaps you are a Developer. 

There are things you enjoy about your work: you enjoy learning and mastering new technoloigies, you get satisfaction from knowing that the systems you design are resource efficient and reliable, and lastly you enjoy using your skills to solve problems with your team members and for your employer. Because you enjoy these things you spend lots of time, considerable free time even, keeping up with the [state-of-the-art](http://puppetconf.com) and you are excited about the possibilities surrounding this nebulous term [DevOps](https://en.wikipedia.org/wiki/DevOps) and all of the exotic crash-landed-in-the-deserts-of-New-Mexico technology which has been flowing out of companies like [Spotify](https://www.youtube.com/watch?v=IkEQb9oHPwk), [Google](https://www.youtube.com/watch?v=YrxnVKZeqK8), [Facebook](https://www.youtube.com/watch?v=C_WuUgTqgOc), and [Twitter](https://www.youtube.com/watch?v=QyS-IIkeFp4) in the last couple of years.

There are also some things you don't like about your work: drifting project requirements, chronically late project delivery, the amount of effort expended propping up poorly implemented legacy systems, and the sheer amount of time you spend cutting [firebreaks](https://en.wikipedia.org/wiki/Firebreak) in the infrastructure for problems which could be avoided if only your team had the time to rampup on [better](http://continuousdelivery.com/), [more modern](http://theleanstartup.com/), [processes](http://kanbanblog.com/explained/) and [tooling](http://jenkins-ci.org/).

What are your options? 

Well, you could sell everything, use that money to buy a [Westie](http://www.gdaykombis.co.uk/cms_images_products/2145_1_large.jpg) campervan and The Canon of DevOps and embark on cross-country odyssey, stopping to learn [multi-pitch sport climbing](http://www.muskettmountaineering.co.uk/wp-content/uploads/2013/10/P1030770-1280x854.jpg) in Yosemite while dropping your body-fat to 9%, and ultimately ending up many months later at the doorstep of a San Francisco startup as a lean, mean, and freshly-annointed Squire of DevOps. Or you could quit being a flake and triage the most pressing issues at your current org and strategize how you might go about fixing them. ( Also, we have plenty of people like that in the Bay Area. Please don't do that! ;) )

After some reading and lot of thinking you come to the conclusion the first problem which must be resolved is eliminating the multi-week, error-prone, and unreproducable infrastructure deployments which undergird your organization's applications. One Thursday evening over beers you and your two best friends from TechOps pronounce to those within earshot 

"No longer shall we hand-configure servers! No longer shall we go without service monitoring! Now and forevermore we shall Puppetize!" 

And then you are kicked out of Applebee's for being dorky.

## And now I shall sing to you the song of my people.

Getting kicked out of Applebee's turns out to be a boon (in so many ways). In your somewhat buzzed bravado your [Confederation of Rogue TechOps](http://users.telenet.be/mydotcom/graph/nmap_matrix5.png) immediately downloads [Puppet Enterprise](http://puppetlabs.com/puppet/puppet-enterprise) and each member burns through the courses in the [Puppet Labs Workshop](http://puppetlabs.com/learn). You learn how to deploy and configure raw new nodes with [Razor](https://www.youtube.com/watch?v=v7jgwPf85BY), how to leverage pre-written modules posted to the [Puppet Forge](http://forge.puppetlabs.com), how to structure your code and classify your nodes with the [Roles and Profiles](http://garylarizza.com/blog/2014/02/17/puppet-workflow-part-2/) pattern, and, perhaps most importantly, how to test and promote Puppet code using [Puppet environments and r10k](http://garylarizza.com/blog/2014/02/18/puppet-workflow-part-3/). On the last day you learn to coordinate multi-node deployments using a clever combination of [MCollective](https://www.youtube.com/watch?v=vOKT35VE_hA) and [dalen/puppetdbquery](http://forge.puppetlabs.com/dalen/puppetdbquery) and it is good. Your [Google Compute Engine](https://cloud.google.com/products/compute-engine/) cloud is a crystal palace and when the wind blows through the [spires](http://www.dvice.com/sites/dvice/files/styles/blog_post_media/public/images/Oobject-15-skyscrapers-on-hold-Dubai-Towers.jpg?itok=bjMpiIs_) it resonates in the frequency of DevOps. And it is purple.

You take what you have learned back to the office and the next application infrastructure delivery proceeds from greenfield to production-grade with only minor bumps and ahead of schedule and in another fit of bravado you announce to the CTO, "We shall now Puppetize All The Things! I shall now have root on the AIX nodes!" at which point you are bludgeoned from behind by the head of the Change Control Board. When you awake you are in your cube and duct-taped to your chair. Your keyboad is out of reach. The world is spinning. Just as you lapse back into unconsciousness you hear the head of the CAB say to your CTO, 

"And how, exactly, are we supposed to know what Puppet is changing on these systems! Does it integrate with our deployment runbook wiki? No!" 

Your last thought before the world goes dark, "I may have overreached. If only Puppet supported a way to [view](http://docs.puppetlabs.com/pe/latest/console_event-inspector.html) pending system changes without actually enforcing them([1](http://docs.puppetlabs.com/references/latest/metaparameter.html#noop),[2](https://nrvale0.github.io/posts/2014/04/the-basics-of-puppet-noop/),[3](https://nrvale0.github.io/posts/2014/04/puppet-noop-via-resource-defaults-resource-collectors/))! And if only I had a way to toggle that functionality from code!"

We might even call it "Deploy to Noop".

## Deploy to Noop

That thing you have envisioned...it's really really easy to implement! Let's start simple and work our way up to something which could be deployed in production.

If you haven't already you should first read up on Puppet's 'noop mode':

* [Puppet Metaparameter Reference - noop](http://docs.puppetlabs.com/references/latest/metaparameter.html#noop)
* [The Basics of Puppet "noop mode"](https://nrvale0.github.io/posts/2014/04/the-basics-of-puppet-noop/)
* [Puppet noop via Resource Defaults and Resource Collectors](https://nrvale0.github.io/posts/2014/04/puppet-noop-via-resource-defaults-resource-collectors/)

Now that you understand Puppet's noop mode, I want to point you to a little-known but super-useful [Puppet Forge](http://forge.puppetlabs.com) module: [trlinkin/noop](http://forge.puppetlabs.com/trlinkin/noop). The thrust of it is this:

>
> Calling the function 'noop()' bundled with trlinkin/noop puts all resources in the current scope and all child scopes into noop mode.
>

If your site.pp looks like so:

/etc/puppetlabs/puppet/manifests/site.pp:

```puppet
noop()
node default {
  notify {'Hello, World! I think you are going to see a noop here --->':}
}
```

you should then see the notify resource 'enforced', not really enforced, in noop:

```console
$ sudo puppet module install trlinkin/noop
Notice: Preparing to install into /etc/puppetlabs/puppet/modules ...
Notice: Downloading from https://forge.puppetlabs.com ...
Notice: Installing -- do not interrupt ...
/etc/puppetlabs/puppet/modules
└── trlinkin-noop (v0.0.2)

$ sudo puppet apply /etc/puppetlabs/puppet/manifests/site.pp
Notice: Compiled catalog for ensenada.nathanvalentine.private in environment production in 0.07 seconds
Notice: /Stage[main]/Main/Node[default]/Notify[Hello, World! I think you are going to see a noop here --->]/message: current_value absent, should be Hello, World! I think you are going to see a noop here ---> (noop)
Notice: Node[default]: Would have triggered 'refresh' from 1 events
Notice: Class[Main]: Would have triggered 'refresh' from 1 events
Notice: Stage[main]: Would have triggered 'refresh' from 1 events
Notice: Finished catalog run in 0.49 seconds
```

So that's pretty interesting! But the noop() call in our site.pp currently sets noop at global scope and therefore turns off enforcement for all resources in all node statements. We have to step back and think about the problem we are trying to solve.

> Because we are developing a Puppet codebase and we have a brownfield environment, we would really like to default all nodes to noop and then selectively enable enforcement on a sample set of a particular server role. For instance, Noop All The Things except for web servers which are not in production. We iterate over that process and eventually every node in every role in our environment has transitioned from noop-Puppet to Puppet runs enforcing our well-smoketested-on-production-nodes codebase!

What if there were a tool that allowed us to set top-scope variable values in Puppet code based on Puppet Facts? Oh!! Hiera!! Let's modify our site.pp like so:

/etc/puppetlabs/puppet/manifests/site.pp:

```puppet
# Force noop mode unless specified otherwise in hiera
$force_noop = hiera('force_noop')
unless false == $force_noop {
    notify { "Puppet noop safety latch is enabled in site.pp": }
    noop()
}

# Allow hiera to assign classes
hiera_include('classes')

node default {
  notify { "The value of 'force_noop is ${force_noop}": }
}
```

and then we'll set up Hiera to reference YAML files and inject a layer in our Hiera router for 'certname':

/etc/puppetlabs/puppet/hiera.yaml:
```YAML
---
:backends:
  - yaml

:yaml:
  :datadir: "/etc/puppetlabs/puppet/data"

:hierarchy:
  - "clientcert/%{::clientcert}"
  - common
```

/etc/puppetlabs/puppet/data/common.yaml:
```YAML
---
force_noop: true
```

and the clientcert YAML for the nodes in our environment:
```console
$ cd /etc/puppetlabs/puppet/data/clientcert
$ tree clientcert 
clientcert
├── master.yaml
└── ubuntu1204-0.yaml
$ cat clientcert/master.yaml clientcert/ubuntu1204-0.yaml 
---
force_noop: false
---
force_noop: true
```

To break it down, the YAML above states:

1. our Puppet master, named 'master', will run in enforcement mode
1. our agent node, named 'ubuntu1204-0.yaml', will run in noop mode

Now, if you understand Hiera, you should see our ability to sift-and-sort nodes
for enforcement or noop is only limited by the routing logic in our Hiera router. 
It's worth noting that the 'force_noop' top-scope variable can also be set from:

1. Hiera backends other than YAML. Perhaps something like hiera-http?
2. an External Node Classifier or the Puppet Enterprise 3.4 Node Classifier Puppet App
3. by editing Puppet Enterprise Groups and Nodes in the PE Console.

Some of you are probably saying 'Nifty!' and others of you are probably saying, "What the hell just happened?!?!". I have good news...I've put together a self-contained Vagrant environment which you can use to play with Deploy to Noop until it clicks. You can find the environment here:

[http://github.com/nrvale0/deploy-to-noop-part1](http://github.com/nrvale0/deploy-to-noop-part-1)

## "This is flipping cool and I'm going to run off and put it into Production immediately!"

Awesome. Glad you like it. I've used this technique at a number of customer sites and I think it is super-useful. There are a few caveats:

When you are making changes to your Hiera data to turn off the noop safety latch on a node or set of nodes, you'd best understand how to either manually query and verify the results from Hiera, or even better, know how to wire up your CI system / deployment orchestration system to test that you aren't accidentally turning off noop in places you don't expect. Here's an example of how to query Hiera from the master command-line assuming the Hiera data and router config shown above:

```console
$ hiera -c /etc/puppetlabs/puppet/hiera.yaml force_noop ::environment=production ::certname=ubuntu1204-0
false
```

Also, noop agent runs have the potential to generate rather large Report Logs every time the agent runs. "These are the things I would like to change but I cannot because I am in noop mode!" If your Puppet master is PuppetDB-integrated (default on Puppet Enterprise) this can result in your PuppetDB consuming quite a bit of storage. PuppetDB data has a TTL so as you disable the noop safety for more and more of your nodes the amount of PuppetDB storage required (assuming a constant number nodes and a steady agent run interval) should decrease. I've had clients who had thousands of nodes in noop mode for months at a time and they have PuppetDB databases in the 100's of GBs range. Be prepared for this when you are laying out the storge for your Puppet master/PuppetDB server. 

I'd also encourage you to move your nodes into enforcement as rapidly as possibly. What's the point of writing Puppet code if you never allow Puppet to converge the configuration of the nodes?!? This is meant as a temporary hack to allow you to get Puppetizing quickly before you have....dum dum dum...developed a full CI-integrated service delivery/deployment pipline. But that's a topic for another blog! ;)

## Conclusion

Feel free to email [me](mailto:nrvale0@gmail.com) if you have questions. Pull Requests and bug reports are welcome and encouraged!

Happy de-brownfielding!
