---
title: Cortafuegos perimetral
author: Paloma R. García Campón
date: 2020-09-11 00:00:00 +0800
categories: [Seguridad]
tags: [iptables, firewall, netfilter]
---

# Introducción
Firewall perimetral o cortafuegos perimetral se utiliza para marcar el perímetro entre unas redes y otras. Por lo general entre una red privada y otra red o internet. 

El Proyecto Netfilter es la comunidad de desarrolladores de software que se dedican al framework disponible en el núcleo de Linux que permite interceptar y manipular paquetes de red, entre otras cosas. Entre los componentes más conocidos de este pryecto se encuentran nftables e iptables. Siendo IPtables la herramienta encargada de inspeccionar, modificar o eliminar paquetes de la red a partir de tablas, cadenas y reglas.

Pero, ¿qué es nftables? Podríamos decir que es lo mismo. Fue creada para sustituir a iptables y mejorar algunos problemas de rendimiento y escalabilidad que ésta tenía. Auna en una sola herramienta iptables, ip6tables, arptables, etc. y con una sintaxis más simple y conatible con la sintaxis de iptables tradicional. Y, aunque parece la misma herramienta, su arquitectura no se parece en nada puesto que no crea tablas, ni cadenas, ni reglas de manera predeterminada. Además, permite hacer múltiples acciones en una sola regla. 

Podríamos decir que nftables es la herramienta definitiva, en estos momentos. Si quieres aprender sobre el uso de nftables, siento comunicarte que éste no es tu ligar, pues en este post se aportan los datos más significativos para poder crearse una idea de como se debe configura correctamente un cortafuego perimetral usando iptables.


# IPtables
Como se ha dicho anteriormente, iptables permite definir reglas para indicar al núcleo qué hacer con los paquetes que atraviesan los interfaces de red. Estas reglas se listan formando tablas. Nerfilter posee tres tablas:
- FILTER: responsable del filtrado de paquetes.
- NAT: establece las reglas para reescribir las direcciones y puertos en los paquetes a través de las acciones PREROUTING o POSTROUTING.
- MANGLES: permite manipular los paquetes.

Estas tablas poseen sus propias cadenas pero se pueden crear nuevas, relacionarlas, anidarlas, etc. para conseguir la configuración deseada.

Las opciones disponibles sobre las reglas son:
* -A: añadir una regla a una cadena (al final).
* -I: añadir una regla a una cadena en una posición determinada.
* -D: eliminar una regla a partir de la posición o coincidiendo con una serie de parámetros.

Y las opciones básicas:
* -s: indica la dirección de origen.
* -d: indica la dirección de destino.
* -p: indica un protocolo.
* -i: indica una interfaz de entrada.
* -o: indica una interfaz de salida.
* -j: inidica la acción a ejecutar sobre el paquete.
* --sport: indica el puerto de origen.
* --dport: inicia el puerto de destino.
* --state: indica el estado del paquete (NEW, ESTABLISHED, RELATED, INVALID).
* -t: indica la tabla sobre la que se quiere actuar. Si no se indica, la tabla por defecto es FILTER.

Ejemplos básicos:
> Con las cuatro órdenes que se explican a continuación se limpian todas las reglas.
Borrar las reglas de la tabla FILTER:
~~~
iptables -F
~~~

Borrar las reglas de la tabla NAT:
~~~
iptables -t nat -F
~~~

Poner los contadores a cero de la tabla FILTER y de la tabla NAT:
~~~
iptables -Z
iptables -t nat -Z
~~~



## Tabla FILTER
Esta tabla posee una lista de reglas ordenadas, cada regla con una cadena que pueden ser de tres tipos: 
- INPUT: por la que pasan todos los paquetes cuyo destino es nuestro sistema.
- OUTPUT: por la que pasan todos los paquetes cuyo origen es nuestro sistema.
- FORWARD: por la que pasan todos los paquetes cuyo origen o destino no es nuestro sistema pero pasan por él, para ser enrutados.

> Para que nuestro sistema pueda enrutar paquetes es necesario tener activado ip_forwarding con `echo 1 > /proc/sys/net/ipv4/ip_forward`.
> Para habilitar el ip_forwarding de forma permanente, se habilita la línea `net.ipv4.ip_forward=1` del fichero **/etc/sysctl.conf** y se ejecuta posteriormente `sysctl -p /etc/sysctl.conf`.

Un punto esencial es tener en cuenta es el orden de estas reglas. Cuando un paquete alcanza una cadena, se comprueba de forma secuencial y se ejecuta la primera regla a la que se ajuste; sin tener en cuentas las posibles reglas a las que se puede ajustar posteriormente. 

