---
title: Guacamole-server
author: Paloma R. García Campón
date: 2020-11-24 00:00:00 +0800
categories: [Implantación de aplicaciones web]
tags: [Apache, Guacamole]
---

# Guacamole-server

Guacamole es un proyecto de código abierto que permite acceder a otros equipos de forma remota, soportando protocolos estándar como VNC, RDP y SSH utilizando HTML5. 

El despliegue de Guacamole es treméndamente sencillo a través de docker. Los pasos los encontramos en [su página oficial](https://guacamole.apache.org/doc/gug/guacamole-docker.html). Personalmente, para una instalación sencilla y rápida, recomiendo la dockerización del [usuario oznu en GitHub](https://github.com/oznu/docker-guacamole).

Otra forma de desplegar guacamole sería de forma nativa, ésta es la forma que se explica a continuación. 

## Dependencias
El primer paso es instalar las librerías necesarias. Para hacerse una idea, algunas de éstas librerías son:
- GCC, G++ y libtool-bin: herramientas de compilación y soporte para librerías. Será necesario para la compilación de guacamole.
- cairo: para la representación gráfica.
- libjpeg-turbo: utilizado por libguac para proporcionar soporte JPEG. La biblioteca necesaria es libjpeg-turbo8-dev o libpng-devel según la plataforma utilizada. 
- libpng: utilizado por libguac para escribir imágenes PNG.
- uuid: se utiliza para asignar ID únicos a cada conexión de Guacamole.
- libavcodec, libavutil y libswscale: biblioteca FFmpeg con decodificadores y codificadores para audio y video o el escalado de imágenes.
- libwebp: para la compresión de imágenes.
- Ogg Vorbis: formato de audio comprimido.
- FreeRDP: compatibilidad con RDP.
- libssh2, OpenSSL y Pango: soporte ssh.
- libVNCServer: compatibilidad con VNC.
- libtelnet: soporte Telnet.

Se instalan dichas librerías:

~~~
sudo apt install gcc g++ libcairo2-dev libjpeg-turbo8-dev libpng-dev libtool-bin libossp-uuid-dev libavcodec-dev libavutil-dev libswscale-dev freerdp2-dev libpango1.0-dev libssh2-1-dev libvncserver-dev libtelnet-dev libssl-dev libvorbis-dev libwebp-dev
~~~

## Compilación de Guacamole
A continuación, se descargará el paquete, se descomprimirá y se compila.

> Para estas acciones serán necesarios los paquetes wget, tar y make:
~~~
sudo apt install wget tar make
~~~

Se descarga el paquete desde la [página oficial de Guacamole](https://downloads.apache.org/guacamole/1.1.0/source/guacamole-server-1.1.0.tar.gz):
~~~
wget https://downloads.apache.org/guacamole/1.1.0/source/guacamole-server-1.1.0.tar.gz
~~~

Se descomprime:
~~~
tar xzf guacamole-server-1.1.0.tar.gz
~~~

Desde el directorio descomprimido se revisa si se necesitan librerías:
~~~
cd guacamole-server-1.1.0/
./configure --with-init-dir=/etc/init.d
~~~

Y se compila:
~~~
make
sudo make install
sudo ldconfig
~~~

Arrancar:
~~~
sudo /etc/init.d/guacd start
sudo /etc/init.d/guacd status

sudo systemctl start guacd
sudo systemctl enable guacd
sudo systemctl status guacd
~~~


## Tomcat Servlet
Es necesario instalar y configurar Tomcat. Para su instalación:
~~~
sudo apt install tomcat9 tomcat9-admin tomcat9-common tomcat9-user
~~~

Y se reinicia:
~~~
reboot
~~~

~~~
sudo ufw allow 8080/tcp
~~~

## Cliente Guacamole
De la misma forma que se instaló el servidor de guacamole hay que isntalar el cliente. PAra ello se va a crear un directorio:
~~~
sudo mkdir /etc/guacamole
~~~

Y se descarga el código fuente:
~~~
sudo wget https://downloads.apache.org/guacamole/1.1.0/binary/guacamole-1.1.0.war -O /etc/guacamole/guacamole.war
~~~

Se crea el enlace simbólico del cliente al directorio de aplicaciones web Tomcat:
~~~
ln -s /etc/guacamole/guacamole.war /var/lib/tomcat9/webapps/
~~~

Y se reinician los servicios:
~~~
sudo systemctl restart tomcat9
sudo systemctl restart guacd
~~~

# Configuración
## Configuración de Apache Guacamole
Crear directorio:
~~~
sudo mkdir /etc/guacamole/extensions
sudo mkdir /etc/guacamole/lib
~~~

Crear variable de entorno y guardar en /etc/default/tomvat9:
~~~
sudo su
echo "GUACAMOLE_HOME=/etc/guacamole" >> /etc/default/tomcat9
~~~

## Configurar conexiones del servidor Guacamole
Creación del fichero de configuración:
~~~
sudo nano /etc/guacamole/guacamole.properties
~~~

~~~
guacd-hostname: localhost
guacd-port:     4822
user-mapping:   /etc/guacamole/user-mapping.xml
auth-provider:  net.sourceforge.guacamole.net.basic.BasicFileAuthenticationProvider
~~~

Vinculación del directorio:
~~~
sudo ln -s /etc/guacamole /usr/share/tomcat9/.guacamole
~~~

## Configuración del método de autenticación
Se genera el hash para el usuarios utilizado para inicial sesion en la web:
~~~
echo -n userguaca | openssl md5
(stdin)= 59b65a7e86fd49f3e97795e5c4c6d4f7

printf '%s' userguaca | md5sum
~~~

Se crea el fichero de configuración /etc/guacamole/user-mapping.xml con el sigueinte contenido:
~~~
<user-mapping>
        
    <!-- Per-user authentication and config information -->

    <!-- A user using md5 to hash the password
         guacadmin user and its md5 hashed password below is used to 
             login to Guacamole Web UI-->
    <authorize 
            username="guacadmin"
            password="59b65a7e86fd49f3e97795e5c4c6d4f7"
            encoding="md5">

        <!-- First authorized Remote connection -->
        <connection name="CentOS-Server">
            <protocol>ssh</protocol>
            <param name="hostname">192.168.56.156</param>
            <param name="port">22</param>
        </connection>

        <!-- Second authorized remote connection -->
        <connection name="Windows 7">
            <protocol>rdp</protocol>
            <param name="hostname">192.168.56.122</param>
            <param name="port">3389</param>
            <param name="username">koromicha</param>
            <param name="ignore-cert">true</param>
        </connection>

    </authorize>

</user-mapping>
~~~

Y se reinician los sistemas:
~~~
sudo systemctl restart tomcat9
sudo systemctl restart guacd
~~~

Si no funciona, ver log:
~~~
cat /var/log/syslog
cat /var/log/tomcat9/CATALINA*
~~~

Y, finalmente, Guacamole está listo en la siguiente URL:
http://<IP>:8080/guacamole/#/

