---
layout: post
title: Zero2Auto – Initial Stagers - From one Email to a Trojan
---
## Preface
This week we have discussed deobfuscating initial stagers and how to unpack their executable payloads. 
And what I’ve decided to do, to practice this week lesson is to find actual malware on any.run and unpack its entire initial stage. 

**Setting up goals:**

1. I’ll choose an infected document from any.run, then I’ll deobfuscate it and extract its payload
2. I’ll take the payload and unpack it from memory, showcasing how to unmap a dumped PE 
3. I’ll attempt to practice the first lesson by finding identifying encryption schemes or known algorithms within the payload.

**Required background:**

1. Knowledge in reversing VBA code
2. Knowledge in reversing PowerShell code
3. Knowledge in various analysis utilities (Process Hacker, PEBear, PEStudio, PEId)
4. Knowledge in IDA Pro
5. Knowledge in x64dbg
6. Basic understanding of stack overflow exploits and how to debug shellcode
***

## <span style="text-decoration:underline;">Choosing a Target:</span>

I decide to go on any.run to find an interesting sample – I have experience in macro deobfuscation so I decide to look for a infected document hosting an exploit which delivers a payload.

I decide to filter MS Office files with an exploit tag, and quickly find a huge batch of files:

![624x295](https://lh3.googleusercontent.com/vUv44khwAaBPMnRoRBjsDAyhztVgrghr36hamHQaKSuVmHUESFFvocTFP4C7clxZq-GsV6IW_rDK3BQIzLvZUsYD9vS91mNadbzAqKutuFKzUiu2nEBusGSd741Qeo5QqCrfxGmFP6Phnc_Tqw)

Its valuable to mention that **CVE-2017-11882**([you can read about it here](https://unit42.paloaltonetworks.com/unit42-analysis-of-cve-2017-11882-exploit-in-the-wild/)) is the same exploit used in the Zero2Auto lesson and it seems as it’s the most used exploit when delivering malicious docs. I’ve also observed that the exploit is sometimes leveraged that attacks the Outlook app directly when I filtered for email message file types:

![624x294](https://lh6.googleusercontent.com/dGdWe8SGaoNQgQ0GOuE1Z1Rq-txScrI0igf_Sjcv_qqhOIQZv2WChttAbzZBz6l3hYl7J0Ql1nbaCUbf5uIRbUZa1l9Xdj9YBwn5zrIHnt4eOxoqV8x2yu_8Okmg5W-XBJF6GdSynTnn3z4Ehg)

I have chosen the following sample: 

[https://app.any.run/tasks/fce8f628-52cf-4f5d-968c-ab59ddeaabda/](https://app.any.run/tasks/fce8f628-52cf-4f5d-968c-ab59ddeaabda/) and the reason for that mainly is because the infection chain is long, contains a wsf script, a vbs script and finally an EXE payload thus it could give me a lot of room to practice my knowledge. 

***

## <span style="text-decoration:underline;">From an email lure to an exploit:</span>

![624x336](https://lh3.googleusercontent.com/KlDF2HJXaiguTwvswWvHjIhXNAHpu1wxhWCw5_Xi_lPEJrcmwKbd7DZSSDbc-L8vpaB2QvdJ8aE1VB4uwwMjPfkqju1wJ3qZan2xldqA8NiPIhlrL7VncmD6lNGswzuny6CJN38IL8DFF5t1GA)

I decide to extract the excel document and inspect it(SHA-256 97CC9023C114013326A97E946B0CA71BB641952208B5BA04463F1DAF15E1A5B5). 

**<span style="text-decoration:underline;">Method 1 – Examining the document by opening it</span>**

The first method is rather, not very safe to say the least but you should be fine if you have macros disabled by default, so I open up the Excel document and I’m met with the following:

![624x286](https://lh5.googleusercontent.com/Xc40sdmRLVzAWyE3KenyktwXJNWGhMAVFNB_T90KCZ67FWkjLA-SvFBpw3C__bhKVgTLOv2FzPuzlOY7UBlWI7pE4PyE-JMZKKVRob4y0RJ2IV4B1DjKpxadS6x1SVU8XscV8ZxBx77WfEcpJg)

I’m definitely not going to enable macros but before I review the macros I decided to check out the rest of the sheets. Sheet number 2 has a very interesting object within it:

![195x101](https://lh6.googleusercontent.com/HNGWNfQYpwPcCHAj3KNIEF9bJMRSAYAU50Q5_4iFhHLfNy7C5mFmXBQ-sq4iMUwSZUVrHOcfjXWO1yAhggRwXahlqNBaUcMiOJBkuSiSl9FSXiw7oFFVO59k-U6wDdEptvKcBsw0StwSoEKRxg)

And if you double click it:

![624x201](https://lh4.googleusercontent.com/bo6PruIKjtYbKoMrj6Q1QTusGlljp4m3adGC9J0Z2gxRryaKmLEosebNsoOGY4AxLY-L6oH_QaYbTPEePX9i-t-ke0P5u4D1_E9aDiVKS18uD-e6_81_8gkHHAnipJjcfkNOn_izLsR7r5UhVg)

This is an equation editor object that was embedded within the document, as I’m running on a fairly new MS Office version and this exploit was long ago mitigated on new versions I would expect I’d run into problems in performing this exploit but I bet I can do some stuff with the shellcode itself if we find it. Let’s continue to Sheet3:

![143x193](https://lh5.googleusercontent.com/SpWpkhHwwQYiHEEWjGcy_yZwTx74SDOt4J_E8wmRIYtX2FPru9YDlG5GlRml8CqyPJ4_2PDsYx0o42OQAWOxOGOmtFLlT5addEqHNSm_PnMbpL4KFrL2T6xw6uHBakvTAzGQG9_ytdu_Mu6HJg)

An observant person like myself (flex much danus?) noticed this two little square and if we double click them, we’ll see that they are actually objects embedded within this document

![518x462](https://lh5.googleusercontent.com/_eDeaa417Vxu1CxNL2EZxdHtmzW9t-1HgojvVtNNY22SdMff_LTQefUjRB3dWOskcmPhkuPR4MHty9-hrBO5uQrStAEvpCnBPv4mNjqQYNyciF557861YtZgTyKPwPwDlmhKKmDo5WdDALyKCA)

These can also be seen being launched in the any.run view:

![624x331](https://lh5.googleusercontent.com/njesZ1dB8Y0TNZJ4WqmdpOsyQdhh4fpO9_2tUwFI0DBzhDWRvbLUiyVG0xV8-IF8n7wbBCfuyuE50CTz8mOlLdTMhDZRRNG3fgTwTdJ_UZI6YFtSVfwfr_hVIbP-ioeJYs0F_pQ4UheGkaoGqg)

What was interesting that if we decide to the view the locations of these embedded objects 

![466x184](https://lh4.googleusercontent.com/yIKxOGhSKa10TqttDoPBDdNfLnu8j4fkp1o9aRxJr7uPCy3Q9ABmZMZmHkAeYXdb6C4XDicm7zXOQQbqcilIgIyJaUqPSXVSao_zGB1LI4VvGBTjH5m1hY7jB9uMUSPWNZUdPEEhWGVxAt2aRw)

They are located within the Temp folder, but I never downloaded them.

To be honest, at this point we can stop the analyzes and simply dump these objects but I would like to review the actual shellcode of the exploit itself just to analyze it. So, I decided to enter the macro editor, and would you like at what I’ve found:

![377x101](https://lh3.googleusercontent.com/0v8uu8N9CrPNBkr8GfQeUkw9XcPCab_kRtLfnYblV1QYy2nQnxmZPFNKdWULqFS43cRlRdGefjIs7GtSWhU9ZkY3tB01TjeFiaXpxoL_WRiliGm-KcTDSfK15Gc8cbtbs6cc8IUEdTBsc-libw)

**Auto_Open** is great news for us!

![624x219](https://lh4.googleusercontent.com/B1TEx9a6aElbEhRL4NqC-nhXdVHSwQruj6L47irZRpK5sowco7dQ535pjlPtID2ZYoGt2idzocAtzOb8dv9s2I9BOzCrlMYAVEGXGx2FooFinPX6EOVOc2Kkq3x6C6pyTBuSmSDln8pb0FNkdA)

I’ve decided to remove the macro deobfuscation from the scope of this blog because it’s pretty straight forward, so I’ll summarize it. This first stage macro contains another macro within itself, it copies this macro inside C:/Programdata/asc.txt and then it runs the newly written macro. There is one problem with this method though, this xlsm file is trashed and I can't modify it. It means that every time I want to edit the xlsm, I have to run the malicious macro.

![624x227](https://lh3.googleusercontent.com/frFsLaMLl3k8fhSgv3NkTzTMphddCbaUSBhsxB2c5YPOrOZfEzAz5mZM-HHnvFStgAYyovPPAJd232Qiez3Upt_QWizhkxzJrd3qslLl42TsSbiK54n2Uh6HC46Ra8TAqWtvV_Fbv8E4rF5KCA)

The second script is odd to say the least, it includes a lot of documentation for some reason and a lot of nonsense comments and text. I found one valuable text though:

![624x62](https://lh4.googleusercontent.com/lX72YlKTMP_CWrOAcnJ8S_-opHlWl7ntlFz6mgRtFB1d3xUTt6k2Eef9S_tvKf6IqfA5epMp5_wp5SWGg54X4uZB3a2ZOvzQYnVKyTJvEPoXrYEuKWVZZWI-Xsfy0i--N6qSCm4aiWn0bJO7pw)

Jokes aside this new macro is extremely obfuscated to the point where I could not use Daniels method of manual deobfuscation, so I decided to look for any valuable info. 

![624x212](https://lh3.googleusercontent.com/LABCDvpr_Sy_sql4LIcNU6EuUNQmH9bIRNXz1DNT1Ii-EgY2HLkAd62nzflJB3XKWl_QPuRUbSYJFYJJNcIijiz3R4HEM_eKARLCIWVBDwmnim5_Ejg7A0ICrkyTMXbbPiOMnyT01Q-0kIGREA)

One of the functions had Http function references and direct calls to some variable called myURL. The final function which I renamed to **DownloadFile** contains these references to the myURL Variable. Alas tho, this concluded with more confusing VBS code that I could not debug, for two reasons – being that this xlsm file is completely broken and I cant save it which doesn’t allow me to edit and debug the code properly and this method is also extremely dangerous as I couldn’t edit the malicious macro I kept executing the malware each time I opened the excel file. I had two more tools in my arsenal, one was to extract the two VBS files I’ve located within the excel and this can be done by manually opening the excel or using OLETools and honestly it’s a lot more safer to use OLETools so I set off to do just that.

**<span style="text-decoration:underline;">Method 2 – Using OLETools to examine the excel</span>**

We know that there are embedded objects with the excel because we saw them and we know they are automatically saved inside the Temp folder as we opened the excel even if Macros are disabled and indeed it was true I found the files in the temp folder but if we want to extract the files safely we should use OLETools for that. I decided to use OLEVba first on the macro itself to see if it can extract anything meaningful from the macro, we located

![624x271](https://lh5.googleusercontent.com/yql50L6V6zkCWnMo5BIkLofA7cpCckwfX19HLe3re4HonIkROOdadLZQ8kYeYo9m1Pt-rBh78z7WZAE6k50UjGNOQ6PdFyQl5os6amG8DEVH6XOjba8YQM1Smw-Qkya3kswVQ9vcN3aGhEZgFw)

Thankfully OLEVba did a lot of the work for us! By detecting the location of the exe payload. I also noticed that there are 4 OLE objects embedded within the xlsm. As we recall we found the Equation Editor, xx, mm so let’s try extracting them.

![624x75](https://lh6.googleusercontent.com/Vtu_yLp7UMgXpZmPfGBir33nexUDDYOSuVNZ4kn33ktACVM7H5eud3tlQ4tqeKgqjz2_0TukDV-rfRZAANGGDgXb_C0I9hpHcxlkPUIwGHipJ1A6wlOWRQ0EHD6l1O8SuH1XqyihCYGcJUWUWw)

Let’s use OLEDump to view the files embedded with in the script:

![624x377](https://lh4.googleusercontent.com/pRJ_tQbDyXHfo9AHMvu6rw6f6KKd7-KcCHoXtFvLXw_oorglwFm-60MMo0KLaEyJ--9LAihXsQPxkXS0Ass_ae_uniyB9GRCOn7Mv4H2Ci_WdQHhkKubkW1A9RhaHV6RkElU2BsrLUyrh1ma8g)

So the M signifies macro, we know that Module2 contains the Auto_Run macro, and from what I can tell there are 3 interesting files:

**B, C, D** I think the sub numerators to each character represent data within these objects. We know that xx, mm and the equation editor are embedded within this excel and because D3 I’d be assuming that object D is the vulnerable equation editor file. I suspect that since Microsoft mitigated the EQ3 exploit the attacker embedded it inside the excel and one of the macros somehow executes it. Well let’s see:

![624x456](https://lh4.googleusercontent.com/cv0QvoD0243GOaTiiuimVAISmx5AR_gGK1neUi-k6UETauZ9BnEfjtmGTLZnZmr1GAaMUGGvvgdtmHEjQZLPbFj1cm6bzTduzzePrf73VSuDnsLynB3FJYELGo-jZ-qU6m2OPnz2a6ed8GnYaw)

Alright, so this **mm** it looks like java script, interesting. 

I decided to view the **C2** stream which looks like the second stage vbs I found by viewing the excel manually it just looks... less obfuscated?

![624x385](https://lh5.googleusercontent.com/dg3eWZW2weaMQXwnMrFIzllBHYbSFUhnTb1gXqYDGjGQgKP4tB3FBkQkfpSX7ZEOTw-xEJgbhyRFwUAY9udPV9bY07qNGtayvUq9_pFeq9hLbgZUjU88bQwiT3y6xyKk23mL2zx9gVCF58kkPw)

The one I viewed looked very similar, after further inspection I realized this is the same script I saw before with less obfuscation. 

Alright, now lets view the final stream which is **D3**

![624x54](https://lh3.googleusercontent.com/InF-scKEW6nDTEzT9oEIfuRg20nEq7p5L7cHQEcjwBy9zS-AGt-r1VVhw256cZesud0GC-Pz0W8NMCW5nsYxtKSZKTZbNGQ-I7fMLGR4Rttmj6RkC1VNMIcnSfj2-TirSoiGLWnw2j-xQlcEYw)

As it looks like, this is the stack overflow implementation of the equation editor. The reason I assume this is because the long stream of bytes after the cmd shell string, also it looks very similar to the example POC given in the article I linked above regarding the CVE. I’ll dump this one as well.

![430x28](https://lh4.googleusercontent.com/rzqSvoKBNRPIorLK5Q5DZ291Q9gzjEsECpeF0LZEOvJVj7tknyysGJassFSACv1k7FgQmTNmk3pQXFVzbmKzy-8b9wXEjqGBybPmSkQZZ8wsd25oGtl1zu5J_LpR67kusDuWDU5r9kJHP0efcQ)

All this does is rename the mm script to be named v, and then it executes. 

So, lets summarize our findings:

1. By examining the excel we know that a macro exists within it that loads another macro, we’ll call them first_stage and second_stage respectively 
2. We also know that there inside the excel file, there are 3 objects embedded within them:
    1. **xx**
    2. **mm**
    3. **E3shellcode**

By examining the E3shellcode we can safely assume its meant to exploit the equation editor, which does not exist in my excel version but we can assume that it is true because if we look inside the any.run flow  
![624x127](https://lh5.googleusercontent.com/f5Z-OFhUwFEr-V8HQm5AqLv86Yg7C_l6mqxmhh8ovlm7RgzPk0TR7tzi4DxdjUfsrOmNZ7OboYrS_uK2_hNnnZySS-OOWyoXw1C73V6lssLIwvTE9EiCtDZ3rQe5MllqwBVxOWD5g0Mf-5y7BQ)

We can see that EQNEDT32.EXE is calling that exact same shellcode, which is supposed to execute **mm**.

![624x263](https://lh5.googleusercontent.com/f8dqQmFFPZF48k8LsXsSMp9Wlgloqs2pj5gWLYqrO3H9KJP5fE7zjE5qC9V8pgbTCKRyIJs15UT1vvVE5ptrZdIAUx4mLXY5B5DS8hEyaI1BJ-2hcfKZQOtbWlVButjNUXRgF9HM6A2GjY1LcQ)

All mm does is execute **xx** and since we have vba script **xx **in its kinda deobfsucated form, so this time I’m really forced to analyze it. But one question arises? Why is there an exploit embedded within this excel AND a malicious macro that directly invokes **xx.vbs**? The answer for this is simple, since Microsoft mitigated the exploit in newer office versions the exploit will not work and why count on that? The threat actor dealt with that problem by additionally including a macro that would run the malware anyway. 

**<span style="text-decoration:underline;">Dealing with long, obfuscated, annoying and blinding VBA:</span>**

We already kinda know that this VBA script is responsible for downloading a file off the internet and executing it but for the sake of practice, I wanted to try and debug this code myself. I started up by cleaning up trash code, giving variables meaningful names and finally cleaning up string fragmentation. 

The first two function called ase64Decode tse ase64Decode decodes base 64 code:

![624x702](https://lh4.googleusercontent.com/M-oWoltCdjd92Eo7B6-W8Q3QiCMYojOiiSQYJqeLLVy43qUaHSVKcBJbozkVZhDUI42PV3y2trtzV9LX6Cps7I3hL7wUqlcoYv62lAO2BkBvAR7oKBx9YPiQMLQ18gB5JHBeUG8WO9qKoNtv9A)

It’s easy to tell because of the name and because it accepts strings as parameters and additionally we see a call to both of these functions passing two strings with comments next to them:

![469x36](https://lh5.googleusercontent.com/likqT7av4w-T8i3idUdBmrFKlU01mBWGALmt_p8ZdFWod-3gSuTiq5lpzVvCl67aZnsEaVcnjnvMwhZWQglcyYywfLnrZORRfYqYSxZJo-kYUqvCb3hBrZHs_uwTFRaVncraE22KmVEmw9FjYA)

“filestring” and “linkstring”, huh I wonder...

![444x515](https://lh3.googleusercontent.com/yYvQxeDRHQsQy-CGaSOlJ8L3yOz_Pu5iHilwSRFRmxSVYI13MtMwhVIAXpakXrG6FTW_jqJOI8xysGFQ9BuBuPjiQ1huyd3rfmC9UI5Nj1mtWgYBspOd59BjYvd04KvHvnp56fbRGqaoui7A2g)

We found the payload location which will be downloaded, then the file would be saved in some location under name putty.exe, then a function called **KTx34hygf37it35hyr** which I renamed to **DownloadFile** downloads the payload to disk, how do I assume this? It’s literally documented within the macro.

![624x120](https://lh4.googleusercontent.com/vQ_9JTjOGQEs6tNCUH9ZLSNScI0d9XiPVqB0CCYsyYHyNDMxRGPIA152HxCPgkICtWImPJgry1h8JqU83LVqK0RqxrtpTb0ug-TSW9uWkgFEjeK2_d5OdUGKFAjVP_FqPoZ5vvo5xrdacUtO-g)

Then the file is saved into C:\ProgramData\ and executed:

![624x88](https://lh5.googleusercontent.com/qRjtZ5mlXFx23B9scCXXl_0WLeMcGy5EbmduH_c1voqcWdKo4NXcthB7hu5RDbRStlzKf665PfdACxjpCCROVfv-m0UDoSDBuNNHE_HXLun_fvpYJwR_aX670X3fcjUGGXTyoTOQ1ZFPtuaaOw)

Awesome! One problem tho, if we try to access to the location of the malware we get:

![624x128](https://lh6.googleusercontent.com/IX1HC0t9Zd0BeygS5Ag8VaWKzh54dTv-J-rEykPu90Wyje7P6_1R9cibnvfCC3ybaaSzYzRH8fyfQT-mt5bfNvzGgls6hv7Q7J1DsiBkf_et6na1SZXpaoZebK5HkkxcdNTwbuNVf6eNEIjBdw)

God damnit, we’ll have to find this file manually online! But I have one more trick up my sleeve, what If I just try to access the website as is, without trying to access the sub domain:

![483x287](https://lh4.googleusercontent.com/uZrKgFP_Wt3tMvL_6POskqTzulLIQUmonhX-vpYQxe8C60fPbiDgpyyuIqBO5EQXAJMm-c2Q7VfgHEeWY-OursX8FhU2ohcAENo4egTk0yqKa2skFTthy66BaQ5dzionhSqr6nMJFBE-AAoR9w)

It’s an open directory! Hosting files called **june11n** and **june11o**! I’m writing this as of date 2020 June 11<sup>th</sup>, so the dates match but the document was viewing now as uploaded at 10<sup>th</sup>! 

After downloading both files, I saw that june11n.exe is a zip file containing an email, and the other file was a regular exe file. I instantly assumed that this is the file that was missing holy.exe but further examining their hashes proved me wrong, but still it might be just packed differently but the final payload would be the same. For that I would need to further analyze the files.

***

## <span style="text-decoration:underline;">Initial Stagers Part 1 Summary:</span>

**Holy.exe** –

Status: Packed 

SHA-256 - 29503feaa5debe98241301a773875a564f67a4183ed681bc90b582824de6944c

[https://bazaar.abuse.ch/sample/29503feaa5debe98241301a773875a564f67a4183ed681bc90b582824de6944c/](https://bazaar.abuse.ch/sample/29503feaa5debe98241301a773875a564f67a4183ed681bc90b582824de6944c/)

[https://www.virustotal.com/gui/file/29503feaa5debe98241301a773875a564f67a4183ed681bc90b582824de6944c/community](https://www.virustotal.com/gui/file/29503feaa5debe98241301a773875a564f67a4183ed681bc90b582824de6944c/community)

[https://analyze.intezer.com/#/analyses/c5194671-cc42-4f93-bdd9-bc8e8b575bd3/sub/78d7877a-362a-46d1-a1ae-0381d7fbbc5b](https://analyze.intezer.com/#/analyses/c5194671-cc42-4f93-bdd9-bc8e8b575bd3/sub/78d7877a-362a-46d1-a1ae-0381d7fbbc5b)

**June11o.exe** – 

Status: Packed

SHA-256 - B539A1F17ED0C58D80F9088A7AC9985454C4922C0126FFBA86A2B0F5C42D9599 

[https://www.virustotal.com/gui/file/b539a1f17ed0c58d80f9088a7ac9985454c4922c0126ffba86a2b0f5c42d9599/detection](https://www.virustotal.com/gui/file/b539a1f17ed0c58d80f9088a7ac9985454c4922c0126ffba86a2b0f5c42d9599/detection)

[https://analyze.intezer.com/#/analyses/eeef3469-99ef-440f-8956-ccb288cb801d/sub/8e9324ce-a470-46ee-abd2-d655e657e5da](https://analyze.intezer.com/%23/analyses/eeef3469-99ef-440f-8956-ccb288cb801d/sub/8e9324ce-a470-46ee-abd2-d655e657e5da)

**boasteel.us** -

[https://whois.domaintools.com/boasteel.us](https://whois.domaintools.com/boasteel.us)

[https://urlhaus.abuse.ch/browse.php?search=boasteel](https://urlhaus.abuse.ch/browse.php?search=boasteel)

***

## <span style="text-decoration:underline;">Dealing with Delphi Packers</span>

In this section, I’ll be unpacking our malware. Dealing with SHA-256: 29503feaa5debe98241301a773875a564f67a4183ed681bc90b582824de6944c. I would not be performing a full malware analysis scenario, which means I would solely attempt to unpack the sample as is. 

**<span style="text-decoration:underline;">Static Analysis </span>**

**<span style="text-decoration:underline;">PEStudio Artifacts:</span>**

Entropy: 7.16, very high 

**PE Type**: PE32

**Compiler Type**: Borland Delphi

**Suspicious Resources:**

![624x183](https://lh3.googleusercontent.com/gUcQPkuhOGFTYkO6teARQ_mln7s81-SEir-i6kE4bQdqEyX6z5BCEZGEVwbBMSw8t8IBzAnX9Frfbv02q-fdGNh1iHqdX4NF3fe2gYP1Udd35hP48ou3geAp3I5541LtBThBw-w1aC5bjxe1wA)

![624x175](https://lh6.googleusercontent.com/Nn7uRfz4cNNZ7gY7ceou_5oCEiaNJCDi5Us-_r_a1jT0nXJWtAgTprmw-nFWzs1_NGw6icw19YqlvJNUPbsrA8gb2PVE7Hg7t1fbMfBjEthjpPjQ37EDxFX-FARtfTQDvQoz3-ykSeuv9Dw_cA)

![624x166](https://lh4.googleusercontent.com/PDW9rapXqouEg19mnHN7JaIf-ZN9C2LP0PCicL4uye10RwwJi0qUU5HnrfvWdIZmjUhSB2L67T2CgNHQ6r6z26B_8JzKV7bebSewmIO-oaWNsDckxDnb9oqp3_H_H63pJLt09N2dryLLrNBIRg)

**<span style="text-decoration:underline;">IDA Artifacts:</span>**

**Meaningful strings**: None

**Meaningful Imports**:

*   Resource Imports
*   VirtualAlloc
*   Create File
*   Write File
*   CreateThread

**<span style="text-decoration:underline;">Dynamic Analysis</span>**

I decide by setting a break point on:

*   All the resource APIs
*   VirtualAlloc
*   VirtualProtect
*   CreateThread
*   WriteFile
*   CreateFile
*   CreateProcessInternalW

Additionally, to deal with Anti-RE I load Syclla-Hide.

I immediately break inside **VirtualAlloc** but only manage to get the allocated memory on the second time I hit **VirtualAlloc**. I decide to set up an hardware breakpoint on the first block of memory allocated and to break the debugger when it's hit.

![512x487](https://lh3.googleusercontent.com/LdLRM-kMqdM0JQBajCuqrRTfLMjRAD2_g7Ipl3aOLYuamo_n8qHQbvsHz0ZHYhi6QgbEMK7afNiW_TI3eCtqmrhEYjKoq1G8KybqNCGUkG60uS4aysv4fjlOdKYtIIHe6VQsVKhB6Kqo49PIpw)

But nothing interesting happens, I continue execution and land on **LockResource** twice, which means I’m accessing two of the resources but for some reason I did not break on **FindResource** which should be run before. Finally, after a few useless calls that lead to no where I landed on **FindResourceA** which passed the handle returned from it to **LockResource** and finally I got a handle to resource **1033:**

![624x81](https://lh3.googleusercontent.com/ZyrRfvf6EnnKP4TwBz6tuBAFsyFTP3mBfXAk87lI0IPHABps3ikmCcRrYyYUYCbGRdW84N3u1ohcRo52a_QpKEWdW-m85pU3OvrZ-iGV6R377o1Hx_FQqKUTQ_hy2keVpBV9vAtKkv3YoZpXmw)

Then the resource is copied to a different location using **VirtualAlloc**:

![590x160](https://lh6.googleusercontent.com/1cfkQen1sX1-EQqrppMP2N-VIP6yuRZk3CtjgIUhPywvDpNjsUsSQ4P_TT4UydgOsJ7XWXxRxAnOTDXg-fruapr97VQ9bUhQqJsvX6UhmiV4ihGNlT9N0U-B11kQgKFI_vo3ZCvL4qPUhUNSlg)

I noticed the malware kept calling more resources again and again, without using strings to access the resource but with a unique ID and when I examined the resource section again I noticed over 200 resources that are trash:

![174x396](https://lh4.googleusercontent.com/GWT6Ta6ocfGtvXXwrKf9vyCFZDIicWLcCgndK3Jdly7VueS3t-YFTx4IfqP5vzHk2B4XM_WXzXzrxeq7c1oy3vMGHgoJUtuX1HHKPVHIGGzyaMoKQBo_5aiQWDV1y8emF4z8zHaSWUtwC8fO9w)

And the packer was just iterating through all of them. I thought that I can just stop debugging the APIs and see if I can somehow exit this loop. I returned from the call to **FindResource**, and returned back to the sample execution and then I thought that if I’d simply look up my location in IDA I could quickly analyze the loop but then I realized:

![624x262](https://lh5.googleusercontent.com/xq_yeis2zD3wyd63ORY8KYyUVC4bv3Z3DTBd7VFfJ6nPVkgk2bIRmOP_EOHTuU05Dwhf_eqLeYTHcpLzsAAb1JnA1v3TUCku7-AM6IZ5TGM67a5NHW29kdwyLU8FCSQ5NTA6HRFJn2Y_W8NbTA)

The sample decrypted itself, so I had to dump the PE and give it a look in IDA again:

![288x547](https://lh3.googleusercontent.com/lXpIZEKUR43J80DJikOXmnadRVWuGYpf7dp1kOQLJWnzyUxJUEjEJ37T2mnTqwIAq8HeocFef1hYLpvqphvJw2elVdLGEI_HYXFwY4zRt5pBOwUk5RKjTS9Nx3lZxsr48yJDOhrQUgMjjNaU_A)

This little function kept calling itself, and I could assume these four blocks simply call FindResource->LockResource->-VirtualAlloc-> memcpy, when I tried to access the functions there were calling this one and view them I got even more confused since the disassembly was completely obfuscated and trashed. I decided to take my chances, remove the breakpoints set on the resources functions and continue execution hoping that **VirtualProtect** would eventually be hit and attempt to change some shellcode or a copied .**text** section to executable. 

![624x120](https://lh6.googleusercontent.com/wRZ5NDnuUdj0IdJBQxJ-6FAb0bMa_2ZO1s6We8xqPBcQN_5Lsrro4bkiC9iuPdSWNAV56dSJ9FlkX0kfUhBRRAC-fao53IMcTrryLM8pbl7gljj-ZEolb2FZGdg6qmaMXj9l3siOQzVlsB8Tgg)

But instead I hit **CreateProcessInternalW**, this API was attempting to relaunch the malware in a suspended state:

![142x79](https://lh6.googleusercontent.com/pw56bqTq_kamigEw3XIzhEIdWT_djgox_wQpscWvZTW-Lf_fIujXFap9CltPNrSg0fwRxgDLSVww_8V-4pF6rKD5aVLi2IuUMlYsTwMcWcU3fp0Oq063ozidvmIv8KOikzpZN4YuNyU35m6Nvg)

![624x34](https://lh6.googleusercontent.com/11mGBG8M5JnEApDqICB1DyFnXd5feH-M-yVCdJMLTPT97IvYqmnSefY0DWvLJGEnr4tpMhI7G8s4hrpzNulyndV6Ga2g5Us35vz_6C5Ez9asnle92FwCNMH-4nPnOE_ZqsBBdsZM8oiKco6BxA)

I decided to return set the breakpoints again on **VirtualAlloc** and **WriteProcessMemory**. I hit run and crashed!

Alright at least I know where I’m crashing, I restarted the sample and waiting till I hit **CreateProcessInternal**. After I hit it, I returned out of the API and kept stepping until I hit a suspicious functions:

![624x78](https://lh6.googleusercontent.com/9bZT6uQG_5DXaFGwZFyJlSL0RI5Iw_Gm0l0dQxAfcpymaataKuASzI3J_UKhjKfy5JsawHD5cdOS5ZtjdrmXpQfOak5L1Ik3k-SC4H2AHzLnhrYhXzEQ0rXEI_2Ujt_WpyJfgEN48yQNnnkl6g)

I decided not to skip it and examined the registers and I noticed one of them was pointing to a string with the value **“PE”**. PE Is the value of the Signature filed inside the IMAGE_NT_HEADERS. I quickly examined where the register was pointing at:

![595x166](https://lh6.googleusercontent.com/6XZCB8Np4BLPnc3PpWH0djt8P8YkCfyCtXpcaT5m5M0VNnT9ve7BP61u8NSU92ohIEVHdPTJWBOF8nCDN3yKsjX8ueo4gU00RKJ4py7H6Op5HP6T2nZobvzf4MKB_2V1Xm2xjkae2-GqarkZyQ)

It’s a valid PE file! Alright so before I jump into my suspicious function I decide to dump this memory region:

![453x24](https://lh5.googleusercontent.com/U56CL3jZZTIZbJjo7evYgY1FM6kkY-UYrJtRs6HSbtEJxaMwphg-uKcGC_VuVBrmP6y8zsHUoE0PFKEGIDML2Ri4rwsSLzcI5tscFllJ3YVB_oMOJBb3_0dOKRMGsBImA71uVgwWvRiM7p3BuQ)

As we can see it is still not executable so perhaps its also unmapped:

![624x184](https://lh5.googleusercontent.com/Y90bkIXlcXjmgCCj-15o5M30Qmz0T_8Xpu5JAMst6rr3gfIY9-KU0sA8M0JFGdoQVLbN0TfmPn8BpTXbx23mUQmfAKG66LMpi0ivhWu9JMNqNFP1lhN4yQRXbZmkUyFVNfmpZlEZ-kh2e0nPgA)

The file is UPX packed! Alright so I can assume this is the second stage packing but before I confirm this I want to know where does my suspicious function lead.

![621x24](https://lh3.googleusercontent.com/LNzBzS8AEFlhHfYs-PKe9cwlT0Eqye_3PDuy0BfarRj_Nw-9lLEwoYnNhjZNoFokon8OizcY9ePq9s_VkeF-HkjCe5eokrrfIVAmAGlrBBv-gDhY3n_IZbASf1G5R8NeV8JAWroak3SbHuS79A)

Our function calls **ZwCreationSection**, which is usually for me the first indicator of process hollowing, it does so for the new process we spawned. Then on the function calls **ZwMapViewOfSection** on new section, next the malware calls **ZwMapViewOfSection** on itself, which means its going to map the **ImageBase.**

![223x115](https://lh4.googleusercontent.com/h_apKV1weiHIe0K9P3PUrLmvgaIqzIQD0r7gP1G3RsXcEVYZtPd9W7jbLsbhsooMJvDs7-9aNGmX9F3-VpVtT1InMgaISwfutAp8D-EhvkgDbdHopXYNswRaTKrfcLcoE0ZsG0cwXeAU6NupbQ) 

![170x18](https://lh4.googleusercontent.com/HXIE842OQc9RKnMB41ovSWpJSWU5f30NxBFxKP9Sm3Pgcn6TNEJFITDPWfOXUrspydljsKNY2znLkZl6gHfJ9aAEHyTZI6QByHwOGsvAWwIxTwYwJy0E_fHC2W-5ZxkLxw3R4Ke0FsZ39E4jnQ)

![624x18](https://lh3.googleusercontent.com/7eIZxMZyTWwH5BQ5tZ78w0j2sQRJIZUeUrCojWfyphXGyTp7z7zrtk1huTtFsTv4HpPcRwhY0E7o0LHbdCT9rHx8SeYPb6ot8MrxqW6F3cn4xuO8vRwuZuiJSE1IH59TymvZKuQ-U2dkFIZ-Yw)

Then the malware does something interesting:

![624x83](https://lh4.googleusercontent.com/EimFVo1wnyGFHQXmo2HLJXaZByxp_l-IPh6eFmvO2y8x5jf_lxUPe6nvgSlGdqpQh-RY3gmSPNhI4lH5tK8ivQe7wuALDqdgA5UMPn7JaeeYnOyCjcspSpk7yQ_9A2zjGZWx1Xxw4gybFM6wUw)

It enters a loop, where it begins to copy the new UPX file we dumped, byte by byte into the new section we created until eventually: 

![583x150](https://lh5.googleusercontent.com/O6dPZn9_cbjwzoUw-bscJ5hunwRwf52pKfmXSdr8sitRWamtyzXDDviDZAzVa5ZDih6XANsbsDKGi6wyah05JFeB5Az6ilTTkyFt35DSqHuhDzgRg97OTjAJqvSU_Co1DyXh-LTQoiAPiWJcFg)

The malware copies the entire new file into this section. After the copying is complete, the malware attempts to get the context of the main thread of the new sub process, how do I know this? First let’s look at the stack before the **GetThreadContext**:

![338x44](https://lh5.googleusercontent.com/G2onMxUojGoc7JWk_5KnMwZ12SJBUCaINmI8AQ2DQB7hI31MNypvVj1SbRidOzupVgV294cadS9NZr0NUPSyhYxpXfXGLsOaWn3jZdCWvWdlSBzYAAOLnldD9fLWbLJb-gLqiU0EHFITUlVdrQ)

![624x17](https://lh3.googleusercontent.com/Rv3QNZVcAVF9MrARdfy3XhJwfi9yWSVrGoSyp4j77pAbDIMooXjJ8J8chfP1eLC9AFQTdr0Fs7UK7B11HlnBuC9Cuv051i0Kr2u5nD1CsGnsdBhOC_terAmQdHuPwujKM6ZhUthFHxoERlirkA)

This thread handle belongs to PID **EF4**:

![380x46](https://lh3.googleusercontent.com/nTx2xNP9S9a7TJcmi793QRn04XAA6TJYdGwEx2P7EdFJVOpy_-0IQlEVK_Rpy_mb5ewmTONc73MP57rkTfp043DhjyQG7RBrN0Bdrg__p8Xg8__IDITJmGSZ04IkQTKcBgUAniO9cEu3EP8qDQ)

![157x77](https://lh4.googleusercontent.com/-THfH1PDfyboltHNDWeap5z99G9SiRCukVwc3lhHcNd7lMQTXZJ71O1RgycY8GLVXPAQ5HUMswbl-JeZgSIt5hlckYPyPastMDmg_RHYsSJb_0I3ql2IpWAKVtNWvYoxe1Ons3dN8OqSQy_Ciw)

It changes the process thread context to a new thread, which will execute the new file and invokes **NtResumeThread** to continue execution of the new subprocess:

![572x35](https://lh5.googleusercontent.com/qbEksvucweGZsxSKfrI12pcrL0MnMN7Q770sPDKFxxgRcQA_bRkbTJrRP1hdxjceLQDaqtnlt1gJMW3sU6BivUZcWaI-TIcyAqK5kdK0Wd8ROVw8SafMVO9XPhh0eR7tzy8yIjEXTGGAIicALQ)

Let’s prove this hypothesis. Let’s access the new sub process in process hacker and view its memory:

![624x32](https://lh6.googleusercontent.com/8JwWAXr6ZM5xOH10ZMpUOXsWhhgf-cq_EItOj5suB6wmyyIs7-GncCpII6-HRCcg1r-V3H1FihDe1XZ4psxIIT73AgKva0t5zgW73zvQFf8mSl5qE1qVrQc35Lac_3eG-XKmIOZ8dosR-HipWw)

And dump it.

Let’s examine our mapped PE: 

![624x190](https://lh3.googleusercontent.com/x8VcZElGtqUFVUyKbKTGJYsXpb3xMfUW-hnh6FVXRmVgfPZwDKdtQMkJosd93V_wxg-8Km_WTf1gKd5Rd7JWulkAUeK3o3DRt0wqex3kaN7rTWY-LWO75_b9vpMEqQSLxcwxu1XTFoXSNXlx5Q)

   
It matches our dumped PE exactly; the only difference is that its mapped to memory. So essentially what happened is very simple, our packed malware started a new sub process of itself, it mapped the original old **ImageBase** to memory and it mapped the other embedded **UPX** PE into memory. then it took the **UPX** PE memory and simply overwritten the new sub processes memory. after it completed it simply returns execution on the new PE. This is classic Process Hollowing. The PE seems unmapped so we can simply dump it.

I’ve used CFF Explorer to unpack it, upon viewing it seems it’s just a launcher. Its code section is small, containing two .text sections, The WinMain function is very linear. Usually there is a PE embedded within such a launcher that is executed in various techniques and if we examine the resource section

![624x125](https://lh3.googleusercontent.com/th0ytA4UCT8llcw6eJ1NIneBK_dCD4X9723vKuqQEp4pQzwIWTY-nceR-BPPQW79cm2b36dINO3n9cAETC8CH0GswD8OzCH5hx51OH4dFBZat3lVg_2sM_Ue9VE58Z1JsitoC-__Ti_FgX-CLw)

Well, this isn’t very stealthy… ahaha alright let’s dump this out.

![516x273](https://lh4.googleusercontent.com/uTFu2KJ4xKhH6ycqw5wToGh5HXP5takMF8e7H1jjqivyNa2AMpiDCV76KNEVOZRQTF1VKiVQvGr6uM2TdZJEEs8WYpRoShXJpLw8_EgvC0UW39cLS_uCBwZpiGGco5Oe84Y9uRjm-19yUMvXtw)

I suspect this is the final payload, as this a .NET file and the verdict on malware bazaar was Agent Tesla. Let’s look this file up in **VirusTotal and Intezer:**

![624x221](https://lh6.googleusercontent.com/xhyYXY0mA4Iq4PFY6JiP9M8Wn1_NTNbXwYIzjS4D0mX-Hj1cE0gnrcixV-4DFZB4Nr_BzWP9yF71h55uIYuX_EQdlZEmLS5vWM8teEeONo7YhOt-A9wcOr2TxBxR9YJI26-RGqqWYY3NrINUAA)

![624x255](https://lh3.googleusercontent.com/UjHWuDV1FEHtdxwzGvvs905_hUMDvm5g2azL1xVYbqO-wcRhZcfeVRpcD3ZxcL1uPcEGS-rX3377UBRnx1sIQjxVFSbt78IiquBlFQ694Pmcc0W_0lqaa9NDa-AxQ85Ig_HH0bnKBQhBNBZtOA)

The verdict is clear, it’s Agent Tesla packed with **ConfuserEx**. We can use de4dot to deobfuscate **ConfuserEx** and continue analysis but that is beyond the scope of this post.

I hope you guys enjoyed this one :slight_smile: 

<span style="text-decoration:underline;">https://analyze.intezer.com/?utm_campaign=website%20to%20community&utm_source=GetStarted%20#/analyses/f704c668-999c-4c7c-9941-9c19a8fb60de/sub/3f32622a-8f38-485a-9add-eea25f2beede</span>

*** 

## <span style="text-decoration:underline;">Unpacking Delphi PE Summary:</span>

**Packed SHA-256:**

29503FEAA5DEBE98241301A773875A564F67A4183ED681BC90B582824DE6944C

**Unpacked SHA-256(UPX PE):**

F3112BD51886FB705F33BEA0DDFA708AE6BABAB6F93162ABFDB931095EC6D116

**Unpacked UPX SHA-256:**

09A21333C61AA0DE5148E97AA238B8F8295E100891AA6A4F1F1108B27752FAD3

**Unpacked Resource SHA-256:**

c9b206aeded0795594a450b5717e65ba934c0dfab6005a8c696bd991ba96db15
