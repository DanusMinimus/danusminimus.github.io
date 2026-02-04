---
layout: post
title: Zero2Auto - Netwalker Walk through
categories: [Zero2Auto]
tags: [netwalker, ransomware, powershell, encryption, reverse-engineering, ida-pro]
---

## Preface
Recently, I’ve joined @[VK](https://twitter.com/VK_Intel) and @[0verflows](https://twitter.com/0verfl0w_) advanced malware analysis course called “[Zero2Auto](https://courses.zero2auto.com/)”. The first lesson was about algorithms in malware; compression, hashing and encryption. The first lesson was supplied with a PDF which is now released as a [post by Vitaly](https://zero2auto.com/2020/05/19/netwalker-re/) based on another post about the [Netwalker sample](https://blog.trendmicro.com/trendlabs-security-intelligence/netwalker-fileless-ransomware-injected-via-reflective-loading/). I was thinking on how I could practice this lesson, and I concluded that a simple thing I could do is expand upon these two posts as they were not detailed enough for most beginner reverse engineers. My main goal would be to prove the assumed findings in Vitaly’s post by expanding on the mechanisms detailed within it and to detail new findings that might relate to the lessons subject which is how to recognize known encoding algorithms and automate them.

***

### Required prior knowledge:

* X86 Assembly
* IDA Pro and x64dbg Knowledge
* Basic Powershell skills

*** 

### Dealing with the first stage PowerShell

The first thing we’ll do is download the sample referenced in both blog posts mentioned above:

SHA-256 hash of the sample: [f4656a9af30e98ed2103194f798fa00fd1686618e3e62fba6b15c9959135b7be](https://any.run/report/f4656a9af30e98ed2103194f798fa00fd1686618e3e62fba6b15c9959135b7be/ca44ad38-0e46-455e-8cfd-42fb53d41a1d)

This is a very long and obfuscated **PowerShell** script, it’s so long that I couldn't even load it into my VM’s PowerShell ISE without it crashing, so I decided that I can circumvent this by loading the script into Sublime(which is the best text editor on earth).

![624x112](https://lh4.googleusercontent.com/EOTOZ1svaSD7lwV9xwU5A7TvVxznQOwQRKQR20j-ftarzwQxUP7WM97i80xQV5yi6OoTvIxKCjKq9lvyIlnt_c6eO9cWGVNicPuRRAjBDvbL2-oT607U9pafljjm9FhC-W7DIga--4SY3HsZaQ)

The script might be long and scary but do not fear, for all we need to do is examine the first line of the code:

![624x17](https://lh3.googleusercontent.com/vu5keseZVPcwlDAp2N6UqND4pJlgalJ5_TGnS-PfHwiDfd-4G5jh6Z_GriNb8PbS4NV1D2rFFAgF4jridEak-COhgBn7LOwHmdLOJrhYdERKpqECUbVeu9grbIsCSurs5qJBTEJbD0BbtgKyZQ)

The first command “Invoke Expression” will simply run the command wrapped inside “$()”, within this statement we can see that the statement will perform base64 decoding. So to decode the first stage obfuscation, we can simply remove the Invoke expression command and pipe the entire decoded string to a text file using **“ Out-File -FilePath .\Process.txt”** and this would result the decode payload stored in Process.txt. The second stage payload is nothing but the same scary mess, but this time a long bytearray is being decrypted within a loop, it simply **XORs** each byte within that array with **0x47**.

Again, all I decide to do is to pipe the final product to a text file:

![580x29](https://lh6.googleusercontent.com/NQCSMjdA50yUtlDXqMJqtqEkq3HXhv7yWa8K2BD1nTW33FffNDpiWep8dSiVUfT1EOMCDLvRKwui1KosrELNxWI1Wj3MdFMogwKbV3mf4TFZ7H7PbCDLiVpLI0jTEFbNX-ngsaQsxihhNQEBuQ)

We reach the second stage payload, which contains 2 long bytearrays – which as stated within the blog posts are two DLL files representing **x86** and **x64** versions of the malware DLL that would be loaded into the memory of **explorer.exe**. I’m merely interested in the bytearray representing the x86 DLL. Using sublime text I can click **cntrl+shift+l** and click on the last line of the first byte array – I’ll copy this bytearray to another script file, then I’ll simply invoke a PowerShell command to write this byte array to a file. It’s worth noting that for some reason sublime text appended newline characters at the last character in each line so running this script wont work unless you replace all new line characters within the script.

![556x72](https://lh6.googleusercontent.com/a0y1L1uOOiYuaGiAxB4KhTB5-8_aX0ZywaLmd5ESOzqn2FcACIvzVo_ZLyignPv28SEUf3walasWR7Cifa_qF0sCn2EaRcvhMdy3mdlRekHuMZcg7_BqM9ofQZyw8lmkFfAoPRpQbbTJwH3njQ)

### Reversing the Netwalker x86 DLL:

Let’s throw the file in **PEBear** and see if we can find anything interesting:

![516x164](https://lh6.googleusercontent.com/pUz5WfrL3rEFQDWg2q7z2ORZbxFtsFkfY4jB_7pgmlTnz5HcsLR6H_ApAqlvIFaDilwvukt1zLUqjv8iZGmZygOtMjk_9D_WsxsEF9Z1z50ZUCF0ySPzZ0XUQYvq2aBQClJASCKMXP7WSZ0v9g)

What is this? Ah yes, do not worry the malware author corrupted the PE File header and replaced the “**MZ**” characters with the header with a word value **0xDEAD** (remember this).

What I’ll do, is replace this value using **HxD** to a proper PE header so we could examine it within IDA and Resource Hacker:

![624x214](https://lh6.googleusercontent.com/9RnnThTyTut3YMwnECB5STRu9U3382vDvc8NqLfJlSKsk-AKj743eSqo3al6n3xGHotOTmyDIk_yIl2lKkRiTWAs2RdWhvwFAv6Sb1G2_wZQJ54J4qjODfIdCeJ6aaLdI36wRD8Wge-u-YKFRw)

Much better!

Usually, upon reaching this point I would perform basic static analysis by examining the file’s strings, view any anomalies within its header and examine its resources. Then I would perform basic dynamic analysis by running the sample and monitor it, but we must remember our initial goal – we must expand upon Vitaly’s findings and find any worthwhile material we can explore ourselves. So first let’s examine Vitaly’s first mention of **CRC32**:

![508x677](https://lh4.googleusercontent.com/cUyOK1_FlPyr84iYOE2_gj1xey_4gfmuzN_gFD7PPhGQO5itWAkER084DexKR6Hz5sspMnTL5DORQh4vQ27s-fjHl02sIXqbEoU0iiN8eMPJJKwxDB-EKFv9u-zCa_nHW5mLe5nmmQM39O_zEg)

How did Vitaly know this is indeed a **CRC32** hashing algorithm? Well lets start by utilizing the **KANA**L plugin within **PEid**, I’ll load the malware into **PEid** and launch the **KANAL** tool:

![372x172](https://lh5.googleusercontent.com/p89cQrPeYnyKXiOJrQp_RASIt5rfmdSwlawDMVddFTFpvgzZq2uzTvs3451Ist71S-oEKm3KJF5lxCKMEBgnNt_GVh6txl90EfzrviMgQG_E8UDxTeGnxsWeRr6ql7zYDdSDgQjR20JXhBlOIw)

![377x60](https://lh3.googleusercontent.com/fEtt-eGNrD7ajzy4Du7Id5OzV4tPsDA9L-QEN7ODaKJdSF7T7BzIdHgUWpNnoiDxAbEvdzbG_rQVugAD0j3FQpnxgUQjB9bDxnb3Ou3_z8ohwPudpTaKq1PilAtLHQ-M-9mUnYwyxTIjItErtw)

As we can see, KANAL recognizes that there is a reference for the CRC32 algorithm within a lot of locations but what exactly did it find there? Let’s jump to **0x1000424F**

![221x72](https://lh5.googleusercontent.com/5L7t9maEfVrngmUJcEPqsm67HD7jekRO2fr0MXsFa3Y0xFN55yLT8zxBIA6_TgGW84rH8ziugOSF3gSyrEoJhKQHyJGVSHf57aPXADBA7V9TJ92pSP9xubnghXNEK0AmtVtMWoIny3CDBFxsSQ)

What is this constant? Let’s google it:

![402x290](https://lh4.googleusercontent.com/EEOneQQtEMk57zVoEnTsGgIxi-h4R-J9FisFoOmEZsO425stm_lG3ZgyXyHHOpO9cOOksvnugDRGn5qp85BMPFxDVMz5rGR3E5BjdzHtDv18p1knjFevDxxt8g4vMk8Edp7vomF8oYRUwmCEhQ)

Aha, alright – even if one would view how [crc2 checksum is calculated](https://stackoverflow.com/questions/2587766/how-is-a-crc32-checksum-calculated) one could quickly see the recognizable division flow at **0x1000421C**.

![173x475](https://lh6.googleusercontent.com/0jbri_HRbZ0MINsCZI3qnm7PtLc-uC7PF8Gz65JPzN1wzP7K3156GLWd2iiKaWkrRGTBZOqNhSyQ7Aq32HDbSvFHLs31yUeXXiZbB5ZWRHg82G5QXo7gGF8pEUHdu6-SloI7VLsn)

When dynamically analyzing the file, at location **0x10001A59** one can see that the value **0x3e006b7a** is resolved as **FindResourceA**.

![695x39](https://lh5.googleusercontent.com/d-033UXzXY_olhuNsZQwwxANLtjxkIwsHFI3lHUPIsd8rC_DsKPHz9D77zn4DT382ZVeWhTxeGmkgHRRXvdBmv4xv846ygDLEBuTI37eDD__cvjcspvCVZh8xehljbOHAURR8GO-gn2LbDya5Q)

![442x54](https://lh5.googleusercontent.com/a9ZwDbwKSy-aNp_j2FhDaKtUHWx-w8f9T8wOpv80E_3lp50cl4S4ExufxZJKgxrLtpkfLJgr9pEVVyDrWvIWEHRcxNsp-Xze9e-JfG9fQ9TwENVivQwycC-MtYbHrhkBGsGW4UsRhKb6PDwiSQ)

## How NetWalker utilizes **PE** Header stomping to break analysis

Let’s examine the following assumption made in Vitaly’s blog:

![624x301](https://lh3.googleusercontent.com/ZxuX_XXC9zAaZmBlzQS8HeIMX6rMfnNh6nFma8PTbXIRz2n5Q76Bb_wT-eevoLYH7Y9l2IckNuxmMfegx-Ot0l3hpMGJxXlnhMGTgZGYkH09tGCXtZY3Rc59aDlyQi-iEbRomnZQ0YTOY3T-Sg)

At location **0x1000A0B0** one can find the **API** resolving function:

![521x188](https://lh3.googleusercontent.com/OhxYLbLVQE9QNjsuoEi1cNzMo9hKcWGP7a22PwshIWot2j-5XwaaKfMrEUMGG-ypimXUwkhFjoACjhNQraLNUsSKARQg7bcp8dcsy5SfOry1FZnQ0Uabd3OnG96DEHyBQ841H3YLlNNBftY8Hw)

So, I assumed Vitaly is referencing the content with **sub_1003710**

![285x445](https://lh6.googleusercontent.com/Z0IyihUGenARa-KNaFpMD5QbxYlzs-H3R1XRTfqj2f-Uu6aIbAyB34KG88wHcejicAlFsW0qq8kvACY9uRaD6kyNaGEaUCvgkcjqu0vbBWtzdOQRcC4R9_OWnsBkp6L4wToLEr2urdhobzNqAA)

And indeed, he was, how ever by simply breaking on this location and running the sample it would crash. I decided to attempt to understand why this was happening. First attempted to skip the call at **0x1000371E** which I renamed to **func_checkValidHeader** but I this function crashed the sample every time with a access violation exception, so lets take a look at it.

![204x265](https://lh5.googleusercontent.com/iYI6nkUIrA_vTQ0cmsOPh_-YaYXo1_naDa2D9v5Z0x1q-PXrGN0sR-KT7zgVk_1IXcofALA-L6Unll0JVv9R7j8ATV-UKd8_RgS-58IUBe3B6M8sA1rC8_uQrfxcYMiKRmSZNgs6Eg14q9jQvg)

First, it loads the offset of the current function into **EAX** and **ANDs** it with **0xFFFF000**. It would then begin to iterate through a loop, subtracting **0x800** from **EAX** and attempting to locate the value **0xDEAD** within the address referenced in **EAX**. Sounds familiar? Sure does – as we recall the DLL PE header was stomped with **0xDEAD**, this code routine is attempting to validate that no one tampered with the sample.

![514x63](https://lh6.googleusercontent.com/uo6iwAWB-77p7_MmjkwavY25ChdFXyPXIA_4k9PMv5h-YSRY6K7uLOdBdBiZy9pHakbJ9pdXHA-U8EXlXLAoI1rcheuggrzl4YeXFxK5bJraCrj1ifl-SQ3mGyyXow1xsZ0sQH3-SwgLfrc0wQ)

Since I modified the header, the sample would get stuck in an infinite loop until **EAX** would point to an invalid memory location resulting an access violation exception – to add to this finding if we go to address **0x1000372D** the sample attempts to fix the stomped header using the value returned by **func_checkValidHeader** which should point to the base address of the DLL. It would then replace **0xDEAD** with “MZ” thus fixing the header.

![601x153](https://lh3.googleusercontent.com/C--jk7Zt7CZ4fcmpVqnYmLEE9BQb6Fm8FCvJaRAzM0LdAOCJ_LOz6SFeMAUw970kgtSh7ftcxccMM4LdQSqtgqveITQfl3xjvBcVXEa0C9bM02WvFsLdp8VcIxvTmBt-jdJzYxbnW0ViQ9ULUg)

![126x22](https://lh6.googleusercontent.com/QZpjoACueRwd42U2ZHXUlKWUIz17swuzMG9LnNVTwuEPnUkWAAB9UE4A53cn9SG0Eo2N4TkQtX-lbYPW0R4r47j9_h1QoWRDNBe3E7YXApkXRYsG_i14KnU7hHKS-Hs6VrQNcKGk6Ctri6GN1g)

To quickly solve this issue, I just patched the binary by removing the header patcher and that solved the problem.

![617x210](https://lh4.googleusercontent.com/3NjOQVKtRGSk5ic_G9_Td_KxcoftcpQ0XG7hQA595z3qQmYIoN3pR8MFaNTzbMpfuOYaJJYpAFYY8_EtQ90gAGuNff13hX3oE3eZg1xcF9QmCzQggyMJWQ13z9BzvJb3SUWJUoXqoLKeeG6Ceg)

We can indeed verify Vitay’s findings after this as the sample doesn’t crash. First the sample loads the resource, locks it.

![285x422](https://lh3.googleusercontent.com/tpaOt8TrIdfEwlc1hPTRGqvhSwFaMHTMGjarCg3hzH7zZ381JPrYAMhZ-dyAZxQCTM4R8LzUm1HvHHGGqAC1faweBX9PZYNI5JCGi07zzcU5yo6tnW1e-dO_LnBvHD5IFkNFQl0WZRdmZHw_Xg)

The malware then loads the resources size and allocates a buffer within the heap to load the resource into it using **memcpy**.

![384x302](https://lh4.googleusercontent.com/IR-UbTSfZzI9Iq7yo8rdYPG1U5QeDF-xWCVaZeBb_9XlJ3Rvilci4CY-ppIj1ckB_3W9br1UR6HMX9kD1gIqcgwqPMmX7MA9RFLmTpul-XpP9Xq0kBR-ZQTFFbcg3CqTG3WCwSOzMg1yA7yelw)

The malware then loads the key length and the key itself and saves it to a stack variable

![398x96](https://lh4.googleusercontent.com/neGwURGqU3_-pIYlrZn_ac-UaxKjeFoK14YPjAbrz9XjEZAgJMDg6yJ0tyrO6dYmGglul4oMXTWvYD03KDOOWlY8NqOOVNH6wCDEZhS5NzNhOA2PaQ-2xe1lRyHkrjhHvUAmEU1tGRJPSnnOmw)

Copied key:

![169x19](https://lh4.googleusercontent.com/bUK9oi9wN5uxKQjoDedod83aB74QQyIUAUbzLpS8VhBs2hldtxO6Fco7ahfaunRABxWbtnOr-u-v2zdYr7KxsVhYUckAHpnRF7DNB6lwLNA0FZsAuhwcQgizcANOtM-40UmIeEMQYw5LrMXO6A)

Afterwards the key value, size and a pointer to the heap buffer containing the resource are saved and pushed into a function I renamed **func_RC4Decrypt**:

![267x340](https://lh4.googleusercontent.com/2jVDQBu6zraniw6Lc4owIx4-EdgkQCwJ45vTnfpe7_LrWUeBN8Hunp-ApAgJvwiaOF60Tg879O5xZ8BtzqCsmUaVlbpE_QPvUyNuOkCjWfqaM4xmC5SRh1GIbcW80G7y_CdCiOqTzf59bkt9Nw)

Vitaly assumes this within the blog:

![624x240](https://lh4.googleusercontent.com/XafOvScrEX1k1afq0uNlTYK-1uxt7typr5VFX3wzWyIFYboTuUfbh31y_UcH21Vssw7OOXrwNjk_nziWI8AmZfXye84RAnDQjNA8ONfexia3EZwxHVAs8sHcZQwVD2hn_LHL3WJGJ7K0iGBgFA)

If one followed the lesson in the course carefully one knows that one of the recognizable features of **RC4 KSA** is a loop flow iterating **256** times:

![548x173](https://lh4.googleusercontent.com/kEXaVxBDUppMf589n9ZRuBL1afW-jU9gVrJ_bTYgAzOhGFINyu_mqM10-wa1-y1dKjKFsj-NjQ6mf1R1bpsWerwsvSsxQJ27nnz0LKqiJqBZ17vtWVp8wk89KIo-vpBm_3gCFqyLV3BwU52_PA)

and if we examine the function located at **0x10009210** we can confirm an example for this at **0x10009281** and at **0x100092CF**:

![287x253](https://lh4.googleusercontent.com/H87LEytZhPV3AK0quKZA4wVYQL81j3NwEPiMlHIvwapmgvQCTelNb8z-5-ZX76CdGfL3If8d8IPwksi4WOFK8A6hsirtVXJ9WAHSrF3E9zWKIiQ5cobi4_DJLRhEKWAil9zFD-YnE1KVTRtVuA)

**EBP** is being loaded with **256** as a preparation of the second **KSA** iteration:

![263x46](https://lh4.googleusercontent.com/HDStPEynpO8WJx2HYMumgeVqR5b6BnMi6UmMefHSegBvZfXL1EO3BhXmsHGppDTemFbqPVXvhNxgBQLLexMWHp9QHlT3Bvvn8V4SoZB9OjeRKxTiIoLKuKve-DtBUrNiac7-bvhkiPq0mU1yWg)

![262x210](https://lh4.googleusercontent.com/m5BbCQrW2PN3eCHBvM7cXTkWQQnYVBpnn8DX0Io5SmSBCFJLwCffo5xvfGPc8M9PGBYsUzlSLw8TCrv0bD5TK6eRRSo0pf6DY58meU7W7951hXk-6LwQVA2tLb-EHREX7hJ3Dz2X3pfD2EdlVQ)

***

### Decryption process as seen within the debugger:

![624x47](https://lh5.googleusercontent.com/leh9SK5nN0s1kveeaYzXp_pQYmomFy2m4swmyvvX5_fKdCpGxP_fPjjYASER6nouhJjhEadRmYYrLBml03apfktYeXch2sJLHqbxPT2TS6HZyJqBY1BFmd6-esC9JMzFJKEaq0iK5e1YQB_4kg)

Size(red), key(blue), resource(rest)

![594x165](https://lh6.googleusercontent.com/S70peBwNPJJsrHbTVO-BFM0YpnqO8yitLFVimbDoO-ESv7MNbp9EvcpokLrN1mP9bzKEKR51Bgz9vLY_o99JGDb2LTd8UMg8isoO6Ta765QZxSM2uVYZ27R4eXQTqR0rL0HEQyTyDzdg-gj_Lg)

After the function decryption is finished:

![596x165](https://lh4.googleusercontent.com/wH5W0ddtCaHsWmeA99iyYmBvjWkyNdqaUqb21AnKItjnhlckVDKYJELL3x8N5rDXzH-8fYLdt9WjdwhQSe1H1dVW1OIOrmuFMVLyJ6QuQgdb-q1SnQCZQvobAmsbcwH4PfSi4lZXpqHevhbkjA)

Finally, at location **0x10003832** the sample restores the Netwalker header back to **0xDEAD**.

![329x143](https://lh4.googleusercontent.com/t8uejiBXZO-CzQ2UHp51q-Vnn9VGh5-k67eO-qzUT6Q_QaB4S0xyMK4xUtm2XDRiFrLIE4Rbg7Kpt5v8z255JBO4jraUhGY2Av_SXle8JLsc1rT3ZsKN3ktz1C4tliAxOkcBdLCG_tybPlqAbg)

I wonder if **0xDEAD** is a cross binary constant across all Netwalker samples ;)

Sources:

https://zero2auto.com/2020/05/19/netwalker-re/

https://blog.trendmicro.com/trendlabs-security-intelligence/netwalker-fileless-ransomware-injected-via-reflective-loading/

https://any.run/report/f4656a9af30e98ed2103194f798fa00fd1686618e3e62fba6b15c9959135b7be/ca44ad38-0e46-455e-8cfd-42fb53d41a1d
