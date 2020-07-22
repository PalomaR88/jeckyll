---
title: Introducción a la administración de Oracle DataBase 
author: Paloma R. García Campón
date: 2020-06-25 00:00:00 +0800
categories: [Bases de datos]
tags: [OracleDB]
---

## Arquitectura general de Oracle
El [Instituto Nacional Estadounidense de Estánderes, ANSI](https://www.ansi.org/) definió la arquitectura que todo sistema gestor de bases de datos deben tener en 3 niveles:
- **Nivel externo o de visión**: Compuesto por la parte de la base de datos que ve un usuario normal. Un usuario no la se completa, solo lo que el usuario necesita. Esto se implementa en la realidad a base de permisos. Cuando se entra en el diccionario de datos desde el usuario system, seguimos siendo usuarios normales y solo vemos vistas de las tablas del diccionario.
- **Nivel conceptual**: Es el nivel que conocen los programadores. Son "tablas" que los usuarios administradores pueden ver a través de vistas. 
- **Niver interno**: DBA, la base de datos ya no son tablas, sino ficheros. Normalmente un DBA sabe como están mapeadas las tablas según los discos.


### Ventajas
- **Independencia lógica**: Se puede cambiar tablas, crear columnas, etc. Sin ser necesario reescribir todas las aplicaciones. Y lo que funcionaba seguirá funcionando.
- **Independencia física**: Esto permite cambier la ubicación de los ficheros que contienen los datos sin que se vean afectadas las aplicaciones. De esta forma, si, por ejemplo, se llena un disco, se puede añadir otro e ir guardando las nuevas tablas en el nuevo disco y el gestor es el que se va a encargar de localizar esa tabla entre los discos. 



## Componentes de un SGBD
### Lenguaje
Lo que permite:
- **DDL**: Crear la estructura. 
- **DML**: Consultar y manipular la información almacenada en la base de datos. 
- **DCL**: Asignar privilegios a usuarios, confirmar o abortar transacciones, etc. 
- En algunos casos, también incluyen formas de manipular aplicaciones. Desarrollo rápido de aplicaciones como Oracle Developer Suite.


### El diccionario de datos
El diccionario de datos contiene los metadatos (datos acerca de los datos) de la base de datos:
- La definición de todos los objeteos existentes en la base de datos: tablas con sus columnas, vistas, procedimientos, triggers, índice, etc...
- La ubicación física de los objetos y el espacio asignado a los mismos..
- Los privilegios y los roles asignados a los usuarios.
- Restricciones de las tablas.
- Etc...

En [ss64](https://ss64.com/ora/) se explican las vistas existentes para gestionar la base de datos en Oracle. Algunas de estas vistas son:

|Nombe de la vista|Despcripción
|----------|----------------------------------------------|
|DBA_TABLES|Se ven todas las tablas de todos los usuarios.|
|ALL_TABLES|Se ven todas las tablas que puedes ver, las que son tuyas y a las que puedes acceder.|
|USER_TABLE|Las tablas de las que eres propietario.|


### Mecanismos de seguridad
- Copias de seguridad.
- Garantizar la proteccion de los datos.
- Restricciones de integridad.
- Recurperar la base de datos hasta un estado consistente en caso de error del sistema. Para esto se neceista un archivo de log, para guardar las acciones que se han realizado y modificado la misma desde la última copia de seguridad. 
- Controlar el acceso concurrente de los usuarios apra evitar errores de integridad. 


### El factor humano
- Usuarios finales.
- Programadores.
- Administradores o DBAs: garantizan el correcto funcionamiento de la base de datos y  gestionan todos los recursos. Tienen el nivel más alto de provilegios y responsabilidades legales en caso de que los datos tengan alǵun tipo de protección. Su objetivo es que las bases de datos estén siempre disponible y con un rendimiento óptimo. 



## Tareas de un DBA
- Decidir el SGBD idóneo, instalarlo y configurarlo inicialmente.
- Supervisar diseño lógico de la BD.
- Realizar diseño físico de la BD: Estructura de almacenamiento.
- Crear y mantener el esquema de la BD.
- Crear y mantener cuentas de usuarios.
- Colaborar en la formación de usuarios y programadores.
- Detectar y resolver problemas de rendimiento de la BD usando herramientas de monitorización.
- Realizar copias de seguridad, migraciones, importaciones y exportaciones, auditorias de seguridad, etc...
- Recuperar isntancias dañadas.
- Instalar y configurar middleware de la BD.



## Arquitectura interna
### Archivos
- **Archivos de datos**: las tablas, el diccionario de datos y el segmento de rollback. Las tablas se agrupan en tablespaces y estos a su vez se reparten en los datafiles. Por defecto Oracle trae una serie de tablespaces donde se guardan cierta información (diccionario de datos y usuarios) estos dos tablespaces se recomiendan que estén en discos distintos.
- **Archivos de configuración**: `.ora`. Hay parámetros y valor asociado. 
- **Archivos de control**: `.ctl`. Se consultan mediante vistas dinámicos y solo son modificados por el servidor. Mantienen la integridad de la BS. 
- **Archivos de log**: Se registra los cambios que se han producido. 


### Memoria
- **Cache de instrucciones**: Si la instruccion está cacheada se ahorra compilarla.
- **Cache de datos**: Lo mismo que lo anterior pero con los datos, también hace lo mismo con datos del diccionario de datos.
- **Cache redo log**: Se guarda lo que se va a guardar en el log, pero se cachea la información hasta que tiene bastante y entonces se vuelca en disco. 


### Vistas dinámicas de rendimiento
Para ver el valor actual de un parámetro de inicialización de arranque se puede usar el comando show parameter.

## Instancias
Una instancia es una base de datos que se está ejecutando.

En la instalación se pregunta:
- DLTP: va a tener muchas consultas pero muy sencillas. Se usan en internet.
- Data Mining: bases de datos en las que hay pocas consultas al dia pero muy complejas. Se usan en soportes de tomas de decisiones.
- Proposito general: el término intermedio.

Según la opción que se elija se otorgan determinados parámetros por defecto. 



## Esquema de usuario en ORACLE
> Esquema: conjunto de objetos que es propiedad de un usuario concreto. 


### Creacion, modificacion y borrado de usuarios
Crear usuario:
~~~
CREATE USER <user>
  IDENTIFIED BY <password>
  [DEFAULT TABLESPACE <tablespace>] # se el tablespace que usará.
  [TEMPORY TABLESPACE <tablespace_temporal>] # los tablespace temporales son determinadas acciones que se guardan en tablespace porque la memoria del equipo a veces no da abasto.
  [QUOTA {<num>{K|M} | unlimited} 
    ON <nombre_tablespace>]
  [PROFILE <perfil>]; # conjunto de límites de cuánto puede usar o conjunto de características que van a tener la contraseña (camibar semanalmente, los caracteres, etc). Los perfiles deben de crearse con anterioridad.
~~~

Modificar:
~~~
ALTER USER <user>
~~~

Borrar:
~~~
DROP USER <user> [CASCADE];
~~~


### Vistas con información de usuarios
|Vista|Descripción|
|-----|-----------|
|DBA_USERS|Todos los usuarios.|
|DBA_TS_QUOTAS|Cuotas de tablespace de los usuarios.|


### Asignacion y revocación de privilegios
Hay dos tipos de privilegios:

**Privilegios sobre objetos**
~~~
GRANT <privilegios> 
    ON <propietario>.<objeto> 
    TO [<usuario> | <rol> | PUBLIC] 
      [WITH ADMIN OPTION]; --> con esta opción puede darle la opción de repartir, a otro usuarios, el permiso al usuario al que se lo esta otorgando.
~~~

**Privilegios del sistema**
Permite realizar determinadas operaciones que no están asociadas a objetos, por ejemplo crear tablas, es una accion pero no es de un objeto.
- create, alter, drop
- create any, alter any, drop any --> para hacer lo que sea con objetos que no son propios sino de otros usuarios. 
- select, insert, update o delete.

~~~
GRANT <privilegios> 
  TO [<usuario> | <rol> | PUBLIC] 
    [WITH ADMIN OPTION];
~~~

Quitar privilegios:
~~~
REVOKE <privilegio> 
  FROM [<usuario> | <rol>];
~~~


Vistas más importantes para un DBA:
|Vista|Descripción|
|-----|-----------|
|DBA_SYS_PRIVS|Todos los privilegios de sistema que se han concedido.|
|DBA_TAB_PRIVS|Todos los privilegios de objetos que se han concedido.|
|DBA_COL_PRIVS|Privilegios sobre columnas.|


### Gestion de roles
Son conjunto de privilegios, se pueden crear desde cero, pero por lo general es con roles predefinidos, los más importantes son:
connect, resource y dba.
- connect: el más básico.
- resource: los programadores. 
- DBA: de administración. Este no se suele usar.

Crear un rol: 
~~~
CREATE ROL <rol>;
~~~

Añadir privilegios: 
~~~
GRANT <privilegio>
  [ON <objeto>] 
  TO <rol>;
~~~

Para quitar privilegios: 
~~~
REVOKE <privilegio>
  FROM <rol>;
~~~

Para dárselo a un usuario: 
~~~
GRANT <rol>
  TO <user>;
~~~

Para quitárselo a un usuario: 
~~~
REVOKE <rol>
  TO <user>;
~~~

Para borrar el rol: 
~~~
DROP role <rol>:
~~~

Para añadir un rol a otro:
~~~
GRANT <rol>
  TO <rol>;
~~~

Vistas importantes con respecto a los roles:
|Vista|Descripción|
|-----|-----------|
|DBA_ROLES|Todos los roles.|
|DBA_ROLE_PRIVS|Roles concedidos a usuarios.|
|ROLE_ROLE_PRIVS|Roles concedidos a otros roles.|
|ROLE_SYS_PRIVS|Privilegios de sistema concedidos a los roles.|
|ROLE_TAB_PRIVS|Privilegios sobre objetos concedidos a los roles.|



## Gestión de perfiles
Cada usuario solo puede tener un perfil. Por defecto están desahibilitados. Para habilitarlos:
~~~
ALTER SYSTEM 
  SET RESOURCE_LIMIT=TRUE
~~~

Crear perfil:
~~~
CREATE PROFILE <nombreperfil> 
  LIMIT {<parametrorecurso>|<parametrocontraseña>} [<valor>[K|M] UNLIMITED]
  ...
~~~



## Caso práctico
### Creación de un usuario
Se crea un usuario llamado Becario:
~~~
create user BECARIO 
  identified by BECARIO;
~~~

Se le da permisos para conectarse a la base de datos:
~~~
grant create session 
  to BECARIO;
~~~


### Quota de registro fallido
Se modifica el número de errores en la introducción de la contraseña de cualquier usuario:
~~~
alter profile DEFAULT limit
  failed_login_attempts 1;
~~~

Se desbloquea la una cuenta que se ha bloqueado por la introducción incorrecta de la contraseña:
~~~ 
alter user BECARIO 
  identified by BECARIO account unlock;
~~~


### Modificación de los índices
Se añade el privilegio para modificar el ínidce en cualquier esquema, con privilegio para pasarlo a otros usuarios:
~~~
grant alter any index 
  to BECARIO 
    with admin option;
~~~

Comprobación:
~~~
SQL> select * from ALL_IND_COLUMNS where TABLE_NAME= 'EMP';


INDEX_OWNER INDEX_NAME TABLE_OWNER TABLE_NAME COLUMN_NAME COLUMN_POSITION COLUMN_LENGTH CHAR_LENGTH DESC

----------- ---------- ----------- ---------- ----------- --------------- ------------- ----------- -------

SCOTT       PK_EMP     SCOTT       EMP        EMPNO​                     1            22           0 ASC
~~~


### Insertar filas
Se agrega el privilegio para insertar filas en scott.emp, pudiendo pasarle el privilegio a otros usuarios.
~~~
grant insert
  on SCOTT.EMP
  to BECARIO
    with grant option;
~~~


### Crear objetos en cualquier tablespace
~~~
grant unlimited tablespace 
  to BECARIO;
~~~


### Gestión completa de usuario
Asignación de los privilegios sobre la gestión de usuario:
~~~
Introduzca el nombre de usuario: becario
Introduzca la contrase±a:

Conectado a:
Oracle Database 11g Enterprise Edition Release 11.2.0.1.0 - 64bit Production
With the Partitioning, OLAP, Data Mining and Real Application Testing options

SQL> create user PRUEBA identified by PRUEBA;

Usuario creado.

SQL> alter user PRUEBA identified by PASSWORD;

Usuario modificado.

SQL> drop user PRUEBA;

Usuario borrado.
~~~


### Gestión completa de privilegios
Asignación de los privilegios sobre la gestión de privilegios:
~~~
SQL> grant select on SCOTT.EMP to PRUEBA;

Concesi¾n terminada correctamente.

SQL> grant create any table to PRUEBA;

Concesi¾n terminada correctamente.
~~~


### Gestión completa de roles
Asignación de los privilegios sobre la gestión de roles:
~~~
SQL> create role BECARIOS;

Rol creado.

SQL> alter role BECARIOS identified globally;

Rol modificado.

SQL> drop role BECARIOS;

Rol borrado.
~~~


### Crear una consulta para obtener un script para quitar el privilegio de borrar registros en alguna tabla de SCOTT a los usuarios que lo tengan
~~~
select 'revoke delete on SCOTT.'||table_name||' from ' || grantee ||';'
  from DBA_TAB_PRIVS
  where OWNER='SCOTT'
  and PRIVILEGE='DELETE';
~~~


### Cración de un tablespace
Se va a crear un tablespace llamado TS2 con tamaño de extensión de 256K. 
~~~
create tablespace TS2 
  datafile 'tbs_ts2.dbf' 
  size 256k;
~~~

El siguiente script asigna un tablespace como tablespace por defeccto a los usuarios que no tienen privilegios para consultar ninguna tabla de SCOT, excepto a SYSTEM:
~~~
select 'alter user "'||username||'" default tablespace TS2;'
  from DBA_USERS
  where USERNAME!='SYSTEM'
  and USERNAME not in (select GRANTEE 
                         from DBA_TAB_PRIVS 
                         where PRIVILEGE='SELECT' 
                         and OWNER='SCOTT');
~~~


### Procedimiento de sesiones abiertas
El siguiente procedimiento recibe un nombre de un usuario y mmuestra cuántas sesiones tiene abiertas en este momento. Además, muestra la hora de comienzo, el nombre de la máquina, sistema operativo y programa desde el que fue abierta:
~~~
create or replace procedure MostrarSesiones(p_user varchar2)
  is
    v_numero number:=0;
    cursor c_sesiones
      is
        select to_char(LOGON_TIME, 'HH24:MI') as HORA, 
               MACHINE, 
               PROGRAM
          from v$session
          where USERNAME=p_user;
    v_cursor c_sesiones%ROWTYPE;
  begin
    for v_cursor in c_sesiones loop
      dbms_output.put_line('Hora: '||v_cursor.HORA||'Maquina: '||v_cursor.MACHINE||'Programa: '||v_cursor.PROGRAM);
      v_numero:=v_numero+1;
    end loop;
    dbms_output.put_line('Numero de sesiones abiertas: '||v_numero);
end MostrarSesiones;
/
~~~

~~~
exec MostrarSesiones('SYS');
~~~




### Procedimiento de concesión de privilegios
El siguiente procedimiento muestra los usuarios que pueden conceder privilegios de sistema a otros usuarios y cuáles son dichos privilegios:
~~~
create or replace procedure PrivilegiosDeAdmin
  is
    cursor c_cursor
      is
        select GRANTEE, PRIVILEGE
          from DBA_SYS_PRIVS 
          where ADMIN_OPTION='YES';
    v_cursor c_cursor%ROWTYPE;
  begin
    for v_cursor in c_cursor loop
        dbms_output.put_line('Usuario: '||v_cursor.GRANTEE||' - '||'Privilegio: '||v_cursor.PRIVILEGE);
    end loop;
end PrivilegiosDeAdmin;
/
~~~

~~~
exec PrivilegiosDeAdmin
~~~


### Añadir privilegios
Asignar privilegio para rastrear los ficheros de log y el estado de estos:
~~~
grant alter database to AYUDANTE;
~~~

Comprobación:
~~~
SQL> alter database clear logfile group 1;

Base de datos modificada.

SQL> archive log list
Modo log de la base de datos		  Modo de No Archivado
Archivado automático             Desactivado
Destino del archivo	       /opt/oracle/product/12.2.0.1/dbhome_1/dbs/arch
Secuencia de log en línea más antigua     0
Secuencia de log actual 	  21
~~~

Asignar privilegio para crear funciones de complejidad de contraseña y asignarlas a usuarios:
~~~
grant create any procedure
  to AYUDANTE;
grant alter profile
  to AYUDANTE;
~~~

Asignar privilegios para eliminar la información de rollback, pudiendo pasárselo a quien quiera:
~~~
grant drop rollback segment
  to AYUDANTE
    with grant option;
~~~

Asignar privilegios para modificar información existente en la tabla dept del usuario scott, pudiendo pasarselo a quien quiera:
~~~
grant update
  on SCOTT.DEPT
  to AYUDANTE
    with grant option;
grant select
  on SCOTT.DEPT
  to AYUDANTE;
~~~

Comprobación:
~~~
SQL> update SCOTT.DEPT set LOC='MADRID'
  2  where DEPTNO=40;

1 fila actualizada.

SQL> select * from SCOTT.DEPT;

    DEPTNO DNAME          LOC
---------- -------------- -------------
        10 ACCOUNTING     NEW YORK
        20 RESEARCH       DALLAS
        30 SALES          CHICAGO
        40 OPERATIONS     MADRID
~~~

Asignar privilegio para realizar pruebas de todos los procedimientos existentes:
~~~
grant execute any procedure 
  to AYUDANTE;
~~~


Asignar privilegio para poner un tablespace fuera de línea:
~~~
grant alter tablespace 
  to AYUDANTE;
~~~

Comprobación, como sysdba se crea un tablespace y como ayudante se modifica:
~~~
create tablespace PRUEBA
  datafile 'prueba.dat' 
    size 20M
  online;
~~~

~~~
SQL> alter tablespace PRUEBA offline;

Tablespace modificado.
~~~


### Script de asignación de privilegio
El siguiente script recibe el nombre de dos usuarios y el primero recibe los privilegios de inserción y modificación sobre todas las tablas del segundo y sobre los procedimientos de éste.
~~~
create or replace procedure OtorgarPrivilegios(p_user1 varchar2,
                                               p_user2 varchar2)
  is
    v_validacion1 number:=0;
    v_validacion2 number:=0;
  begin
    v_validacion1:=ComprobarUsuario(p_user1);
    v_validacion2:=ComprobarUsuario(p_user2);
    if v_validacion1=0 and v_validacion2=0 then
      PrivilegiosSobreTablas(p_user1, p_user2);
      PrivilegiosSobreProcedimientos(p_user1, p_user2);
    end if;
end OtorgarPrivilegios;
/

create or replace function ComprobarUsuario(p_user varchar2)
  return number
  is
    v_resultado varchar2(30);
  begin
    select USERNAME into v_resultado
      from DBA_USERS
      where USERNAME=p_user;
    return 0;
  exception
    when NO_DATA_FOUND then
      dbms_output.put_line('No existe el usuario '||p_user);
      return 1;
end ComprobarUsuario;
/


create or replace procedure PrivilegiosSobreTablas(p_user1 varchar2, 
                                                   p_user2 varchar2)
  is
    cursor c_tablas
      is
        select OBJECT_NAME
          from DBA_OBJECTS
          where OBJECT_TYPE='TABLE'
          and OWNER=p_user2;
    v_cursor c_tablas%ROWTYPE;
  begin
    for v_cursor in c_tablas loop
      dbms_output.put_line('GRANT INSERT ON '||p_user2||'.'||v_cursor.OBJECT_NAME||' TO '||p_user1||';');
      dbms_output.put_line('GRANT UPDATE ON '||p_user2||'.'||v_cursor.OBJECT_NAME||' TO '||p_user1||';');
    end loop;
end PrivilegiosSobreTablas;
/


create or replace procedure PrivilegiosSobreProcedimientos (p_user1 varchar2, 
                                                            p_user2 varchar2)
  is
    cursor c_procedimientos
      is
        select OBJECT_NAME
          from DBA_OBJECTS
          where OBJECT_TYPE='PROCEDURE'
          and OWNER=p_user2;
    v_cursor c_procedimientos%ROWTYPE;  
  begin
    for v_cursor in c_procedimientos loop
      dbms_output.put_line('GRANT EXECUTE ON '||p_user2||'.'||v_cursor.OBJECT_NAME||' TO '||p_user1||';');
    end loop;
end PrivilegiosSobreProcedimientos;
/
~~~

### Procedimiento para crear script que cree un rol
El siguiente procedimiento crea un script que crea un rol conteniendo todos los permisos que tenga el usuario cuyo nombre reciba como parámetro, hayan sido asignados de forma directa o a través de roles:

> Para este procedimiento se va a usar la función ComprobarUsuario del ejercicio anterior.
~~~
create or replace procedure CrearRol (p_user varchar2)
  is
    v_validacion number:=0;
    v_nuevoRol varchar(50):='BackupPrivs'||p_user;
  begin
    v_validacion:=ComprobarUsuario(p_user);
    if v_validacion=0 then
      dbms_output.put_line('CREATE ROLE BackupPrivs'||p_user);
      PrivilegiosDelSistema(p_user, v_nuevoRol);
      PrivilegiosSobreObjetos(p_user, v_nuevoRol);
    end if;
end CrearRol;
/



create or replace procedure PrivilegiosSobreObjetos(p_user varchar2, 
                                                    p_nuevoRol varchar2)
  is
    cursor c_privTab
      is
      select distinct PRIVILEGE, 
                      OWNER, 
                      TABLE_NAME
        from DBA_TAB_PRIVS
        where GRANTEE=p_user
        or GRANTEE in (select distinct granted_role
                         from DBA_ROLE_PRIVS
                         start with GRANTEE=p_user
                         connect by GRANTEE=prior GRANTED_ROLE);
    v_cursor c_privTab%ROWTYPE;
  begin
    for v_cursor in c_privTab loop
      AgregarPrivilegioSobreObjeto(v_cursor.PRIVILEGE, v_cursor.OWNER, v_cursor.TABLE_NAME, p_nuevoRol);
    end loop;
end PrivilegiosSobreObjetos;
/


create or replace procedure AgregarPrivilegioSobreObjeto(p_privilegio DBA_TAB_PRIVS.PRIVILEGE%TYPE, 
                                                         p_propietario DBA_TAB_PRIVS.OWNER%TYPE, 
                                                         p_nombreTabla DBA_TAB_PRIVS.TABLE_NAME%TYPE, 
                                                         p_nuevoRol varchar2)
  is
  begin
    dbms_output.put_line('grant '||p_privilegio||' on '||p_propietario||'.'||p_nombreTabla||' to '||p_nuevoRol||';');
end AgregarPrivilegioSobreObjeto;
/


create or replace procedure PrivilegiosDelSistema(p_user varchar2,
                                                  p_nuevoRol varchar2)
  is
    cursor c_privSis
      is
        select distinct PRIVILEGE, ADMIN_OPTION
          from DBA_SYS_PRIVS
          where GRANTEE=p_user
          or GRANTEE in (select distinct granted_role
                           from DBA_ROLE_PRIVS
                           start with GRANTEE = p_user
                           connect by GRANTEE = prior GRANTED_ROLE);
    v_cursor c_privSis%ROWTYPE;
begin
    for v_cursor in c_privSis loop
      AgregarPrivilegioDelSistema(v_cursor.PRIVILEGE, v_cursor.ADMIN_OPTION, p_nuevoRol);
    end loop;
end PrivilegiosDelSistema;
/



create or replace procedure AgregarPrivilegioDelSistema(p_privilegio USER_SYS_PRIVS.PRIVILEGE%TYPE,
                                                        p_opcAdmin USER_SYS_PRIVS.ADMIN_OPTION%TYPE,
                                                        p_nuevoRol varchar2)
is
begin
    if p_opcAdmin='YES' then
        dbms_output.put_line('grant '||p_privilegio||' to '||p_nuevoRol||' with admin option;');
    else
        dbms_output.put_line('grant '||p_privilegio||' to '||p_nuevoRol||';');
    end if;
end AgregarPrivilegioDelSistema;
/
~~~


### Función de verificación de contraseña
La siguiente función verifica la contraseña para que difiera en más de tred caracteres de la anterior y que la longitud sea diferente a la anterior. Además, debe tener mínimo dos letras y mínimo dos números.

~~~
create or replace function VerificacionPSW (p_usuario varchar2,
                                            p_pswnueva varchar2,
                                            p_pswvieja varchar2)
  return boolean
  is
    v_sumaRepe number:=0;
    v_letraigual number:=0;
    v_numNum number:=0;
    v_numLetra number:=0;
    v_validar number:=0;
  begin
    if length(p_pswnueva)=length(p_pswvieja) then
      raise_application_error(-20003, 'La nueva contraseña no debe tener la misma longitud que la anterior');
    end if;
    for i in 1..length(p_pswnueva) loop
      ContarNumerosYLetras(substr(p_pswnueva, i,1), v_numNum, v_numLetra);
      CompararCaracteres(substr(p_pswnueva, i,1), p_pswvieja, v_letraigual);
      if v_letraigual=0 then
        v_sumaRepe:=v_sumaRepe+1;
      end if;
      v_letraigual:=0; 
    end loop;
    v_validar:=LanzarErrores(v_sumaRepe, v_numNum, v_numLetra);
    return TRUE;
end VerificacionPSW;
/

create or replace function LanzarErrores(p_sumaRepe number, 
                                         p_numNum number,
                                         p_numLetra number)
  return number
  is
  begin
    case
      when p_sumaRepe<4 then
        raise_application_error(-20002, 'La nueva contraseña debe tener al menos 3 caracteres diferentes con respecto a la antigua');
      when p_numNum<2 then
        raise_application_error(-20003, 'La nueva contraseña debe tener al menos 2 letras');
      when p_numLetra<2 then
        raise_application_error(-20004, 'La nueva contraseña debe tener al menos 2 caracteres numericos');
      else
        return 1;
    end case;          
end LanzarErrores;
/


create or replace procedure CompararCaracteres(p_caracter varchar2,
                                               p_psw varchar2,
                                               p_letraigual in out number)
  is
  begin
    for i in 1..length(p_psw) loop
      if substr(p_psw,i,1)=p_caracter then
        p_letraigual:=1;
      end if;
    end loop;
end CompararCaracteres;
/


create or replace procedure ContarNumerosYLetras (p_caracter varchar2,
                                                  p_numero  in out number,
                                                  p_letra   in out number)
  is
  begin
    if p_caracter=REGEXP_REPLACE(p_caracter,'[0-9]') then
      p_numero:=p_numero+1;
    else
      p_letra:=p_letra+1;
    end if;
end ContarNumerosYLetras;
/
~~~

Comprobación:
**Creación del perfil**
~~~
create profile PswSegura limit 
    PASSWORD_VERIFY_FUNCTION VerificacionPSW;

drop profile PswSegura;
~~~

**Asignación al usuario**
~~~
alter user PRUEBAPALOMA profile PswSegura;
~~~

**Comrpobación** 
~~~
SQL> alter user pruebapaloma identified by asdfghjklñ45 replace asdfghjklñ456;
alter user pruebapaloma identified by asdfghjklñ45 replace asdfghjklñ456
*
ERROR en línea 1:
ORA-28003: fallo en la verificación de la contraseña especificada
ORA-20002: La nueva contraseña debe tener al menos 3 caracteres diferentes con
respecto a la antigua

SQL> alter user pruebapaloma identified by 12345745 replace asdfghjklñ456;
alter user pruebapaloma identified by 12345745 replace asdfghjklñ456
*
ERROR en línea 1:
ORA-28003: fallo en la verificación de la contraseña especificada
ORA-20003: La nueva contraseña debe tener al menos 2 letras

SQL> alter user pruebapaloma identified by zxcvbnm replace asdfghjklñ456;
alter user pruebapaloma identified by zxcvbnm replace asdfghjklñ456
*
ERROR en línea 1:
ORA-28003: fallo en la verificación de la contraseña especificada
ORA-20003: La nueva contraseña debe tener al menos 2 caracteres numericos

SQL> alter user pruebapaloma identified by asdfghjklñ456 replace aaa2222;

Usuario modificado.

SQL> alter user pruebapaloma identified by zxcvbnm123 replace asdfghjklñ456;

Usuario modificado.

SQL> alter user pruebapaloma identified by qwertyu789 replace zxcvbnm123;
alter user pruebapaloma identified by qwertyu789 replace zxcvbnm123
*
ERROR en línea 1:
ORA-28003: fallo en la verificación de la contraseña especificada
ORA-20003: La nueva contraseña no debe tener la misma longitud que la anterior
~~~


### Procedimiento que muestre los privilegios de un rol
A continuación, se va a crear un procedimiento que mmuestra los privilegios de sistema y los privilegios sobre objetos de un rol:

~~~
create or replace procedure MostrarPrivileciosdelRol(p_rol varchar2)
  is
    v_validacion number:=0;
  begin
    v_validacion:=ComprobarSiRolExiste(p_rol);
    if v_validacion=0 then
      BuscarPrivilegiosDelSistema(p_rol);
      dbms_output.put_line(' ');
      dbms_output.put_line('--------------------------------------------------------------------------------');
      dbms_output.put_line(' ');
      BuscarPrivilegiosSobreObjetos(p_rol);
    end if;
end MostrarPrivileciosdelRolPaloma;
/

create or replace procedure BuscarPrivilegiosDelSistema(p_rol varchar2)
  is
    cursor c_sys
      is
        select distinct PRIVILEGE
          from ROLE_SYS_PRIVS
            where ROLE in (select distinct ROLE 
                             from ROLE_ROLE_PRIVS 
                             start with ROLE=p_rol
                             connect by ROLE=prior GRANTED_ROLE)
          or ROLE=p_rol;
    v_sys c_sys%ROWTYPE;
  begin
    dbms_output.put_line('PRIVILEGIOS DEL SISTEMA');
    dbms_output.put_line('--------------------------------------------------------------------------------');
    for v_sys in c_sys loop
        dbms_output.put_line(v_sys.PRIVILEGE);
    end loop;
end BuscarPrivilegiosDelSistema;
/


create or replace procedure BuscarPrivilegiosSobreObjetos(p_rol varchar2)
  is
    cursor c_tab
      is
        select distinct PRIVILEGE, TABLE_NAME, OWNER
          from ROLE_TAB_PRIVS
          where ROLE in (select distinct ROLE 
                           from ROLE_ROLE_PRIVS 
                           start with ROLE=p_rol
                           connect by ROLE = prior GRANTED_ROLE)
          or ROLE=p_rol;
    v_tab c_tab%ROWTYPE;
  begin
    dbms_output.put_line('PRIVILEGIOS SOBRE OBJETOS');
    dbms_output.put_line('--------------------------------------------------------------------------------');
    for v_tab in c_tab loop
        dbms_output.put_line(v_tab.PRIVILEGE||' sobre la tabla '||v_tab.TABLE_NAME||' del usuario '||v_tab.OWNER);
    end loop;
end BuscarPrivilegiosSobreObjetos;
/


create or replace function ComprobarSiRolExiste(p_rol varchar2)
  return number
  is
    v_resultado varchar2(30);
  begin
    select ROLE into v_resultado
      from DBA_ROLES
      where ROLE=p_rol;
      return 0;
  exception
    when NO_DATA_FOUND then
      dbms_output.put_line('No existe el rol '||p_rol);
      return -1;
end ComprobarSiRolExiste;
/
~~~


