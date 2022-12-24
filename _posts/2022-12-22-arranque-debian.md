---
title       : "Problemas con arranque en Debian"
excerpt:    "Hoy hablaremos de algunos problemitas relacionados con el arranque en Linux, derivados de algún cambio en las particiones como un redimensionamiento de disco."
header:
  teaser: /assets/images/kernelpanic/prev.png
  teaser_home_page: true
classes		: wide
categories:
  - Sistemas
tags:  
  - Kernel panic
  - Howto
  - Debian
  - Initramfs
---

![](/assets/images/kernelpanic/prev.png){: .align-center}

Hoy hablaremos de algunos problemitas relacionados con el arranque en Linux, derivados de algún cambio en las particiones como un redimensionamiento de disco.

En mi caso concreto, tras ver que salió el nuevo kernel de Debian, fui a actualizarlo... pero tras reiniciar apareció en la pantalla el mensaje que todos queremos ver: <span style="color: red">**Kernel panic** </span>

Lo que ocurre, es que me había quedado sin espacio en disco, por lo que no se pudo instalar correctamente el kernel. Ante esta situación, la cosa estaba clara, tocaba cargar el kernel anterior, liberar algo de espacio para poder operar y aumentar el disco.

De esta forma, con las herramientas de VirtualBox le asigné más espacio al disco a través de su interfaz. Bien, ahora toca asignar ese espacio a la partición primaria de Debian, así que utilicé Gparted.

Como el espacio sin asignar se genera después de todas las particiones, quedaba en el medio la del **swap**, por lo que tendría que deshabilitarla, eliminarla, redimensionar la partición primaria y volver a crear la del swap. Así lo hice.

Tras reiniciar, en primer lugar, el sistema se tira como un minuto con la pantalla en negro sin decir nada, luego carga el kernel, se tira otro minuto y medio con el mensaje ***"a start job is running for dev-disk-by"***... y finalmente carga todo correctamente.

El problema inicial del *Kernel panic* ha sido subsanado, pero... creo que tenemos algo más que afinar para que el sistema vuelva a cargar en segundos.

La cuestión está en que, aparentemente no se hizo ningún cambio, pero si nos damos cuenta, la partición del swap fue eliminada y vuelta a crear, por lo que el UUID no coincide.

De esta forma, vamos a revisar los UUID que tenemos actualmente:

~~~
❯ lsblk -f
NAME   FSTYPE FSVER LABEL UUID                                 FSAVAIL FSUSE% MOUNTPOINT
sda                                                                           
├─sda1 ext4   1.0         36a75b30-73ef-42e9-ad90-c1d5e7da30f4    3,8G    60% /
├─sda2                                                                        
└─sda5 swap   1           21186a3b-1ed7-4dca-9d06-d4cd4c1ee018                [SWAP]
sr0   
~~~

Si ahora miramos el fichero */etc/fstab* (es el que indica los sistemas de archivos que se van a montar en el arranque):

~~~
❯ cat /etc/fstab
# /etc/fstab: static file system information.
#
# Use 'blkid' to print the universally unique identifier for a
# device; this may be used with UUID= as a more robust way to name devices
# that works even if disks are added and removed. See fstab(5).
#
# systemd generates mount units based on this file, see systemd.mount(5).
# Please run 'systemctl daemon-reload' after making changes here.
#
# <file system> <mount point>   <type>  <options>       <dump>  <pass>
# / was on /dev/sda1 during installation
UUID=36a75b30-73ef-42e9-ad90-c1d5e7da30f4 /               ext4    errors=remount-ro 0       1
# swap was on /dev/sda5 during installation
UUID=aa69ce1f-fd0f-4ab8-adb3-3219c67326da none            swap    sw              0       0
/dev/sr0        /media/cdrom0   udf,iso9660 user,noauto     0       0
~~~

Podemos comprobar que no coincide el UUID del swap, por lo que lo modificaremos con `sudo nano /etc/fstab`

Tras reiniciar, comprobaremos que hemos quitado el mensaje *"a start job is running for dev-disk-by"*.

A continuación, si añadimos el parámetro "noresume" en el Grub `sudo nano /etc/default/grub` quedando la línea así:

~~~
GRUB_CMDLINE_LINUX_DEFAULT="quiet noresume"
~~~

Y luego regeneramos el fichero con `sudo update-grub`, tras reiniciar habremos quitado esa espera de un minuto antes de cargar el kernel. Y con esto ya estaría todo resuelto.

No obstante, creo que es mejor solventar el motivo por el cual hace esa espera y dejar el grub por defecto, por tanto, vamos a ahondar un poco más.

>La inicialización del sistema operativo comienza cuando el gestor de arranque carga el núcleo en la RAM. El núcleo abrirá el *initramfs* (initial RAM filesystem), que como se puede deducir, es un fichero que contiene un sistema de archivos raíz temporal durante el proceso de arranque. A partir de aquí, el núcleo montará todos los sistemas de
archivos configurados en */etc/fstab*.

Bien, pues teniendo esto en cuenta, vamos a revisar el siguiente fichero:
~~~                                                                        
❯ cat /etc/initramfs-tools/conf.d/resume
RESUME=UUID=aa69ce1f-fd0f-4ab8-adb3-3219c67326da
~~~

Al igual que antes, vemos que no coincide, por lo que lo editaremos con nano por el UUID actual y posteriormente ejecutaremos `update-initramfs -u`.

Tras reconstruir el *initramfs* podemos volvemos a dejar el Grub como lo teníamos. 

~~~
GRUB_CMDLINE_LINUX_DEFAULT="quiet"
~~~

Reiniciamos, y ahora sí, nos carga en segundos y tenemos todo perfecto como al principio.