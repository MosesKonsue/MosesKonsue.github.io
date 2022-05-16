---
title: "SecureShell BTLO"
date: 2022-05-16T17:30:30
categories:
  - challenge
tags:
  - Blue Team Labs Online
  - Forensics
classes: wide
---
This challenge is a hard rated Digital Forensics challenge from [BlueTeamLabsOnline](https://blueteamlabs.online/home/challenge/17).

>*"Hey! We had a SSH service on a system and noticed unusual change in size of the log file. Don’t panic, it was the new IT guys’ daughter who said she was able to break into the system. I had given her permission to test some of these services. I am giving you the log file, can you solve the following queries? "*

The challenge has guided questions to answer to obtain points.

---
We begin by unzipping the challenge files:

<img src="/assets/images/SecureShell/sh0.PNG" alt="unzip and see the files.">

We see a file named `sshlog.log`, I think I'll see if I can just `cat` , `cut` , `grep` my way through this challenge. We quickly glance at the contents with cat:

<img src="/assets/images/SecureShell/sh1.PNG" alt="Cat.">

Now on to the questions:

<h4>Is it an internal or external attack, what is the attacker IP? </h4>

Alot of the way I worked through these questions involved `grep` for phrases in the log that I thought would contain the answer and looking at the output. In this case I `grep` 'd for `Connection from` and see a recurring internal IP. **Internal:192.168.1.17**

<img src="/assets/images/SecureShell/sh2.PNG" alt="Connection from.">

<h4>How many valid accounts did the attacker find, and what are the usernames?</h4>

We see a way to figure out which accounts the attacker attempted:

<img src="/assets/images/SecureShell/sh3.PNG" alt="connection search.">

We do the same for the phrase that states the account does not exist:

<img src="/assets/images/SecureShell/sh4.PNG" alt="User does not exist.">

We compare the lists manually and determine which accounts were attempted, and which returned *"User does not exist"* and see that 3 users are not in both lists. `administrator, guest, sophia`. We attempt this answer but it's incorrect. We go back for a closer look:

<img src="/assets/images/SecureShell/sh5.PNG" alt="Guest failed to resolve.">
<img src="/assets/images/SecureShell/sh6.PNG" alt="administor failed to resolve.">

We see that the `administrator` and `guest` accounts do not resolve. Sophia however we see indications of the account existing:

<img src="/assets/images/SecureShell/sh7.PNG" alt="administor failed to resolve.">

**1:sophia**

<h4>How many times did the attacker login to these accounts?</h4>

We see **2** log events for `Accepted password`: 

<img src="/assets/images/SecureShell/sh8.PNG" alt="Accepted password.">

<h4>When was the first request from the attacker recorded?</h4>

The attacker's IP is 192.168.1.17 (Internal) so we simply grep `Connection from` and see the first log entry for it: 

<img src="/assets/images/SecureShell/sh9.PNG" alt="First connection from attacker IP.">

**2021-04-29 23:52:25.989**

<h4>What is the log level for the log file?</h4>

I have to do a little [research](https://www.freebsd.org/cgi/man.cgi?sshd_config(5)) here, it seems loglevels indicate how intensive/invasive the logs are:

<img src="/assets/images/SecureShell/sh10.PNG" alt="Documentation for sshd.">

We grep the log to find the most intrusive level:

<img src="/assets/images/SecureShell/sh11.PNG" alt="Debug3.">

**DEBUG3**

<h4>Where is the log file located in Windows?</h4>

This was a little hard to find because I first thought to search through the logfile for traces of the path. However checking the [microsoft documentation](https://docs.microsoft.com/en-us/windows-server/administration/openssh/openssh_server_configuration) :

<img src="/assets/images/SecureShell/sh12.PNG" alt="Microsoft documentation.">

We google for the path offered by microsoft:

<img src="/assets/images/SecureShell/sh13.PNG" alt="Thanks google.">

Thanks `zack-snyder` on github for asking your question!

**C:\ProgramData\ssh\logs\sshd.log**


<img src="/assets/images/SecureShell/sh14.PNG" alt="Victory.">

<h4>For next time:</h4>

- Need to read the log details deeper, administrator and guest accounts existed but did not resolve. Should have been caught quicker. 



