---
title: Primeros pasos en MongoDB
author: Paloma R. García Campón
date: 2020-09-13 00:00:00 +0800
categories: [Bases de datos]
tags: [MongoDB]
---

MongoDB es un sistema de base de datos NoSQL, orientado a documentos y de código abierto.

# Instalación de MongoDB en Linux
La instalación del gestor de base de datos Mongo se va a realizar sobre un sistema Debian 9. En la [página oficial de Mongo](https://docs.mongodb.com/manual/administration/install-on-linux/) se encuentran los pasos fundamentales para las distintas distribuciones de Linux. Los comandos necesarios son los siguientes:
~~~
sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv 9DA31620334BD75D9DCB49F368818C72E52529D4

echo "deb http://repo.mongodb.org/apt/debian stretch/mongodb-org/4.0 main" | sudo tee /etc/apt/sources.list.d/mongodb-org-4.0.list

sudo apt-get update

sudo apt-get install -y mongodb-org

sudo apt-get install libcurl3 openssl
~~~

Para continuar la instalación es necesario descargar el paquete desde la página de oficial de Mongo.
~~~
tar -zxvf mongodb-src-r4.4.0.10.tar.gz

export PATH=mongodb-src-r4.4.0.10/bin:$PATH
~~~

Por último, para iniciarlo se usan los comandos:
~~~
service mongod start

service mongod stop

service mongod restart

mongo
~~~

# Importación de documentos
Empleando la utilidad mongoimport introduce los documentos correspondientes a las colecciones productos y zips (códigos postales).

Para usar el comando mongoimport tenemos que indicar una serie de parámetros:
* --db: indica la base de datos donde queremos guardar los datos.
* --collection: indica la colección o tabla donde se guardarán los datos.
* --drop: para borrar todos los datos que hubiese en la colección.
* --file: indica dónde está el fichero en formato json.

~~~
mongoimport --db practica --collection producto --file /home/paloma/CICLO/BASE\ DE\ DATOS/Mongo/Products.json 

mongoimport --db --file practica --collection /home/paloma/CICLO/BASE\ codpostal DE\DATOS/Mongo/zips.json
~~~

Pero antes hay que descargar el paquete de herramientas de MongoDB desde la página oficial.

# Introducción en el manejo de datos
En MongoDB se trabaja con colecciones que, en una especie de traducción, son las tablas en los gestores de bases de datos relacionales. Pero estas colecciones no poseen columnas concretas, sino etiquetas y los valores de estas etiquetas, que a su pueden ser otras etiquetas con otros valores.

Los datos se introducen en los esquemas con insert seguido de los datos en esquema json:
~~~
db.<coleccion>.insert({"<etiqueta>":"<valor>"})
~~~

Para buscar entre los datos se utiliza find. Si se utiliza `pretty()` la salida de los datos es más amigable:
~~~
db.<coleccion>.find({"busqueda"}).pretty()
~~~

Para hacer búsquedas donde un valor sea a 10 (siendo 10 un ejemplo):
~~~
db.<coleccion>.find({"<etiqueta>" : 10})
~~~

Para hacer una búsqueda de un valor mayor a otro se utiliza `$gt` y para que sea menor `$lte`. En el siguiente ejemplo se buscan los valores menores a 10:
~~~
db.productos.find({"<etiqueta>" : {$lte : 10}})
~~~

También se pueden contar los casos en los que un valor coinciden con la búsqueda con count. Por ejemplo, contar los valores donde un valor es menor a 10:
~~~
db.<coleccion>.count({"<etiqueta>" : {$lte : 10}})
~~~

Imaginemos que queremos mostrar dos etiquetas concretas (etiqueta1 y etiqueta2) de los elementos cuya etiqueta valor sea 10:
~~~ 
db.<coleccion>.find({"valor" : 10}, {"etiqueta1" : 1, 
                                     "etiqueta2" : 1}).pretty()
~~~

Si se quiere elegir la cantidad de salidas que queremos que aparezcan se utiliza limit(<numero>). Por ejemplo, mostrar los dos primeros datos que coincidan con la búsqueda:
~~~
db.<coleccion>.find({"<etiqueta>":"<valor>"}).limit(2).pretty()
~~~

Esto es muy útil si además ordenamos los datos con sort. Por ejemplo, enseñar las etiqueta name de los dos elementos cuya etiqueta valor esan las dos mayores:
~~~
db.<coleccion>.find({"name":1}).limit(2).sort({"valor":-1}).pretty()
~~~


## Casos prácticos
### Introducir un registro con los datos de tu teléfono móvil.
~~~
db.productos.insert({"id" : "XMi8", 
                     "name" : "Mi 8", 
                     "brand" : "Xiaomi", 
                     "type" : "phone", 
                     "price" : 300, 
                     "warranty_years" : 1, 
                     "available" : true  })
~~~      

### Introducir un registro con los datos de tu tarifa.
~~~
db.productos.insert({"id" : "SyT7", 
                     "name" : "Phone Service 7", 
                     "type" : "service", 
                     "monthly_price" : 7, 
                     "limits" : { "voice" : { "units" : "minutes", 
                                             "n" : 100, 
                                             "over_rate" : 0.05 }, 
                                 "data" : { "n" : "1,5GB", 
                                         "over_rate" : 0 }, 
                                 "sms" : { "n" : "unlimited", 
                                         "over_rate" : 0 } }, 
                     "sales_tax" : true, 
                     "term_years" : 2 })
~~~

### Mostrar los documentos correspondientes a productos cuyo precio es 200.
~~~
db.productos.find({"price" : 200}).pretty()
~~~

### Mostrar el número de productos de precio menor o igual que 10.
~~~
db.productos.count({"price" : {$lte : 10}})
~~~

### Mostrar el nombre y precio de los productos cuyo tipo es teléfono.
~~~
db.productos.find({"type" : "phone"}, {"name" : 1, 
                                       "price" : 1, 
                                       "_id" : 0}).pretty()
~~~

### Mostrar los datos de los cargadores que sirven para el teléfono AC3.
~~~
db.productos.find({"type" : "charger", 
                   "for" : "ac3"}).pretty()
~~~

### Mostrar las tarifas que permiten un tráfico de datos ilimitado.
~~~
db.productos.find({"type" : "service", 
                   "limits.data.n" : "unlimited"}).pretty()
~~~

### Mostrar el nombre de los dos teléfonos más caros.
~~~
db.productos.find({"type":"phone"}, {"name":1, 
                                     "_id":0}).limit(2).sort({"price":-1}).pretty()
~~~

### Mostrar el nombre de la ciudad más poblada de Alabama.
~~~
db.codpostal.find({"state":"AL"}, {"city":1, 
                                   "_id":0}).limit(1).sort({"pop":-1}).pretty()
~~~

### Mostrar el nombre de las ciudades de Michigan situadas al norte del paralelo 46.
~~~
db.codpostal.find({"state": "MI", 
                   "loc.1":{$gt:46}}, {"city": 1,
                                       "_id":0})
~~~

### Mostrar el número de ciudades agrupadas por estado.
~~~
db.codpostal.aggregate([{$group:{_id:"$state",
                                 ciudades:{$sum:1}}}])
~~~

### Mostrar la ciudad más poblada de cada estado.
~~~
db.codpostal.aggregate([{$group:{_id:"$state",
                                 ciudad:{"$city":1}}}])
~~~

### Mostrar la población total de cada estado.
~~~
db.codpostal.aggregate([{$group:{_id:"$state",
                                 poblacion:{$sum:"$pop"}}}])
~~~

