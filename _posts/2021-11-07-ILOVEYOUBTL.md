---
title: "ILOVEYOU BlueTeamLabs "
date: 2021-11-07T17:30:30
categories:
  - challenge
tags:
  - Blue Team Labs Online
  - Reverse Engineering
classes: wide
---
This challenge is an easy rated reverse engineering challenge from [BlueTeamLabsOnline](https://blueteamlabs.online/home/challenge/14).

>*"ILOVEYOU the 3 magical words which have an impact in most of the people's life.
On the other hand, these 3 words don't need any introduction for the people in the Infosec industry. Let's relive history by analysing the ‘ILOVEYOU’ malware.
This challenge should be completed in a virtual machine as it contains real malware. "*


The challenge has guided questions to answer to obtain points.

---

We begin by unzipping the challenge files:

<img src="/assets/images/iloveyou/ily0.PNG" alt="Unzipping the file.">

We see a file named `LOVE-LETTER-FOR-YOU.TXT.vbs.txt`. Seems like we won't have to use GHIDRA:


<img src="/assets/images/iloveyou/ily1.PNG" alt="Catting the file.">

Now on to the questions:


<h4>What is the text present as part of email when the victim received this malware?</h4>

<img src="/assets/images/iloveyou/ily2.PNG" alt="Body of the messsage.">

We see the message body is `kindly check the attached LOVELETTER coming from me.`

<h4>What is the domain name that was added as the browser's homepage?</h4>

<img src="/assets/images/iloveyou/ily3.PNG" alt="Domain name.">

We see the domain skyinet, the answer is in format: `http://www.skyinet.net/`

<h4>The malware replicated itself into 3 locations, what are they?</h4>

<img src="/assets/images/iloveyou/ily4.PNG" alt="3 Locations.">

The BlueTeamLabsOnline answer must be in format `C:\Windows\System32\MSKernel32.vbs, C:\Windows\System32\LOVE-LETTER-FOR-YOU.TXT.vbs, C:\Windows\Win32DLL.vbs`

<h4>What is the name of the file that looks for the filesystem?</h4>

<img src="/assets/images/iloveyou/ily5.PNG" alt="WinFAT32.exe">

Reading the explanation of the LoveLetter worm on [f-secure](https://www.f-secure.com/v-descs/love.shtml) it seems the file is `WinFAT32.exe`.

<h4>Which file extensions, beginning with m, does this virus target?</h4>

<img src="/assets/images/iloveyou/ily6.PNG" alt="File extensions it targets.">

Ah, mp3 and mp2.

<h4>What is the name of the file generated when the malware identifies any Internet Relay Chat service?</h4>

<img src="/assets/images/iloveyou/ily7.PNG" alt="if irc then script.ini.">

We see that the script looks for `mIRC` and if present it generates and writes to a file named `script.ini`. The script written to the script is an IRC command to send a message with the worm html page as the user $ME. This is done in order to bait people to click and propagate the worm further. 


<h4>What is the name of the password stealing trojan that is downloaded by the malware? </h4>

<img src="/assets/images/iloveyou/ily8.PNG" alt="Right at the top!">

I luckily read the f-secure page on the LoveLetter worm thoroughly, it mentions [Barok](https://www.f-secure.com/v-descs/barok.shtml).

<h4>What is the name of the email service that is targeted by the malware?</h4>

<img src="/assets/images/iloveyou/ily9.PNG" alt="Outlook.">

<h4>What is the registry entry responsible for reading the contacts of the logged in email account?</h4>

<img src="/assets/images/iloveyou/ily10.PNG" alt="WAB.">

<h4>What is the value that is stored in the registry to remember that an email was already sent to a user?</h4>

<img src="/assets/images/iloveyou/ily11.PNG" alt="Value stored.">

The value stored is `1`. The `regv` variable is filled by the `regedit.RegRead` command. When `regv` is read initially, it checks the WAB and then looks through the address list. If there isn't a stored value in the registry for that address, then `regv` for that position is set to 1 by `regv = 1`.

<img src="/assets/images/iloveyou/ily12.PNG" alt="Victory!">

