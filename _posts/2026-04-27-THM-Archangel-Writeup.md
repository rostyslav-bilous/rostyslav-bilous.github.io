---
title: Archangel Writeup | TryHackMe
date: 2026-04-29 
categories: [TryHackMe]
tags: ["local file inclusion", poison, "cron jobs", suid, "path hijacking"]     # TAG names should always be lowercase
image:
  path: assets/img/archangel/archangel_thm_logo.jpeg
  width: 100
  height: 100
---

## 1. Room info
* **Name:** Archangel
* **Platform:** TryHackme
* **Difficulty:** Easy
* **Link:** [Archangel room on TryHackMe](https://tryhackme.com/room/archangel)

## 2. Reconnaissance
I started with a standard service scan to identify open ports.

```terminal
kali@kali:~/thm$ nmap -sV -sC 10.128.185.88 -oN nmap/initial.txt
Starting Nmap 7.98 ( https://nmap.org ) at 2026-04-27 05:24 -0400
Nmap scan report for 10.128.185.88
Host is up (0.024s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 9f:1d:2c:9d:6c:a4:0e:46:40:50:6f:ed:cf:1c:f3:8c (RSA)
|   256 63:73:27:c7:61:04:25:6a:08:70:7a:36:b2:f2:84:0d (ECDSA)
|_  256 b6:4e:d2:9c:37:85:d6:76:53:e8:c4:e0:48:1c:ae:6c (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-title: Wavefire
|_http-server-header: Apache/2.4.29 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
SSH is listening on port 22 and an Apache server is running on port 80. 
## 3. Enumeration
I quickly skimmed the webpage and its source code but haven't found anything useful (that's not the case on this machine as I would realize later), so I started a directory scan with `gobuster` using `dirb/common.txt`. 
```bash
gobuster dir -u http://<Target-IP> -w /ush/share/wordlists/dirb/common.txt
```
Accessible (status `200` or `301`) were the following entries:
```
flags               (Status: 301) [Size: 314] [--> http://10.128.185.88/flags/]
images              (Status: 301) [Size: 315] [--> http://10.128.185.88/images/]
index.html          (Status: 200) [Size: 19188]
layout              (Status: 301) [Size: 315] [--> http://10.128.185.88/layout/]
pages               (Status: 301) [Size: 314] [--> http://10.128.185.88/pages/]
```
First, I quickly checked `flags/`, but it turned out to contain only an HTML RickRoll redirect.
![Getting RickRolled](assets\img\archangel\1-flags_html_redirect.png)

`pages/` and `layout/` also did not contain anything useful. At this point I decided to go back to the home page  and scan it again for any clues. And here it is: a hostname, at the very top of the page.
![A hostname](assets\img\archangel\2-hostname.png)

If I wouldn't have found anything on the home page I would consider using a bigger list such as `dirbuster/directory-list-2.3-medium.txt` or searching for files with extensions like `php`,`html` or `bak`. Luckily, this wasn't necessary.


**Virtual Hosting** allows a single server to host multiple websites on a single IP address. When a client sends an **HTTP request**, the server uses the **Host header** to determine which specific website to serve. To access the website with the hostname `mafialive.thm` add the following line to `/etc/hosts`:

```text
<Target-IP> mafialive.thm
```

After accessing `http://mafialive.thm` I got the first flag.
![flag-1](assets/img/archangel/3-flag_1.png)

So I ran `gobuster` on the new webpage and these are the accessible recources:
```
index.html          (Status: 200) [Size: 59]
robots.txt          (Status: 200) [Size: 34]
```

`robots.txt` prevents search engine bots from indexing sensitive or irrelevant portions of the website. Looking at the file itself I saw that there exits `/test.php` page.

![robots](assets/img/archangel/4-robots.png)

After navigating to `http://mafialive.thm/test.php` and clicking the button I saw two things. First, the message `Control is an illusion` which might be a hint towards improper input validation logic, and, second, that the system dynamically loads content via `?view=`. 

![button](assets/img/archangel/5-button.png)

In `php`, `include()` is used to import and **paste** the contents of a file in the place where `include()` was called. It's might be useful when a developer wants to reuse a piece of text/code across multiple pages. The imported file is then not just treated as plaintext but is **evaluated**. In practical terms it means that if an attacker manages to write a malicious piece of `php` code inside a file that is being subject to `include()`, that code would be executed and the system will be compromised.


...

Well, normally, `www-data` user that is being used in Apache servers to retrieve data and does not have any write permissions in the web server directory. That's why I use globally-writable `/tmp` folder to store the reverse shell.


