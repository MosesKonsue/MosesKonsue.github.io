---
title: "Bruteforce BTLO"
date: 2022-04-30T17:30:30
categories:
  - challenge
tags:
  - Blue Team Labs Online
  - Incident Response
  - Forensics
  - RDP
classes: wide
---
Working on some more incident response, a medium rated IR challenge from [Blue Team Labs Online.](https://blueteamlabs.online/home/challenge/40)

>*" Can you analyze logs from an attempted RDP bruteforce attack?
One of our system administrators identified a large number of Audit Failure events in the Windows Security Event log.
There are a number of different ways to approach the analysis of these logs! Consider the suggested tools, but there are many others out there!*"

The challenge has guided questions to answer to obtain points.

---

First we unzip the challenge:

<img src="/assets/images/bruteforce/bru0.PNG" alt="Unzipping the file.">

After reading the Read Me file we have a lok at the challenge files, firstly the `csv`:

<img src="/assets/images/bruteforce/bru1.PNG" alt="Windows audit log.">

We see a `csv` full of event 4625 "An account failed to log on" error audit log events. The other `txt` file contains the same audit events in case you want to parse through them another way. And the .evtx file is the windows event log file. We'll try and work with the `txt` file just by my preference.

Now on to the questions:

<h4>How many Audit Failure events are there?</h4>

We can find how many events there are by looking for the full title of the event: "Audit Failure" with grep, and using `wc -l` to count the lines. **3103**

<img src="/assets/images/bruteforce/bru2.PNG" alt="3103">


<h4>What is the username of the local account that is being targeted?</h4>

Looking at the first failure event we immediately see the name. **administrator**
<img src="/assets/images/bruteforce/bru3.PNG" alt="administor">

<h4>What is the failure reason related to the Audit Failure logs?</h4>

<img src="/assets/images/bruteforce/bru4.PNG" alt="Unknown user name or bad password">

<h4>What is the Windows Event ID associated with these logon failures?</h4>

Looking back through the Event ID associated with the failure reasons we see **4625**.

<img src="/assets/images/bruteforce/bru5.PNG" alt="4625">

<h4> What is the source IP conducting this attack?</h4>

Glance back to the logs and we see the Source Network Address. 

<img src="/assets/images/bruteforce/bru6.PNG" alt="113.161.192.227">

<h4>What country is this IP address associated with?</h4>

We visit [Cisco Talos](https://talosintelligence.com) and paste in the IP to get the geo information. **Vietnam**

<img src="/assets/images/bruteforce/bru7.PNG" alt="Vietnam.">

<h4>What is the range of source ports that were used by the attacker to make these login requests? (LowestPort-HighestPort - Ex: 100-541) </h4>

We do some inefficient and probably cringe-worthy bash work but get what we want. Get the Source Ports from the logfile, remove all mention of "Source Port:" and then remove the spaces. Putting the port numbers into a file called "ports.txt". From here we could use a script to grab the max and min values, but instead `sort -u` sorts and keeps only unique port numbers. We view the min at the top of the file and max at the bottom for a range of: 
**"49162-65534"** 

<img src="/assets/images/bruteforce/bru8.PNG" alt="The command. ">


<img src="/assets/images/bruteforce/bru9.PNG" alt="The file sorted.">



<img src="/assets/images/bruteforce/bru10.PNG" alt="Victory.">