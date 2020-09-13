---
title: Configurar el protocolo HTTPS en un servidor nginx
author: Paloma R. García Campón
date: 2020-09-13 00:00:00 +0800
categories: [Seguridad]
tags: [HTTP, HTTPS, Nginx]
---

HTTPS es un protocolo que permite establecer una conexión segura entre el servidor y el cliente, impidiendo que pueda ser interceptada por una persona no autorizada. En este post se va a explicar cómo configurarlo en un servidor nginx con dos aplicaciones web.

# Certificado wildcard
Se ha creado un certificado wildcard que ha sido firmada por una entidad certificadora. Para crear el certificado, en primer lugar se instala la herramienta openssl:
~~~
sudo dnf install mod_ssl
~~~

Se crea la clave con openssl con la siguiente sintaxis:
~~~
sudo openssl genrsa -out <nombre_clave>.pem 4096
~~~

En nuestro caso:
~~~
sudo openssl genrsa -out gonzalonazareno.pem 4096
Generating RSA private key, 4096 bit long modulus (2 primes)
........................................++++
................................................................................................................................++++
e is 65537 (0x010001)
~~~

Y se crea un fichero de configuración para la configuración del certificado:
~~~
[req]
default_bits = 4096
prompt = no
default_md = RSA
distinguished_name = dn

[ dn ]
C=<iniciales_pais>
ST=<ciudad>
L=<localidad>
O=<organización>
OU=<departamento>
emailAddress=<correo>
CN = <direccion_web>
~~~

Siguiendo el esquema anterior, en nuestro caso este es el fichero que ha resultado:
~~~
[req]
default_bits = 4096
prompt = no
default_md = RSA
distinguished_name = dn

[ dn ]
C=ES
ST=Sevilla
L=Dos Hermanas
O=IES Gonzalo Nazareno
OU=Informatica
emailAddress=palomagarciacampon08@gmail.com
CN = *.paloma.gonzalonazareno.org
~~~

Utilizando este fichero se ha a crear el fichero .csr con la siguiente sintaxis:
~~~
sudo openssl req -new -sha256 -nodes -out <nombre_csr>.csr -newkey rsa:4096 -keyout <clave> -config <fichero_configuracion>
~~~

En nuetro caso la instrucción es la sigueinte es la siguiente:
~~~
sudo openssl req -new -sha256 -nodes -out paloma.gonzalonazareno.org.csr -newkey rsa:4096 -keyout gonzalonazareno.pem -config paloma.gonzalonazareno.org.conf
~~~

Este certificado se envía a la unidad certificadora.

Tras obtener el certificado firmado se mueve a **/etc/pki/tls/certs/** y la clave a **/etc/pki/tls/private**:
~~~
sudo mv paloma.gonzalonazareno.org.c* /etc/pki/tls/certs/
sudo mv gonzalonazareno.pem /etc/pki/tls/private/
~~~

Y se modifican las siguientes líneas del fichero de configuración de Nginx indicando las rutas del certificado y la clave y redirigiendo el servicio http al https:
~~~
server {
    listen 80;
    server_name  www.paloma.gonzalonazareno.org;
    rewrite ^ https://$server_name$request_uri permanent;
}

server {
    listen 443;
    server_name  www.paloma.gonzalonazareno.org;
    ssl on;
    ssl_certificate /etc/pki/tls/certs/paloma.gonzalonazareno.org.crt;
    ssl_certificate_key /etc/pki/tls/private/gonzalonazareno.pem;
~~~

Y se descomenta toda la parte relacionada con https del fichero de configuración de nginx, **/etc/nginx/nginx.conf**, además de indicar las rutas del certificado y la clave:
~~~
    server {
        listen       443 ssl http2 default_server;
        listen       [::]:443 ssl http2 default_server;
        server_name  _;
        root         /usr/share/nginx;

        ssl_certificate "/etc/pki/tls/certs/paloma.gonzalonazareno.org.crt";
        ssl_certificate_key "/etc/pki/tls/private/gonzalonazareno.pem";
        ssl_session_cache shared:SSL:1m;
        ssl_session_timeout  10m;
        ssl_ciphers PROFILE=SYSTEM;
        ssl_prefer_server_ciphers on;
        location / {
        }

        error_page 404 /404.html;
            location = /40x.html {
        }

        error_page 500 502 503 504 /50x.html;
            location = /50x.html {
        }
    }

}
~~~

# Configuración SELinux y firewall
Si está habilitado SELinux, por ejemplo si se está trabajando en un CentOS, hay que indicarle a éste que debe permitir que Ningx pueda acceder al directorio que contiene la clave y el certificado:
~~~
sudo restorecon -v -R /etc/pki/tls/
~~~

En el caso de que el firewall se interponda en la conexión:
~~~
sudo firewall-cmd --zone=public --add-port=443/tcp --permanent
sudo firewall-cmd reload
sudo systemctl restart firewalld
~~~