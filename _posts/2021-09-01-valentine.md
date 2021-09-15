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
classes: wide
---
Valentine is a easy rated retired box released in 2018 from [HackTheBox](https://app.hackthebox.eu/machines/Valentine). 
<h2>Enumeration:</h2>

**Nmap scan results:**

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

- Port 22 OpenSSH 5.9p1 on Ubuntu linux. 
- Port 80 is running Apache/2.2.22 webserver. This means dirbuster/gobuster and nikto are our next scans.
- Port 443 is running ssl/http Apache 2.2.22 which is secure socket layer over HTTP. Also known as HTTP Secure or HTTPS. So an HTTPS port for the apache web server. 
- These services are utilising syn-ack 3 way TCP handshakes.
- We also see the domain valentine.htb and just in case we add valentine.htb to our /etc/hosts file.

I realised that I have never really looked up what ssl/http actually refers to.
> [Parablu.com](https://parablu.com/what-is-port-443-and-why-it-is-imperative-to-your-dr-plan/) says: *Information that travels on the port 443 is encrypted using Secure Sockets Layer (SSL) or its new version, Transport Layer Security (TLS) and hence safer*. 
> According to [wikipedia](https://en.wikipedia.org/wiki/Transport_Layer_Security#Key_exchange_or_key_agreement) TSL works by handshake protocol and key/cipher combinations. *"Before a client and server can begin to exchange information protected by TLS, they must securely exchange or agree upon an encryption key and a cipher to use when encrypting data".*

**Nikto** finds nothing of note.

**Gobuster:**

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


The /dev directory looks suspicious, looking inside we see a notes.txt and hype_key file. I download them both.
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

Being completely stuck, we check the HTB writeup!

<h2>Foothold</h2>

It seems that this particular version of openssl is vulnerable to the well known heartbleed bug, it even has its own [website](https://heartbleed.com/) by the Synopsys Software Integrity Group which says:
> *"The Heartbleed Bug is a serious vulnerability in the popular OpenSSL cryptographic software library. This weakness allows stealing the information protected, under normal conditions, by the SSL/TLS encryption used to secure the Internet. SSL/TLS provides communication security and privacy over the Internet for applications such as web, email, instant messaging (IM) and some virtual private networks (VPNs)."*

[Wikipedia](https://en.wikipedia.org/wiki/Heartbleed) has further details and refers to heartbleed as **CVE-2014-0160**:
>*" It resulted from improper input validation (due to a missing bounds check) in the implementation of the TLS heartbeat extension.[3] Thus, the bug's name derived from heartbeat. The vulnerability was classified as a buffer over-read, a situation where more data can be read than should be allowed."*

The heartbeat extension for TLS seems to have been developed as a way to *"test and keep alive secure communication links without the need to renegotiate the connection each time.*". The way it works from my understanding (reading wikipedia) is that a client can send a Heartbeat Request message with a payload as well as the payload's length. The server would then send the exact same payload back. 

The Heartbleed bug (which is a great play on heartbeat) would exploit this by sending a request with a small payload and a large payload length. The server would then send the payload back, and then whatever characters it had in active memory to the client to fit the payload length. It would do this because of bounds checking failure. The limit to this seems to be 64kb due to 16-bit integer payload size limits. 

> Wikipedia continues: *"Where a Heartbeat Request might ask a party to "send back the four-letter word 'bird'", resulting in a response of "bird", a "Heartbleed Request" of "send back the 500-letter word 'bird'" would cause the victim to return "bird" followed by whatever 496 subsequent characters the victim happened to have in active memory. Attackers in this way could receive sensitive data, compromising the confidentiality of the victim's communications. Although an attacker has some control over the disclosed memory block's size, it has no control over its location, and therefore cannot choose what content is revealed*

We google around and find a [script](https://github.com/sensepost/heartbleed-poc) from SensePost which explains that these tools are outdated and that there are metasploit and burp versions of this exploit which may be easier. I felt like trying this python script and thus I run it against our box. However I don't really seem to get much on the first 10 runs of it, the cause of which I understand because of the great background reading from earlier. 

Reading the github we see options to increase the number of heartbeats to send with `-n`.  I try again and set the `-n` option to 2 and coincidentally seem to get a base64 encoded password, I guess this because of the presence of the double = symbols at the end. This process could be automated to output the responses to a file and filter for unique strings. 

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

The password is in the form of "aGVhcnRibGVlZGJlbGlldmV0aGVoeXBlCg==" which base64 decodes to heartbleedbelievethehype.

With this password we try to ssh as user hype with the file hype_key and we get in!

The user flag is found on hype desktop.

<h2>Privilege Escalation</h2>

```bash
hype@Valentine:~/Desktop$ id
uid=1000(hype) gid=1000(hype) groups=1000(hype),24(cdrom),30(dip),46(plugdev),124(sambashare)
```
From here I try the usual, it seems we can't cat shadow or passwd, crontab doesn't seem to reveal any cronjobs that are exploitable to my eye.

Next step, running linpeas shows a shared tmux session /.devs/dev_sess running. we run `tmux -S /.devs/dev_sess` to attach to a shared session. It seems that the root user started a shared tmux session, which by nature of being started by the root user had root permissions. From here we have root, and can `cat /root/root.txt` to obtain the root flag. 

<h4>For next time:</h4>

- If an sshkey has x_key x is the user.
- Nmap scans can run scripts, and can even check for heartbleed for you.
- Pay attention to linpeas 95% highlights.

