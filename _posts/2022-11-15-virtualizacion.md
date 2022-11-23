---
title       : "Problemas con máquinas virtuales en Windows 11"
excerpt:    "Hace poco, decidí formatear porque llevaba años sin hacerlo y me apetecía, por lo que decidí de paso, cambiarme a Windows 11. Todo funcionó bien y sin problemas... o eso pensaba, hasta que vi que las máquinas virtuales que tenía creadas con VirtualBox iban extremadamente lentas, e incluso alguna no era operativa."
header:
  teaser: /assets/images/win/win.png
  teaser_home_page: true
classes		: wide
category    : [ Sistemas ]
tags        : [ Windows ]
---

![](/assets/images/win/win.png){: .align-center}

Hace poco, decidí formatear porque llevaba años sin hacerlo y me apetecía, por lo que decidí de paso, cambiarme a Windows 11. 

Todo funcionó bien y sin problemas... o eso pensaba, hasta que vi que las máquinas virtuales que tenía creadas con VirtualBox iban extremadamente lentas, e incluso alguna no era operativa.

Toda la información que salía en Google cuando buscaba, era en relación al problema al actualizar a Windows 11 con estos softwares de virtualización y derivados.

Algo tenía que estar pensando, pero no entendía el qué, hasta que un día de casualidad vi por donde podía ir el tema.

Pues como sabréis, os sonará el famoso [**TPM 2.0**](https://support.microsoft.com/es-es/windows/habilitar-tpm-2-0-en-el-equipo-1fd5a332-360d-4f46-a1e7-ae6b0c90645c) necesario para instalar Windows 11, pues este tenía la culpa de esta situación.

Al instalarlo tuve que activar en la UEFI de mi placa dicho módulo, pues no estaba por defecto.

Ahí estaba el problema, al activarlo se activan nuevas medidas de seguridad por defecto de Windows 11: **Device Guard y Credential Guard**.

De esta forma, os dejo el enlace de un buen artículo que seguí para desactivarlos y que también explica más acerca de los mismos:

<https://www.softzone.es/2019/06/18/solucion-problema-vmware-device-credential-guard-windows-10/>

Tras realizar estos pasos, las máquinas funcionaron con la normalidad habitual.

