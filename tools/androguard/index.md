---
layout: wiki
category: pages
title: Wiki
---

# Tools :: Androguard

Author: Anthony Desnos  
Website: [http://code.google.com/p/androguard/](http://code.google.com/p/androguard/)  
Current version: 1.9  
Last updated: December 05, 2012  
Direct D/L link: [https://androguard.googlecode.com/files/androguard-1.9.tar.gz](https://androguard.googlecode.com/files/androguard-1.9.tar.gz)  
License type: LGPL  


### Description

Androguard (Android Guard) is primarily a tool written in full python to play with:

* .class (JavaVM)
* .dex (DalvikVM)
* APK
* JAR
* Android's binary xml

### Features

* Map and manipulate DEX/ODEX/APK/AXML/ARSC format into full Python objects
* Diassemble/Decompilation/Modification of DEX/ODEX/APK format
* Decompilation with the first native (directly from dalvik bytecodes to java source codes) dalvik decompiler (DAD)
* Access to [static code analysis](http://code.google.com/p/androguard/wiki/Analysis). Includes basic blocks, instructions, permissions (with database from [http://www.android-permissions.org/](http://www.android-permissions.org/))
* Allows creating your own static analysis tool
* Analysis a bunch of Android apps
* Analysis with ipython/Sublime Text Editor
* [Diffing](http://code.google.com/p/elsim/wiki/Similarity#Diffing_of_applications) of Android applications
* [Measure](http://code.google.com/p/elsim/wiki/Similarity#Similarity_of_applications_(aka_rip-off_indicator)) the efficiency of obfuscators (proguard, ...)
* [Determine](http://code.google.com/p/elsim/wiki/Similarity#Similarity_of_applications_(aka_rip-off_indicator)) if your application has been pirated (plagiarism/similarities/rip-off indicator)
* Check if an Android application is [present](http://code.google.com/p/androguard/wiki/DetectingApplications) in a database (malwares, goodwares ?)
* Open source [database](http://code.google.com/p/androguard/wiki/DatabaseAndroidMalwares) of Android malware (done in limited free time, so help is welcome!)
* Detection of ad/open source librairies (WIP)
* Risk indicator of malicious application
* [Reverse](http://code.google.com/p/androguard/wiki/RE) engineering of applications (goodwares, malwares)
* [Transform](http://code.google.com/p/androguard/wiki/Usage#Androaxml) Android's binary xml (like AndroidManifest.xml) into classic xml
* [Visualize](http://code.google.com/p/androguard/wiki/Visualization) your application with [gephi](http://www.gephi.org/) (gexf format), or with [cytoscape](http://www.cytoscape.org/) (xgmml format), or PNG/DOT output
* Integration with external decompilers (JAD+dex2jar/DED/fernflower/jd-gui...)

