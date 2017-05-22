---
layout: post
title:  "A Simple Tool for Linux Kernel Audits"
date:   2017-05-22 23:00:00
author: jduck
categories: blogs
---

In Android, the Linux kernel is the crux of security. It is responsible for enforcing access control to just about everything in the system. If an attacker can gain arbitrary code execution in kernel mode, they can bypass application sandboxing, access hardware directly, and more.


Motivation
----------
Over the years, the Linux kernel source code has grown significantly. It contains code for all supported functionality including drivers for a wide variety of hardware across several supported architectures. A modern Linux kernel source tree is quite large. The tree we will use in our demonstration within this post is 729 megabytes.

```
dev:0:/android/source/kernel/msm$ du -hs . --exclude .git
729M    .
```

Now, you may be wondering why size matters. To better understand, let us put ourselves in the shoes of Bob. He works professionally as a source code auditor. Companies hire him to read through vast amounts of source code to discover and fix bugs and/or vulnerabilities in the code. Bob's success depends on devising and executing a strategy to use his time most effectively. Some auditors may opt to use automated tools while others may choose to manually read through the code. 

Bob prefers to work smarter instead of harder so he has developed a suite of regular expressions that he runs over the kernel code to identify potential problem areas. After sifting through the results and analyzing the surrounding code, he discovers a rather serious looking problem in a driver. He pulls out the test device for the project and quickly finds that the driver is not loaded. Oh no! Now Bob becomes curious. He reaches out and asks which Android devices in the "Droid Army" are using the driver. Unfortunately, it turns out no devices use that driver. In my opinion, this represents an error in Bob's strategy that led him to wasting his time. Is there anything Bob can do differently to avoid this fate in the future? The answer is "Yes."


The Tool
--------
Enter the Linux Kernel reducer, or *lk-reducer* for short. This tool helps avoid some of the problems we have discussed thus far. It works by monitoring file system access while building the Linux kernel. (By no means is its utility limited to the Linux Kernel, but it is simply where we found a need.) This is possible because providing the Linux Kernel source code is a legal requirement under the Linux GNU Public License (LGPL). By monitoring the build process, we are able to determine which files from within the source tree have been used to build the final kernel image. Some files will inevitably not get used and thus we can determine that they are not needed. Therefore, they can be eliminated from review during a source code audit. This saves time during both manual and automated source code reviews.

The current incarnation of the tool is a slight modification of an implementation by Jann Horn. The first incarnation consisted of using strace(1) and a bunch of shell scripts to monitor calls to the open(2) system call. Jann developed his version around the Linux *inotify* subsystem. His implementation is much more clean and performant. My only modifications were to let the user decide how to process the monitoring results based on a data file. The tool generates a file called "lk-reducer.out" that shows whether each file was **A**ccessed, **U**ntouched, or **G**enerated. Let us see it in action!


A Demonstration
---------------
Once the tool is downloaded and compiled, give it the path to your Linux kernel source tree:

```
dev:0:lk-reducer$ ./lk-reducer /android/source/kernel/msm
dev:0:msm$
```

Now, build the Linux kernel to track which files are accessed. Remember to not only configure and build the kernel, but also clean the build too. Otherwise, we might miss files used during the cleaning process and end up with a tree we can build but not clean.

```
dev:0:msm$ export ARCH=arm64 SUBARCH=arm64 CROSS_COMPILE=aarch64-linux-androidkernel-
dev:0:msm$ make marlin_defconfig
  HOSTCC  scripts/basic/fixdep
  HOSTCC  scripts/kconfig/conf.o
  SHIPPED scripts/kconfig/zconf.tab.c
  SHIPPED scripts/kconfig/zconf.lex.c
  SHIPPED scripts/kconfig/zconf.hash.c
  HOSTCC  scripts/kconfig/zconf.tab.o
  HOSTLD  scripts/kconfig/conf
drivers/soc/qcom/Kconfig:381:warning: choice value used outside its choice group
drivers/soc/qcom/Kconfig:386:warning: choice value used outside its choice group
#
# configuration written to .config
#
dev:0:msm$ make
scripts/kconfig/conf --silentoldconfig Kconfig
drivers/soc/qcom/Kconfig:381:warning: choice value used outside its choice group
drivers/soc/qcom/Kconfig:386:warning: choice value used outside its choice group
  CHK     include/config/kernel.release
[... further build output omitted for brevity ...]
  DTC     arch/arm64/boot/dts/htc/msm8996-v3-htc_sailfish-xb.dtb
  GZIP    arch/arm64/boot/Image.gz
  CAT     arch/arm64/boot/Image.gz-dtb
dev:0:msm$ make mrproper
  CLEAN   .
  CLEAN   arch/arm64/kernel/vdso
  CLEAN   arch/arm64/kernel
  CLEAN   crypto/asymmetric_keys
  CLEAN   kernel/time
  CLEAN   kernel
  CLEAN   lib
  CLEAN   net/wireless
  CLEAN   security/selinux
  CLEAN   usr
  CLEAN   arch/arm64/boot/dts/htc
  CLEAN   arch/arm64/boot
  CLEAN   .tmp_versions
  CLEAN   scripts/basic
  CLEAN   scripts/dtc
  CLEAN   scripts/kconfig
  CLEAN   scripts/mod
  CLEAN   scripts/selinux/genheaders
  CLEAN   scripts/selinux/mdp
  CLEAN   scripts
  CLEAN   include/config include/generated arch/arm64/include/generated
  CLEAN   .config .version include/generated/uapi/linux/version.h Module.symvers
dev:0:msm$
```

With the build and clean process completed, simply exit the sub-shell to generate the data file:

```
dev:0:msm$ exit
exit
processing remaining events...
inotify event collection phase is over, dumping results to "lk-reducer.out"...
cleanup complete
```

From here, you can analyze the data file and choose whether to delete the files from the current source tree or copy them to another place. I tend to copy the files to another directory and audit from there.

NOTE: If you're working with a tree from Git, you might want to filter out the *.git* subdirectory as I've done here. You can always consult the history in the original repository.

```
dev:0:msm$ grep ^A lk-reducer.out | cut -c 3- | grep -v '\./\.git/' > lk-reducer-keep.out
dev:0:msm$ mkdir ../msm-marlin-reduced
dev:0:msm$ tar cf - -T lk-reducer-keep.out | tar xf - -C ../msm-marlin-reduced/
dev:0:msm$ du -hs ../msm-marlin-reduced/
132M    ../msm-marlin-reduced/
```

As you can see, we've reduced the source code from 729 megabytes down to only 132 megabytes. More importantly, we know that there's no code left that is not built into the final kernel image.


Limitations
-----------
Of course, this is not the only problem faced when auditing the Linux kernel. Even though we have eliminated code that isn't accessed at build time, some remaining code may not get used due to probing, system configuration, or other states that may be out of our control. Also, code within a file might still be thrown out by the pre-processor or optimized out during compilation. Further, this tool does nothing to help understand the threat model or determine attack surfaces that might be interesting for an audit.


Conclusion
----------
This post defines one problem Linux kernel auditors face and presents a tool designed to help solve it. Jann and I are pleased to publish this tool to assist the community in their source code audits of the Linux kernel. It is our hope that the tool will save you time and make you more effective. You can find the tool on GitHub [here](https://github.com/jduck/lk-reducer/). Thank you for your time and best of luck and your kernel auditing endeavors!


