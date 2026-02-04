---
layout: post
title: The short guide for the Freeloader Researcher
categories: [Security Research]
tags: [threat-intelligence, osint, virustotal, ransomware, malware-hunting, any-run]
---

# The guide for a freeloader Threat Intelligence Analyst and Malware Researcher

# Chapter Zero – Prologue

Recently I saw a blog post by [Trend Micro](https://www.trendmicro.com/en_us/research/22/h/ransomware-actor-abuses-genshin-impact-anti-cheat-driver-to-kill-antivirus.html) being posted in the [Curated Intelligence](https://twitter.com/CuratedIntel) Discord group. The blog post describes a rather interesting ransomware incident discovered by Trend Micro where [(yet again)](https://www.mandiant.com/resources/blog/unc2596-cuba-ransomware) a legitimate driver was being utilized to terminate security related processes from the kernel space. I was curious and looked in the IoC index.

![](/assets/img/freeloader_blog/1.jpg)

I decided to look up the IoCs in a VirusTotal and unfortunately the only hash that was listed in the VT database(to this posts date) was the driver **mhyprot2.sys**.

As I am lucky enough to have access to a premium VTI account and determined to provide the juicy hashes to my community I started researching.

![](/assets/img/freeloader_blog/2.jpg)

And quickly enough I found more campaigns from the same ransomware actor and found the tools and ransomware that were associated with the Trend Micro report. However, as my ego was being stroked by myself (after all I did just find all those missing hashes) I remembered the days when I was just starting out with Malware Research, and I begged for samples.

And so, I decided to challenge myself, I will find and download all the missing samples and attempt to attribute the ransomware actor using just OSINT and free tools.

Let's dive in.

![](/assets/img/freeloader_blog/3.jpg)

# Chapter One – Finding the samples

Let's look up the driver **mhyprot2.sys** hash(0466E90BF0E83B776CA8716E01D35A8A2E5F96D3) in a community VirusTotal account and look up the **relations** tab( I avoided the community tab as the collections and comments are not useful at all in this case)

![](/assets/img/freeloader_blog/4.jpg)

There are a lot of execution parents for this driver as it is a legitimate gaming driver, and the drop-down list of execution parents is crazy long. However, we do have some clues to look for. From the report we know that **avg.msi** and **avg.exe** are responsible for dropping the driver onto disk:

![](/assets/img/freeloader_blog/5.jpg)

I decided to keep dropping more files within the execution parents list and what do you know –

![](/assets/img/freeloader_blog/6.jpg)

Interesting, lets see what this hash is about. I decided to click on the file name (MD5: **b6373b520a21c2e354b805d85a45a92d** ) and it's a jackpot!

![](/assets/img/freeloader_blog/7.jpg)

Two files are missing however the ransomware file itself called **svchost.exe** and **logon.bat**. The first is quite easy to find, by just clicking the dropped files within the MSI I found that VirusTotal displayed a misleading name for the ransomware file. By clicking on the 6th file (with the long hash name) we get transferred to the following sample( **5143bbdf1f53248c7743f8634c0ddbc** ).

![](/assets/img/freeloader_blog/8.jpg)

Great! Now for the missing batch file and the ransomware note. The only way I could think of is to trigger the MSI installation on a sandbox like AnyRun and not only would I gain access to all the files as AnyRun allows anyone who pleases to download files from their reports but I would also theoretically gain access to the **logon.bat** file and the ransomware note, which could aid in attribution.

# Chapter Two – Downloading the samples

Back in the day, I used open malware repositories, or I begged from my researcher friends to use their VTI access and download samples for me. The latter will obviously hurt our fragile ego so lets start looking up avg.exe and avg.msi in open malware repositories.

I searched the hashes in AnyRun, Malshare, Malware Bazaar and Google. Finally, I hit the jackpot with Hybrid Analysis which allows registered users to download samples!

![](/assets/img/freeloader_blog/9.jpg)

The next step is to download the file and trigger an execution on AnyRun which should provide me easy access to all the samples within this ransomware campaign.

Once I got my hands on the **avg.msi** sample I ran it on AnyRun. There's quite a cool trick in AnyRun that I only discovered when I laid my hands on a premium account. Its possible to increase the machine run time by clicking the button **Add Time** at the top right, for a free account it will increase the run time from 1 minute to 5 minutes which could change quite a bit.

![](/assets/img/freeloader_blog/10.jpg)

Anyway, I ran the sample and what do you know! We have the entire infection process with all the files that were missing!

Here is **logon.bat** which was missing in VT.

![](/assets/img/freeloader_blog/11.jpg)

Here is the ransomware note:

![](/assets/img/freeloader_blog/12.jpg)

# Chapter Three – Attribution

This chapter will be quite short sadly. Spoilers, I couldn't find the actor responsible for this ransomware.

I first tried to upload the ransomware note to [ID Ransomware](https://id-ransomware.malwarehunterteam.com/) which can attribute ransomware notes to ransomware actors by just uploading the ransomware note. Which claimed this ransomware note belongs to Nemucod.

![](/assets/img/freeloader_blog/13.jpg)

I was doubtful so I uploaded the ransomware file **svchost.exe** (since its on anyrun now I can download it for free) to **Intezer** which does a community account version. Intezer attempts to attribute files to malware and malicious actors by code "genes".

![](/assets/img/freeloader_blog/14.jpg)

That also wasn't very helpful. Well.. I did my best right?

See you guys next time!

**Update 8/27/2022:**
Thanks to [@1ZRR4H](https://twitter.com/1ZRR4H) who took a good look at the ransomware note saw that the TOX-ID in the ransom note is the same one found in [Rever Ransomware notes](https://www.enigmasoftware.com/reverransomware-removal/). A simple google search of the TOX-ID confirms it:

![](/assets/img/freeloader_blog/15.jpg)

[@Amigo_A_](https://twitter.com/Amigo_A_) has written about Rever ransomware. Read more about Rever Ransomware here:
https://id-ransomware.blogspot.com/2022/07/rever-ransomware.html

# Chapter Four – IoC appendix

AVG.MSI - b6373b520a21c2e354b805d85a45a92d

AVG.exe - 44961feb7fd9eeabdb67e5eeb15b9c8a

HelpPane.exe - d33dac29513dcc1027f29d5e9e901369

Svchost.exe - 5143bbdf1f53248c7743f8634c0ddbc1

Logon.bat - 160b427081688e677d0136a42dddc2d9

Mhyprot2.sys - 4b817d0e7714b9d43db43ae4a22a161e
