---
layout: post
title:  "On the WebView addJavascriptInterface Saga"
date:   2014-02-26 18:41:00
author: jduck
categories: news
tags: [ sdk, addJavascriptInterface, addjsif, WebView, browser, aosp ]
---


In the last month, several new facts came to light in the saga of security issues with using *addJavascriptInterface* in Android WebView objects. While the dangers associated with this method are well documented, the full extent and reach of associated issues was not known until recently. These issues continue to be a plague on the security of the Android ecosystem to this day.

We decided to write this post for many different reasons. First, we want to set the facts straight. The interactions involving this method and the security issues that result from it are complex. Understandably, some published articles contain technically inaccuracies. Second, we have been doing more testing and think that some of the test results are interesting. Finally, we want to make some recommendations to various parties within the ecosystem and document open questions in and future directions for our research.

It's important to note that this post does not discuss a number of other important Android security issues. Many egregious privacy leaks stem from Android applications and advertising SDKs. Stock and third party browsers are often vulnerable to publicly disclosed vulnerabilities. There are other security issues that can stem from using *addJavascriptInterface* too. These topics are not covered in this post, but deserve attention too.

*NOTE: This post ballooned significantly. As such, we will be working to publish more details in the future.*


## Background

In this section, you will learn about the different technologies and paradigms that come together to create the issues that are the topic of this post. This includes introductions to software handling on Android, the vulnerable API, and important properties of these vulnerabilities.

### Software on Android

Software on Android devices consists of firmware and applications. Each type of software are versioned, distributed, and updated using different mechanisms.

The firmware for a particular device is built by the party responsible for maintaining the device. The firmware is put onto the device by that party and later updates (via OTA usually) must go through that party as well. In addition to the *Android version*, OEMs and carriers have their own versioning schemes. This part of the Android software update process is discussed in depth elsewhere, so we won't elaborate further.

Applications are either distributed as part of the firmware or via Google Play. In both cases, the applications can be updated directly by the vendor via Google Play. Applications are versioned by their respective vendors.

In addition to the *Android version*, Google uses a versioning mechanism to indicate the precise availability and behavior of objects and methods. These objects and methods comprise the developer API, and hence this version is called the *API level*. As the API levels have increased, many changes have been made. The developer documentation contains lots of notes about changes throughout the evolution of Android. 

Application developers **choose** which API level they use for their application at compile time. When a new API level is released, developers don't have to do anything unless they want to take advantage of functionality from the new API. That is, app developers are free to select an older API level when they build their application. Google has tried to dissuade this practice in newer versions of the SDK by printing warnings at compile time. Also, Google created Google Play Services to ease some of the pain caused by API evolution. However, no technical barrier prevents using older API levels. In fact, some developers do this intentionally as a way to maximize device compatibility. Once an application is built against a particular API level, it cannot be changed without recompiling and redeploying.


### WebViews and addJavascriptInterface

