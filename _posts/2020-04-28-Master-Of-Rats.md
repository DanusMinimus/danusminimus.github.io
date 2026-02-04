---
layout: post
title: Master of Rats
categories: [Malware Analysis]
tags: [quasar-rat, mozi-botnet, csharp, network-protocol, master-of-puppets, malware-tracking]
---

# **Preface**
One day I was skimming through [abuse.ch](https://urlhaus.abuse.ch/browse/). This website collects user submitted malicious or suspicious URLs and I&#39;ve stumbled through something very interesting. I saw that a user that goes by the twitter handle [@Gandylyan1](https://twitter.com/Gandylyan1) is uploading huge amounts of daily samples of the same malware variant called **Mozi** ([You can read about it here](https://blog.netlab.360.com/mozi-another-botnet-using-dht/)). This botnet is an IoT P2P botnet that seems to spread like crazy. Gandy is uploading samples as I write this article and there are currently 24,709 IPs uploaded to abuse.ch, and it seems gandy is the only one uploading them. 

The malware is very interesting and not too complicated too understand. In the basic gist it spreads through IoT devices using known exploits and brute forcing attacks, if it manages to connect to an IoT device it starts an http service on that device and uploads itself to a random port and hosts the sample on the IoT device&#39;s IP address. This peer scans and attacks the network and when it takes over another device this newly infected device will receive the mozi sample from the previously infected peer. This is a parasite. I was so excited about this that I&#39;ve decided to set off with a simple goal in mind – to build a tracker for this botnet. 

But alas my Linux knowledge can be summed up with the fact that I know that &quot; **ls -la**&quot; should print the contents of a directory. But the idea of creating a malware tracking tool was eating me up day and night. After a short search I&#39;ve stumbled upon the following tool created by [Intezer](https://intezer.com/?utm_campaign=Brand%20ads&amp;utm_source=Google%20Ads&amp;utm_content=Brand&amp;utm_term=%2Bintezer&amp;utm_campaign=0719+RLSA&amp;utm_source=adwords&amp;utm_medium=ppc&amp;hsa_acc=6013738437&amp;hsa_cam=9281250457&amp;hsa_grp=94310168375&amp;hsa_ad=417088374601&amp;hsa_src=g&amp;hsa_tgt=aud-877982745810:kwd-765886859489&amp;hsa_kw=%2Bintezer&amp;hsa_mt=b&amp;hsa_net=adwords&amp;hsa_ver=3&amp;gclid=CjwKCAjwv4_1BRAhEiwAtMDLsu6-ldJnan200Cnv305zpq8XjqohXDXEJnVf-i9srcKGV46nfLgwJBoC5ioQAvD_BwE). This tool is found here [https://github.com/intezer/MoP](https://github.com/intezer/MoP). This tool is a small python project allows a researcher to fake an infected malware client by simulating an OS environment. All the researcher must do is reverse a malware network protocol. No honeypots, no virtual machines no nothing. In addition, Master of Puppets allows a researcher to track multiple malware families in one single run, all just with a python interface! Sounds easy right? I though the same. I looked up any open source malware tools on GitHub and found [Quasar](https://github.com/quasar/QuasarRAT), which is an open source RAT which is used by people for malicious purposes. This is a great way to learn about malware, reversing open source malware and just understanding how everything works under the hood from the networking side. Great candidate for our little experiment! And so, I have set of to become the Master of RATs.

![624x351](https://lh4.googleusercontent.com/Qwx_s0RIMeJivhbBg3rp2ZUB62Zwkm7OP4UayF03krIeaoRHq6IH3eWTisMajLtKIyBuratlwbnAjXkRH4TW88w0TRO5Lt2f3DbmZeNYDyiWDMCtg0RO6cuN0DoTEae_dfD-_2FZnFMf1fDFMQ)


**Required Knowledge:**

1. Basic knowledge of Wireshark
2. Basic knowledge of programming
3. Intermediate knowledge in python
4. Basic knowledge in C#

**Required Tools:**

1. VMWare
2. Visual Studio Community
3. Python 3.8
4. Sublime Text Editor 3
5. Dnspy
6. De4dot
7. Brain

**Setting up some goals:**

Before we even think about using Intezers tool we must reverse Quasar. What are we looking for?

1. We want to understand how a Quasar client connects to a server
2. We want to understand how Quasar constructs the messages it sends to the sever
3. We want to understand if there is an encryption/decryption process for processing messages
4. We also want to understand how the server processes client messages (which is possible in this case since we hold the source code for Quasar)

***

## **Learning to read C#**

We load up the downloaded Quasar source into Visual Studio 2019 Community which can be downloaded for free [here](https://visualstudio.microsoft.com/vs/community/) and we are greeted with this:

![306x181](https://lh4.googleusercontent.com/bjp6tSwJwPiOsWYY7bzO0XvOSGMYeLh8iysNeCkIJW910CiElu5OuIyI84wA5TfsecCRAbjjfEuvQ8DitTVth_2ht7TH-raUOmyI_87kvhntZvsvWqJMv0NfcvFRJ4bGmL52-auAF23Q7p489w)

We are interested in the client code, how ever just for reference and as we will be dealing with the others – Common contains various utilities and Server contains the code for the Server application. All C# programs start with **Program.cs** as far as I managed to figure out so we will start there as well, let&#39;s open **Quasar.Client** and find **Program.CS**.

![624x431](https://lh4.googleusercontent.com/fB6kQj3U5RffBJoA1kJrZgbIBD9kztd_KtB51rzNZAkcrfBmAXfHuaSXPag4_MtdEjyZIwD-0_snoReesrML9MdvOuDXis1U8YPtfRxT9dftOkioOWMCFnO1E2fHhpgDWIf7ZsbpXpmICn8GRQ)

Ah, I wish this was C ☹. Anyway, we can see a few things of interest in this little statement if we right click **QuasarClient** and then click on go to implementation we will drop on where most of the juice happens in this code.

![624x163](https://lh3.googleusercontent.com/_Gjm24tlu7jJQN3AqEJUiM7Y5cm8FDzTiENF217HNg9qshIoyrUOGxjGx3Ovj_B2nNlfiPRzF7YtrXA_f-Xb92bJ29DwQ6VFRlENyPBOj1Oh0Ck2wecTKj7IphYagEOtXnbCrEEONfsy4xILdQ)

We&#39;ll start by explaining the **QuasarClient** class which inherits from the **Client** Class. The job of this class is to manage all the events that that accrue within the client, It has a special function to handle the registration of the bot ( **OnClientState** ), it has a function to handle message reading events ( **OnClientRead** ) and failing events ( **OnClientFail** ).

![624x513](https://lh6.googleusercontent.com/TpyG68yezt5zEX26kicbLaeIF1zNudan5r5D5bo4GzgyBr8JNVex92IfHPGy7IMnbJfTWLJVGaxQ5VyGQ7cOy7NgYwaHWJ7czGsRcpmUAgwIrfZFy5QEwaXfiMCk4WQa1AyIaJh4tIJ7Wqp9qg)

The **OnClientState** function attempts to send an identification packet to the server. To explore how this message is created we can access the constructor of the **ClientIdentification** Class

![399x626](https://lh4.googleusercontent.com/1h1QlQZp0279hEllqI4IyRc45Tqmy5McUa_Fh2VPLhXG-uX3obCk95sFcE4uLMxFOpa3k5qSpzHLHtZxYt9TIZ_TAm5HqZbu6tEb0faGIhXrNLhCM2F5Ud6LmQPQ1nm8RqkSrsTZ6ZMpiAyBSA)

Everything here seems rather normal except for these Proto declarations.

Let&#39;s get back to the **Program**. **cs** code block and look at **ConnectClient.Connect**

![511x347](https://lh6.googleusercontent.com/cPAMPKZOE6VZsK6MHZYtwpSZoWu_gyPqjZuWtwW3kjDUJ1gW_8EMekdlLFW27yyr7IIeNSq94GIZh_jEGAt9CgFV8pcW6Nbjxvqq8m9fQd9gj_k1lm74vNyh9GxxyQKf5Gzo74Z1ThIrH4Zibw)

Which leads us back to the base **Client** class

![624x303](https://lh6.googleusercontent.com/3JXPrGpkRyuY4pU4TgTrQWyCrmKlzRviDvt0JEpddCpH5vAQI8nqvaE-GwmZWdi8WTi22_kj7_IR78-zAiSAFs93p0EcfyNP8M5ZQTL5UenodvWUwkQKWtegzrI2FkkZ82nxgJcYPRnGBRr3Mw)

Alright! Seems like this the answer to our first goal! It seems that to initiate a connection the client first establishes an SSL Stream, and then it looks like some sort of validation is happening using **ValidateServerCertificate** callback and **AuthnticateAsClient**. Let&#39;s leave these for now as we are just mapping how the code works. Now what happens next? If we access **OnClientState** through the **Client** base class that would lead us to the event handler itself, to find the actual function that triggers on this even we must go to **QuasarClient.cs** and access the reimplementation of the function through there (Gosh I hate OOP). As we saw before, The **OnClientState** function triggers the **client.Send** function


![580x489](https://lh5.googleusercontent.com/v9az9crWWvnm_P_Lo14DC9yIHL-jKrFoONuNeFzDImBhW4CN1ItPz1LjJwKz-H_vqtv_jSD-wNYWOuGAb_s2k3PSxmrOrejt5uOobcG1DhDpotXdBk1eg2jc5rrgxNsbtK1claKG4_kzL4qYIg)

I&#39;ll be honest I don&#39;t know C# but I&#39;m working with my instincts here (much like in assembly haha) and the only thing of value that I see here is **ProcessSendBuffers** so let&#39;s access that and see if it would yield us any results.

![445x526](https://lh4.googleusercontent.com/W4qorHICKbAVM7TOWoqPS0pT9kLJ6GZ5iv2bHrbvLL8_AmVBWfBNEnAGNhxGppHwowLMm7i4BDftmRAUShCOrUqM2tXB3gBz7DxYyy1UMh7VFbrRxbhvGuNElU1gRUm8LLQEY4jQghTMPbOH4w)

Again, using the same strategy as before, lets access **SafeSendMessage** and see where it takes us.

![624x412](https://lh5.googleusercontent.com/B7oWhv_HaDMjR-whqmKqZwDsYlh3H21pvpwXVfT3O81iBhDTg2FiSD50wNE0HcDcq3yd-1UWwBVF136192xl-kw_y3Yr2-vog-3jJHnaYr9efBbA6ldPsFnpah2XRjxZqdVp00QzG7kdkSdl-g)

Alright, now we don&#39;t want to access **OnClientWrite** as I fear it won&#39;t take us where we want but instead lets access **WriteMessage** which is located within the class called **PayloadWriter**.

![624x349](https://lh6.googleusercontent.com/uB5JRcYrIjuCSWj6eWMVvyNb7YEYuVgYEKg9MIAwujvKdPUwj0RUVnl19mHrftOACvib2L35hssJI02MBnRh2ByXNBl8AxPzBdEdYOvTm4FfbRejhgID7lQxREtAAZzY1vy5C98z_cTLwW9a8w)

Jackpot. This function writes a serialized message(I&#39;ll explain what serialization is, don&#39;t worry) to the SSL stream! So, let&#39;s make a small diagram detailing our findings:

![209x602](https://lh6.googleusercontent.com/usEtJ0jMfqEgKr5zIzZvuhh-ov8wrompiZyHWPuG-WmB2HKYjnD4QTYdLY8qbU06s2d3UsdZfLAuKVIH5qj9RvgbRVmQrmE69rEDV4MRvgq-kOd4n6oC1-_DOU1WR-Hz6_ZFmf44XVKeAYcCbg)

Alright, this is obviously very shallow and incomplete and as we progress with our dynamic analysis, we could expand on this diagram so let&#39;s compile this Quasar Project on Release settings in Visual Studio and move this entire thing to a virtual machine and start playing around with it.

***

## **Building and analyzing a sample:**

After you compile Quasar and moved it to an isolated environment you can start the Quasar.

![624x422](https://lh4.googleusercontent.com/0kEoD51R9Gt51LT04Zj98hLYb5IdgOgJoF2zHwqIMeon42LaEVLxOlzooHG6GV4eoKiaYUfSS5hdMPG4fkPIpuhCydICwCz4N9xGsy8jc2hItCqAYzgPQKXZhWqEKXB1a_WBbBPFXE-v0it0OA)

This screen should pop up, and this is actually very important. This pop message is a builder for the **X509** Certificate which is responsible for creating a valid SSL stream between the client and the server. Quasar will generate a **X509** cert and bind this cert to all generated clients. You can learn more about SSL here: [https://www.youtube.com/watch?v=iQsKdtjwtYI](https://www.youtube.com/watch?v=iQsKdtjwtYI)

After you generate a certificate is time to build a sample, after generating the certification click on the **Builder** and you should be promoted with the builder menu, the most important part is this one:

![624x522](https://lh3.googleusercontent.com/MasV7fFhE_dKFQk8fYZiaPjQhP0VlT94ajpC5wnuKM2oBG5KmecmHdIaQ9nD0P5ux5VpH0lB1j1JrFzjxoXPHa9leCMFhUEeZ7yovqdnyqEcO_dWu8dbTkhAskFFehH-VQPP1_5AQsNLNOlluw)

I have 2 IPs here; one is the loop back address and the other is the local IP of this Virtual Machine. I would suggest binding the client to the IP of the current virtual machine as it would be possible to emulate connections to the server from the current virtual machine and outside through the host (since the host is also a member within the VMWare local network). You can use any port you like but I&#39;ve used port **27015** because Minecraft. After you built the client you should see it within the current directory of the installed Quasar client. Let&#39;s take it out and open it in **dnspy** which is a **.NET decompiler**

![624x242](https://lh4.googleusercontent.com/F_6zPAYSLoLsLYe0hfaugLU3Ji3uff_2jKEObK_sq6oFQxCshsydyLtckNgo3hrdvTjPbf4HCztg7JHBO4y_zu74tW2MFYi9Croqy_tE23NAaDoKMXVp3EXA76Zk11BUuMK3m6OfPDqPS3hExg)

But we are met with this garbage, but do not worry! We can use **de4dot** which a .NET binary de-obfuscator so let&#39;s run it and we should be met with a clean Quasar client:

![624x292](https://lh5.googleusercontent.com/qcAMJ53v7_BvtOBI0wD_VjJ-BkNB0UpQKGF1CWguPIRMvF6mYEBGdB8CeKzNpn_nWj6Xc38bsxKdud0Euo-NJGYJ6g6JRyNpQx5iF21DMJYvjM13ug3XCUKZz45ivxpnUun22K9DA8oc44QfMw)

So what you can see here, is although our Quasar client is clean the symbols are gone but don&#39;t worry as we hold the full source so let&#39;s start debugging. we just want to see if our diagram is correct so lets click start and place a break point on the entry point (I highly suggest renaming these functions and class names according to the source but since I&#39;ve debugged this so many times I already know this know block like my right hand). PLEASE MAKE SURE QUASAR SERVER IS RUNNING. We&#39;ll encounter our first problem with in **Class0.smethod\_3()** which is the second Initialization method:

![340x121](https://lh5.googleusercontent.com/J2hbthquImhedf6f9Esiwn7WGlwyFFvCEf4UVN31oRGRJkgDh6VrJ0I97WwJ_61jEDhMgUf8b7htSu1LunL1eTmDnOw7AjAXqLoHVcZdDubHaJfVi-4ONj3s9XlMLfshYY4JjxSL5qQQQxf3vg)

It will not return **True** , thus causing the client not to execute and exit. but why?! Let&#39;s look inside our source code:

![624x131](https://lh6.googleusercontent.com/fOChdOHZEgf6QhZKP_XjC6gzE0cu2UXYnv729EbvEpvFreURpcapXC-dSxsboj5gPFM-mJ98BzTcX3XXm_UbaHJVTYKIkGKkVuJsIbwppwdCKaeLu0oHF5frGd227TEsL_DJRYLCxmWSDL8EPg)

This if statement which is marked in red, install and connects our client to the server by returning **true** after initializing but it seems it will not execute as the **current path** the client is running from is not equal to the **install path**. To understand what I mean let&#39;s go back to the builder:

![624x516](https://lh4.googleusercontent.com/2btsAECRMbak4HRiYWpcOP-wrOZvj_ngXsLgR2juoPYxWXs6lpk9RCzW1PrufnzjbbJVNJ7OE7Bt3KJY8gMvTyYwIOZ_qypQxv1AsNEJJi_H_JCG98WJCRBWciJa90KY-1mQ8MzUX1NulKGESw)

So, this code block checks if the client is currently being run from inside **Appdata\Romaing** (In this specific case) if its not there it would execute the following code:

![478x125](https://lh3.googleusercontent.com/0k8p8GxT_3uAU-1GL6ZrSOADNQfeSXXUanPVskqvUXK1A_9uLFfLCrIk89UusFc9Iy_U0Jg2QD6TkIxaM3Kg0vbCcNTIjf4nf9_DMcan0vp84ZqVCSSIyCsw2W4t7K8b8YUaScEqSS0hKm8VgA)

This code handles two problems, one is that the client has detected that another instance of Quasar is running because it detected the same **mutex** that was used within the current client and the other is to Install the client into the computer but adding persistence, killing and deleting the current file and process and relaunching it after it has been moved to our designated install folder. You can enter the **Install** method and read for yourself as the code is very documented. This is very cool cause it gives a researcher a real insight in how malware might be developed. With this knowledge in mind let&#39;s do two things:

1. Update our diagram
2. Move our client to the designated install directory and start it from there

![624x638](https://lh6.googleusercontent.com/XbSsZLySHdkaMwq3rQXJwSHwtkXmhr7ZyrYRkFTEes7Gt_5bO0rbtkyOX9oKsEk2JzJXBerOdnm8xGm1G5WKXa854aI-aM6oeP-s-BDApbqNQoWhHbFUU6oywidFtW7PZOJYNYzc3mkALgmB6Q)

Let&#39;s debug our client from our preferred install directory and see what happens, remember to make sure the quasar server is running in addition I would like to fire up Wireshark to monitor the network(This are my own settings, and the IP address and Port will be different on your machine):

![621x23](https://lh4.googleusercontent.com/6cayQ43j7YiGeowBUT2GzU71Rbb7ltbHhLrYjUR-0By2OolZ-Vf0kPs1aQ7Ylm_1DcEdq7UttVXEC1scH2KuZrdvwI-HqXTWS7DMsfJdob3ecMGdMN9y542Tbet7AYEDcySfzaUSPo5W-JaT_Q)

I&#39;ll restart the client from the **Appdata\Roaming** directory and jump straight into the **Client.Connect** function:

![624x46](https://lh6.googleusercontent.com/3winTBHVcbDNJV1ZO1mngbTE5YEUDOm7lvpN9VzL8ZZUxiagnPmA3R45pNY1WfBLlcsm6li89m5kZSWbY22fVaPHvzQeWr1iamEcWBspVZKJnrfQ5GlY8pV8cxUwLjFaJFmojv6HG1KiKNGeWQ) 

This time we hit exactly where we want **(Pro tip, you can right click dnspy objects and change their names but hitting Edit method, after hitting enter to confirm the change it would send you inside the edited function – to go back, press backspace).**

![624x97](https://lh5.googleusercontent.com/ZQ70JmLOGLxUcQR0BY_6DTwX5SA9iG1VZCWdGMu34uNfj9LF6HOBwlIIaA8roSolNAwXtcAg61VY7K1ywQpyZDLObRSqY2Yowyo6_Vw77l-iE6hXzGfdWKaaTuzNhvXfhuQ760juSKu86HCl8Q)

We have three places that are of value here, first is the **RemoteCertificationValidationCallBack** which would validate the certification received from the server, the stream reading function and **OnClientState** function that as stated before should send us to the **QuasarClient** registration handler.

![624x297](https://lh3.googleusercontent.com/g9AW0XsQ_5VG4CgQqfkYnSIB4X7OhIdVSl3urJ7HZFrFitor668IDoUojJS6UjC3kA_cpMmDylPQWvSA9scgCW_yzI-z-cF15_B-sl2SNEEwA4ztmqz_wByiEdmz5TeFr4qhVpfWQKLL3q-iqw)

So **socket.Connect** function should connect me to the server successfully and initiate the first TCP handshake :

![624x29](https://lh4.googleusercontent.com/_nVXKrt_soEgiZYFF57rfCSRw3CY1tVWR7mK3vJ30PvWp7QfTeqrRHTkoQAyX6LCQg2afzDNctv_Stl4nsq9LdJke_Gf8L84dHcSPS4NDltbEyDYgsDOcmQbcBoQRkdy9TGxhGqTjAt3hozvIQ)

![624x297](RackMultipart20200427-4-nb5son_html_9f1831d338e48753.png)Next I want to examine what happens when we execute line **287**.

![624x87](https://lh6.googleusercontent.com/2xVH4HvN2wIu3viO9P8LD6uUo-Pt6XqD0BI5JXTouqt7PGjQsbUvV_x0awza97s_2qRuMmgZlTMMqWPlUWrlV0kGRv9h2gPxGNQPa1CI7uGomptJj-_s9j9N76X5PpmbxCjGa2baf7xJ4viNrg)

This is an SSL handshake, but what happened is that the server passed its **X509** cert to the client and the client approved this certification, and this is happening inside **RemoteCertificationValidationCallBack**. Let&#39;s examine how it looks inside the source code

![624x101](https://lh5.googleusercontent.com/HUQZJLpF13m5kgFCNo7iGzDusRpedbBTwm35dlRapd_hojLvbbyyp_-e79kcuXp8NaTpZunHbL4dmXh616jFJgAA3xx_iOQMv05F6e6CDmdgxYsEdXH0SUay4Y0JHwOtwLIJbagTileJYGklug)

As you can see within the # **else** statement which happens when the binary is compiled with debug mode off, there is a function that checks if the clients and the servers certificate match. But look at what happens within the debug mode, it just returns true and because this happens on the client side... our client emulator can do the same to initiate a valid SSL communication with the server. Let&#39;s keep this in mind and continue. What happens next is a bit tricky, in line **290** inside the **Client** Class, **OnClientState** would be called but because it is called from the **Client** class the event registration function would hit and not the event handler function.

![624x164](https://lh4.googleusercontent.com/JDzXo-4xolAmLdhfrZOHXCiUQdryPXVaztFOtBAfj-xICmZ5Y7MUHUQmFo6gIavmsTq_zsyPrzYGAK4diVhujKKdU3Li6vgfsjh-nTi7vXGSU9YjUXBp2DsVbIdEZtZOCQ4TPwO5AwJK5Ky5RA)

We must find the **QuasarClient** class manually and from there navigate to the **OnClientState**** function**(I advise the reader to read this a few times and to play around with the source code to fully understand what this means as this is very important to understand how the client behaves, and having the source code is just a privilege to expand on our researching and coding skills). &quot;But Danus! How will we find it in this mess of unnamed functions? &quot; The answer to that is very simple, let us return to**Class0 **which is** Program.cs:**

![624x114](https://lh5.googleusercontent.com/p674Wu51WcQI_lfnEcRfUNQMV9JqTE5iTzgGW-pShVGMUD5koE8wAXNqBW7-ihA_qTkLZq2UT4nd1m0Tm-PcQiNgYIBbFjQB_WYF2HAx1TOEiNOusLku2DgZpN1qBh8auTUuxsi5IyjAeV_ubw)

![427x215](https://lh3.googleusercontent.com/BzVVFEhmDVxT_eOmF2FEaSqAYuD3Q5FBsW2u_lYKUDP7BLUQ-pTCmnzXdkFz6DCnY5WgPtM1YSOOQDVRCMEkW7gAitSivcl-tiIZhJeeGIWiAoVTapbypXhNMv6BuUguboCaT3l9nYxR7dM6NA)

So **Gclass27** is **QuasarClient** , lets rename it so it would be easier to navigate to it, then we&#39;ll access this class by double clicking it and try to find **OnClientState** Manually.

![624x406](https://0x00sec.s3.amazonaws.com/original/2X/c/cdc06de0621729514ee87eec2ad0efa61e26e449.png)

Here it is, let&#39;s place a breakpoint on line **79** , and set a breakpoint inside the **PayloadWriter WriteBytes** function we found earlier which is located inside dnspy under **Stream1, method02**. On line 79 **Class18** is created, and then passed into the send function. Class 18 is called **ClientIdentification** within the source code of Quasar:

![624x406](https://lh4.googleusercontent.com/DfyeA0y3IoaTpbrUvr7VKCJT3KvGcufsB_aubRnTGlaYNtWwCK2DX2Xiw0ZAsX7kvI1eZgHVcEesHk7dkRdTs7wPDO1lsvc9s_bZdH7Z5SS4Ou3pcfQ0D7S1TD4OnC9RAEofgPq15Eu_Ni3b1Q)

Which is the message constructor and if we continue the execution up until the payload writer, we can see the contents of this message:

![624x420](https://lh3.googleusercontent.com/YWRLXasRCOZWhzhYbZYx9gKiGthhP8n7SI_KSQJ5BBTCnE5B6TLFAZ0_YbGFPX2WHimNV1IQLF5D31BvjXxhBtAkxeel3J6m67aCZ37-wfv7SGsD2iVBykqLtUQtmdGO9wHSa2wK50U7k-T1MA)

![624x381](https://lh5.googleusercontent.com/NRLopRJ2-rF6KNTqgrGdLO-CM6vcG7owULG8lRq5KnCYqA0THjlMgd0S1EU4mCF9wTzeYCa2pNy96izyJ0iVsi_rI3auWl6tkn6glweMuHk3S3hp4pWD7CsVNgGpiC4v3B0o5KaJp4YxX4zc8w)

And we just intercepted the entire message. Easy. But how ever I do want to note something strange, there are only 14 members inside the **ClientIdentification** class but here inside the debugged message there are 28?

![502x101](https://lh5.googleusercontent.com/9vgAaMFirDbEPUcGSQ2tDLAtVnUGgRH7jQq_xVEV9dj0qLCdnXrENo4yVUaaBzJRHMVQYQ5z6wlURkZcokkUY0NEN8U7noEIsmW-AMb_HQhjwF5zAznuWzG5pAROu-yuIVHXQLAUmSLGYnBjXw)

In addition, in line 38 the message gets copied into a stream and **serialized** then the length of the message is sent and then the raw bytes returned from the **serializer** function are sent. What in the hell is a **Serializer** and why are there **28** items in the message protocol when there should be only **14**? Also look at the contents of the message after it gets serialized. First let&#39;s debug the program until line **40** and view the contents of the **array** variable by right click it and then clicking on **show memory window**.

![624x223](https://lh4.googleusercontent.com/1mUnHF2DbZ7v0PKOyiSspmU3leQKDEkvgW8K2m8yG7_lPpaBRoi2jqpk6jEMJVWERMuUZCIT5wl7dZNBN2et9Y8Lthgyj7cR8tXGE0rezSBWcWbItambsgSJJ6Fs1MaBSXjfeCaSDclDmV_V9Q)

One can recognize some the message text but there are so many extra bytes here that just don&#39;t make sense at all.

***

## **Message Serialization and Google Protocol Buffer**

So, to save the reader the time it took me to fully understand what the hell is going on here, serialization is a method to compress a message size and make its processing more efficient. Many types of serialization exist, one can compress a message to a json format or an XML format and send it to a socket, the receiving side would de-serialize the message on a pre agreed protocol. You can learn about serialization here: [https://www.youtube.com/watch?v=uGYZn6xk-hA](https://www.youtube.com/watch?v=uGYZn6xk-hA)

Quasar uses something Protobuf which is developed by google, Protobuf Is also a message serializer (which I hope you know after watching the video I&#39;ve sent above)

Alright, this brings as to these **ProtoMembers** we saw earlier:

![403x557](https://lh5.googleusercontent.com/0T4kahaBl-IhziQ7-yIAxcqIUDd74ZHAHmzpLK2el-dg04loSxSK01CO2UA8g9nRPl1qA-bRWV5CFDnwoPNUHj_56yQ6uzIhXomjD6KYX7FyeuY0y7EuOUpMC6V39WXOGigfDiyb_vs7NNRw8g)

This C# class is defined under the **ProtoContract** which lets the compiler know that upon generating these 14 members, to generate a google **protobuf** message with 14 members. That&#39;s why for each class member there is a proto member. Now the protobuf protocol compresses each type (int32, int64, string) differently and that&#39;s why we see a lot of strange bytes in our message. This essentially answers goals 1-3 we set up before.

It&#39;s clear what we must do next:

1. Create a python script that initiates an SSL connection
2. Validate the connection between the client and the server
3. Generate a protobuf message to match exactly the message that is sent by our generated client.

Important note on number 3, We can safely assume that the only important parts of the message are the **Tag** , **Signature** and **EncryptionKey** members and from the tests I&#39;ve conducted they must remain static and match the generated client exactly. Which makes sense as **Signature** and **EncrpytionKey** are the members which contain information from the **X509** Certificate.

***

## **Learning to write Pythonic Protobuf messages**

So luckily for us, Google has made a programming interface for creating protobuf message with python and a tutorial which can be found here:[https://developers.google.com/protocol-buffers/docs/pythontutorial](https://developers.google.com/protocol-buffers/docs/pythontutorial)

I&#39;ve crafted a message and script that initiates an SSL connection to our server:

![286x365](https://lh6.googleusercontent.com/hDc86-Pf09HgR4YV5pwzH2FSSs5I7pDAjL6baFU-GIsfAn4V_WfS-1KTvwqKR-ZKuzzEVSVFFNztxOdvc__NCdGQmLDTEoS3jTROFeyp-kEAOEJ4bFMvcTE7VJIKi5FG9RoMBzrLOfsZ6IFJ2A)

First the message is rather simple, it contains 14 members which match the exact same members in the Quasar client. One can follow the tutorial I&#39;ve linked on how to compile this message.

![527x240](https://lh5.googleusercontent.com/UKkSM_z1_RZvBksXi_KJz65oXJxSxf7dEnWbaZjkan_NwPxQbPMddFkkZ6oZ_VcGB2NO0WmurNaFQUTPFSw1MZNISRIMZlusdnisw8FhICL34RYyvh7lMeQtH7PdBDD9_-1SknZGVSaK0l9H2w)

This little script creates an SSL socket and deals with the verification problem, by always returning 1 on each certificate returned. This is cheating but hey, we are hackers :3

![462x64](https://lh4.googleusercontent.com/awk8QKwcdkqYpEnUPw6B7BRiJcVZuKENiCwM6zi3qWeR8f1QrzrTvwWB7mWOLOoTe2pvyyGgXIWkxmOb5gB0Lr3hIntvCfNPKPlXbxY73wmf8LBNhA45heynOYJfmOUZBngKo330e9YNEwmMjw)

Now to check our code, lets generate a message and serialize it. Remember that it must be appended with 4 little endian bytes that would represent the total size of our message.

![418x45](https://lh6.googleusercontent.com/CFeG0EmhCm2cSfKrB0UT3PsJfmwI_HQh-WS1evORYrzwyXXrT1hFdne5Bv_wGpkocIXNW_0Cjbk0yIpgL4BDYsoozg_aXh9VIUnlpmmhCsDuEun5x-I8CNOKLEa79b5vfvqQzy2-ExoEXMoCyA)

![440x90](https://lh3.googleusercontent.com/KF40T81N2JOe4cqZzjeGgPtUdJ-kc6J2dW1Zf0fBbb1Zw2I8A9uJb1A2L87b6cORXAmsjVkz3XUpJJ01qfbFGc-3p4HIq5aVT9m47yFbcFadbXL7eLslVX1cMFAZikqHlOchLBKVYprbngmVBg)

So, I&#39;ve created a little function that would return exactly that and prefix it to the serialized message, now let&#39;s generate our message, serialize and print it and see if it matches to what Quasar generates.

![323x25](https://lh6.googleusercontent.com/YHOFYmZp5y55eLR5IEZVnPcE8YguAvQepZ7kPVSjCVgI8x5wCfkO2I_YfBYxOmjLMFnd-h0S13er9ZhUbA6Ldg1max06BtWETaFdWtOnI0LL5edOqZDocnurFRoXZyLZyKUEKZbk9xu13eiDvw)

![378x22](https://lh5.googleusercontent.com/uywBG8r4A4jc9LaXabWGxBLf2NhC17P3MNahPF8BHWLH5KYjmsA5wURjurjl1CGDinouBjZJcBmcL85Cx71IeV8LaZCzfJWR-erJUnOgcbjrL9hfmLbH5zhvbxyMr8hjN7LIPkPEguT5folj2w)

As you can see, there is a problem with the first bytes. The message generated by python matches exactly (ignore the prefixed size 0xdf 0x02 0x00 0x00) to the C# message besides the first three bytes. **0x0A, 0xCF, 0x05.** What are these prefixed bytes? I didn&#39;t know so I&#39;ve asked on stack overflow:

[https://stackoverflow.com/questions/61412249/protobuf-net-unrecognized-stream-prefix/61425656#61425656](https://stackoverflow.com/questions/61412249/protobuf-net-unrecognized-stream-prefix/61425656%2361425656%20)

After a small research, I determined that this prefix can only be the length of the message. As the rest of the python generated message matches the C# generated python message exactly. In addition, if we edit the variables the Quasar Client message is sending, this first prefix field would chance as we increase the length of our message. For example – lets debug our Quasar client instance again and break on the Payload Writer function at line 36 just before the message is sent and edit the message contents by adding an arbitrary amount of **&#39;A&#39;** characters to one of the fields.

![624x105](https://lh6.googleusercontent.com/CbO2IGbFu4FDeNh3BgfqL5Qw4bQFgcsMGhW1M-FAPjYnl2ULgcJLDCL_FzMS4pqw0Iv5r1e4dc5WaolpNAfmqfUpQNQ1V6cT_El1a8kma4AtsudtCVzxViAWUd6Ha2qkZcUiy6w5VlyoP2vZ4g)

![548x108](https://lh6.googleusercontent.com/UMh2_yAZPmkNJa4GzEQEooShHbzNA-I6SJ51z62H_E6d-IjecdaXeA2YZyl5qG91_NDIir6RSxc29DnYmvQvPGnqKEo1SchRjcr-4CyMF1-k3B5qpXc2JkUkJ9Z_tABDg-4IwStGDW5AEP_MQw)

As you can see, the message was changed and from 0xcf the second byte turned to 0xf5. The difference between 0xF5-0xCF is 38 decimal which is the exact amount of &#39;A&#39; chars I&#39;ve added. To craft this kind of integer length encoder we must first understand how this byte sequence is encoded and luckily for us Google won&#39;t keep that a secret:

[https://developers.google.com/protocol-buffers/docs/encoding](https://developers.google.com/protocol-buffers/docs/encoding) and in addition this was explained to me in length in the answer here [https://stackoverflow.com/questions/61412249/protobuf-net-unrecognized-stream-prefix/61425656#61425656](https://stackoverflow.com/questions/61412249/protobuf-net-unrecognized-stream-prefix/61425656%2361425656%20)

I won&#39;t go into detail explaining how this entire thing works, I&#39;ve did that for my own self-interest (and I advise you do that as well, [@marcgravell](https://twitter.com/marcgravel) on stack overflow explained it perfectly) but understanding this interferes with our main goal. Do we have to reimplement the entire thing from scratch when Google Protobuf is open source? The answer to that is no.

![624x33](https://lh6.googleusercontent.com/GJqEIzclJnWdn1Jj5IItz7RHLg2ByJrfSDDYpxbdn32HIEt5n-J8gnfwotMd7MzgWAnN1MjEtqJF_88nUqRNPoKz7ViRyjfbw-b3xuz3TiI3euqs1yxJKebnEJj7wNff-rUM3sESpHoggoDoUg)

![363x261](https://lh6.googleusercontent.com/Otc6mWcx_fAJ1QE0lCnbrmzm-eUHZyYMf0262v84z5BGEY__qbJC5Jk9mhHpVWK-w35sf5QMYzvEsj6W-_TC5FXhlzogKR59WXR3DvRtp1YfPUvnNEAPdF39TNDFwngYGwUFyHRHRrliDjdQTA)

In these two code blocks, I&#39;m serializing a message and passing its length into a function I&#39;ve &quot;Borrowed&quot; [from the encoder script on protobuf GitHub](https://github.com/protocolbuffers/protobuf/blob/master/python/google/protobuf/internal/encoder.py). I&#39;ve edited it a little bit by making it always prepend the **0x0A** byte to final result which represents the field number and field type. Since this message is prefixed to our generated message its always going to be of type 2 ( **length prefix** ) and field number 1 this combo will always yield byte **0x0A**.

![289x18](https://lh6.googleusercontent.com/IF4yAAEfTxJIgZJq8GJQNZYpQh-5WPXZMv78340J0xPcxHVBoIusl2NhKq83REv-ryIm5hUzlmGffnoTuZEJ28Dk5KtJErdntuFZEsIochAeKzjD44iMukcqB--ITNdhn2zN0HD6IWtFQEIdPw)

Boom. Now our pythonic message matches the C# Quasar generated message.
Now, on to the connection part:

![289x134](https://lh6.googleusercontent.com/pGqQSaLtzS8Xt-DDl9bhz-jAxpBGtOBITE_Pt_z10-cVJwvl7oIltKb0ti-xIHpVaccaMCrN8b6WDSdgBciJ0LREmsXz3TaSa5Rt69-BE39xlAxXDi0rpdC0atffSMM4F8CMRpwB7fAdXumJlg)

This little code block will connect to our server and perform a handshake with it. Which will trigger the server to pass its **X509** certification, then our verification function will trigger:

![442x78](https://lh4.googleusercontent.com/VtPDZb0TmyANU9lBoLvCUdqjhp5qlIpBvZ4SQeNcetgz6BNHhQ3HG0JOPc7qKaDFBX70R_iVRraOe3Z7JHAdvP__4kMSziNyQqNe23DP2F2NHZYzKvLyqirJ2lirfQJO_4sbxw_2MLsqWQ2deA)

Which will always return True. Then we&#39;re going to write our serialized message, prefixed with the length of the message + the length of the message serialized using this code block:

![470x134](https://lh5.googleusercontent.com/K8_A5wMsCUcWkSX7I4_ZtvNXQWLlCB1luM3Oh3_TIINt6ZAJ7sIGlNo-ifZMCBLLJrA4KRNE8XOAK4PUqmoKFPxcKbbNcHUQJPErI3XzIfFz29UjnxRsrLB_r5i7T8K2iAs_IblpPsjvV-XrSQ)

First at line 97, I serialize the message. Then I append a prefix which represents the size of the message to the message. Finally, I calculate the entire size of the message including the prefix, convert the size to little endian byte format and append that as well to the serialized message. The message is sent over to the server and we can view this inside Wireshark:

![624x92](https://lh4.googleusercontent.com/l6YHUarKXpP1j6QI-CkpEFdBEOyS5zHrjk0R9tJzM08hl5I5tBZPlCIn1PVByTSOFXSarhBF5eJEMGYf9HaeA0r-6aV4th_pMCkyJ1IAqsHQlG4ZrUCR0QriJn9vpxs9FUaOZ8XiDqL3u9aNVA)

Amazing! and if we check our quasar server, we can see that:

![624x73](https://lh6.googleusercontent.com/L13jALva-CskIBrBiraWE454KGxT21zxuXlCevA2Kh0np2dYLNkjH2g1Pz1rqcPoCKCBb4hz_CkaWKEUkLBIYtM5gDilZY9GNKJ-1o8cy-L-XX-83jDp3RkpWQ5uRhhDj-8P23dt-jYSO3XgNg)

I&#39;ve sent a custom message that was registered with no problem! Now I headed out to learn how Intezers tool works so I could turn my little test python script to a full Quasar RAT tracker.

The entire script can be found here https://github.com/DanusMinimus/Master-Of-RATs/blob/master/test_script.py

***

## **Creating a Quasar Plugin for Intezers Master of Puppets**

This part is a bit trickier as Intezer is not actively developing this tool, but I would go to length as to explain how this tool works and save the hard work for you as I do believe this tool holds great value for tracking malware.

Master of Puppets is a collection of python scripts which allow to emulate a whole blown OS environment thus, saving the researcher the time and resources needed to set up honey pots or virtual machines. Now the cool thing is that we don&#39;t need to set up a whole blown OS + Kernel and a boot loader as most malware would want to interact with specific user mode applications and the file system and those are easy to emulate! How does it work? Master of Puppets comes packaged with utilities scripts and handlers that emulate and deal with basic functions to handle the malware such as connect, register, send, receive, and loop. The only job the researcher has is to create plugins for specific malware and the tool will handle the rest of the problems. I would like to walk the reader through the code and together create a diagram to help developers understand how this tool works. First download [https://github.com/intezer/MoP](https://github.com/intezer/MoP) and my custom plugins quasar.py, targets.yaml, utils.py and bring our also previously created proto class from here [https://github.com/DanusMinimus/Master-Of-RATs](https://github.com/DanusMinimus/Master-Of-RATs)

Next please follow the installation guide here [https://intezer.github.io/MoP/docs/html/index.html](https://intezer.github.io/MoP/docs/html/index.html)

If you are using sublime text editor, you can click on **Project-\&gt;Add Folder to project-\&gt;navigate to the downloaded tool** and this should open a comfortable view of the project.

![624x472](https://lh5.googleusercontent.com/HLjpp-MGOh3cnulI_5aDxiKTC0DRdncKMIRKc7lxzwjUGV796CxHil4ojCdc9OeT4gkLP0HcVKBfeAad0DJanNdvAMi7hWwtixfxcHioWerB_hCKrysm6sB2DuJEaIAatej57cqj)

Please open **orchestrator.py** and remove the first line &quot;#!/usr/bin/env python3.6&quot; as it causes the tool not to run on any version other than python 3.6. so, this is our main script for this tool. Scroll to line 46, as you can see this a command parser and in our case we&#39;ll be using the option described in line 53 – **targets-config** which allows the tool to connect to multiple clients, so please open **targets.yaml** and remove the hashes with in it.

![486x142](https://lh5.googleusercontent.com/5aIS8iwwlmDJMQvy6MxRUR7qzVplw76ZZaDpB4ORWWYbG9nS6cEQflM1ZibGP0IgL8_1xZGWCw2FU-3Xf7ZthF3LU6g2rsjIPs07MWoEVzO8kN2r5f_upn6VZzhaPhcEY8sgy9YY)

I&#39;ve already set it up to run with my instance of quasar but you can keep yours as is for now as we haven&#39;t set up our plugin yet. This tool sets up multiple targets, by specifying the IP, Port and the plugin for the target.

If we go back to the main script:

![624x182](https://lh5.googleusercontent.com/Gw9nEZ5s4uE5ubDIvecsg_FlG0JoXLGz9xU3g0xu0OH2sYUVoFcgsIIiCDUjL2CN2HekRWqZGDzoO43v-0Qsv9Cbyg0gmf3KHtdcs50KHruQYJ_d0yQA87d9qXBdFRbKBkQc3VC-)

What happens is at line 54, the **targets.yaml** file is parsed, and the contents of it are extracted. Then function connect\_targets() is called:

![612x78](https://lh5.googleusercontent.com/VHDs3ZblIYqYDBZY-CYC6iClMwYcQEetbuC8XGkm6wX80FPUF0b8qlxG56BbVTNnxgFRm_7Ja-mq1hxCVHSovLZypge5NO5C-ciuBJATxP_3lKoijElqBsTRAFwu_lkmz5azx35D)
Which for each target found in the targets.yaml file executes the connect() function:

![624x215](https://lh4.googleusercontent.com/z6EtZ28XZzB0IFxm4z107AdcdqghboIZCR1eIL7aeXuFq1IFkkvQfUJPDdqbFnxlrBizyzBNwyzOsCh-38bByFhERJyk78yqSkHbxvYJxrpGBX8vCj3pyxmcql_PDmUKBno9Z48H)

The connect function is found at line 22, it starts a thread with a callback function called \_connect(), it passes the ip, port and the plugin into this function. \_connect is called at line 27. First it imports a plugin from the plugin folder, then at line 29 it uses the plugin constructor to connect to the rat and then uses the plugins connect, register and loop functions. All the plugins that are created extend all the properties from the puppet\_rat.py class. So, let&#39;s open it up:

![624x524](https://lh5.googleusercontent.com/lYeP5D3Q5doUnqBr5vQBrBN9VZ04dQnU1EpNi7JX5OvXuXPsRbvDCVBrcgfnuemHezfFRbDnqB5YsnQalRovAshgZZilLdVG9RA9S_hhQlUTA0pLUx4JSVpXSzcA_SF7naAgY-L4)

At line 16, we can see the constructor for this class which as we saw before at the main script is triggered in line 29. Here at line 16 the constructor sets up various client attributes. The ip, port, the fake process id, the logger which is used to log all events and the conn variable which is used to represent a socket. The most important functions are:

1. **connect** which is supposed to implement a connection to the server from the client.

2. **register** which is supposed to implement the registration function

3. **loop** which is supposed to mimic the infinite loop between the client which is waiting for orders from the server client

It&#39;s clear what we must bring from our test script. First lets start with the easiest thing, let&#39;s edit the targets.yaml file to match our needs. An example can be found in the images above. Next please download [quasar.py and clientidentity\_pb2.py](https://github.com/DanusMinimus/Master-Of-RATs) from my repo linked above and place them inside the plugins folder.

Now I&#39;m going to review my plugin code line by line:

![624x307](https://lh5.googleusercontent.com/VkCaj_Sxyk6edMYcTOesc4t7WB4XdQnMz8B58tTlt-r94iqwQuf-zW8g7eAGzg7kilRpaMdjPtv6kZJztnxVYkRq1yDUyZ78yQLcZhDGjJr2gsdC0GTwL9h9Cs_RH6_d6z0O2thv)

First let&#39;s discuss the constructor which as you can see extends the PuppetRat class, thus inheriting all its properties. I&#39;ve added the message members so they can be edited on the fly. The only messages that a user has to set by himself are the **Tag**, **EncryptionKey** and the **Signature** as these interchange between client to client. In addition, I&#39;ve added a protobuf message member called message which can be seen in line 70. It creates an uninitialized Quasar message.

![507x494](https://lh4.googleusercontent.com/IAgQVxiE89lclTZkUMXUymTK51K2W-b4key_3qQvGt_FNSepLvsukoM5UdKZvpFt2ivsL6iGIYZzzTGYVtlDQ6o6k2SpZ7IitbYXcNTcoZxKckshvmjjAxLsCRxYykBLG_nDKphA)

I&#39;ve added 4 more custom functions that would allow the researcher to change the tag, id, key, and signature as he would seem fit and function that would create and set a protobuf message. The function \_\_del\_\_ is a standard python function which executes when the object gets destroyed and in this case it would just close the socket connection created for Quasar.

![624x199](https://lh5.googleusercontent.com/vDRsXZwKZQkhitTCCRU8V4KKdTBoykvgSHZ1w63vMd6uZGotxaioy116NevjaT3voXl0ngXFua5DE4eZA5ZLtOL6inU_9f8lv9bZD1XbWPpeGAV7vopN_IF6taTMLBgpoUIulbkj)

The re implementation of the connect function which is very similar to the one in the test script. Before we review it, please download the utils.py file from my repo and place it inside the **stage props** folder as I&#39;ve added several functions to it. First in line 112 I create a tcp socket, then I bind that socket to a SSL socket and context. The **create\_ssl\_sock** is a custom functions ive added to the **utils** script. Much like in our test script all it does is create an SSL socket and binds a context to it so we can verify the quasar certificate. Then, a connect is started to the Quasar server and a handshake is attempted. If all goes well, the logger should display a proper message.

![465x165](https://lh4.googleusercontent.com/gdQG89g0imrc-Sodt6_fILEmAOntchyznHeqbRFP7EhcmncLhzxCL37IN2hELt7a5kVAFe7X5sLjm45IzSp8N0f5Ax2kIdf8y5uhT71RBB5VA_PfCa8uTViIVS_o7bYVDTppift_)
Then we move on to the loop function, which is very primitive at the moment, all it does is receive messages from the server and displays them.

![624x165](https://lh4.googleusercontent.com/MJbVe-VLLh9Yy3pn_jIqS5UJ7u-uC8UezHmhj5wq27Uc0_iLIavkqdnY5Z2fqkmX-V5gRehB3DCsM2CsMB3QjfT_nQFwY0-UDKvmUk1SL5ap2ZoQFcws9yvgMb_BX4sP9Ey_j--e)

We move on to the register function, which is the final one, all it does is mimic exactly what would test script does. It constructs a Quasar message and sends it to the server. I wouldn&#39;t go into length about this as we discussed this already. Now finally, let&#39;s see how this run!

Start your virtual machine and launch the Quasar server, then start a shell within the MoP project and input the following command.

&quot; **py orchestrator.py –targets-config targets.yaml&quot;**

![624x66](https://lh5.googleusercontent.com/TpZn_CJZLxwW3dP60PDvGLylzB1ebeL1OFT-ok3Z-nfxXtBsg1toz0P-SXPFSDFwUg05xbfztPj9KDRDCui6tjO6PJBfYxvbq-Taz2unngWtHfdU4Uo--5sthYwoLL1YlDj9Zji_)

**YES!** Alright let&#39;s see what happens if we try to execute functions from the server itself by forcing the client to open a message box:

![624x253](https://lh3.googleusercontent.com/ppOOgDC0bzazl8UVmjRHuq_nkaL9fPUcJW6HfG9NS-gVcXPAaS5C5Nv6zNMGsUIxRAtkaauLaGH55BnrPkAeiS8lR0ys7xhbGi4e6BWD6Ly5b5Qt2O6BL6wphLYAtutTU1euPHOd)

![624x127](https://lh5.googleusercontent.com/y7RBLFrjuatuy4a4RweHHrPUwtUJB5SHjfTnnVwkvVGGBKoqmKhFn--IhH9JZ3gv_3bdGQHYDux04EVo_Oe8272iHO-g3zDZ9L9VjgYumKtR7dHdBcJeUS7buYwPgkWaJ4A-uzSK)

Amazing! As you can see, we received a new message which needs to be deserialized. This would require some more effort into reversing more messages but from our gained knowledge it shouldn&#39;t be too hard 😊

This sums up the post, I hope you enjoyed it! I&#39;ll try to release a full version for this plugin sometimes this month so stay tuned and in the meantime happy researching!

Special thanks to [Intezer](https://intezer.com/?utm_campaign=Brand%20ads&utm_source=Google%20Ads&utm_content=Brand&utm_term=%2Bintezer&utm_campaign=0719+RLSA&utm_source=adwords&utm_medium=ppc&hsa_acc=6013738437&hsa_cam=9281250457&hsa_grp=94310168375&hsa_ad=417088374601&hsa_src=g&hsa_tgt=aud-789535425270:kwd-765886859489&hsa_kw=%2Bintezer&hsa_mt=b&hsa_net=adwords&hsa_ver=3&gclid=CjwKCAjwqJ_1BRBZEiwAv73uwKTr5ccCuc8K2tgIYyav9-D83icOLLWYiHSsR4hI67yxn0xqzMEiYRoCmf0QAvD_BwE), [Marc Gravell](https://twitter.com/marcgravell?lang=he) for this post as I probably wouldn&#39;t finish it without you.

And thanks to my best friend [x24whoamix24](https://twitter.com/comeREwithme)  as without your encourgment I would still be avoiding these pesky network projects :)

*** 
## Sources

https://github.com/intezer/MoP/

https://stackoverflow.com/questions/61412249/protobuf-net-unrecognized-stream-prefix/61425656#61425656

https://developers.google.com/protocol-buffers

https://github.com/quasar/QuasarRAT