Las acciones más comunes son:
- ACCEPT: acepta el paquete.
- DROP: elimina el paquete.
- LOG: genera LOG de la conexión.

Algunas de las operaciones disponibles para el manejo de las cadenas son:
* -L: lista las reglas de una cadena.
* -P: Estable la política por defecto de una cadena.
* -F: borrar todas las reglas de una cadena.
* -Z: Poner a cero todos los contadores (paquetes y bytes) de una cadena.

Ejemplos básicos:
Permitir el tráfico para la interfaz loopback:
~~~
iptables -A INPUT -i lo -p icmp -j ACCEPT
iptables -A OUTPUT -o lo -p icmp -j ACCEPT
~~~

Permitir la conexión ssh desde la red 172.22.0.0/16: 
~~~
iptables -A INPUT -s 172.22.0.0/16 -p tcp -m tcp --dport 22 -j ACCEPT
iptables -A OUTPUT -d 172.22.0.0/16 -p tcp -m tcp --sport 22 -j ACCEPT
~~~

Borrar las políticas por defecto de una cadena:
~~~
iptables -P [INPUT | OUTPUT | FORWARD] DROP
~~~

Permitir ssh desde el cortafuegos a la LAN, siendo la red 192.168.100.0/24:
~~~
iptables -A OUTPUT -p tcp -o eth1 -d 192.168.100.0/24 --dport 22 -j ACCEPT
iptables -A INPUT -p tcp -i eth1 -s 192.168.100.0/24 --sport 22 -j ACCEPT
~~~

Permitir peticiones y respuestas con el protocolo ICMP:
~~~
iptables -A OUTPUT -o eth0 -p icmp -j ACCEPT
iptables -A INPUT -i eth0 -p icmp -j ACCEPT
~~~

Permitir ping a la LAN:
~~~
iptables -A OUTPUT -o eth1 -p icmp -j ACCEPT
iptables -A INPUT -i eth1 -p icmp -j ACCEPT
~~~


### Reglas FORWARD
Imaginemos un escenario compuesto por una LAN sin políticas FORWARD. En este caso los equipos que componen la LAN se encuentran incomunicados, ya que no se permite el paso de paquetes por el cortafuegos. Por lo tanto, habrá que configurar los pares de reglas, FORWARD en ambas direcciones, para ir permitiendo distintos protocolos, puertos, etc. 

Ejemplos básicos:
Permitir hacer ping, protocolo icmp, desde la LAN.  
~~~
iptables -A FORWARD -o eth0 -i eth1 -s 192.168.100.0/24 -p icmp -j ACCEPT
iptables -A FORWARD -i eth0 -o eth1 -d 192.168.100.0/24 -p icmp -j ACCEPT
~~~

Consultas y respuestas DNS, protocolo udp por el  puerto 53, desde la LAN:
~~~
iptables -A FORWARD -i eth1 -o eth0 -s 192.168.100.0/24 -p udp --dport 53 -j ACCEPT
iptables -A FORWARD -o eth1 -i eth0 -d 192.168.100.0/24 -p udp --sport 53 -j ACCEPT
~~~

Permitir la navegación web, por los puertos 80 y 443, desde la LAN:
~~~
iptables -A FORWARD -i eth1 -o eth0 -s 192.168.100.0/24 -p tcp --dport 80 -j ACCEPT
iptables -A FORWARD -o eth1 -i eth0 -d 192.168.100.0/24 -p tcp --sport 80 -j ACCEPT
iptables -A FORWARD -i eth1 -o eth0 -s 192.168.100.0/24 -p tcp --dport 443 -j ACCEPT
iptables -A FORWARD -o eth1 -i eth0 -d 192.168.100.0/24 -p tcp --sport 443 -j ACCEPT
~~~



## Tabla NAT
### Source NAT
Para cambiar la dirección origen por la externa del dispositivo de NAT.Se hace como último paso antes de enviar el paquete. Se define con la cadena POSTROUTING en la tabla NAT.

Ejemplo:
**Tras activar el ip_forwarding, Suponemos que la red interna es la 192.168.100.0/24, la interfaz externa del dispositivo de NAT ka eth0 y la IP estática externa la 80.14.1.2

~~~
iptables -t nar -A POSTROUTING -s 192.168.100.0/24 -o eth0 -j SNAT --to 80.14.1.2/32
~~~
**


#### Enmascaramiento IP (IP Masquerade)
Cuando se realiza SNAT pero la IP origen del dispositivo es dinámica se debe comprobar la IP externa del dispositivo de NAT antes de cambiarla en el paquete. Se utiliza la opción MASQUERADE.

