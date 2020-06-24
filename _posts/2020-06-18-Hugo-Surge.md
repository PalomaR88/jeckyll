---
title: Creando una página con Surge y Hugo
author: Paloma R. García Campón
date: 2020-06-18 00:00:00 +0800
categories: [Implantación de aplicaciones web]
tags: [Surge, Hugo]
---

![surge](/assets/img/sample/surge-hugo/surge.png)

![hugo](/assets/img/sample/surge-hugo/hugo.png)

Imaginemos que queremos crear un sitio web con páginas estáticas como puede ser un blog. Con conocimientos mínimos de HTML no es muy complicado, ¿verdad?. No, no es necesario conocer HTML, además, mantener el sitio se convertiría en un trabajo tedioso. Para eso hay multitud de herramientas que nos pueden ayudar con nuestro proyecto. Pero si lo que se quiere es crear un sitio con páginas estáticas la mejor opción es elegir un generador de páginas estáticas. Hay multitud de aplicaciones para ello como pueden ser Wordpress, Jekyll o, la que se explica a continuación, Hugo que crea las páginas a través de ficheros markdown. Y, una vez creemos nuestro sitio, ¿donde lo colocamos?. Para el despliegue se usará Surge, aunque esta elección de herramientas permite infinidad de combinaciones.

## Instalación del Hugo
En la [página oficial de Hugo](https://gohugo.io/) se encuentra toda la documentación necesaria para su instalación. A continuación se indican los pasos fundamentales:

Instalación:
~~~
sudo apt-get install hugo
~~~

Creación del primer sitio:
~~~
$ hugo new site <nombre_del_sitio>
Congratulations! Your new Hugo site is created in /home/paloma/DISCO2/CICLO II/IMPLANTACIÓN DE APLICACIONES WEB/1.Generador_pag_estatica/quickstart.

Just a few more steps and you're ready to go:

1. Download a theme into the same-named folder.
   Choose a theme from https://themes.gohugo.io/, or
   create your own with the "hugo new theme <THEMENAME>" command.
2. Perhaps you want to add some content. You can add single files
   with "hugo new <SECTIONNAME>/<FILENAME>.<FORMAT>".
3. Start the built-in live server via "hugo server".

Visit https://gohugo.io/ for quickstart guide and full documentation.
~~~

Lo más sencillo para instalar un tema en nuestro sitio web es [buscar en el repositorio oficial](https://themes.gohugo.io/) y copiar el repositorio. A continuación, indicar el tema que se va a utilizar en el fichero config.yoml. En este ejemplo el tema usado se llama *terrassa*.
~~~
$ $ git submodule add https://github.com/danielkvist/hugo-terrassa-theme.git terrassa
Clonando en '/$USER/quickstart/themes/terrassa'...
remote: Enumerating objects: 5, done.
remote: Counting objects: 100% (5/5), done.
remote: Compressing objects: 100% (5/5), done.
remote: Total 1416 (delta 0), reused 1 (delta 0), pack-reused 1411
Recibiendo objetos: 100% (1416/1416), 4.15 MiB | 3.55 MiB/s, listo.
Resolviendo deltas: 100% (764/764), listo.
$ echo 'theme = "hugo-terrassa-theme"' >> config.toml
~~~

En el fichero **config.toml** se modifican los parámetros principales para configurar el sitio web:
~~~
baseURL = "/"
title = "Un pez llamado Wanda"
author = "Paloma R."
googleAnalytics = ""
defaultContentLanguage = "en"
language = "en-US"
paginate = 3

theme = "hugo-terrassa-theme"
...
~~~


Para poblar el blog de entradas se añaden los ficheros con extensión .md en el directorio /content/posts/ y utilizar el comando hugo para generar las páginas en HTML:
~~~
$ hugo

                   | EN  
+------------------+----+
  Pages            | 38  
  Paginator pages  |  6  
  Non-page files   |  0  
  Static files     |  3  
  Processed images |  0  
  Aliases          |  6  
  Sitemaps         |  1  
  Cleaned          |  0  

Total in 82 ms
~~~

## Instalación de surge
De nuevo, en el [sitio oficial de Surge](https://surge.sh/) se encuentra la documentación necesaria para la sencilla instalación de esta herramienta:
~~~
$ sudo npm install --global surge
npm WARN npm npm does not support Node.js v10.15.2
npm WARN npm You should probably upgrade to a newer version of node as we
npm WARN npm can't make any promises that npm will work with this version.
npm WARN npm Supported releases of Node.js are the latest release of 4, 6, 7, 8, 9.
npm WARN npm You can find the latest version at https://nodejs.org/
/usr/local/bin/surge -> /usr/local/lib/node_modules/surge/lib/cli.js
+ surge@0.21.3
added 137 packages from 113 contributors in 22.737s
~~~

Y para desplegar el sitio web solo hay que introducir el comando surge y completar los datos que se piden.

## Automatización
Cuando se quiera modificar el sitio web se deberán volver a usar los comando hugo y surge además de guardar los cambios en el repositorio. Para automatizar todas estas acciones se ha generado un sencillo script:

~~~
#! bin/sh

commit=$(git commit -am 'nuevo' | egrep 'content/')
touch ../nuevo.txt
echo $commit > ../nuevo.txt

git add $(cat ../nuevo.txt)
git commit -m 'añado'
git push

rm ../nuevo.txt

hugo
surge public/ el_pez_llamado_wanda.surge.sh
~~~

## Resultado
Los ficheros de configuración resultantes se encuentran en el [siguiente repositorio de GitHub]( https://github.com/PalomaR88/El_pez_llamado_Wanda "El pez llamado Wanda(GitHub)"). Y [El_pez_llamado_Wanda]( http://el_pez_llamado_wanda.surge.sh/ "El pez llamado Wanda") es la sitio web que se ha creado.
