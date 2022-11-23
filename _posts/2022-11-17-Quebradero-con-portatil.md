---
title       : "Problemas con portátil antiguo"
excerpt:      "Hoy comentaremos los quebraderos de cabeza que tuve con un portátil antiguo de unos 16 años, el cual no se actualizaba e iba super mal."
header:
  teaser: /assets/images/portatil/prev.png
  teaser_home_page: true
classes		: wide
categories:
  - Sistemas
tags:  
  - Debian 11
  - Windows 11
  - Howto
---

![](/assets/images/portatil/prev.png){: .align-center}

Hoy comentaremos los quebraderos de cabeza que tuve con un portátil antiguo de unos 16 años, el cual no se actualizaba e iba super mal.


## Problemática

Tras realizar comprobaciones, se ponía al 99% el procesador en idle, por lo que tampoco me dejaba trabajar y no podía entender por qué ocurría esto, así que lo mejor en estos casos como sabéis... **formatear**.

De esta forma, tras formatear y probar con nuevas versiones de Windows (la que trae de serie está obsoleta) la cosa seguía muy parecida recién arrancado, por lo que no era cuestión de virus o de software, directamente no tenía pontencia para soportar Windows (10) de forma adecuada.


## Instalando Debian

Harto ya, (aunque el portátil no es mío) decidí que instalar un buen Linux puede ser la salvación, así que me puse a ello con Debian 11.

Tras una larga espera, comienzan nuevamente problemas, en este caso la pantalla no coge la resolución adecuada.

~~~
lspci | grep VGA
00:01.0 VGA compatible controller: Advanced Micro Devices, Inc. [AMD/ATI] Mullins [Radeon R2 Graphics]
~~~

De esta forma, hablando con mi buen amigo [Trazi](https://rubenhortas.github.io/) me recomienda instalar unos drivers genéricos de la comunidad, y para ello, debemos activar los repositorios privativos.

Por tanto, vamos a modificar la lista de fuentes de software, que se encuentra en el archivo "sources.list" bajo el directorio /etc/apt, con el fin de añadir las ramas *contrib* y *non-free* en Debian.

``sudo nano /etc/apt/sources.list``

~~~
deb http://deb.debian.org/debian bullseye main contrib non-free
deb-src http://deb.debian.org/debian bullseye main contrib non-free

deb http://deb.debian.org/debian-security/ bullseye-security main contrib non-free
deb-src http://deb.debian.org/debian-security/ bullseye-security main contrib non-free

deb http://deb.debian.org/debian bullseye-updates main contrib non-free
deb-src http://deb.debian.org/debian bullseye-updates main contrib non-free
~~~

Tras esto, ya nos va a encontrar el paquete necesario, así que antes de instalar los drivers:

``sudo apt update``

Y ahora ya sí, instalamos los drivers:

``sudo apt firmware-amd-graphics``

> Tras realizar un reinicio, chimpún, ya cogió la resolución correctamente.


Pero... no había terminado todo, ya que el usuario cuando desconectó el cable de red, no detectaba las wifis...

Aquí perdí mucho tiempo, tratando de realizar configuraciones, instalando, desinstalando...

Pero vi que no podía ser de eso, tenía que ser algo más simple, por lo que vi que tenía el controlador Realtek RTL8723BE, así que googleando vi que otros usuarios tenían problemas también, así que aunque pensaba que sí lo reconocía, la solución parecía estar igual que antes en instalar el driver adecuado.

Como ya tenía el sources.list actualizado, solamente era localizar el paquete que incluía mi controlador.

``sudo apt install firmware-realtek``

> Nuevo reinicio... y chimpún, ya detecta las redes con normalidad.


Así es como salvamos el portátil, con un buen Linux sin tanta tralla.

