---
title: Hack The Box - The Notebook
author: Gustavo Bonito
date: 2021-03-07 17:30:00
categories: [Capture The Flag, Hack The Box]
tags: [Capture The Flag, Hack The Box]
pin: false
---

# Introduction

---

This is a write-up for the [The Notebook](https://www.hackthebox.eu/home/machines/profile/320) (Medium) machine, released on Hack The Box on 06/03/21.

# Reconnaissance

---

## Service Enumeration

![](/assets/img/content/f17dcdc1b1d3429db013a5466506ceef.png)
*Ports / Services enumeration*

Since there were only 2 exposed services I started by looking at the web server running on port 80/tcp. There was a web application running that allowed a user to register an account and login to take notes.

## Web Application (The Notebook)

After registering an account I found that there were very limited options to an unprivileged user.

**Available Functionalities**

- Submit/Edit notes

# Vulnerability Analysis

---

## Authentication

Since there were very few functionalities available to exploit, I started by looking at the authentication portion of the application and noticed it was using a JWT token.

### JWT Token

After decoding the token I noticed that the "kid" parameter pointed to an internal address with the private key, used to sign the token.

![](/assets/img/content/97ac1b6ea0224864901c4bfce6cb73b8.png)
*JWT base64 decode*

Objective: Sign the JWT token with a private key hosted in my server and modify the payload to set a regular user as an administrator.

# Exploitation

---

## Custom JWT token

I developed a small python script ("jwt-token.py") to generate a custom JWT token using my private RSA key, to set the current user as an administrator.

![](/assets/img/content/5c14c77ab01f40569670dabc2a9a43fb.png)
*"jwt-token.py" source code*

## Administration Panel

With the generated JWT token I managed to access the administration panel.

![](/assets/img/content/14b57204e8a934e2d97128b1a86c2c3j2.png)
*"jwt-token.py" generated token*

### Administration Panel - Web Shell Upload

There was a note created by the administrator saying that PHP file uploads were allowed.

![](/assets/img/content/8d6e726591ba4a06a1211fbc4115ec81.png)
*Administration panel note*

Getting a reverse shell as "www-data" from the malicious PHP file

![](/assets/img/content/43b93207e8a034e2d90234h6j8cc2c3j3.png)
*Netcat reverse shell*

# Post-Exploitation

---

## From "www-data" to user ("noah")

After compromising the system with web server privileges only I executed the LinPEAS enumeration script and noticed that there was a home folder backup in "/var/backups/" directory. The "home.tar.gz" file contained an SSH private key that allowed me to login as the user "noah".

![](/assets/img/content/8b57109e8a034e1d97138b1a8cc2c3a1.png)
*"noah" user private SSH key*

## Privilege Escalation

With access to the "noah" via SSH, the only thing left was to get root access to the system, so I executed the LinPEAS script again and noticed that any user could run the command ```"docker exec -it webappdev01*"``` as administrator.

![](/assets/img/content/3f74939dfbd447abadbec139a544fb14.png)
*Listing user privileges*

The container only contained the source code of the application running on port 80/tcp, but the web server was running on the host system. After looking at the source code I found to SHA3-256 hashes for the user "admin" and "noah" but it was not possible to crack them using a regular wordlist.

## CVE-2019-5736

With no useful information on the host neither on the container I started searching for Docker container escape vulnerabilities and found the following exploit on Github:

- [https://github.com/Frichetten/CVE-2019-5736-PoC](https://github.com/Frichetten/CVE-2019-5736-PoC)

### Modifying the Exploit

A few modifications to the exploit were needed to get a reverse shell as root.

![](/assets/img/content/c8fc0f174fe34323af3c11b997f20611.png)
*Modifying the exploit to get a reverse shell*

After transferring the compiled exploit to the target machine and its container, the only thing left was running it.

![](/assets/img/content/3753a5af77c041d0a2b00ad8f26d8e54.png)
*Running the exploit on the container*

![](/assets/img/content/b4fa4cca0cdf4dd4bce882f3d116de25.png)
*Running "docker exec -it webappdev01 /bin/sh" on the host*

Success! I was able to get root access on the Netcat listener.

![](/assets/img/content/98440df5864f4886ba515b9ab6a1e01f.png)
*Privileged access to the server (root)*