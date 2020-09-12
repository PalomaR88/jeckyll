---
title: Introducción a MicroVM con Firecracker
author: Paloma R. García Campón
date: 2020-08-10 00:00:00 +0800
categories: [Administración de sistemas operativos]
tags: [KVM, microVM, Firecracker]
---

¿Sabías qué en la China del siglo IX, intentando crear una poción para la inmortalidad, los taoístas inventan la pólvora que pronto fue usada como espectáculo visual mediante lo que hoy conocemos como petardos? ¿Sabías qué el tema de este post no es original, sino que decidí indagar en el tema a partir de un trabajo de un compañero? Nada de esto es importante para acercarse a Firecracker y la creación de MicroVM que es lo que intentaré en este post.


![kubernetes](https://github.com/PalomaR88/jeckyll/blob/master/assets/img/sample/firecracker/firecracker.jpg?raw=true)

Como se indica en la [página oficial de Firecracker](https://firecracker-microvm.github.io/), es una tecnología de virtualización de código abierto diseñada para crear y administrar micro máquinas virtuales. 

El siguiente post indica los puntos principales para desplegar Firecracker.

# Instalación

Para la instalación se va a descargar lel binario del [repositorio oficial de GitHub de Firecracker](https://github.com/firecracker-microvm/firecracker). Para ello, en primer lugar se va a establecer la variable `latest` que recoge la última versión disponible:
~~~
latest=$(basename $(curl -fsSLI -o /dev/null -w  %{url_effective} https://github.com/firecracker-microvm/firecracker/releases/latest))
~~~

**Explicación de la instrucción anterior:**
`basename`: elimina el directorio y el sufijo de los nombres de archivo.

`curl`: herramienta para transferir datos desde o hacia un servidor.
- f: sin salida de errores.
- s: para que no muestre el medidor de progresos ni los mensajes de error.
- S: para que muestre un mensaje de error si falla.
- L: si el servidor informa que la página solicitada se ha movido, esta opción hará que curl rehaga la solicitud a la nueva ubicación.
- I: para obtener solo los encabezados.
- o: para indicar la salida.
- w: muestra información en la salida estándar después de una transferencia completa.


A continuación, se descarga el binario:
~~~
curl -LOJ https://github.com/firecracker-microvm/firecracker/releases/download/${latest}/firecracker-${latest}-$(uname -m)
~~~

Y se cambia el nombre del fichero descargado:
~~~
mv firecracker-${latest}-$(uname -m) firecracker
~~~

Se le da permiso de ejecución:
~~~
chmod +x firecracker 
~~~

Con el binario de Firecracker en su lugar, es hora de crear el socket para la API. Para ello, en primer lugar hay que borrarlo, por su existiera:
~~~
rm /tmp/firecracker.socket
~~~

Y se crea el nuevo socket:
~~~
./firecracker --api-sock /tmp/firecracker.socket
~~~

Con estos pasos ya tendremos disponibles las funciones de Firecracker que se ejecutarán en una shell diferente.


# Configuración del kernel y sistema de ficheros
La configración de la instancia que se lanzará se define mediante solicitudes HTTP. 

Para obtener el kernel y el sistema de ficheros, si no tiene ninguno disponible, se ejecuta el siguiente script:
~~~
arch=`uname -m`
dest_kernel="hello-vmlinux.bin"
dest_rootfs="hello-rootfs.ext4"
image_bucket_url="https://s3.amazonaws.com/spec.ccfc.min/img"

if [ ${arch} = "x86_64" ]; then
    kernel="${image_bucket_url}/hello/kernel/hello-vmlinux.bin"
    rootfs="${image_bucket_url}/hello/fsfiles/hello-rootfs.ext4"
elif [ ${arch} = "aarch64" ]; then
    kernel="${image_bucket_url}/aarch64/ubuntu_with_ssh/kernel/vmlinux.bin"
    rootfs="${image_bucket_url}/aarch64/ubuntu_with_ssh/fsfiles/xenial.rootfs.ext4"
else
    echo "Cannot run firecracker on $arch architecture!"
    exit 1
fi

echo "Downloading $kernel..."
curl -fsSL -o $dest_kernel $kernel

echo "Downloading $rootfs..."
curl -fsSL -o $dest_rootfs $rootfs

echo "Saved kernel file to $dest_kernel and root block device to $dest_rootfs."
~~~

Con el siguiente script se configura el kernel invitado:
~~~
arch=`uname -m`
kernel_path=$(pwd)"/hello-vmlinux.bin"

if [ ${arch} = "x86_64" ]; then
    curl --unix-socket /tmp/firecracker.socket -i \
      -X PUT 'http://localhost/boot-source'   \
      -H 'Accept: application/json'           \
      -H 'Content-Type: application/json'     \
      -d "{
            \"kernel_image_path\": \"${kernel_path}\",
            \"boot_args\": \"console=ttyS0 reboot=k panic=1 pci=off\"
       }"
elif [ ${arch} = "aarch64" ]; then
    curl --unix-socket /tmp/firecracker.socket -i \
      -X PUT 'http://localhost/boot-source'   \
      -H 'Accept: application/json'           \
      -H 'Content-Type: application/json'     \
      -d "{
            \"kernel_image_path\": \"${kernel_path}\",
            \"boot_args\": \"keep_bootcon console=ttyS0 reboot=k panic=1 pci=off\"
       }"
else
    echo "Cannot run firecracker on $arch architecture!"
    exit 1
fi
~~~

Y con el tercer script se configura la imagen de la micro máquina:
~~~
rootfs_path=$(pwd)"/hello-rootfs.ext4"
curl --unix-socket /tmp/firecracker.socket -i \
  -X PUT 'http://localhost/drives/rootfs' \
  -H 'Accept: application/json'           \
  -H 'Content-Type: application/json'     \
  -d "{
        \"drive_id\": \"rootfs\",
        \"path_on_host\": \"${rootfs_path}\",
        \"is_root_device\": true,
        \"is_read_only\": false
   }"
~~~

Con esta configuración ya se podría ejecutar el último script donde se ejecuta la máquina:
~~~
curl --unix-socket /tmp/firecracker.socket -i \
  -X PUT 'http://localhost/actions'       \
  -H  'Accept: application/json'          \
  -H  'Content-Type: application/json'    \
  -d '{
      "action_type": "InstanceStart"
   }'
~~~

La respuesta al script anterior será la siguiente, con un 204 que indica que la solicitud se realizó correctamente:
~~~
$ sh maquina.sh 
HTTP/1.1 204 
Server: Firecracker API
Connection: keep-alive
~~~

Por defecto la máquina creada tiene 1 CPU y 128 MiB de RAM. Pero podría establecerse los parámetros que se quieran antes del paso anterior con el siguiente script:
~~~
curl --unix-socket /tmp/firecracker.socket -i  \
    -X PUT 'http://localhost/machine-config' \
    -H 'Accept: application/json'            \
    -H 'Content-Type: application/json'      \
    -d '{
        "vcpu_count": <nº_CPU>,
        "mem_size_mib": <RAM>,
        "ht_enabled": false
    }'
