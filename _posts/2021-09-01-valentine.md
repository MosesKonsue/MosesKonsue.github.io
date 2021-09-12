---
title: "Valentine HTB"
date: 2021-09-01T17:30:30
categories:
  - study
tags:
  - Linux
  - HacktheBox
  - Patch Management
  - Web
---

<h2>Enumeration:</h2>
Nmap scan results:

```
22/tcp  open  ssh      syn-ack OpenSSH 5.9p1 Debian 5ubuntu1.10 (Ubuntu Linux; protocol 2.0)
80/tcp  open  http     syn-ack Apache httpd 2.2.22 ((Ubuntu))
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.2.22 (Ubuntu)
|_http-title: Site doesn't have a title (text/html).
443/tcp open  ssl/http syn-ack Apache httpd 2.2.22 ((Ubuntu))
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.2.22 (Ubuntu)
|_http-title: Site doesn't have a title (text/html).
| ssl-cert: Subject: commonName=valentine.htb/organizationName=valentine.htb/stateOrProvinceName=FL/countryName=US
| Issuer: commonName=valentine.htb/organizationName=valentine.htb/stateOrProvinceName=FL/countryName=US
```

<h4>What is the scan telling us?</h4>
- 22
- 80
- 443
- we find domain valentine.htb, Apache/2.2.22, OpenSSH 5.9p1, Debian 5ubuntu1.10 (Ubuntu Linux; protocol 2.0)

We add valentine.htb to our /etc/hosts file.

gobuster:

```
/index (Status: 200)
/index.php (Status: 200)
/dev (Status: 301)
/encode (Status: 200)
/encode.php (Status: 200)
/decode (Status: 200)
/decode.php (Status: 200)
/omg (Status: 200)
```

The /dev directory looks suspicious, looking inside we see a notes.txt and hype_key file.
The notes file says:

```
To do:

1) Coffee.
2) Research.
3) Fix decoder/encoder before going live.
4) Make sure encoding/decoding is only done client-side.
5) Don't use the decoder/encoder until any of this is done.
6) Find a better way to take notes.
```

Being completely stuck, we check the writeup!

<h2>Foothold</h2>

It seems that this particular version of openssl is vulnerable to the mighty known heartbleed bug:
The Heartbleed Bug is a serious vulnerability in the popular OpenSSL cryptographic software library. This weakness allows stealing the information protected, under normal conditions, by the SSL/TLS encryption used to secure the Internet. SSL/TLS provides communication security and privacy over the Internet for applications such as web, email, instant messaging (IM) and some virtual private networks (VPNs).

We google around and find https://github.com/sensepost/heartbleed-poc. We set the -n flag to 2 and coincidentally seem to get a base64 encoded password

```
/decode.php..Co
  00f0: 6E 74 65 6E 74 2D 54 79 70 65 3A 20 61 70 70 6C  ntent-Type: appl
  0100: 69 63 61 74 69 6F 6E 2F 78 2D 77 77 77 2D 66 6F  ication/x-www-fo
  0110: 72 6D 2D 75 72 6C 65 6E 63 6F 64 65 64 0D 0A 43  rm-urlencoded..C
  0120: 6F 6E 74 65 6E 74 2D 4C 65 6E 67 74 68 3A 20 34  ontent-Length: 4
  0130: 32 0D 0A 0D 0A 24 74 65 78 74 3D 61 47 56 68 63  2....$text=aGVhc
  0140: 6E 52 69 62 47 56 6C 5A 47 4A 6C 62 47 6C 6C 64  nRibGVlZGJlbGlld
  0150: 6D 56 30 61 47 56 6F 65 58 42 6C 43 67 3D 3D 2B  mV0aGVoeXBlCg==+
  0160: 67 27 97 21 D5 BF BB 29 64 4B 9E BB FB 2F D9 65  g'.!...)dK.../.e
```

which base64 decodes to heartbleedbelievethehype

we try to ssh as user hype and we get in!

flag is found on hype desktop.

<h2>Privilege Escalation</h2>

```bash
hype@Valentine:~/Desktop$ id
uid=1000(hype) gid=1000(hype) groups=1000(hype),24(cdrom),30(dip),46(plugdev),124(sambashare)
```
can't cat shadow or passwd.

Running linpeas shows an active tmux terminal /.devs/dev_sess​ running. we run tmux -S /.devs/dev_sess to attach to a shared session. It seems that the root user started a shared tmux session with root access. From here we have root, and can cat from /root/root.txt

<h4>For next time:</h4>
- If an sshkey has x_key x is the user.
- Pay attention to linpeas 95% highlights.
- Read up on heartbleed.

