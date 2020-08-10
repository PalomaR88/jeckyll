---
title: Introducción a MicroMV con Firecracker
author: Paloma R. García Campón
date: 2020-06-24 00:00:00 +0800
categories: [Administración de sistemas operativos]
tags: [KVM, microVM, Firecracker]
---

![kubernetes](/assets/img/sample/firecracker/firecracker.jpgsample/firecracker/firecracker.jpg)

Como se indica en la [página oficial de Firecracker](https://firecracker-microvm.github.io/), es una tecnología de virtualización de código abierto diseñada para crear y administrar micro máquinas virtuales. 

El siguiente post indica los puntos principales para desplegar Firecracker.

# Instalación
La descarga se puede realizar desde el [repositorio oficial de GitHub](https://github.com/firecracker-microvm/firecracker/releases/download/v0.21.1/firecracker-v0.21.1-x86_64):
~~~
wget https://github.com/firecracker-microvm/firecracker/releases/download/v0.21.1/firecracker-v0.21.1-x86_64
~~~

Se le da permiso de ejecución:
~~~
chmod +x firecracker-v0.21.1-x86_64 
~~~

Y, para que sea accesible por todos los usuarios, se mueve a la ruta /usr/local/bin/:
~~~
sudo mv firecracker-v0.21.1-x86_64 /usr/local/bin/firecracker
~~~

Con el binario de Firecracker en su lugar, es hora de crear el socket para la API. Para ello, en primer lugar hay que borrarlo, por su existiera:
~~~
rm /tmp/firecracker.sock
~~~

Y se crea el nuevo socket:
~~~
/usr/local/bin/firecracker --api-sock /tmp/firecracker.sock
~~~

Con estos pasos ya tendremos disponibles las funciones de Firecracker que se ejecutarán en una shell diferente.


# Configuración del kernel invitado
La configración de la instancia que se lanzará se define mediante solicitudes HTTP. 

En primer lugar, hay que difinir la fuente de arranque y el kernel a usar. 
*
*
*
*
*
*
*
*
***************esto no me sale, estoy configurando el kernel. TENGO QUE INICAR EL SOCKET!!!
*
*
*
*******este primer chorizo es de la pagina




curl --unix-socket /tmp/firecracker.sock -i \
    -X PUT 'http://localhost/boot-source'   \
    -H 'Accept: application/json'           \
    -H 'Content-Type: application/json'     \
    -d '{
        "kernel_image_path": "./hello-vmlinux.bin",
        "boot_args": "console=ttyS0 reboot=k panic=1 pci=off"
    }'

La respuesta 204 Sin contenido indica que la solicitud se realizó correctamente.

*
*
*
*
*
***********y este segundo es de lo del fernando


kernel_path=$(pwd)"/hello-vmlinux.bin"

  curl --unix-socket /tmp/firecracker.socket -i \
  -X PUT 'http://localhost/boot-source'   \
  -H 'Accept: application/json'           \
  -H 'Content-Type: application/json'     \
  -d "{
        \"kernel_image_path\": \"${kernel_path}\",
        \"boot_args\": \"console=ttyS0 reboot=k panic=1 pci=off\"
   }" PUT 'http://localhost/boot-source'   \
  -H 'Accept: application/json'           \
  -H 'Content-Type: application/json'     \
  -d "{
        \"kernel_image_path\": \"${kernel_path}\",
        \"boot_args\": \"console=ttyS0 reboot=k panic=1 pci=off\"
   }"



*
*
*
*
*
*
*
*
*
*
https://www.katacoda.com/firecracker-microvm/scenarios/getting-started


https://github.com/ftiradob/MicroVM_firecracker#1-introducci%C3%B3n-1