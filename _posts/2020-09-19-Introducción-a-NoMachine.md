---
title: Introducción a MicroVM con Firecracker
author: Paloma R. García Campón
date: 2020-08-10 00:00:00 +0800
categories: [Administración de sistemas operativos]
tags: [NoMachine, microVM, Firecracker]
---

NoMachine es un gestor de conexiones remotas, escritorios remotos, rápido, seguro y fácil de usar. Además de ser multiplataforma, aporta una versión gratuita para sistemas operativos basados en GNU/Linux.

Permite desde un ordenador a otro lograr conexiones estables y rápidas y gestionar su propio servidor privado de forma totalmente gratuita. La versión **Esntreprise** es de pago, puede descargarse y funcionar full durante 30 días de demostración. Esta versión de pago, a diferencia de la gratuita, permite saltar los posibles firewalls que hayan en las redes. Además, incorpora un servidio de servidor en la nube.

# Instalación
El paquete .deb está disponible en la [web oficial de NoMachine](https://www.nomachine.com/es).

Tras descargarse el paquete, se instala:
~~~
sudo dpkg -i Descargas/nomachine_6.11.2_1_amd64.deb
~~~

La aplicación consta de una GUI muy explicativa:
![inicio](https://raw.githubusercontent.com/PalomaR88/jeckyll/master/assets/img/sample/nomachine/1.png)

Automáticamente, aparecerán las máquinas accesibles desde tu red, en este caso la máquina tlaloc:
![maquina](https://raw.githubusercontent.com/PalomaR88/jeckyll/master/assets/img/sample/nomachine/2.png)

Se inicia el escritorio remoto a esa máquina:
![remoto](https://raw.githubusercontent.com/PalomaR88/jeckyll/master/assets/img/sample/nomachine/3.png)

En ese momento, en la máquina tlaloc aparece una nota informativa de la conexión:
![tlaloc](https://raw.githubusercontent.com/PalomaR88/jeckyll/master/assets/img/sample/nomachine/4.png)
