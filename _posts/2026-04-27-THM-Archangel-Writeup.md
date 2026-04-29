---
title: Archangel Writeup | TryHackMe
date: 2026-04-29 
categories: [TryHackMe]
tags: [lfi, poison, "cron jobs", suid, path-hijacking]     # TAG names should always be lowercase
---

## 1. Room info
* **Difficulty:** Easy
* **link:** https://tryhackme.com/room/archangel

## 2. Reconnaissance
I started with a standard service scan to identify open ports.

```text
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
SSH is listening on port 22 and an Apache server is running on port 80. I quickly skimmed the webpage but havn't seen anything useful (that's not the case on this machine as I would realize later), so I started a directory scan with the following command:
```text
gobuster dir -u http://<Target-IP> -w /ush/share/wordlists/dirb/common.txt
```
Accessible were the following entries:
```text
flags               [36m (Status: 301)[0m [Size: 314][34m [--> http://10.128.185.88/flags/][0m
images              [36m (Status: 301)[0m [Size: 315][34m [--> http://10.128.185.88/images/][0m
index.html          [32m (Status: 200)[0m [Size: 19188]
layout              [36m (Status: 301)[0m [Size: 315][34m [--> http://10.128.185.88/layout/][0m
pages               [36m (Status: 301)[0m [Size: 314][34m [--> http://10.128.185.88/pages/][0m
```
First, I quickly checked `flags/`, but it turned out to contain only an HTML RickRoll redirect.
![Getting RickRolled](assets\img\archangel\1-flags_html_redirect.png)

`pages/` also did not contain anything useful. At this point I decided to go back to the home page of the website and scan it again for any clues. And here it is: a hostname, at the very top of the page.
![A hostname](assets\img\archangel\2-hostname.png)

**Virtual Hosting** allows a single server to host multiple websites on a single IP address. When a client sends an **HTTP request**, the server uses the **Host header** to determine which specific website to serve. To access the website with the hostname `mafialive.thm` add the following line to `/etc/hosts`.

```text
<Target-IP> mafialive.thm
```

