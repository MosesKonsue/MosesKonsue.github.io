---
title: "Insider - CyberDefenders "
date: 2021-10-13T17:30:30
categories:
  - challenge
tags:
  - CyberDefenders
  - Forensics
  - Incident Response
  - FTK
classes: wide
---
This challenge is an incident response challenge from [CyberDefenders](https://cyberdefenders.org/labs/64)

>*"After Karen started working for 'TAAUSAI,' she began to do some illegal activities inside the company. 'TAAUSAI' hired you to kick off an investigation on this case.
You acquired a disk image and found that Karen uses Linux OS on her machine. Analyze the disk image of Karen's computer and answer the provided questions."*

The challenge has guided questions to answer to obtain points.

---

We unzip the challenge files and see what is going on:

<img src="/assets/images/insider/ins1.PNG" alt="Unzipping the file.">

We see a `.txt` and `.ad1` file, looking inside `FirstHack.ad1.txt` we see:

<img src="/assets/images/insider/ins2.PNG" alt="Contents of the .txt file.">

It seems to be the details of the disk image, creation information, hashes and other important chain of custody information. 
Weirdly no case or evidence number though, I remember that being important. 

Moving on we open `FirstHack.ad1` in FTK Imager and see:

<img src="/assets/images/insider/ins3.PNG" alt="Opening the disk image in FTK imager.">


Now for the questions!

<h4>What distribution of Linux is being used on this machine?</h4>

We look around the boot directory of the disk image in FTK and see:

<img src="/assets/images/insider/ins4.PNG" alt="Inside the boot directory">

Seems like Kali was used on this machine.

<h4>What is the MD5 hash of the apache access.log?</h4>

Looking at the apache2 directory we find the `access.log` file. From here we use FTK imager to export the hash like so:

<img src="/assets/images/insider/ins6.PNG" alt="Exporting the access.log hash">

Once we've exported the hash of the `access.log` file we open it up in our spreadsheet viewer of choice and see:

<img src="/assets/images/insider/ins7.PNG" alt="There is the MD5 hash.">

Which is correct!

<h4>It is believed that a credential dumping tool was downloaded? What is the file name of the download?</h4>

Navigating to the Downloads directory we see an infamous credential dumping tool, [mimikatz](https://www.varonis.com/blog/what-is-mimikatz/).

>*"Mimikatz is an open-source application that allows users to view and save authentication credentials like Kerberos tickets."*

<img src="/assets/images/insider/ins8.PNG" alt="This is the downloaded zip file.">

This particular file is named `mimikatz_trunk.zip`

<h4>There was a super-secret file created. What is the absolute path?</h4>

We see in the root directory that there is a `.bash_history` file that could contain references to a created file, as we do not see an obviously created file anywhere else:

<img src="/assets/images/insider/ins9.PNG" alt="Bash history!">

`/root/Desktop/SuperSecretFile.txt`

<h4>What program used didyouthinkwedmakeiteasy.jpg during execution?</h4>

We continue to check further down the `.bash_history` file and find reference to `didyouthinkwedmakeiteasy.jpg`:

<img src="/assets/images/insider/ins10.PNG" alt="Bash history again!">

Program used was `binwalk`!

<h4>What is the third goal from the checklist Karen created?</h4>

On the root Desktop we see a `checklist` regular file:

<img src="/assets/images/insider/ins11.PNG" alt="Profit!">

Seems Karen aims to profit from her hack.

<h4>How many times was apache run?</h4>

Navigating back to the apache `access.log` file we see that the file is empty with 0 byte size. This points to apache having never been run. An answer of 0 is correct.

<img src="/assets/images/insider/ins12.PNG" alt="Empty file.">

<h4>It is believed this machine was used to attack another. What file proves this?</h4>

On the root desktop we find a file called `irZLAohL.jpeg` which appears to be a screenshot of another user's windows machine. Considering our image is running Kali linux, this suggests an attack on another machine. 

<img src="/assets/images/insider/ins13.PNG" alt="Suspicious screenshot.">

<h4>Within the Documents file path, it is believed that Karen was taunting a fellow computer expert through a bash script. Who was Karen taunting?</h4>

Looking at the Documents file path we see a file called `firstscript_fixed` which contains the following:

<img src="/assets/images/insider/ins14.PNG" alt="She can do it too apparently.">

Seems Karen is taunting `Young` about writing bash.

<h4>A user su'd to root at 11:26 multiple times. Who was it?</h4>

Navigating back to the logs directory, we find `auth.log` file. Which according to [DigitalOcean](https://www.digitalocean.com/community/tutorials/how-to-monitor-system-authentication-logs-on-ubuntu):

>*"A fundamental component of authentication management is monitoring the system after you have configured your users.
Luckily, modern Linux systems log all authentication attempts in a discrete file. This is located at "/var/log/auth.log"*

<img src="/assets/images/insider/ins15.PNG" alt="auth.log 11:26">

We see a user named `postgres` (reference to PostgreSQL?) mentioned here:

```
Successful su for postgres by root
```


<h4>Based on the bash history, what is the current working directory?</h4>

Looking through the bash history again, we see the last reference to a directory is `/root/Documents/myfirsthack/`.

<img src="/assets/images/insider/ins16.PNG" alt="bash history">

This is correct, and we finish the challenge!

<img src="/assets/images/insider/ins17.PNG" alt="Success!">