Ejemplo:
SNAT para que los equipos de la LAN puedan acceder al exterior, siendo la red interna 192.168.100.0/24 y la interfaz externa del dispositivo de NAT la eth0, tras activar el ip_forwarding:
~~~
iptables -t nat -A POSTROUTING -s 192.168.100.0/24 -o eth0 -j MASQUERADE
~~~

Permitir hacer conexiones ssh, protocolo tcp, desde los equipos de la LAN:
~~~
iptables -t nat -I POSTROUTING -s 192.168.100.0/24 -o eth0 -p tcp --dport 22 -j MASQUERADE
~~~
Además, hay que permitir el tráfico de paquetes en el cortafuegos por el puerto 22:
~~~
iptables -A FORWARD -i eth1 -o eth0 -p tcp --dport 22 -j ACCEPT
iptables -A FORWARD -i eth0 -o eth1 -p tcp --sport 22 -j ACCEPT
~~~


### DNAT con iptables
Se utiliza para exponer un puerto de un equipo interno al exterior. Para ello se cambia la dirección destino de la externa del dispositivo a la del equipo que queremos exponer, se puede cambiar también el puerto destino, definiendo la cadena PREROUTING de la tabla NAT.

Se hace como primer paso antes de tomar la decisión de encaminamiento. 

Ejemplo:
Suponemos que queremos enviar las peticiones al puerto 22/tcp al equipo 192.168.100.2 de la red interna. Tras habilitar ip_forwarding:
~~~
iptables -t nat -A PREROUTING -p tcp --dport 22 -i eth0 -j DNAT --to 192.168.100.2
~~~

Si fuera a un puerto diferente, por ejemplo el 2222/tcp:
~~~
iptables -t nat -A PREROUTING -p tcp --dport 22 -i eth0 -j DNAAT --to 192.168.100.2:2222
~~~

Permiso de acceso a nuestro servidor web de la LAN al exterior.
En un primer momento tenemos que permitir que la consulta pase por el cortafuegos:
~~~
iptables -A FORWARD -i eth0 -o eth1 -d 192.168.100.0/24 -p tcp --dport 80 -j ACCEPT # en este caso lo correcto sería indicar la IP del servidor 
iptables -A FORWARD -i eth1 -o eth0 -s 192.168.100.0/24 -p tcp --sport 80 -j ACCEPT
~~~
Y necesitamos configurar una regla DNAT:
~~~
iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 80 -j DNAT --to 192.168.200.10
~~~


# Ejemplos de usos
Permitir el acceso desde el esterior y desde el cortafuegos a un servidor de correo. 
~~~
iptables -A FORWARD -i eth0 -o eth1 -d 192.168.100.10/24 -p tcp --dport 25 -j ACCEPT
iptables -A FORWARD -i eth1 -o eth0 -p tcp --sport 25 -j ACCEPT
iptables -A OUTPUT -o eth1 -d 192.168.100.10/24 -p tcp --dport 25 -j ACCEPT
iptables -A INPUT -i eth1 -p tcp --sport 25 -j ACCEPT
iptables -t nat -I PREROUTING -i eth0 -p tcp --dport 25 -j DNAT --to 192.168.100.10
~~~

Se comprueba ejecutando un telnet al puerto 25 tcp.
~~~
debian@router-fw:~$ telnet 192.168.100.10 25
Trying 192.168.100.10...
Connected to 192.168.100.10.
Escape character is '^]'.
220 lan.novalocal ESMTP Postfix (Debian/GNU)
~~~

Permitir hacer conexiones ssh desde el exterior a la LAN:
~~~
iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 22 -j DNAT --to 192.168.100.10
iptables -A FORWARD -i eth0 -o eth1 -p tcp --dport 22 -j ACCEPT
iptables -A FORWARD -i eth1 -o eth0 -p tcp --sport 22 -j ACCEPT
~~~

Modificar la regla anterior, para que al acceder desde el exterior por ssh se tenga que conectar al puerto 2222, aunque el servidor ssh esté configurado para acceder por el puerto 22:
~~~
iptables -t nat -I PREROUTING -i eth0 -p tcp --dport 2222 -j DNAT --to 192.168.100.10:22
sudo iptables -I FORWARD -i eth0 -o eth1 -p tcp --dport 22 -j ACCEPT
sudo iptables -I FORWARD -i eth1 -o eth0 -p tcp --sport 22 -j ACCEPT
~~~

Permitir hacer consultas DNS sólo al servidor 192.168.202.2:
~~~
iptables -t nat -A POSTROUTING -s 192.168.100.0/24 -o eth0 -p udp --dport 53 -j MASQUERADE
iptables -I FORWARD -i eth1 -o eth0 -d 192.168.202.2 -p udp --dport 53 -j ACCEPT
iptables -R FORWARD 2 -i eth0 -o eth1 -p udp --sport 53 -j ACCEPT
~~~