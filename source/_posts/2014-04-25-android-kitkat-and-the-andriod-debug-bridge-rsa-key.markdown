---
layout: post
title: "Android KitKat and the Andriod Debug Bridge RSA Key"
date: 2014-04-25 00:00:00 -0800
comments: true
author: nrvale0
categories: [andriod, rsa, adb, cyanogenmod, htc-one]
---
I recently ran into the dreaded ["Insufficient storage available"](http://www.itworld.com/mobile-wireless/403955/insufficient-storage-available-android-and-how-fix-it-aka-unix-y-man-behind-c) problem when trying to reinstall the Google Drive app on my HTC One running [Cyanogenmod's](http://www.cyanogenmod.com) spin on KitKat. In the usual [shaving-the-yak](http://www.urbandictionary.com/define.php?term=yak%20shaving) fashion the proposed fix for this problem (fixes in previously-linked article did not resolve :() required an upgrade of ADB and the Android SDK. As it turns out KitKit now requires a click-to-accept on your phone when you've connected to USB debugging with an unrecognized RSA key. This is news to me in a number of ways as I didn't even know there **was** an RSA key built-out as part of ADB debugging but, regardless, my phone was not prompting in any way about any keys. It took about 20 minutes to figure out the following series of steps to get ADB working:

1. Settings/Developer Options/Revoke USB debugging authorization
1. Settings/Storage/USB computer connection(top-right of the screen for me)/Media device(MTP) = checked
1. Optionally, Settings/Android debugging = unchecked and then re-checked

Now, where were we....oh, yes, Google Drive...

