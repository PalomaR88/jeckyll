---
title: Instalación de Ansible
author: Paloma R. García Campón
date: 2021-12-30 00:00:00 +0800
categories: [Administración de sistemas operativos]
tags: [Ansible]
---
# Instalación de Ansible

Este post puede considerarse una guía rápida para instalar Ansible en un nodo que hará las funciones de anfitrión en un despligue con esta herramienta. Para ello, he usado una máquina Debian 11.

Aunque hay varias formas de instalar Ansible, en mi caso utilizaré PIP. En primer lugar, nos aseguramos que el sistema tenga python3 (en Debian 11 y anteriores versiones viene instalado por defecto) y pip:
~~~
sudo apt install python3 python3-pip
~~~

Y se instalado ansible desde pip:
~~~
sudo pip3 install ansible
~~~

Como verificación, comprobamos la versión que hemos instalado:
~~~
paloma@davinci:~$ ansible --version
ansible [core 2.12.1]
  config file = None
  configured module search path = ['/home/paloma/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/local/lib/python3.9/dist-packages/ansible
  ansible collection location = /home/paloma/.ansible/collections:/usr/share/ansible/collections
  executable location = /usr/local/bin/ansible
  python version = 3.9.2 (default, Feb 28 2021, 17:03:44) [GCC 10.2.1 20210110]
  jinja version = 3.0.3
  libyaml = True
~~~

## Configuración básica

El nodo anfitrión ebe llegar por ssh a los demás nodos usando una key. También será necesario guardar la huella digital y para ello realizaremos una primera conexión a los nodos por ssh.

Creación de key:
~~~
ssh-keygen -t rsa
~~~

Copia de la key en los nodos:
~~~
ssh-copy-id -i ~/.ssh/id_rsa.pub root@davinci
~~~

El paso anterior debe repetirse tantas veces como nodos estén participando en la implantación de la nueva herramienta.
