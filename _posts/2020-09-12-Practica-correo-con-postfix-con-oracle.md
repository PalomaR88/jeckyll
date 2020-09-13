---
title: Práctica correo con Postfix y Oracle
author: Paloma R. García Campón
date: 2020-09-12 00:00:00 +0800
categories: [Bases de datos]
tags: [OracleDB, Postfix]
---

# Introducción
Imaginemos que una empresa hotelera quiere enviar correos automáticamente cuando en su base de datos se rellena el campo Fecha de la Factura. A continuación, se explica cómo conseguirlo usando OracleDB, como gestor de bases de datos, y Postfix, como cliente de correo.

El modelo de creación de tablas para el ejercicio se encuentra en el [GitHub personal](https://github.com/PalomaR88/BD_hotel/blob/master/Hotel.sql), donde también está el documento para [cargar los datos](https://github.com/PalomaR88/BD_hotel/blob/master/Datos_hotel.sql). 

# Instalación del cliente de correo Postfix
Instalación:
~~~
sudo apt-get install postfix
~~~

Se habilita el servicio de Postfix:
~~~
sudo systemctl start postfix 
~~~

Se solicita una lista del contenido de la cola de correo:
~~~
mailq
~~~

Y para el manejo de correo electrónico, se descarga mailutils, que es un conjunto de utilidades y demonios:
~~~
sudo apt-get install mailutils
~~~

# Instalación y configuración del paquete UTL_MAIL
El paquete UTL_MAIL de OracleDB es una utilidad para administrar el correo electrónico que incluye funciones de uso común como archivos adjuntos, CC, CCO, etc.

En primer lugar hay que descargar el paquete:
~~~
@$ORACLE_HOME/rdbms/admin/utlmail.sql
@$ORACLE_HOME/rdbms/admin/prvtmail.plb
~~~

Y se indica el servidor de correo con la siguiente instrucción:
~~~
alter session set SMTP_OUT_SERVER='<servidor_correo>';
~~~

Se otorgan los permisos necesarios al usuario de oracle para que puedan usar UTL_MAIL:
~~~
grant execute on UTL_SMTP to <usuario_oracleDB>;
grant execute on utl_mail to <usuario_oracleDB>;
grant execute on sys.UTL_TCP to <usuario_oracleDB>;
grant execute on sys.UTL_SMTP to <usuario_oracleDB>;
~~~

Se crea, añade y asigna ACL para el uso de la red a través de un procedimiento:
~~~
create or replace procedure ACLCorreo
  is
    begin
      dbms_network_acl_admin.create_acl	(acl	     => 'www.xml',
	  				 description => 'WWW ACL',
					 principal   => '<usuario_oracle>',
					 is_grant    => true,
					 privilege   => 'connect');
      dbms_network_acl_admin.add_privilege (acl	      => 'www.xml',
	                            	    principal => '<usuario_oracle>',
					    is_grant  => true,
					    privilege => 'resolve');
      dbms_network_acl_admin.assign_acl	(acl  => 'www.xml',
					 host => '<servidor_correo>');
end ACLCorreo;
/
~~~

Y se ejecuta el procedimiento:
~~~
exec ACLCorreo;
~~~

Por último, se crea un procedimiento que permita mandar correos:
~~~
create or replace procedure Enviar(p_envia varchar2, 
				   p_recibe varchar2, 
   				   p_asunto varchar2, 
  				   p_cuerpo varchar2, 
   				   p_host varchar2) 
  is
    v_mailhost varchar2(80) := ltrim(rtrim(p_host));
    v_mail_conn utl_smtp.connection;
    v_crlf varchar2(2):= CHR(13) || CHR(10);
    v_mesg varchar2(1000);
  begin
    v_mail_conn := utl_smtp.open_connection(mailhost, 25); 
    v_mesg:= 'Date: ' ||TO_CHAR( SYSDATE, 'dd Mon yy hh24:mi:ss' )|| v_crlf ||'From:  <'||p_envia||'>' || v_crlf || 'Subject: '||p_asunto || v_crlf ||'To: '||p_recibe || v_crlf ||'' || v_crlf || p_cuerpo;
    utl_smtp.helo(v_mail_conn, v_mailhost);
    utl_smtp.mail(v_mail_conn, p_envia);
    utl_smtp.rcpt(v_mail_conn, p_recibe);
    utl_smtp.data(v_mail_conn, v_mesg);
    utl_smtp.quit(v_mail_conn);
end Enviar; 
/
~~~

# Procedimientos de recolección de información
Lo primero que se necesata conocer es el correo del cliente, para ello se utiliza la siguiente función:
~~~
create or replace function DevolverEmail (p_codEst FACTURAS.COD_ESTANCIA%type)
  return PERSONAS.EMAIL%type
  is
    v_email PERSONAS.EMAIL%type;
  begin
    select EMAIL into v_email
      from PERSONAS
      where NIF=(select NIF_CLIENTE
                   from ESTANCIAS
                   where CODIGO=p_codEst);
    return v_email;
  exception
    when NO_DATA_FOUND then
      return '-1';
end DevolverEmail;
/
~~~

En el correo se envía un resumen de la factura al cliente, incluyendo los datos fundamentales de la estancia, el importe de cada apartado y el importe total. Para ello se va a crear un paquete que recoja toda esta información.
~~~
create or replace procedure RellenarPaqueteFactura(p_codEst ESTANCIAS.CODIGO%type,
    						   p_fechaInicio ESTANCIAS.FECHA_INICIO%type,
    						   p_fechaFin ESTANCIAS.FECHA_FIN%type,
    						   p_codReg ESTANCIAS.COD_REGIMEN%type,
    						   p_codTipoHab HABITACIONES.COD_TIPO%type)
  is
  begin
    RellenarDatosEstancia(p_codEst, p_fechaInicio, p_fechaFin, p_codReg, p_codTipoHab);
    RellenarGastosExtras(p_codEst);
    RellenarActividades(p_codEst);
end RellenarPaqueteFactura;
/
~~~

El procedimiento anterior contiene, a su vez, tres procedimientos que internamente usan un cuarto procedimiento llamado `CrearFilaPaqueteFactura` que va añadiendo la información al paquete:
~~~
create or replace procedure CrearFilaPaqueteFactura(p_concepto varchar2,
                                                    p_cuantia number)
  is
  begin
    PkgFactura.v_TabFactura(PkgFactura.v_TabFactura.last+1).concepto:=p_concepto;
    PkgFactura.v_TabFactura(PkgFactura.v_TabFactura.last).cuantia:=p_cuantia;
  exception
    when value_error then
      PkgFactura.v_TabFactura(1).CONCEPTO:=p_concepto;
      PkgFactura.v_TabFactura(1).CUANTIA:=p_cuantia;
end CrearFilaPaqueteFactura;
/
~~~

El primer procedmiento que utiliza `RellenarPaqueteFactura` recopila los datos de la estancia, que a su vez utiliza tres funciones para obtener el número de dias, la temporada y el precio por día:
~~~
create or replace procedure RellenarDatosEstancia(p_codEst ESTANCIAS.CODIGO%type,
  						  p_fechaInicio ESTANCIAS.FECHA_INICIO%type,
      	  					  p_fechaFin ESTANCIAS.FECHA_FIN%type,
 						  p_codReg ESTANCIAS.COD_REGIMEN%type,
						  p_codTipoHab HABITACIONES.COD_TIPO%type)
  is
    v_dias number;
    v_codTemp TEMPORADAS.CODIGO%type;
    v_precioPorDia TARIFAS.PRECIO_DIA%type;
  begin
    v_dias:=ObtenerNumeroDias(p_fechaInicio, p_fechaFin);
    for i in 1..v_dias loop
      v_codTemp:=ObtenerTemporada(p_fechaFin+i);
      v_precioPorDia:=ObtenerPrecioPorDia(v_codTemp, p_codReg, p_codTipoHab);
      CrearFilaPaqueteFactura('Dia '||i, v_precioPorDia);
    end loop;
end RellenarDatosEstancia;
/

create or replace function ObtenerNumeroDias(p_fechaInicio ESTANCIAS.FECHA_INICIO%type,
	  				     p_fechaFin ESTANCIAS.FECHA_FIN%type)
  return number
  is
    v_dias number;
  begin
    v_dias:=trunc(p_fechaFin-p_fechaInicio);
    return v_dias;
end ObtenerNumeroDias;
/

create or replace function ObtenerTemporada (p_fecha ESTANCIAS.FECHA_INICIO%type)
  return temporadas.codigo%type
  is
    cursor c_temporadas
    is
      select FECHA_INICIO, FECHA_FIN, CODIGO
        from TEMPORADAS;
      v_temporada c_temporadas%rowtype;
  begin
    for v_temporada in c_temporadas loop
      if p_fecha between v_temporada.FECHA_INICIO and v_temporada.FECHA_FIN then
        return v_temporada.CODIGO;
      end if;
    end loop;
end ObtenerTemporada;
/

create or replace function ObtenerPrecioPorDia (p_codTemp TEMPORADAS.CODIGO%type,
						p_codReg ESTANCIAS.COD_REGIMEN%type,
						p_codTipoHab HABITACIONES.COD_TIPO%type)
  return number
  is
    v_cuantia TARIFAS.PRECIO_DIA%type;
  begin
    select PRECIO_DIA into v_cuantia
      from TARIFAS
      where COD_TIPO_HAB=p_codTipoHab
      and COD_REG=p_codReg
      and COD_TEMP=p_codTemp;
    return v_cuantia;
end ObtenerPrecioPorDia;
/
~~~

El segundo procedimiento obtiene la información relativa a los gastos extras:
~~~
create or replace procedure RellenarGastosExtras (p_codEst ESTANCIAS.CODIGO%type)
  is
    cursor c_gastosextras
      is
	select CONCEPTO, CUANTIA
	  from GASTOS_EXTRAS
	  where COD_ESTANCIA=p_codEst;
    v_gastoextra c_gastosExtras%rowtype;
  begin
    for v_gastoextra in c_gastosExtras loop
      CrearFilaPaqueteFactura(v_gastoextra.CONCEPTO, v_gastoextra.CUANTIA);
    end loop;
end RellenarGastosExtras;
/
~~~

Y el tercero devuelve la información sobre las actividades realizadas:
~~~
create or replace procedure RellenarActividades(p_codEst ESTANCIAS.CODIGO%type)
  is
    cursor c_actividades
      is
	select a.PRECIO_PERS as PORPERSONA, 
	       a.NOMBRE as ACTIVIDAD, 
	       ar.NUM_PERSONAS as NUMPERSONA
          from ACTIVIDADES a, ACTIVIDADES_REALIZADAS ar
	  where ar.COD_ESTANCIA=p_codEst
	  and ar.COD_ACTIVIDAD=a.codigo
	  and ar.ABONADO=0;
    v_actividades c_actividades%rowtype;
    v_coste number(6,2);
  begin
    for v_actividades in c_actividades loop
      v_coste:=v_actividades.PORPERSONA*v_actividades.NUM_PERSONAS;
      CrearFilaPaqueteFactura(v_actividades.ACTIVIDAD, v_coste);
    end loop;
end RellenarActividades;
/
~~~

Y se crea un procedimiento para redactar el correo y enviar toda la información al procedimiento `Enviar`, indicando el servidor de correos:
~~~
create or replace procedure MandarCorreo(p_NIF PERSONAS.NIF%type,
					 p_NOMBRE PERSONAS.NOMBRE%type,
 					 p_APELLIDOS PERSONAS.APELLIDOS%type,
 					 p_FECHAINICIO ESTANCIAS.FECHA_INICIO%type,
 					 p_FECHAFIN ESTANCIAS.FECHA_FIN%type,
 					 p_correo PERSONAS.EMAIL%type)
  is
    v_cont number(6,2):=0;
    v_cuerpomedio varchar2(500);
    v_cuerpo varchar2(1000);
  begin
    for i in PkgFactura.v_TabFactura.FIRST .. PkgFactura.v_TabFactura.LAST loop
      v_cuerpo:=(v_cuerpo||PkgFactura.v_TabFactura(i).concepto||'  -	'||PkgFactura.v_TabFactura(i).cuantia||chr(10));
      v_cont:=v_cont+PkgFactura.v_TabFactura(i).CUANTIA;
    end loop;
    Enviar('oracle@servidororacle', p_correo, 'Hotel Rural', 'Estimado cliente '||p_nombre ||' '||p_apellidos||chr(10)||'Su factura para la estancia en Hotel Rural durante los dias '||p_fechainicio||' '||p_fechafin||' ya está disponible.'||chr(10)||v_cuerpo||'Total: '||v_cont||chr(10)||'Atentamente, la empresa'||chr(10)||sysdate, 'babuino-smtp.gonzalonazareno.org');
end MandarCorreo;
/
~~~

Para que se active el procedimiento `Enviar`, se crea un trigger:
~~~
create or replace trigger EnviarCorreoCliente
  after insert or update of fecha on facturas
  for each row
  declare
    v_correo PERSONAS.EMAIL%type;
    select p.NIF as v_nif, 
           p.NOMBRE as v_nombre, 
	   p.APELLIDOS as v_apellidos, 
	   e.FECHA_INICIO as v_fechaInicio, 
	   e.FECHA_FIN as v_fechaFin, 
	   e.COD_REGIMEN as v_codReg, 
	   h.COD_TIPO as v_tipoHab
      from PERSONAS p, ESTANCIAS e, HABITACIONES h
      where e.CODIGO=:new.COD_ESTANCIA
      and e.NUM_HABITACION=h.NUMERO
      and p.NIF=e.NIF_CLIENTE;
  begin
    v_correo:=DevolverEmail(:new.COD_ESTANCIA);
    if v_correo!='-1' then
      RellenarPaqueteFactura(:new.COD_ESTANCIA, v_fechaInicio, v_fechaFin, v_codReg, v_tipoHab);
      MandarCorreo(v_NIF, v_NOMBRE, v_APELLIDOS, v_FECHAINICIO,v_FECHAFIN, v_correo);
    end if;
end CorreoInvestigadorPuntuacion;
/
~~~


