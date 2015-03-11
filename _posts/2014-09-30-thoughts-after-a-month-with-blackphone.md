---
layout: post
title:  "Thoughts after a Month with Blackphone"
date:   2014-09-30 15:00:00
author: jduck
categories: news
---

About a month ago, I decided to order a Blackphone.
The product web site makes some tall claims about security, even calling it a "secure smartphone."
This kind of proclamation is rather bold, perhaps even disingenuous, and often leads to intense scrutiny in the security community.
For example, consider the the response to [Oracle calling their product "Unbreakable."](http://www.cnet.com/news/oracle-unbreakable-no-more/)
I'm a bit of a skeptic, so over the last month I've spent some of my free time researching Blackphone as a company and evaluating the security of its flagship smartphone.
I wrote this post to present some of my observations and the opinions formed as a result of my research.

Before diving in, I want to point out that I'm not the first person to take a look at Blackphone.
The device was announced in January and was finally made available in June.
At that time, Ars Technica [reviewed a pre-release version of the device](http://arstechnica.com/security/2014/06/exclusive-a-review-of-the-blackphone-the-android-for-the-paranoid/) and their review was [cross-posted on Bruce Schneier's blog](https://www.schneier.com/blog/archives/2014/06/blackphone.html).
In August, Blackphone had a booth in the DEF CON 22 vendor area and the CSO even did [an interview discussing the device](https://www.youtube.com/watch?v=dbtSNAD4JJY).
At that event, fellow droidsec researcher jcase bought a Blackphone and subsequently [rooted it on the same day](https://twitter.com/TeamAndIRC/status/498257627176923137).
After Black Hat/DEF CON, companies like [Malwarebytes](https://blog.malwarebytes.org/hackedunpacked/2014/08/blackphone-privacy-centric-device/), [Bluebox](https://bluebox.com/blog/technical/blackphone-review-security-and-privacy/), and [viaForensics](https://www.viaprotect.com/blog/blackphone-security-cert-pinning/) have since posted their takes on the device.
These articles and myriad of associated reader comments raise many valid points; many of which echo my own sentiments.

I purchased the Blackphone shortly after returning from DEF CON.
It shipped directly from Hong Kong and arrived in only two days!
Once it arrived, I quickly added it to the [droid](http://www.accuvant.com/labs/research/researching-android-device-security-with-the-help-of-a-droid-army) [army](https://www.blackhat.com/us-14/archives.html#Drake) and starting taking a look.


## Getting Root

My first order of business, as is the case with any new device, was to get root on the device.
It shipped with PrivatOS 1.0.1, which left it vulnerable to the chain of bugs jcase used to root Blackphone at DEF CON.
The steps in reproducing this method are as follows:

1. Enable third-party app installs
2. Host/install an [apk that pops the debugging menu](https://twitter.com/TeamAndIRC/status/498589122458427392) (be sure to set Content-type)
3. Run the app and enable USB debugging
4. Use **adb jdwp** with **jdb** to [debug the "remotewipe" system app](https://twitter.com/TeamAndIRC/status/498589595177467904)
5. Inject some code to spawn **telnetd** as *system*
6. Find a way to get from *system* to *root*

I made fairly short order of reproducing the first five steps of the path to root, leaving me with *system* privileges.
The last part involved getting *root* from the *system* account.
jcase didn't disclose his method for achieving this, so I started auditing to find my own way.
It didn't take long before I found what I was looking for.

The following excerpt from **/init.qvs.rc** illustrates the problem.

```
service boot_script /data/boot_script.sh
        oneshot
        user root
        disabled
[...]
on property:sys.boot_completed=1
[...]
        start boot_script
```

On Android, the *system* user owns the **/data** directory and therefore can easily create the **/data/boot_script.sh** shell script.
On the next boot, **init** will execute the script as *root*.
This allows not only escalating privileges from *system* to *root*, but also allows persisting *root* access across subsequent vulnerable system updates.
I was able to use this issue to keep root access even after installing the PrivatOS 1.0.2 and 1.0.3 updates.


### Reporting the Issue

I reported the issue on August 25th and it was quickly acknowledged.
A fix was released as part of PrivatOS 1.0.4 on September 9th.
The Blackphone staff fixed the issue within 14 days and graciously credited me in their [release notes](https://support.blackphone.ch/customer/portal/articles/1684699-privatos-1-0-4-release-notes).
Such a short turn around is pretty impressive for a security issue in the initial ramdisk of a smartphone.
The fix for the issue is as follows:

{% highlight diff %}
diff -ubr 1.0.3/boot/root/init.ceres.rc 1.0.4/boot/root/init.ceres.rc
--- 1.0.3/boot/root/init.ceres.rc       2014-08-26 19:46:49.029329552 -0500
+++ 1.0.4/boot/root/init.ceres.rc       2014-09-09 14:06:38.526742249 -0500
@@ -591,9 +591,6 @@
     user system
     group system inet net_admin

-# Customers should remove this line
-import init.qvs.rc
-
 # log save to files
 service nvlog_to_file /system/bin/nvlog_to_file.sh
     class main
Only in 1.0.3/boot/root: init.qvs.rc
{% endhighlight %}


### Reflecting on Exploited Issues

Although Blackphone fixed these issues quickly, it's unclear why they shipped in the first place.
The excerpt above included removing a comment that says, "Customers should remove this line".
This comment was likely left by NVIDIA, the System-on-Chip (SoC) manufacturer who provides the Board Services Platform (BSP) for this device.

In steps 4 and 5, I exploited a security issue reported by Sebastián and Marco from viaForensics (also droidsec members).
jcase independently discovered and exploited this same bug at DEF CON 22.
The root cause of the issue was that Blackphone shipped a system app that was debuggable.
It's quite unfortunate actually, because the [Android Compatibility Test Suite (CTS)](https://source.android.com/compatibility/cts-intro.html) would find this type of security issue quickly.

Unfortunately, the staff at Blackphone didn't catch these issues on their own.
The presence of such issues is distressing.
A solid Security Development Lifecycle (SDL) should make the likelihood of such issues small.
Perhaps Blackphone doesn't have an SDL, or perhaps they just missed these issues.
In any case, the fact that such rookie mistakes shipped detracts from Blackphone's security claims.


## Differences from Android/AOSP

After rooting the device, I began investigating exactly what changes Blackphone made to Android/AOSP to make their device more secure and/or private.
People have been making a bit of stink about this [on Twitter](https://twitter.com/McGrewSecurity/status/505366191146536960) and in reader comments on various articles.
Most of Blackphone's responses point to process differences like quicker patching rather than hardening or increased privacy features.
Keep in mind that although some of the differences from other Android devices are apparent, they aren't necessarily differences from AOSP.


### PrivatOS is NOT Open Source

Unfortunately, Blackphone has not made any of their source code for PrivatOS available; despite [making promises to make Blackphone "open source all the way."](http://mashable.com/2014/01/15/blackphone/)
They made the [source to their Linux kernel](https://github.com/sgp-blackphone/Blackphone-BP1-Kernel) available, which they are legally encumbered to do due to GPL.
Much of the code in AOSP is released under a BSD or Apache license, which does not have this legal requirement.
I'm not terribly surprised given the fact that Silent Circle, one the companies behind Blackphone, also [was very slow to keep their open source promise](http://www.quora.com/Will-Silent-Circle-silence-critics-who-demand-they-uphold-their-promise-to-release-their-source-code-If-not-can-they-be-trusted).

The reasons for not releasing code are unclear.
Perhaps the bureaucracy that plagues the mobile operating system ecosystem is rearing its ugly head.
The cause could be internal logistics issues, NVIDIA holding them back, or an attempt to protect company IP.
Whatever the case, the important thing to realize is that keeping the code closed hurts Blackphone.

Opening the source code will increase trust in the Blackphone product and the company behind it.
"Many eyes" arguments aside, auditing open source software is easier than reverse engineering.
For example, in June 2013 [Azimuth Security reviewed](http://blog.azimuthsecurity.com/2013/06/attacking-crypto-phones-weaknesses-in.html) the open source ZRTP library used by Silent Circle's apps.
They identified several vulnerabilities which were subsequently fixed.
Without taking a look at the Blackphone code, we don't know if they introduced additional security issues, which is unfortunately common for Android device manufacturers.
The ideal way to review these changes from AOSP would be to compare the source code against Android 4.4.2.
Without the code, the amount of reverse engineering time and effort required is enough to dissuade most researchers (including me, so far).
Apart from making auditing easier, opening the source code greatly improves transparency.
Researchers and analysts can easily review the code to verify no backdoors are present.


### Not Android Compatible

Although Blackphone is based on Android 4.4.2 from AOSP, it is not [Android compatible](https://source.android.com/faqs.html#compatibility). 
That means Blackphone isn't allowed to use the Android name and cannot ship with access to Google Play.
The former is largely unimportant, but the latter actually has interesting security ramifications -- both good and bad.

Excluding Google Play, or any other app store for that matter, removes a huge attack surface.
In fact, in Blackphone's default configuration, you can't install apps at all.
I think this is fantastic.
**After all, installing an app is effectively equivalent to giving the author of that app a shell account on your most personal machine.**
Nobody would give a complete stranger a shell, now would they?

On the flip side, not having access to Google Play means Blackphone doesn't benefit from the resources that Play provides.
Features like automatic app updates, [remote kill](https://jon.oberheide.org/blog/2010/06/25/remote-kill-and-install-on-google-android/), and other ecosystem wide mitigation potential.
Further, it probably means Play Services and Google Cloud Messaging are not present, which may break apps that depend on those features.
Omitting this feature effectively separates Blackphone from the rest of the Android ecosystem; for better and for worse.

Not going the "compatible" route also means that Blackphone probably doesn't have access to the Open Handset Alliance.
While not much has been stated publicly about the OHA since its inception, it is believed to be the channel through which Google and other Android OEMs share important vulnerability information.
Not having access to privately reported vulnerability advisories and code fixes ahead of time puts Blackphone at a slight disadvantage.
Because disclosure practices are so terrible in the Android ecosystem, Blackphone may miss out on important fixes entirely.


### Considerations of Forking AOSP

As a fork of AOSP, Blackphone's PrivatOS incurs significant maintenance costs but can also realize some amazing benefits.
Standing at over 25 gigabytes of source code, backporting patches can be a nightmare.
This is probably the biggest reasons that OEMs and carriers take so long to release firmware updates.
In some cases, difficulties arise resulting in such updates being scrapped and never released at all.

Just think of the insane amount of code that will change when Android L is finally released.
To remain secure, Blackphone will have to do one of two things.
Option one is to comb through all released changes and backport security relevant fixes.
This applies to not only Android-specific projects, but also to external projects that are included in AOSP like OpenSSL and WebKit/Blink/Chromium.
Failure to do so could leave Blackphone users susceptible to publicly disclosed security issues such as those regularly published on the [Google Chrome Releases](http://googlechromereleases.blogspot.com/) blog.
For example, [the stable channel update on August 26](http://googlechromereleases.blogspot.com/2014/08/stable-channel-update_26.html) fixed over eight security issues, four of which were rated High and one rated Critical.
Blackphone will need to review such changes and keep their fork updated to keep users secure.
This is no small feat.

Now, the awesome part of being a fork is that they can do this **quicker** than Google itself.
Being decoupled from AOSP, they don't have to wait for Google's fix to come down in the next major version release.
This is something that Blackphone is already doing, and doing fairly well.
For example, they were able to fix serious vulnerabilities like [FakeID](https://bluebox.com/technical/android-fake-id-vulnerability/) and [futex](https://hackerone.com/reports/13388)/[Towelroot](https://towelroot.com/) on their own, accelerated time line.
This is certainly a good thing, but as mentioned earlier in this section, the sheer amount of code to track and maintain remains a herculean challenge.


### Observed Changes

While using the Blackphone, the following changes from AOSP were observed:

* The bootloader on the Blackphone is easily unlockable, but immediately re-locks itself after booting up.
This is annoying when developing, but is a great feature for those that might forget to re-lock the bootloader.

* The recovery mode on the device appears changed, but it's not clear how exactly at this point.
First off, it's difficult to get into.
To do so, you have to use "adb reboot recovery" or quickly hit Vol-Up and Vol-Dn after powering the device on.
Once you get in to recovery mode, the buttons don't appear to do anything at all.
This means no sideloading updates and so on.

* Permissions handling code within the Android Framework must have been modified to support the Permissions Privacy feature.
This is presumably the biggest change that Blackphone has made to AOSP.

It's important to note that these changes have not been verified by comparing code to AOSP or otherwise; mostly because the source code is not available.
This is by no means a complete list of changes, or even possible changes.
I am leaving a more in-depth review for a later date or an exercise to the reader.


## Bug Bounty Program

On September 23rd, Blackphone [announced their bug bounty program](https://blog.blackphone.ch/2014/09/23/blackphones-bug-bounty-program/).
Surprisingly, this is **the first Bug Bounty program for any Android-based smartphone**.
Although Google's Patch Rewards Program (PRP) covers AOSP, it is focused on hardening and does not reward for individual vulnerability reports/fixes.
Chrome offers a bug bounty, which covers Chrome for Android, but that's a far cry from paying for bugs in the entire Android OS.

Previously, Blackphone publicly [stated that they were against Bug Bounties](https://gigaom.com/2014/09/23/hackers-will-get-paid-for-finding-blackphone-flaws-after-all/).
In fact, CEO Toby Weir-Jones [told Ars Technica that "bug bounties are contrary to the company's philosophy."](http://arstechnica.com/security/2014/08/blackphone-goes-to-def-con-and-gets-hacked-sort-of/)
Although it's strange that they have changed their mind, I welcome and applaud their new approach.
Working with the security research community at large and compensating researchers for their time is the right move.


## Conclusions

After only a little over a month with a Blackphone, I've gotten a feel for the device, the company behind it, and the software that it runs.
Along the way, I noticed several differences to most other Android-based devices.
I tried to discern whether or not the device lives up to its claims of being focused on security and privacy.
While I made some headway in this, I think there is much left to be done.

When it comes to security, I feel that Blackphone's claims are overstated.
Blackphone made several rookie mistakes when shipping their device, one of which was completely avoidable by simply running the Android CTS.
Further, the Blackphone contains no additional OS or kernel hardening features when compared against other Android devices.
Comparing the device against Samsung's Galaxy S5, Blackphone lags quite a bit behind.

Of course, it's not all bad news.
Their commitment to fast patching, which I've witnessed first hand, deserves great acclaim and is a refreshing improvement over the rest of the Android ecosystem.
With the addition of their bug bounty program, they are poised to truly make a difference by leveraging the community at large.
Blackphone still has security challenges to conquer, but they seem to be headed in the right direction.

I would be remiss if I didn't point out Blackphone's flakiness when it comes to keeping their word.
The company's inability to keep its promises raises doubt and will likely cause many would-be customers to lose interest.
The type of people that value Blackphone's key features need to be able to trust the device and the company behind it.
I implore Blackphone to rectify this problem, uphold their word, and deliver the source code to PrivatOS.
I'm convinced that much good will come of it.

Privacy is where Blackphone really shines.
From permissions modifications to custom communications apps, Blackphone's privacy features truly make snooping on a user's private information more difficult.
Unfortunately, achieving privacy requires achieving solid security.
All bets are off if someone compromises the device, regardless of secure cryptography or otherwise.

All said and done, I can honestly say that I like the device.
I admire and applaud what the company is trying to do and wish them the best.
