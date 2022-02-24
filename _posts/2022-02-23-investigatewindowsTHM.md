---
title: "Investigate Windows THM"
date: 2022-02-23T17:30:30
categories:
  - room
tags:
  - TryHackMe
  - Windows
  - Forensics
  - Incident
  - Powershell

classes: wide
---
Investigating Windows is a room on TryhackMe that allows for basic windows forensic practice. It can be found [here.](https://tryhackme.com/room/investigatingwindows)

>*"This is a challenge that is exactly what is says on the tin, there are a few challenges around investigating a windows machine that has been previously compromised. Connect to the machine using RDP"*

The room has guided questions to answer to complete it. We're using [rdesktop](https://www.kali.org/tools/rdesktop/) to connect to the machine from Kali. However, I'm trying to learn Powershell so will be using it where possible. 

---

<h2>Questions</h2>

<h4>What's the version and year of the windows machine?</h4>

This can be seen with Powershell command `systeminfo`.

<img src="/assets/images/investigatingwindows/iw1.PNG" alt="systeminfo.">


**Windows Server 2016**

<h4>Which user logged in last?</h4>

We can use `net user` to see what users exist on the local system. From here we can run `net user <username>` to look at their last logins and see who logged in before us.
<img src="/assets/images/investigatingwindows/iw2.PNG" alt="net user Administrator">

It's **Administrator**!

<h4>When did John log onto the system last?</h4>

We can use `net user john` to query about the user on the local system in order to find out details of their account. We see that their last login was: **03/02/2019 5:48:32 PM**.
<img src="/assets/images/investigatingwindows/iw3.PNG" alt="net user">

<h4>What IP does the system connect to when it first starts?</h4>

Some quick googling shows us that startup applications appear in the registry at `[HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Run]`. So we run `regedit` and navigate to the registry location.

<img src="/assets/images/investigatingwindows/iw4.PNG" alt="regedit">

**10.34.2.3**

<h4>What two accounts had administrative privileges (other than the Administrator user)?</h4>

We can once again use `net user <username>` to look for the local groups that the users are in, we ran `net user` earlier to grab the list of users.

<img src="/assets/images/investigatingwindows/iw5.PNG" alt="Group membership.">

**Jenny,Guest**.

<h4>Whats the name of the scheduled task that is malicious?</h4>

Scheduled task? Gotta go to `Task Scheduler`. We see a few tasks scheduled to run malicious actions here. The first one being **`Clean file system`** which is the correct answer. This task actually runs a netcat listener on port 1348 with `nc.ps1 -l 1348`. The other commands `falshupdate22` runs a hidden `powershell.exe` window. `GameOver` seems to run `mim.exe` and outputs the result to `o.txt`, I'm gonna assume this is `mimikatz`.

<img src="/assets/images/investigatingwindows/iw6.PNG" alt="Netcat listner.">

*What file was the task trying to run daily?*

**nc.ps1**

*What port did this file listen locally for?*

**1348**

<h4>When did Jenny last logon?</h4>

We run `net user Jenny` and see:

<img src="/assets/images/investigatingwindows/iw7.PNG" alt="Never...">

**Never**

<h4>At what date did the compromise take place?</h4>

We can see that the suspicious tasks were created on **`03/02/2019`**

<img src="/assets/images/investigatingwindows/iw8.PNG" alt="Tasks were created 03/02/2019.">

<h4>At what time did Windows first assign special privileges to a new logon?</h4>

Special privilege assignments would be logged by Windows `Event Viewer`. We google for `Event ID special privileges` and see that we are looking for `Windows logs > Security` with ID: [`4672`](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/). We look for the first event on the day of compromise and find that the answer is incorrect. The hint lets us know that the room is looking for xx:xx:49 and that was the correct entry:
<img src="/assets/images/investigatingwindows/iw9.PNG" alt="Event viewer audit.">

**03/02/2019 4:04:49 PM**

<h4>What tool was used to get Windows passwords?</h4>

We remember seeing earlier that a task called `GameOver` ran the .exe `mim.exe`. We guess that this stands for **mimikatz** and that is correct.

<h4>What was the attackers external control and command servers IP?</h4>

To find this we can look at the windows `hosts` file which can be found at: `c:\Windows\System32\Drivers\etc\hosts`. Here we see that the domain `google.com` is pointing at **76.32.97.132** which is definitely not supposed to be the case. 

<img src="/assets/images/investigatingwindows/iw10.PNG" alt="Hosts file.">

<h4>What was the extension name of the shell uploaded via the servers website?</h4>

The Windows webserver is commonly `IIS` we know that uploaded files are stored in  `c:/inetpub/wwwroot`:

<img src="/assets/images/investigatingwindows/iw11.PNG" alt="shell.gif">

And see that a file called shell.gif is there, which is a nice red herring, but the actual answer is **.jsp**.

<h4>What was the last port the attacker opened?</h4>

We check the `Windows Firewall with Advance Security` `Inbound Rules` and see right at the top... **1337**. Nice.

<img src="/assets/images/investigatingwindows/iw12.PNG" alt="Group membership.">

<h4>Check for DNS poisoning, what site was targeted?</h4>

We saw it earlier, the domain **google.com** was pointed to another IP. More information [here](https://www.cloudflare.com/en-gb/learning/dns/dns-cache-poisoning/).

---

<img src="/assets/images/investigatingwindows/iw13.PNG" alt="Victory!">

<h4>For next time:</h4>

- Learn more Powershell, it's hard to use.
- Learn the different logs that are in event viewer. There is a lot of useful forensic information there. 