~~~

Para destruir la máquina creada:
~~~
pkill firecracker
~~~

# Obtener Firectl
Firectl es la herramienta básica de línea de comandos que permite ejecutar Firecracker a través de la línea de comandos.

Para obtener firectl se necesita compilar [el repositorio de GitHub](https://github.com/firecracker-microvm/firectl). Para ello se necesita el comando **git**:
~~~ 
git clone https://github.com/firecracker-microvm/firectl
~~~

En el caso de no tener instaladas las herramientas Go necesatias se puede user un contenedor docker para contruir y copiar el binario de la siguiente forma:
~~~
cd firectl 
sudo make build-in-docker
~~~

Se va a crear una variable que será un número aleatorio para la ruta del socket de la nueva máquina:
~~~
NMV=$(shuf -i 0-10 -n 1)
~~~

Y se crear microVM, con la imagen del kernel y el sistema de ficheros creados en el punto anterior. Se han copiado en el fichero /tmp:
~~~
sudo ./firectl \
  --firecracker-binary=./firecracker \
  --kernel=/tmp/hello-vmlinux.bin \
  --root-drive=/tmp/hello-rootfs.ext4 \
  --cpu-template=T2 \
  --kernel-opts="console=ttyS0 noapic reboot=k panic=1 pci=off nomodules ro" \
  --socket-path=./firecracker-$NMV.socket
~~~

Estas son las opciones que se han utilizado:
- **firecracker-binary**: ruta del binario de firecracker.
- **kernel**: ruta a la imagen del kernel.
- **root-drive**: ruta a la imagen del disco raíz.
- **cpu-template**: plantilla de CPU. Actualmente, Firecracker dispone de C3 y T2.
- **kernel-opts**: línea de compandos del kernel. 
- **socket-path**: ruta para usar el socket de Firecracker. Es un fichero único.

Tras loguarse con el usuario root y misma contraseña este es el resultado:
~~~
Welcome to Alpine Linux 3.8
Kernel 4.14.55-84.37.amzn2.x86_64 on an x86_64 (ttyS0)

localhost login: root
Password: 
Welcome to Alpine!

The Alpine Wiki contains a large amount of how-to guides and general
information about administrating Alpine systems.
See <http://wiki.alpinelinux.org>.

You can setup the system with the command: setup-alpine

You may change this message by editing /etc/motd.

login[856]: root login on 'ttyS0'
localhost:~# 
~~~

Para parar la máquina:
~~~
reboot
~~~

## Agregar un disco externo
En primer lugar se va a crear una imagen qcow2 de 100M:
~~~
qemu-img create -f qcow2 file.qcow2 100M
~~~

Se configura la imagen para que utilice el sistema de ficheros ext4:
~~~
sudo mkfs.ext4 file.qcow2
~~~

Y se inicia la MicroVM, con las mismas opciones que en el punto anterior, agregando el disco:
~~~
sudo ./firectl \
  --firecracker-binary=./firecracker \
  --kernel=/tmp/hello-vmlinux.bin \
  --root-drive=/tmp/hello-rootfs.ext4 \
  --cpu-template=T2 \
  --kernel-opts="console=ttyS0 noapic reboot=k panic=1 pci=off nomodules ro" \
  --socket-path=./firecracker-$NMV.socket \
  --add-drive=file.qcow2:rw
~~~

Una vez iniciada la máquina se verifica que contiene el disco:
~~~
localhost:~# fdisk -l
Disk /dev/vda: 30 MiB, 31457280 bytes, 61440 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/vdb: 192 KiB, 196608 bytes, 384 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
~~~


Como en cualquier otra máquina, el disco puede montarse para ser usada:
~~~
localhost:~# mount /dev/vdb /mnt/
[  119.899149] EXT4-fs (vdb): mounted filesystem without journal. Opts: (null)
localhost:~# lsblk
NAME MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
vda  254:0    0   30M  0 disk /
vdb  254:16   0  192K  0 disk /mnt
~~~


## Configuración de una interfaz de red
En primer lugar se necesita un punto de acceso que en nuestro caso será una interfaz de red virtual que se creará usando tap:
~~~
sudo ip tuntap add tap0 mode tap # user $(id -u) group $(id -g)
~~~

Y se configura se configura la red y se levanta:
~~~
sudo ip addr add 172.20.0.1/24 dev tap0
sudo ip link set tap0 up
~~~

A continuación, se proporcionan las reglas de iptables para habilitar el reenvío de paquetes. En primer lugar, comprobando el nombre del dispositivo de interfaz principal y añadiendolo a una variable:
~~~
DEVICE_NAME=wlp1s0
~~~

Se activa el, conocido cómo, "IP forwarding":
~~~
sudo sh -c "echo 1 > /proc/sys/net/ipv4/ip_forward"
~~~

Regla SNAT para que los equipos de la LAN puedan acceder al exterior, especificando la tabla, `-t`, añadiendo la regla al final, `-A` e indicando la interfaz de red de salida,`-o`:
~~~
sudo iptables -t nat -A POSTROUTING -o $DEVICE_NAME -j MASQUERADE
~~~

La siguiente regla carga un módulo, `-m`, con estados concretos:
~~~
sudo iptables -A FORWARD -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
~~~

Y, por último, se permite la tranmisión de paquetes desde el tap, `-i`, y el dipositivo de interfaz principal, `-o`:
~~~
sudo iptables -A FORWARD -i tap0 -o $DEVICE_NAME -j ACCEPT
~~~

Es necesario conocer la MAC del tap que se ha creado, además, se agrega en una variable para ser usada posteriormente:
~~~
MAC="$(cat /sys/class/net/tap0/address)"
~~~

Y se inicia la microVM indicando el tap y la mac:
~~~
sudo ./firectl \
  --firecracker-binary=./firecracker \
  --kernel=/tmp/hello-vmlinux.bin \
  --root-drive=/tmp/hello-rootfs.ext4 \
  --cpu-template=T2 \
  --kernel-opts="console=ttyS0 noapic reboot=k panic=1 pci=off nomodules ro" \
  --socket-path=./firecracker-$NMV.socket \
  --tap-device=tap0/$MAC
~~~

Una vez en la microVM se valida:
~~~
localhost:~# ifconfig -a
eth0      Link encap:Ethernet  HWaddr DA:7F:BA:76:6D:AB  
          BROADCAST MULTICAST  MTU:1500  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

lo        Link encap:Local Loopback  
          LOOPBACK  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

localhost:~# ifconfig -a
eth0      Link encap:Ethernet  HWaddr DA:7F:BA:76:6D:AB  
          BROADCAST MULTICAST  MTU:1500  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

lo        Link encap:Local Loopback  
          LOOPBACK  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
~~~

Se configura la interfaz:
~~~
ifconfig eth0 up && ip addr add dev eth0 172.20.0.2/24
ip route add default via 172.20.0.1 && echo "nameserver 8.8.8.8" > /etc/resolv.conf
~~~

Y se comprueba desde dentro haciendo ping a google.com:
~~~
localhost:~# ping google.com
PING google.com (172.217.17.14): 56 data bytes
64 bytes from 172.217.17.14: seq=0 ttl=117 time=17.866 ms
~~~


Y se comprueba haciendo ping desde fuera:
~~~
paloma@coatlicue:~$ ping 172.20.0.2
PING 172.20.0.2 (172.20.0.2) 56(84) bytes of data.
64 bytes from 172.20.0.2: icmp_seq=1 ttl=255 time=0.469 ms
~~~
