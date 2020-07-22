---
title: Introducción a la administración de PostgreSQL
author: Paloma R. García Campón
date: 2020-07-03 00:00:00 +0800
categories: [Bases de datos]
tags: [PostgreSQL]
---

## Privilegios del sistema en PostgreSQL
Los privilegios de sistema en Postgresql se deteminan a través de los roles, que a su vez son los usuarios y pueden tener asignados otros roles. 

Para realizar una correcta administración de los privilegios del sistema sobre los usuarios se aconseja agrupar los usuarios en diferentes roles siendo la diferencia principal entre los usuarios y los roles de grupo el privilegio de login que tendrían los primeros. 

Estas opciones pueden asignarse a los roles en la propia creación del rol o con posterioridad:
~~~
CREATE ROLE <nombre_rol> 
  WITH <opcion>;
ALTER ROLE <nombre_rol> 
  WITH <opcion>;
~~~

Algunas de estas opciones, y sus contrario, son:
- SUPERUSER/NOSUPERUSER: se agregan los privilegios de superusuario.
- CREATEDB/NOCREATEDB: opción para crear bases de datos.
- CREATEROLE/NOCREATEROLE: opción para crear nuevos roles.
- VALID UNTIL: indica la expiración del rol/usuario. 
- [ENCRYPTED] PASSWORD: asigna una contraseña al rol/usuario.
- LOGIN/NOLOGIN: opción para crear sesiones.
- INHERIT/NOINHERIT: opción para determinar si hereda los privilegios de los roles de los que es miembro.
- REPLICATION/NOREPLICATION: opción para controlar la transmisión.
- BYPASSRL/NOBYPASSRLS: opción para omitir los sistemas de seguridad de fila de las tablas.
- CONNECTION LIMIT: limita el número de sesiones concurrentes. 
- IN ROLE: opción para indicar los roles de los que formará parte.
- ADMIN: opción para indicar el rol o los roles de los formará parte con derecho a agregar a otros roles en este. 

## Asignar y revocar privilegios sobre una tabla concreta
La sintaxis que se utiliza para otorgar privilegios sobre una tabla en Postgrest es:
~~~
GRANT <nombre_privilegio> 
  ON <nombre_tabla> 
  TO <nombre_rol | nombre_grupo_rol | PUBLIC> 
  [WITH GRANT OPTION]
~~~

Los roles que pueden otorgar estos privilegios son:
- Rol superusuario.
- Rol propietario de la tabla.
- Rol que ha recibido el privilegio con la opción WITH GRANT OPTION.

Para revocar privilegios de una tabla concreta la sintaxis es:
~~~
REVOKE <nombre_privilegio> 
  ON <nombre_tabla> 
  FROM <nombre_rol | nombre_grupo_rol | PUBLIC>
~~~

También se puede eliminar la opción WITH GRANT OPTION, no el privilegio, de un rol concreto con GRANT OPTION FOR.


## Rol en PostgreSQL y diferencias con OracleDB
Una de las diferencias entre ORACLE y Postgres es que en el segundo, aunque en ocasiones se utiliza el término usuario, solo se trabaja con roles. Mientras que en ORACLE los roles son grupos de usuarios o/y de otros roles, en Postgres los roles son los propietarios de las bases de datos y pueden estar compuestos a su vez de otros roles. 

La creación de roles y la asignación de privilegios sobre objetos a estos comparten la misma sintaxis en ambos gestores:
~~~
CREATE ROLE <nombre_rol>;
GRANT <privilegio> 
  ON <nombre_objeto> 
  TO <nombre_rol>;
~~~

Mientras que la asignación de roles a otros roles es igual, en Postgres no se permite asignar roles a un usuario porque no existe este término.

> Asignación de un rol a un usuario en ORACLE: 
~~~
GRANT <nombre_rol> 
  TO <nombre_usuario>;
~~~

> Asignación de un rol a otro rol en ORACLE y Postgres:
~~~
GRANT <nombre_rol> 
  TO <nombre_rol>;
~~~

Para listar los roles, y los privilegios de estos, en ORACLE se debe consultar, en el diccionario de datos, las vistas DBA_ROLES, DBA_ROLE_PRIVS y ROLE_ROLE_PRIVS. En Postgres se utiliza \du+.

## Perfil en PostgreSQL y diferencias con OracleDB
En Postgres no existe el concepto perfil porque, mientras este término en ORACLE trabaja sobre los usuarios, en Postgres todas las delimitaciones se resuelven sobre los objetos.


