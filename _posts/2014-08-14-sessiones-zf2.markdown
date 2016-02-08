---
layout: post
title:  "Sesiones en Zend Framework 2"
date:   2014-08-14 21:12:00
categories: php
tags: zf2 sesiones
image: /assets/article_images/2014-08-14-sessiones-zf2/bag.jpg
comments: true
disqus_id: a691ba29-6d49-4337-b9a8-7b3ca2bc51c9
---

Para usar sesiones en Zend Framework 2 podríamos usar la funcionalidad nativa de PHP para manejar sesiones, pero rápidamente nos vamos a encontrar con que conforme la complejidad de la aplicación crece, se nos va a salir de las manos el manejo de todos los datos concernientes a las sesiones de nuestros usuarios. Es ahí donde entra en juego el componente `Zend\Session`, que hace parte de Zend Framework 2. En esta entrada explicaré cómo usar `Zend\Session` para iniciar, guardar y consultar información de una sesión. Debo notar que lo expuesto acá solo representa el uso más básico de `Zend\Session`, e información más completa acerca de todas las posibilidades ofrecidas se encuentra en la <a href="http://framework.zend.com/manual/2.3/en/modules/zend.session.config.html">documentación oficial</a>, así como en diferentes lugares en la red. Para empezar, sin embargo, considero que lo expuesto en esta entrata consiste en un buen punto de partida.

Para este ejemplo no usaré la aplicación esqueleto de Zend, sino sólamente los módulos necesarios para el manejo de sesiones. Usando composer, mi archivo `composer.json` es el siguiente:

~~~ javascript
{
	"require": {
		"zendframework/zend-session" : "2.3.*@dev",
		"zendframework/zend-eventmanager": "2.3.*@dev"
	}
}
~~~

Después de ejecutar `php composer.phar install`, procedemos a requerir `vendor/autoloader.php`. Usaré un solo archivo, `index.php`, para ejemplifcar el uso del componente. Primero, añadimos los siguientes usos:

~~~ php
<?php
require "vendor/autoload.php";

use Zend\Session\Config\SessionConfig;
use Zend\Session\Container;
use Zend\Session\SessionManager;
~~~

`SessionConfig` maneja los diferentes parámetros de configuración de la sesión, `SessionManager` maneja acciones de manejo de sesión como Inicio, Destruír, etc., y `Container` es la interfaz priparia para el manejo de los datos de sesión. Ahora iniciamos el `SessionManager` con las opciones deseadas:

~~~ php
<?php
$sessionConfig = new SessionConfig();
$sessionConfig->setOptions(array(
	'remember_me_seconds' => 180,
	'use_cookies' => true,
	'cookie_httponly' => true
	)
);
$sessionManager = new SessionManager($sessionConfig);
$sessionManager->start();
Container::setDefaultManager($sessionManager);
~~~

`Container::setDefaultManager($sessionManager)` lo usamos en caso de que estemos usando varios `SessionManager`, o prefiramos ser explícitos. A continuación vamos a mostrar al usuario la página en caso de que no se encuentre logeado. Hay que tener que en cuenta que una cosa es la sesión, y otra cosa es el login. La sesión siempre la iniciamos, y la información sobre si un usuario se encuentra autenticado en el sistema es información que se encuentra en la sesión. Se puede pensar en la sesión como una _sesión de navegación_. Es decir, usuarios del sitio que no estén autenticados también tendrán una sesión, sólo que ésta no estará asociada a un usuario en particular.

~~~ php
<?php
if (!isset($_REQUEST['action'])) {
	$session = new Container('userdata');
	if ($session && $session->username) {
		echo "Username: " . $session->username;
		?>
		<a href="index.php?action=logoff">Salir</a>
		<?php
	}
	else {
		?>
		<form action="index.php">
			<label for='username'>Username</label>
			<input type="text" name="username" id="username" />
			<input type="submit" value="login" name="action" />
		</form>
		<br/>
		<?php
	}
}
~~~

Para verificar que se trata de la página principal, verificamos que la acción no exista, y que el nombre de usuario (atributo `username`) no esté iniciado. Para acceder a los datos de la sesión es necesario iniciar el contenedor, `$session = new Container('userdata')`. Los contenedores son usados para dividir los datos, siendo en este caso `username` el nombre de estos datos. Más contenedores pueden ser creados, con otros nombres y para otros datos.

Si el usuario se encuentra autenticado, vamos a mostrar su nombre y proveer un vínculo para salir. Si no se encuentra autenticado, mostramos un formulario en el que pedimos el nombre; en nuestro caso "autenticarse" se refiere a dar el nombre. Ahora vamos a mirar cómo se realiza la autenticación. De acá lo que nos interesa es cómo se guardan datos en el contenedor:

~~~ php
<?php
if ($_REQUEST['action'] == 'login') {
	$session = new Container('userdata');
	$session->username = $_REQUEST["username"];
	echo "Sesion iniciada. Username: " . $session->username;	
	echo "<a href='index.php'>Volver</a>";
}
~~~

Acá igual, iniciamos el contenedor y guardamos la información del nombre de usuario directamente desde `$_REQUEST`; también proveemos un vínculo para ir a la página principal. Ahora veamos cómo funcionaría el _cierre de sesión_:

~~~ php
<?php
if ($_REQUEST['action'] == 'logoff') {
	$session = new Container('userdata');
	unset($session->username);
	echo "Sesion cerrada.";
	echo "<a href='index.php'>Volver</a>";
}
~~~

Simplemente borramos la información del contenedor con `unset`.

Como podemos ver, el sistema de contenedores que utiliza Zend Framework es muy útil porque nos permite tener la información que necesitemos de manera organizada. Con las opciones de configuración tenemos más control de nuestra sesión, además. En un entorno MVC como el de la aplicación esqueleto de Zend Framework 2, la inicialización del `SessionManager` podría ir en `Module.php`, mientras que el acceso y guardado de datos en los contenedores iría en acciones de los controladores.