---
title: "Memory Forensics 1"
collection: writeups
permalink: /writeups/mem1
date: 2023-05-30
excerpt: ''
---

I recently completed a CTF challenge on a site that covered some of the basics of memory forensics. The challenge wasn't particularly difficult, but I feel like good memory forensics puzzles are few and far between. I'm not going to give specifics on the challenge, but I did want to write about a few of the interesting things I discovered while looking for the flag. Plus, for anyone who doesn't know about Volatility and memory forensics I hope to give you some insight into how it works. The tool I'm most familiar with, and the one I will be using, is the Volatility framework which is available [here](https://github.com/volatilityfoundation/volatility3) (requires Python).

## Initial Analysis

The challenge comes as a downloadable zip file containing a few artifacts, one of those being the memory image in a *.elf* format. An ELF file, also known as the *Executable and Linkable Format*, is a command file standard for certain executables, core dumps, or other Unix specific binaries[^1]. The other two files contain some supporting information regarding the memory dump, but since this isn't a write-up in the traditional sense I'm going to omit them. 

![](/images/writeups/mem1/rem1.JPG)

Something important to note about Volatility is that the older version required you to use an Operating System "profile" in order to scan the memory dump. This could be a bit of a headache to get around if you're learning Volatility for the first time. The newer Volatility 3 Framework doesn't have this requirement, so we can skip it. If you were to need a profile type, the older Volatility has a the [imageinfo](https://github.com/volatilityfoundation/volatility/wiki/Command-Reference#imageinfo) command that will give you it's suggested OS profile. Just a tip!

At this point I looked at the supporting files and got an idea for what I was dealing with. Using the Volatility 3 Framework we can check the processes that were running when the image file was taken. The plugins `windows.psscan.PsScan`, `windows.pslist.PsList`, and `windows.pstree.PsTree` basically provide the same information with slight variations. We'll run `PsTree` to get the processes that were running, and the parent/child relationships between the processes. An initial scan of the processes reveals a potential red flag, PowerShell.

![](/images/writeups/mem1/rem2.JPG)

## Digging Deeper

PowerShell is generally considered safe by Windows because it's specifically a Windows scripting language and command-line tool. It's extremely powerful and as such is used extensively by malware and malicious actors in "[Live off the Land](https://www.crowdstrike.com/cybersecurity-101/living-off-the-land-attacks-lotl/)" (LotL) attacks. It's built-in, it's trusted, and it's powerful, what else could a hacker need? It's definitely something to look at first in any incident... just to be sure. Volatility contains a plugin called *Malfind* that will list memory ranges that could contain malicious code. Running the plugin `windows.malfind.Malfind` on our image confirms our suspicion that PowerShell is probably doing more than a little admin work here.

During the initial analysis of the files provided there was a reference to a PDF document that was downloadable from an external source. I assume it has something to do with our PowerShell command here. Luckily Volatility has a plugin for that too and it's called `windows.filescan.FileScan`. We just run the plugin and grep for the file name and see what comes up, and guess what, we get some hits! The file listings contain memory addresses, so we can use `windows.dumpfiles.DumpFiles` to dump whatever is at that address. 

![](/images/writeups/mem1/rem3.JPG)

The file dump gives us a couple files to choose from that contain data from the memory address being dumped. Sure enough we have a super suspicious PowerShell command running with the *hidden* argument (to prevent the PowerShell window from showing up when the command is run), setting the Execution Policy to Bypass to allow scripts to run, and then Base64 text encoding. Yea, nothing happening here just a bunch of encoded text being passed as a PowerShell command.

## Sidebar: What's happened so far?

If you've been following along and this is your first time with memory forensics you're probably thinking this is moving along pretty fast. Maybe, maybe not, but I wanted to add some detail to what's been done so far. Memory forensics is like observing various parts of your computer OS as you would normally, i.e. with Task Manager, netstat, etc., but in a kind of "suspended animation". The CPU cycles have stopped, the network connections are static, the processes are frozen in time, all of which can be observed with the right tools.

Thus far we've used Volatility to list processes, find malicious code, and dump files, but there's so much more you can do. We can list the connections that were active at the time when the memory dump was taken, list all the loaded DLL's, list registry hives, dump registry hives, scan using Yara rules... and this is just Windows! Volatility is also capable of analyzing Linux and Mac memory dumps with much of the same tools.

We'll continue on and see how we can now take information out of Volatility and run it through other tools to put together the rest of the picture.

## Deobfuscation

There are actually two sections of obfuscated code in the file we've dumped. I used [Cyberchef](https://cyberchef.org/) to work out this part by the way. The first part is just straight Base64, and once decoded appears to be a decoder for the main body. A big giveaway is that the final variable is passed to *iex* which is PowerShell shorthand for *Invoke-Expression*[^2], or the cmdlet you use to run another PowerShell command or expression.

![](/images/writeups/mem1/rem4.jpg)

This still leaves us with the main body which took a little more work to get through (image not included). The PowerShell in this section is hidden inside some encoding, which only reveals another call to PowerShell, refer back to the initial process list where PowerShell was running twice, and another block of encoded text. This encoded text has a second quirk to it that exists because the Window Operating System doesn't use UTF-8 for Unicode encoding. Once you get this last block figured out the final block of PowerShell code will contain the flag. I think this last code block is a downloader, but I'm not sure. I just grabbed the flag.

## Summary

This challenge was a bit old, but I thought it was a great reintroduction to using Volatility. It deals with a number concepts relating to memory forensics and obfuscation techniques that are great for a fairly easy challenge. It doesn't go too deep to make it inaccessible to newcomers, but it also doesn't go too easy and makes you learn how to use the tools available to you. I'm going to look out for more of these memory forensic specific challenges because they're fun and make for interesting write-ups! 


[^1]:  If you're into technical illustrations, check out [this one](https://upload.wikimedia.org/wikipedia/commons/e/e4/ELF_Executable_and_Linkable_Format_diagram_by_Ange_Albertini.png) of the ELF file format.
[^2]: "The `Invoke-Expression` cmdlet evaluates or runs a specified string as a command and returns the results of the expression or command. Without `Invoke-Expression`, a string submitted at the command line is returned (echoed) unchanged."      Source: [Microsoft](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-expression?view=powershell-7.3)
