---
layout: post
title:  "Depuración web remota de PHP con Xdebug"
date:   2012-11-15 23:06:00
categories: php
tags: php xdebug depuración
image: /assets/article_images/2012-11-15-depuracion-remota-xdebug/remote.jpg
comments: true
disqus_id: f8e66f0e-c75a-403e-a5ea-60e10e42dd6a
---

En este post explicaré los pasos que seguí para configurar la depuración remota sobre Internet de una aplicación web en PHP, usando Netbeans. Primero, la configuración de mis máquinas:
<ul>
	<li>Por un lado, tengo una o varias máquinas de desarrollo. Estas máquinas por lo general tienen Windows 7, aunque eso no viene al caso. El desarrollo lo hago usando Netbeans 7.</li>
	<li>Por otro lado, mi servidor de pruebas es una máquina modesta con Linux, Ubuntu. PHP 5.3 y Apache 2.</li>
</ul>

El hecho es que mi servidor es fijo, está en mi residencia, y muchas veces me llevo el portátil para adelantar trabajo desde donde me encuentre, usando la conexión móvil de mi celular.

De modo que necesitaba una manera de usar la funcionalidad de depurado de Netbeans, para que se justifique en algo los cerca de 400MB en memoria que consume (bromeo, sí se justifica, además estoy acostumbrado al IDE). Configurar Xdebug para un entorno local es relativamente sencillo, y ya lo había hecho en varias ocasiones en máquinas con Windows. De modo que me lancé de lleno a la tarea de configurarlo para depuración remota. Primero sobre red local, luego sobre Internet.

<strong>Instalar y activar Xdebug en Linux</strong>

La página de soporte de Netbeans <a href="http://wiki.netbeans.org/HowToConfigureXDebug#How_to_on_Linux">provee </a>cierta información sobre cómo configurar Xdebug en Linux para que funcione con el IDE instalado en la misma máquina del servidor, pero hasta ahí llega; es decir, asume que Xdebug ya se encuentra instalado. Tuve que basarme en <a href="http://www.spiration.co.uk/post/1372/how-to-install-xdebug-on-linux">otro recurso</a> para poder instalarlo. Resumo el procedimiento para Ubuntu:

Primero, hay que asegurarse que el comando <code>pecl</code> está disponible. Simplemente con escribirlo en consola, el sistema tratará de ejecutarlo y, de no encontrarlo, dirá qué paquete hay que instalar para que esté disponible. En el tutorial dice que se trata de php-dev, pero en mi caso el comando estaba en el paquete php-pear. Sudo apt-get bla bla bla.

Ahora viene lo bueno. Una vez está instalada la aplicación que nos provee <code>pecl</code>, tenemos que ejecutar dicho comando para instalar Xdebug:

~~~ bash
# pecl install xdebug
~~~ 

Esto descargará, compilará e instalará la extensión de PHP que tanto anhelamos. Al final de mensajes crípticos y un poco abrumadores por parte de la terminal, se nos informará la nueva ruta en la que quedó el archivo xdebug.so recién compilado. Debemos anotar esa ruta, pues la usaremos después.

Ahora tenemos que añadir las correspondientes lineas de configuración al archivo php.ini, para activar la extensión. Para ubicar el archivo php.ini que vamos a modificar tenemos dos opciones:
<ul>
	<li>Escribir un <em>script</em> PHP y correrlo en el servidor con el siguiente código: <code>&lt;?php phpinfo() ?&gt;</code>, o</li>
	<li>en la terminal, <code>php -i | grep php.ini</code></li>
</ul>
Una vez ubicado el archivo, lo editamos y añadimos (o descomentamos) la línea que reza más o menos:

~~~ 
zend_extension = /ruta/que/habiamos/anotado/xdebug.so
~~~ 

Eso instalará y activará Xdebug en Linux, previo reinicio del servidor, claro. Para probar podemos correr el script que dan en el tutorial en el cual me basé.

<strong>Configurar Xdebug para depuración remota</strong>

Como lo que yo quería hacer no era desarrollar en mi servidor y depurar a golpe de comando, sino en otras máquinas y depurar por medio de Netbeans, no vamos a parar acá.

En el archivo php.ini que hemos editado, el tutorial oficial de Netbeans nos dice que pongamos además las siguientes líneas:

~~~ 
xdebug.remote_enable=1
xdebug.remote_handler=dbgp
xdebug.remote_mode=req
xdebug.remote_host=127.0.0.1
xdebug.remote_port=9000
~~~ 

Estas líneas están bien si se quiere usar Netbeans en la misma máquina que tiene el servidor. Eso lo vemos en <code>xdebug.remote_host = 127.0.0.1</code>. Ahí nos está diciendo que el cliente de Xdebug está en 127.0.0.1, es decir en localhost. Lo que vamos a hacer es comentar esa línea, y en vez añadir la siguiente, para que quede todo igual, menos:

~~~ 
; xdebug.remote_host=127.0.0.1
xdebug.remote_connect_back=1
~~~ 

Esta directiva le está diciendo a Xdebug que trate de conectar a todos los clientes que inicien una sesión de depuración. Esto permitirá hacer depuraciones desde equipos remotos cuyas direcciones IP pueden cambiar. Ahora, sólo es cuestión de configurar los puertos.

<strong>Puertos</strong>

Tenemos que abrir o redirigir externamente el puerto 9000 a nuestro servidor desde Internet, y a nuestro cliente. Si el cliente (Netbeans) está bajo un router o un firewall, debemos redirigir las conexiones entrantes por dicho puerto en TCP hacia nuestra máquina.

Bono: una buena aplicación que me sirvió para redirigir el puerto de mi teléfono Android hacia mi portátil es <a href="https://play.google.com/store/apps/details?id=at.bherbst.net">Port Forwarder</a>.