## Casos pŕacticos
### Consultas al diccionario
A continuación, se va a realizar una consulta al diccionario de datos de PostgreSQL para averiguar todos los privilegios que tiene un usuario concreto:
~~~
select 'Privilegio '||PRIVILEGE_TYPE||
       ' en la tabla '||TABLE_NAME||
       ' del esquema '||TABLE_SCHEMA||
       ' de la base de datos '||TABLE_CATALOG
  from INFORMATION_SCHEMA.TABLE_PRIVILEGES 
  where GRANTEE=<nombre_rol>;
~~~

Consulta para averiguar qué usuarios pueden consultar una tabla concreta:
~~~
select GRANTEE
  from INFORMATION_SCHEMA.TABLE_PRIVILEGES
  where TABLE_NAME = <nombre_table>
  and PRIVILEGE_TYPE = 'SELECT';
~~~

### Crear un usuario llamado becario y dar privilegios
Creación del usuario con la opción contraseña:
~~~
create user BECARIO
  with password 'BECARIO';
~~~

Creación de una base de datos cuyo propietario es becario:
~~~
create daatabase BDBECARIO;
grant CONNECT 
  on database BDBECARIO 
  to BECARIO;
~~~

### Conectarse a la base de datos
Al crear el "usuario"/"rol" con CREATE USER se agrega la opción login. Pero si el usuario se ha creado con CREATE ROL habría que añadir dicha opción para que pueda crear sesiones:
~~~
alter role BECARIO
  with LOGIN;
~~~

Además, hay que modificar el fichero /etc/postgresql/11/main/pg_hba.conf de tal forma que la opción para el método de loguearse en local sea md5:
~~~
local   all             all                                     md5 
~~~

Y se inicia sesión con las opciones -U (usuario):
~~~
psql -U <usuario> -d <base_datos> -h <direccion_host> -W
~~~

~~~
postgres@servidor:~$ psql -U becario -d bdbecario -h localhost -W
Password: 
psql (11.5 (Debian 11.5-1+deb10u1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
ario: 
psql (11.5 (Debian 11.5-1+deb10u1))
Type "help" for help.

bdbecario=> 
~~~


### Insertar filas en SCOTT.EMP, privilegio que podrá pasarlo a quien quiera
~~~
grant insert on SCOTT.EMP 
  to BECARIO
    with grant option;
~~~


### Crear objetos en cualquier tablespace
En Postgres se puede dar permisos sobre un tablespace concreto con la siguiente sintaxis:
~~~
GRANT CREATE ON TABLESPACE <nombre_tablespace> 
  TO becario;
~~~


### Gestión completa de usuarios, privilegios y roles
Hay que asignar al rol la opción de superuser:
~~~
alter role BECARIO
  with SUPERUSER;
~~~

Pero esta opción otorga más permisos de los que se pide en el enunciado. Por lo tanto, lo correcto sería otorgarle los privilegios específicos que serían:

Para administrar roles:
~~~
alter role BECARIO
  with CREATEROLE;
~~~

Para otorgar el privilegio de login y de crear bases de datos a otros usuarios hay que añadir las siguientes opciones al usuario:
~~~
alter role BECARIO
  with LOGIN
  with grant option;
alter role BECARIO
  with CREATEDB
  with grant option;
~~~


### Consulta que un script para quitar el privilegio de borrar registros en alguna tabla de SCOTT a los usuarios que lo tengan
En Postgres no se puede indicar el usuario propietario de las tablas. Por el contrario, permite filtrar las tablas por la base de datos o/y el esquema a la que pertenece.
~~~
select 'REVOKE DELETE ON '||table_catalog||'.'||table_name||' from '|| grantee||';'
  from INFORMATION_SCHEMA.ROLE_TABLE_GRANTS
  where PRIVILEGE_TYPE='DELETE'
  [and TABLE_CATALOG='SCOTT']
  [and TABLE_SCHEMA='SCOTT'];
~~~


### Procedimiento que recibe un nombre de usuario y muestra cuántas sesiones tiene abiertas en este momento
~~~
create or replace function MostrarSesiones (p_user varchar)
  returns void as $$
  declare
    v_cont numeric:=0;
    c_sesiones cursor is
	    select to_char(BACKEND_START,'DD/MM/YYYY HH24:MI') HORA,
             CLIENT_HOSTNAME, 
             APPLICATION_NAME
	      from PG_STAT_ACTIVITY
	      where USENAME=lower(p_user);
  begin
    for v_cursor in c_sesiones loop
        raise notice 'Hora: %', v_cursor.HORA;
        raise notice 'Maquina: %', v_cursor.CLIENT_HOSTNAME;
        raise notice 'Programa: %',v_cursor.APPLICATION_NAME;
        raise notice ' ';
        v_cont:=v_cont+1;
    end loop;
    raise notice 'Numero de sesiones abiertas: %', v_cont;
end;
$$ language plpgsql;

select MostrarSesiones('postgres');
~~~




