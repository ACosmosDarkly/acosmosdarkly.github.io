---
title: "HTB - Cap"
collection: writeups
permalink: /writeups/comingsoon
date: 2021-11-20
excerpt: ''
---
           

### Recon / Enumeration

The box is located at 10.10.10.245, so I open up with `nmap -sV` flag to get service and version information from open ports on the machine. Something I learned after the fact is including -sC is the equivalent of `--script=default` to run additional scripts to enumerate further. I'm learning new things all the time here, so I'll be including that from now.

Nmap returns ports open for File Transfer Protocol (FTP/21), Secure Shell (SSH/22), and gunicorn (HTTP/80) which is a Python web server.

![/images/writeups/cap/Cap2.JPG]

I also run `gobuster` to get a list of directories from the webserver. Gobuster returns three directories: data, ip, and netstat. Data has an HTTP status of `302 Found` that means ["the URI of the requested resource has been changed temporarily"](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/302). This actually has some interesting implications which I'll talk about in a different write-up. I will definitely be checking in on that directory.

![/images/writeups/cap/Cap10.JPG]

### User - Website

Initial inspection of the website reveals a dashboard for a security application. The directories enumerated earlier are pretty easy to guess; "IP Config" to the /ip directory, "Network Status" to /netstat, but what about /data?

![/images/writeups/cap/Cap4.JPG]

I check out the "Security Snapshot" link which apparently gives us a five second capture and downloadable pcap of the server. The page shows zero packets total, but I've found my /data directory. The directory shows the pcap number, 2, as a file inside the directory. Changing that number to 0 gives us a pcap from a different session...

_Disclaimer: I had to retake this screenshot, so my actual pcap count was more like 10 or 20-something rather than 2. I played around with this page for a little while._

![/images/writeups/cap/Cap3.JPG]

![/images/writeups/cap/Cap5.JPG]

Pcap with a total of 72 packets. I only managed to get a few packets in while messing with the tool, so I'm guessing this pcap was longer than five seconds. Sure enough, the TCP data it contains is an FTP session between the user "nathan" and the server. Unfortunately for nathan, FTP is a cleartext protocol and his username and password were captured in the process.

![/images/writeups/cap/Cap6.JPG]

I take the username and password and plug them into SSH and.....

![/images/writeups/cap/Cap7.JPG]

Got the User flag.

### Root - Capabilities + GTFOBins

Checking `sudo -l` to list available sudo commands returns nothing, nathan has no sudo rights. However, after doing some reading, I found the command `getcap` that returns "capabilities" for queries files. I run the following query:

`getcap -r / 2>/dev/null`

![/images/writeups/cap/Cap8.JPG]

Python3.8 has the capability `cap_setuid`, and reading the man page for `capabilities` indicates that this allows for arbitrary manipulation of process UIDs. This is how we're going to get root.

Using a python script taken from the [GTFOBins](https://gtfobins.github.io/gtfobins/python/) python playbook, I run the following:

`python3 -c 'import os; os.setuid(0); os.system("/bin/bash");'`

![/images/writeups/cap/Cap9.JPG]

We used the `setuid` option available through python to open a bash shell as root (UID 0). This box is done.

### Summary

I've been picking up HTB more lately and this was my first box in a while. I remembered GTFOBins from a previous box I worked on before. That made things a little easier, though this box is a particularly easy one already.

It was my first time making use of Gobuster to enumerate directories on a webserver, and learning about capabilities and what that means. For more information on that, might I suggest RTFM or [checking out this website](https://book.hacktricks.xyz/linux-unix/privilege-escalation/linux-capabilities) where I got the `getcap -r` command from.

_G_
