---
title: Creación de un escenario básico con KVM y libvirt
author: Paloma R. García Campón
date: 2021-29-12 00:00:00 +0800
categories: [Administración de sistemas operativos]
tags: [KVM, VM, libvirt]
---
# Creación de un escenario básico con KVM y libvirt

Se va a construir un cluster para prácticas en una máquina Debian 11.1. 

## Redes
Una opción para configurar las redes de las máquinas virtuales sería crear una red interna en el servidor y conectar todas las máquinas a esa red. Pero, en este caso, queremos que las máquinas virtuales se conecten a la red local donde está conectado el servidor. Para ello, se va a configurar un bridge. En primer lugar se descargan las herramientas para crear bridges:
~~~
sudo apt install bridge-utils
~~~

La configuración del bridge será muy simple, puesto que se va a utilizar un servidor DHCP. Se confugura el fichero /etc/network/interfaces donde se indica 
~~~
auto lo
iface lo inet loopback

auto br0
iface br0 inet dhcp
        bridge_ports enp2s0
~~~

Para aplicar los cambios se reinicia el servicio networking:
~~~
sudo systemctl restart networking
~~~

## Instalación de KVM y libvirt y configuración

Para crear el escenario son necesarios los paquetes de KVM y libvirt:
~~~
sudo apt install qemu-kvm virtinst libvirt0 libvirt-clients libvirt-daemon-system virt-manager
~~~

Las configuraciones relacionadas con redes, almacenamiento, etc. se lleva a cabo a través de fichero de configuración xml.



### Redes

Como las máquinas virtuales usarán el bridge que hemos creado anteriormente, debe crearse un fichero xml con la configuraciónd el mismo:
~~~
<network>
 <name>bridge</name>
 <forward mode="bridge"/>
 <bridge name="br0"/>
</network>
~~~

Y se define:
~~~
virsh net-define bridge.xml
virsh net-autostart bridge
virsh net-start bridge
~~~

Para comprobar que la red se ha definido correctamente se puede comprobar listando las redes:
~~~
paloma@tlaloc:~$ virsh net-list
 Nombre   Estado   Inicio automático   Persistente
----------------------------------------------------
 bridge   activo   si                  si
~~~

### Almacenamiento

El almacenamiento de las MV puede configurarse de diversas formas. En nuestro caso, se crea un fichero qcow2 donde se instalará el sistema operativo y servirá de base para futuras máquinas. 

En primer lugar, se crea un pool de volúmenes por defecto a través de un fichero XML:
~~~
<pool type='dir'>
  <name>default</name>
  <target>
    <path>/var/lib/libvirt/images</path>
  </target>
</pool>
~~~

Y se añade e inicia:
~~~
virsh -c qemu:///system pool-define almacenamiento.xml
virsh -c qemu:///system pool-start default
virsh -c qemu:///system pool-autostart default
~~~

De esta forma se comprueba que el grupo se ha creado:
~~~
paloma@tlaloc:~$ sudo virsh pool-list 
 Nombre    Estado   Inicio automático
---------------------------------------
 default   activo   no
~~~

Se crea un fichero xml con la siguiente información:
~~~
<volume type='file'>
  <name>debian.qcow2</name>
  <key>/var/lib/libvirt/images/debian.qcow2</key>
  <source>
  </source>
  <allocation>0</allocation>
  <capacity unit="G">10</capacity>
  <target>
    <path>/var/lib/libvirt/images/debian.qcow2</path>
    <format type='qcow2'/>
  </target>
</volume>
~~~

Y se añade el volumen al pool indicando el fichero de configuración anterior:
~~~
sudo virsh -c qemu:///system vol-create default debian-qcow2.xml
~~~

Para comprobar que el volumen se ha creado correctamente:
~~~
paloma@tlaloc:~$ sudo virsh vol-list --pool default
 Nombre         Ruta
------------------------------------------------------
 debian.qcow2   /var/lib/libvirt/images/debian.qcow2
~~~

Otra forma de crear ficheros qcow2 es con el siguiente comando:
~~~
qemu-img create -f qcow2 <nombre.qcow2> <tamañoG>
~~~

## MV base

### Creación de la primera MV

Se va a crear un fichero qcow2 base con el sistema operativo instalado que más adelante será utilizado para crear otras máquinas virtuales. Para crear esta primera máquina virtual se va a utilizar el comando virt-install. Las opciones que se van a emplear se explican a continuación:

--connect: permite indicar el hipervisor al que se va a conectar. En nuestro caso qemu:///system. 
--name: nombre de la instancia.
--memory: memoria que se asignará en MiB.
--disk: se especifica el medio de almacenamiento.
--vcpus: se especifica la cantidad de cpu virtual que se le asigna a la instancia.
-c: para indicar la ruta del fichero ISO que se utilizará como medio de instalación.
--vnc: opcional. Esta opción configura un servidor VNC en el host. Otro protocolo que admite es spice.
--os-type y --os-variant: aunque también son parámetros opciones, indicar el sistema operativo es muy recomendable para optimizar la configuración. Para ver los sistemas disponibles se utiliza el comando **osinfo-query os** que se encuentra en el paquete **libosinfo-bin**.
--network: se elige la red que va a usar la instancia de las que se han configurado anteriormente. 
--noautoconsole: en el caso de crear una instancia a partir de una imagen ISO debe emplearse esta opción.
--hvm: solicita el uso de una virtualización completa.
--keymap: especifica el idioma del teclado.

Finalmente, usaremos el siguiente comando con sus opciones:
~~~
virt-install --connect qemu:///system \
--name debian \
--memory 2048 \
--vcpus=2 \
--disk path=/var/lib/libvirt/images/debian.qcow2 \
-c /home/paloma/ISO/debian-11.1.0-amd64-netinst.iso \
--network network=bridge \
--os-type linux --os-variant=debian10 \
--vnc --noautoconsole --hvm --keymap es
~~~

Ya se ha creado la máquina y con el comando virt-manager pordemos ver que se está ejecutando y a través de aquí podemos acceder a ella para realizar la instalación del sistema operativo. Además, voy a realizar algunas configuraciones iniciales como la actualización, la instalación de sudo, etc. De esta forma, ya podemos usar este fichero qcow2 para otras MV.

### Reutilización de la MV

Como he dicho anteriormente, tenemos un fichero qcow2 con el sistema operativo y la configuración básica que quiero que tengan mis MV. Para ser utilizado, lo primero será redimensionar el disco:
~~~
qemu-img convert -O qcow2 /var/lib/libvirt/images/debian.qcow2 /var/lib/libvirt/images/debian-min.qcow2
~~~

A continuación, hacemos un aprovisionamiento ligero del disco compartido:
~~~
qemu-img create -b /var/lib/libvirt/images/debian-min.qcow2 -f qcow2 davinci.qcow2
~~~

Y este será el nuevo disco que usaremos para una nueva MV:
~~~
virt-install --connect qemu:///system --name davinci \
--memory 2048 --disk path=/var/lib/libvirt/images/davinci.qcow2 \
--vcpus=1 --boot hd --vnc --os-type linux --os-variant=debian10 \
--network network=bridge --noautoconsole --hvm --keymap es
~~~

A partir de aquí, sería el inicio de la creación de escenarios.
