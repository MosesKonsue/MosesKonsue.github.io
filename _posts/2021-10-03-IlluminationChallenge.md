---
title: "Illumination"
date: 2021-10-03T17:30:30
categories:
  - challenge
tags:
  - HacktheBox
  - Forensics
classes: wide
---
To diversify my skillset I have decided to work on CTF style challenges also, and write about them. The first: Illumination from [HackTheBox](https://app.hackthebox.eu/challenges/illumination)

>*"A Junior Developer just switched to a new source control platform. Can you find the secret token?"*

This is also a test of adding images to these posts, and doing challenge writeups.

---

First we unzip the challenge files with the provided password. 
<img src="/assets/images/illumination/illu1.PNG" alt="Unzipping the challenge files.">


Next we have a look at what was inside the zip file. At first I forgot to run `ls -alh` which will list all files and directories including hidden ones. I nearly missed the .git directory, though I should have noticed it during unzipping anyway.
![What do we have?](https://github.com/MosesKonsue/MosesKonsue.github.io/blob/master/assets/images/illumination/illu2.PNG "What do we have?")
