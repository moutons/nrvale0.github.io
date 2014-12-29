---
layout: post
title: .gitignore to allow 'empty' directory
date: 2014-10-31 18:00:00 -0800
author: nrvale0
comments: true
categories: [git]
---
I've had a couple of situations where I wanted to be able to commit
an empty directory to a git repo. Git doesn't allow you to add+commit
empty directories so the usual way around this is to include a 
.gitinclude file in the 'empty' directory -- thus no longer empty.

```console
$ mkdir /tmp/foo && cd /tmp/foo
$ git init
Initialized empty Git repository in /tmp/foo/.git/
$ mkdir bar
$ git add -A -n
$ touch bar/.gitinclude
$ git add -A -n
add 'bar/.gitinclude'
```

Okay, that seems better.

But in my case I would like 'bar' but not any subdirectories. Let's
see how that shakes out:

```console
$ mkdir bar/baz
$ git add -A -n
add 'bar/.gitinclude'
```

That looks about right -- BUT IT ISN'T! Git isn't including 'baz' because
baz is a directory with no content.

```console
$ touch bar/baz/somefile
$ git add -A -n
add 'bar/.gitinclude'
add 'bar/baz/somefile'
```

Well, crap. Hows about I just add a .gitignore into 'bar':

```console
$ echo '*' > bar/.gitignore
$ git add -A -n
```

Uh, now we don't get anything on an 'add'?!?

Here's the right way to get an 'empty' 'bar' with no subdirectories
whether those subdirectories have content or not:

```console
$ rm bar/.gitinclude
$ (cat <<HERE
*/
!.gitignore
HERE
> ) > bar/.gitignore
$ git add -A -n
$ add 'bar/.gitignore'
$ tree
.
└── bar
    └── baz
        └── somefile

2 directories, 1 file
```

That took way to long to figure out! :\ Here's a repo if you don't believe it worked:

[nrvale0/gitinclude_tricks](https://github.com/nrvale0/gitinclude_tricks)
