---
title: 'Process Investigation with strace'
date: 2023-10-23
permalink: /posts/2023/10/straceinvestigation/
excerpt: Using strace to uncover the reasons behind stuck processes
---

I was working on upgrading an application and ran across an issue that unexpectedly led me to a few valuable learning experiences in Linux OS processes. The upgrade I was performing involved upgrading the appliance from an RPM.

The process is typically fairly simple, just stop the services from the terminal and run the package updates using the `rpm -U {package name}` command. The first part of the update went off without a hitch, but the problems started when I attempted the second update.

I ran the update command again, got a few steps in, then..... nothing. The process just stopped with zero feedback. The process *appeared* to be running, but it sat there for a long while doing nothing. This situation is not uncommon in IT, where systems may seem unresponsive for an extended period but are, in fact, working in the background (looking at you diskpart).

I wasn't being particularly patient because the first update completed in seconds, so I had the expectation that this would do the same. Since this was not the case I decided to look into it.

### Is it really stuck, or is it just me?
Off the top of my head, I recalled a couple of checks I could perform: `ps aux` for showing all processes on the system, and `lsof -p {PID}` for listing the files opened under the specific PID for the RPM. In the end, all this got me was confirmation the RPM process was running, and it was using the files I was expecting. It was still stuck though, so I went searching for other answers.

I got a few suggestions and checked out `/proc/{PID}/status` to see what the state of the process was. To my surprise, the RPM process was sleeping! 

![proc file process example]({{/images/posts/strace/LinuxProc1.JPG" | absolute_url }})

(example process above. Note the "sleeping" state of the process)

This provided at least some explanation. If it's sleeping it must be waiting on something, but what could it possibly be waiting for? Fortunately, `strace -p {PID}` helped solve that problem by informing me that it was waiting for a different process. The form of the output from this command was:

`wait4({PID}`

Okaaay... so I'll run through the process with this new PID which happened to be /bin/bash doing some other RPM processing. Turns out this one is sleeping too! What gives? I run the `strace -p {PID}`on this PID as well and get something a little more cryptic:

`wait4(-1,`

The first one had the PID, which made more sense, but -1 is not a PID, and I have no mental reference for it. Dr. Google it is, then! I end up finding that `wait4(-1,` is the same as `wait3()` which according to the [Linux Foundation](https://refspecs.linuxfoundation.org/LSB_1.3.0/gLSB/gLSB/baselib-wait4-2.html) means "wait for any child process...". Great, so the thing causing my primary RPM task to sleep, is itself sleeping because it's waiting on a different child process!

Since I wasn't given a PID for the new process I have to go digging in the process list again. This time I'm using `pstree -p` to give a visual representation of the process parent/child relationships, and include the PID for the processes. I grep and find the processes are part of a long chain ending in a `chmod` process. Chmod is used for modifying file system object access attributes. Interesting, why is chmod holding up the effort?

I proceed this time straight to *strace* and I immediately get a running list of function calls. Finally some good news! The process is running, but what is it doing? I spend a couple seconds looking at the output and notice some familiar file names. Turns out chmod is updating the attributes of a bunch of files as what I came to assume was part of the migration process that it runs during an update. I could see the files were running in alphabetical order, so I decided to be patient and wait.

Eventually the chmod process did finish and the whole RPM update completed successfully. As an IT professional I should know by now that patience is a virtue, but why wait when you can investigate!

The takeaways I got out of this whole endeavor are:
- To check the state of a process, check the process info in `/proc/{PID}/status`. You can quickly see what the state of the process is. There are other ways to do this I'm sure, as there's always a dozen ways to complete the same task. BTW, this is a file so you'll need to `cat` or `less` to see the contents, i.e. `less /proc/{PID}/status`.
- I've used strace before, but this gave me a greater appreciation for the command as a tool for checking hung processes. It can be a great troubleshooting tool, so [here's some more info.](https://strace.io/)
- I need to remember pstree as a command for checking parent/child process relationships. There are times in problem investigations where this is important, such as it was for me.
- Keep asking why. Why is the process sleeping? Why is this other process sleeping? Why is strace showing wait4(-1,? Why is chmod running? 5 Whys for root cause analysis.