In Android, many browsers web-based mobile applications rely on the [WebView component](http://developer.android.com/reference/android/webkit/WebView.html). The documentation for this component is pretty good, so check it out if you want to fully understand it's purpose and usage. Suffice to say that it's a part of the Android Framework that provides a working, embeddable Web browser. On versions prior to Android KitKat (4.4) it's based on the WebKit engine. With KitKat and later, it's based on Chromium. It doesn't include the UI portion which is also referred to as "the chrome" (not to be confused with the Chrome browser).

The *addJavascriptInterface* method is one of the ways that developers that embed a WebView into their application (including people that build browsers) to expose Java functionality to Javascript. It has existed since the first release of Android (API level 1). It provides a method to achieve synchronous (meaning Javascript waits for a response) communication with the hosting application. There's plenty of resources showing how to use this functionality out there, so we won't go into further detail here. For a more high-level explanation, see the *Our findings make uncomfortable reading* section of [Dave Hartley's article](https://www.mwrinfosecurity.com/articles/ad-network-research/) on the MWR blog.

### Security Issues

Multiple security issues exist involving the *addJavascriptInterface* method of Android WebView objects. Several issues involve applications that expose an object to untrusted Javascript within a WebView using the vulnerable API. Another issue, on which the exploitation of the aforementioned issues rely, is that untrusted Javascript can execute arbitrary Java code (and thus shell commands and native code). More details about these issues are presented in the following sections. The important point to understand here is that this is not just one issue. There are multiple, interrelated vulnerabilities.

In all cases, exploiting vulnerable apps gives an attacker the privileges of the app itself. However, wide public availability of privilege escalation exploits for Android devices can lead to a full device compromise.

### Attack Vectors

Each vulnerability can be exploited via one or more [attack vectors](http://searchsecurity.techtarget.com/definition/attack-vector). Exactly which attack vectors allow exploitation depends on the context in which the affected WebView is used. That is, it depends where untrusted Javascript comes from. From our point of view they break down into the following categories:

1. URL-based attack vectors such as sending a URL via email, IM, SMS, etc.
2. Malicious advertisements (aka Malvertising)
3. Injecting an exploit into a compromised trusted site (e.g. a watering hole attack)
4. Man in the Middle attacks (MitM) such as DNS hijacking, rogue AP/BTS, etc.
5. Local attacks against Javascript cached on external storage (SD card)

The most severe issues can be exploited in all scenarios while slightly less severe can only be exploited via a subset.


## New Developments

Back in September 2013, jduck developed a Metasploit exploit module called [addjsif](https://github.com/jduck/addjsif) that targets vulnerable uses of *addJavascriptInterface* within advertising SDKs. He created this exploit in hopes that having it in Metasploit would bring additional attention and visibility to the seriousness of these issues. It is specifically designed to be used as a MitM proxy server that will inject its payload into passing traffic. The addjsif project's [README](https://github.com/jduck/addjsif/blob/master/README.md) documents how administrators can set up and use the exploit to test devices on their networks.

While testing his module, jduck successfully exploited the wildly popular [Fruit Ninja](https://play.google.com/store/apps/details?id=com.halfbrick.fruitninja) and [Angry Birds](https://play.google.com/store/apps/details?id=com.rovio.angrybirds) games installed on a Nexus 4 running Android 4.3 (!!). He also researched the names of other objects exposed via *addJavascriptInterface* and embedded their names into the module.

The decision to open source jduck's exploit module at the end of January 2014 set in motion the sequence of events leading to this post. Though MWR Labs published an exploit in December 2013, their Drozer tool simply doesn't have as large of user base as Metasploit does. The original goal stood and so the module was released publicly.

Starting in early February, the [Metasploit](http://www.metasploit.com) engineers took jduck's work and started looking to integrate it into their penetration testing framework. Some of their efforts are documented in the [pull request](https://github.com/rapid7/metasploit-framework/pull/2942) initiated to merge it. As seen in description of the pull request, Rapid7's Joe Vennix discovered that after minor modifications jduck's exploit module was able to remotely execute arbitrary code on older versions of the stock Android Browser app. This was news to us, as it hadn't previously been reported publicly by anyone (including Google!).

Further testing by Tim Wright showed that the exploit also works on the current version of Google Glass (XE12 as of this writing). Even worse, an exploited Google Glass browser yields camera permissions. That's right. This exploit allows an attacker to see whatever someone who is wearing Google Glass sees. On top of that, several publicly available exploits are capable of gaining root privileges on a Google Glass device. These facts combined should cause quite a concern for early adopters of this exorbitantly priced gadget.

Following these events, another media blitz ensued with many articles claiming that 70% of Android devices are vulnerable to this particular exploit. A lengthy discussion on [a post to the /r/Android sub-reddit](http://www.reddit.com/r/Android/comments/1xtjbq/scan_a_qr_code_get_exploited_metasploit_just/) had us defending our research against vehement opponents. Truth be told, the 70% number came from an assumption that all devices prior to Android 4.2 were affected. We had tested against only a handful of devices at that point and thus decided to do some more thorough testing against the stock browser of various devices. 

First, jduck set out to create [a simple and safe test page](/tests/addjsif/) that would detect whether or not the WebView in which it was loaded was vulnerable. The Rapid7/Metasploit team assisted in organizing some crowd-sourced testing on Twitter while he tested against the droidsec [droid army](http://opencfp.immunityinc.com/talks/27/). Each of jduck's devices run stock firmware (in most cases the latest available), so the tests are a decent sampling of the over all device pool. Further, Joe Vennix ran some tests against [one of the public testing services](http://www.appthwack.com/) to see how their devices fared. The results of this round of testing were quite interesting and several new facts were uncovered.


### Key Findings

Through our testing, we have discovered several important facts worth discussing. The key findings are:

1. Early testing with ad-supported apps revealed that even current and fully up-to-date devices can be successfully exploited in specific circumstances. Applications that (1) insecurely use *addJavascriptInterface* to render untrusted content and (2) are compiled against an API level less than 17 remain vulnerable. The popular apps tested are only two such apps; it's likely that many many more exist. This issue was assigned [CVE-2012-6636](http://www.cve.mitre.org/cgi-bin/cvename.cgi?name=2012-6636).

2. More recent testing revealed that certain versions of stock browsers, including AOSP, are also vulnerable. This issue is due to insecure use of *addJavascriptInterface* involving the *searchBoxJavaBridge_* object. Digging in deeper we discovered exactly when the vulnerable object was introduced ([9497c5f](https://android.googlesource.com/platform/frameworks/base/+/9497c5f8c4bc7c47789e5ccde01179abc31ffeb2%5e%21) in Android 4.0) and removed ([d773ca8](https://android.googlesource.com/platform/frameworks/base.git/+/d773ca8ff2a7a5be94d7f2aaa8ff5ef5dac501a8%5e%21/) and [0e9292b](https://android.googlesource.com/platform/frameworks/base.git/+/0e9292b94a3cb47374a8ac17f6287d98a426b1a8%5e%21/) in Android 4.2) within AOSP code. It was assigned [CVE-2014-1939](http://www.cve.mitre.org/cgi-bin/cvename.cgi?name=2014-1939) per [our request](http://openwall.com/lists/oss-security/2014/02/11/2).

3. Some devices using versions of Android within the affected range (4.0 < x < 4.2) were not vulnerable. This indicates that OEMs/carriers back-ported the patch to their firmware somewhere along the way. It's possible (maybe even likely) that Google notified these partners of the issue to spur movement without notifying the general public. Determining the entirety of vulnerable device + firmware combinations remains an open problem (and one that we are actively seeking to address).

4. An additional insecurely exposed object was found within certain HTC device firmware versions. In particular, testing against the One V and One X+ on Android 4.0.3 and 4.1.1 showed that an object called *HTMLOUT* was exposed. Interestingly, this is the exact name of an object that was discussed in a blog by Aleksander Kmetec in 2009. (NOTE: A quick search while writing this post turned up [this advisory](http://en.wooyun.org/bugs/wooyun-2010-013) on WooYun. We don't believe a CVE has been assigned yet.)

5. Certain versions of third-party browsers are also vulnerable. In particular, certain versions of the Dolphin Browser tested vulnerable. Unfortunately, the exact version and list of exposed objects are not available.

Despite the fact that these issues have existed for years, several of these findings have only come to light in the last month. These issues are important and need more attention. The next section provides some suggestions about what various parties in the Android ecosystem can do to help.


## Suggestions

These issues have not been holistically addressed. Thankfully, there are a number of things that various groups within the Android ecosystem can do to improve the situation. Without intervention or assistance from Android vendors, we are largely left to our own device to protect ourselves and our users.

That leads to the first, and most important, recommendation for everyone in the ecosystem. **Communication is key.** We as a community need to spread the word about what brings these issues about, how to avoid them, how to locate and fix instances of them, and so on. Please do your part to help make the Android ecosystem a more secure place.

**Google** - You took a positive step when you made that security enhancement in Android 4.2. However, the original "fix" and messaging surrounding it leave much to be desired.

On the messaging side, that post represents a missed opportunity to convey the seriousness of the matter. You could have strongly recommended targeting apps that depend on *addJavascriptInterface* to the latest API level. You could have reiterated the peril of improperly using this API. Unfortunately, you opted to only casually explain the change as an enhancement.

On the "fix" side, a more aggressive approach would be far more effective. Had you taken such an approach, you might not be reading this post. Because of the approach you chose, developers can still build, and are still building, vulnerable apps today. These apps can even be exploited on the most up-to-date devices. These are serious vulnerabilities and they should be given the respect and urgency that they deserve. We plead with you to take more aggressive actions to remedy this blight.

**OEMs and Carriers** - Some of you have done a good thing by back-porting the fix to older firmwares. That's great for the devices that did get updated, but that leaves users of older devices exposed. We dream of a day when you don't leave anyone behind. The time from a security fix to it being on users' devices really should be on the order of days, not months or weeks. We implore you to find ways to continue to improve this process.

**App Developers** - If you're currently distributing a vulnerable app/SDK, please take immediate steps to correct the issue and get an update deployed. If possible, avoid using *addJavascriptInterface* at all. If you *must* use it, target your app/SDK to API level 17 or higher. If you're making an SDK, consider forcing your users to use API level 17 or higher. If you must have a synchronous connection to Java-land, consider using *shouldAllowURLOverriding* to override *onJsAlert* or *onJsPrompt*. App developers need to go the extra mile to be sure they are not including advertising SDKs that put users at risk. Finally, use the best practices (HTTPS and certificate pinning) to help keep your WebView content trustworthy in the face of MitM attacks. 

**Researchers** - Join us in doing additional research, testing, and reporting so that we can eliminate these dangerous issues. Help discover exactly which devices and/or apps that are out there are vulnerable. Seek out additional unsafe uses of *addJavascriptInterface* and report your findings. It's my understanding that finding such a thing will win you a shiny CVE. The list in the addjsif MSF module might even be a good starting place. Work with vulnerable device/app vendors to get the issues fixed. Start dialogues with your customers in the mobile space to raise awareness. Finally, you can help protect users by creating third-party solutions!

**Users** - The security and privacy of your device is ultimately in your hands. Keep your eyes and ears open for information about vulnerable apps. Don't connect to potentially malicious Wi-Fi access points. Remove vulnerable apps from your device until they can be updated. Give 1-star ratings to vulnerable apps. As a precaution, play ad-supported games in airplane mode. Pay for your apps so that advertising network code is never activated.


## Future Work / Open Questions

While putting together this research we have identified a number of gaps and outstanding questions. We have additional details that we plan to publish in the coming days and weeks. Ideas for future work include:

1. More testing - Our test results are fairly limited. We'd like to solicit more high quality test results from the community at large. We're thinking of the best way to enable this testing, but for now following the directions on [the test page](/tests/addjsif/) would help.

2. More devices - Our test bed is fairly small. We'd love to add more devices, and have added a plea for devices to our [donation page](/donate/).

3. More statistics - Extracting statistics from app markets is very resource intensive. Determining which apps use *addJavascriptInterface* insecurely requires looking at each app individually, possibly even multiple versions of each.

4. More research - Our tests looking into whether or not compiling a library for Android apps causes apps that use that library to be vulnerable were inconclusive. Do they build against a particular API level? Can they use a different API level than the app they are included in? These are questions we'd like to answer.

There's more than enough for us to do. We would love your help!


## Conclusion

This post documents some history and our latest findings in our ongoing research of the WebView *addJavascriptInterface* vulnerability saga. Though these issues have received a great deal of attention in the press, most users remain vulnerable. The latest articles focus on the new discoveries that some stock browsers are vulnerable. However, they fail to explain the full risk. Once an attacker gains access, a variety of publicly available exploits allow them to fully compromise the device. Most importantly, application developers often still expose users' devices by insecurely using *addJavascriptInterface*. These issues can be exploited, even on a fully updated and current device (such as a Nexus 5 - tested today with Fruit Ninja).

The perilous situation of more than 51% of the Android device pool running woefully outdated software remains. As mentioned previously, many of the tested devices are fully updated to their latest stock firmware. In our testing of 44 devices, 13 were running browsers in the vulnerable version range. Of those, 6 had a vulnerable stock browser. Extrapolating these numbers out accurately is impossible without the exact numbers of each device/firmware combination, but a fair estimation is something like 25 percent (46% of 51%). This is much less than the 70% quoted in many articles, but also doesn't take into account the other issues surrounding this method (which are much larger issues).

In closing, there is much to do before this saga will end. We've provided some recommendations for various parties in the ecosystem and hope that they will not fall on deaf ears. That said, we're not naive. We recognize that there are many challenges on the road ahead of us. Through our initiatives we hope to overcome these issues and usher Android security into a new era.

Be safe out there!


