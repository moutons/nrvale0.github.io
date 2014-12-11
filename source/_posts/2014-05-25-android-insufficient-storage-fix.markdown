---
layout: post
title: "Android 'Insufficient storage' fix"
date: 2014-05-25 00:00:00 -0800
author: Nathan Valentine
comments: true
categories: [android, adb, cyanogenmod, htc-one]
---
So lately I"ve been having intermittent problems when upgrading Android
apps on my HTC One running CyanogenMod; the dreaded "Insuffucinent storage"
nonsense. I'm not going to pretend I've spent the time to dig into this
issue to find the root case but I will tell you which of the numerous
suggested work-arounds have been successful in my case. For instance,
in the case of the AirBNB app:

(of course the usual #include &lt;disclaimer.h&gt; applies)

```console
$ adb shell
shell@m7:/ $ su -
root@m7:/ # cd /data/app-lib
root@m7:/data/app-lib # find ./ -type f -ipath '*airbnb* -iname '*.so' -exec rm -f {} \;
```

and then make another pass at upgrading the app from Google Play. 

If the update still fails then you can go Big Hammer with the following command: 

```console
$ adb shell
shell@m7:/ $ su -
root@m7:/ # cd /data/app-lib
root@m7:/data/app-lib # find ./ -name '*airbnb*' -exec rm -rf {} \;
```

Note that this often resets the data for the app such that you may have
to login to the app again. 
