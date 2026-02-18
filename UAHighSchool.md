<p align="center">
  THM : UA High School<br>
  Difficulty : Easy<br>
  Room link : https://tryhackme.com/room/uahighschool<br>
  <img src="https://i.imgur.com/kYl4w77.png">
</p>

## Summary

- [Introduction](#introduction)
- [Reconnaissance](#reconnaissance)
- [Initial Access](#initial-access)
- [Lateral Movement](#lateral-movement)
- [Privilege Escalation](#privilege-escalation)
- [Conclusion](#conclusion)

## Introduction

```
"U.A. High School" CTF is one of the "easy" rooms in THM. 
It involves basic web enumeration, steganography, and linux privilege escalation.
```

In this room, we are tasked with hacking into U.A. High School. Let's go Plus Ultra!

## Reconnaissance
I started with an Nmap scan to check for open ports.
```
nmap -sC -sV -oN nmap.txt 10.49.170.113
```

The scan results indicated that port 22 (SSH) and port 80 (HTTP) were open :
![](https://i.imgur.com/KI9a7fi.png)

Upon navigating to the web server, a static, clue-free landing page was observed :
![](https://i.imgur.com/fV2287h.png)

To identify hidden directories, a directory brute-force scan was performed using feroxbuster :
```
feroxbuster -u http:10.49.170.113/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt
```

and found a bunch of pages :
![](https://i.imgur.com/Z1nopVS.png)

The enumeration revealed an /assets directory. Manual exploration of different URLs uncovered a generic forbidden page at /assets/images :
![](https://i.imgur.com/DxynSl9.png)

## Initial Access

Further investigation of the /assets directory led to the discovery of /assets/index.php. Testing for command injection by appending ?cmd=whoami resulted in a successful output :
![](https://i.imgur.com/wuBEKc4.png)

The output d3d3LWRhdGEK was identified as Base64 encoded. Decoding it confirmed the user identity as www-data.
```
Echo d3d3LWRhdGEK | base64 -d
# Output: www-data
```

Recognizing the potential for a reverse shell, a payload was generated using an online reverse shell generator :
![](https://i.imgur.com/PdKW9Pk.png)

A Netcat listener was set up to capture the connection. After executing the payload via the URL parameter, a connection was successfully established :
![](https://i.imgur.com/xlF7Ekm.png)

## Lateral Movement

Following the initial compromise, research was conducted based on clues provided by the machine. Exploration of the file system revealed a passphrase.txt file within /var/www/Hidden_Content containing a Base64 encoded string.
```
echo QWxsbWlnaHRGb3JFdmVyISEhCg== | base64 -d >> passphrase.txt
# Output: AllmightForEver!!!
```

An initial attempt to login to the deku account using this passphrase failed. 
![](https://i.imgur.com/8Ukg1ob.png)

However, two JPG files were discovered which served as additional clues. These files were downloaded to the attacking machine. One image could be opened normally, while the other required analysis using a hex editor :
![](https://i.imgur.com/1YZ8LCE.png)

Steganography tools were employed to check for hidden data.
```
steghide extract -sf oneforall.jpg
```

Using steghide and the previously recovered passphrase, a hidden file named **creds.txt** was successfully extracted from **oneforall.jpg**.
![](https://i.imgur.com/iRlLmAn.png)

The creds.txt file revealed the password for the user deku.
![](https://i.imgur.com/UXTK1tf.png)

Using these credentials, a successful SSH login to the deku account was achieved. The first flag, **user.txt**, was located and captured in the home directory :
![](https://i.imgur.com/QsBuBTY.png)


## Privilege Escalation
To identify privilege escalation vectors, the **sudo -l** command was executed. The output showed that the user could run **/opt/NewComponent/feedback.sh**. 

The script was found to be vulnerable to command injection. By manipulating the input during the feedback prompt, permissions were modified to allow access.
```
sudo ./feedback.sh
```
![](https://i.imgur.com/fB2xurj.png)

This exploit successfully granted root user permissions :
![](https://i.imgur.com/Y95e6Iu.png)

Finally, the root directory was accessed, and the final flag root.txt was retrieved :
![](https://i.imgur.com/B1fllHv.png)


## Conclusion

This assessment demonstrated a complete compromise of the U.A. High School server, moving from external web reconnaissance to root-level access. Key vulnerabilities included a web-based command injection, information leakage via steganography, and an insecure script allowing for privilege escalation.