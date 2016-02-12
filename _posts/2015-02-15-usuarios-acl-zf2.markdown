---
layout: post
title:  "Implementando usuarios y permisos en Zend Framework 2"
date:   2015-02-15 15:41:25
categories: php
tags: zf2 usuarios permisos acl
image: /assets/article_images/2015-02-15-usuarios-acl-zf2/kid.jpg
comments: true
disqus_id: 9b1888cd-58d4-433b-b50f-8fa94aa96369
---

El día de hoy vamos a construír la primera parte de un sistema de tickets en Zend Framework 2, específicamente la parte de usuarios y permisos, usando dos módulos muy populares para ello:

- `Zfc-user`
- `BjyAuthorize`

Además, lo haremos usando Doctrine para la comunicación con la base de datos.

### La aplicación

La aplicación que vamos a desarrollar es la forma más básica de un sistema de _tickets_ de soporte. Tendremos dos roles, `usuario` y `agente`. Los usuarios pueden crear tickets. Los agentes pueden leer y cambiar de estado a los tickets, pero no pueden modificar nada más. Las personas no se pueden registrar en el sitio.

Eso es todo lo que hará esta pequeña aplicación. En esta entrada solo nos vamos a concentrar en la parte de roles y permisos, y en siguientes entradas iremos ahondando en otros temas como eventos, servicios o formularios.

## Módulos necesarios

Para empezar, vamos a crear nuestra aplicación con base en la aplicación esqueleto de Zend Framework 2. Para obtenerla, vamos a la página oficial y seguimos las instrucciones: https://github.com/zendframework/ZendSkeletonApplication.

Una vez tengamos la aplicación instalada, es hora de añadir los módulos que vamos a usar. Estos se añaden a el archovp `composer.json`, de modo que debe quedar más o menos así:

~~~ json
{
    "name": "seperez/application",
    "description": "Implementing a simple application with users ans permissions",
    "license": "BSD-3-Clause",
    "keywords": [
        "zf2",
        "framework",
        "users",
        "permissions"
    ],
    "homepage": "http://example.com",
    "require": {
        "php": ">=5.5",
        "zendframework/zendframework": "~2.5",
        "doctrine/doctrine-orm-module": "0.9.2",
        "doctrine/doctrine-module": "0.10.0",
        "zf-commons/zfc-user-doctrine-orm": "1.0.1",
        "bjyoungblood/bjy-authorize": "1.4.0"
    }
}
~~~ 

Guardamos el archivo, y corremos `php composer.phar update`, para actualizar e instalar los módulos necesarios.

### Activar los módulos en nuestra aplicación

Ahora que hemos añadido y descargado los módulos necesarios, vamos a tomar un vistazo a lo que hemos instalado:

- `doctrine/doctrine-module` y `doctrine/doctrine-orm-module`: estos son los módulos que proveen la integración con el ORM Doctrine a nuestra aplicación en Zend Framework 2.
- `zf-commons/zfc-user-doctrine-orm`: este módulo provee la funcionalidad de `zfcuser`integrada con Doctrine. Al instalar este módulo, el sistema de resolución de dependencias de Composer instalará también los módulos `zfc-base`y `zfc-user`. Más acerca de `zfcuser` más adelante.
- `bjyoungblood/bjy-authorize`: es el módulo que usaremos para proveer la funcionalidad de permisos. Este módulo es básicamente una manera más fácil de usar  la ACL (Access Control List) proveída por Zend Framework 2 por defecto. Ya veremos cómo se configura.

Para activar dichos módulos en nuestra aplicación, vamos al archivo `config/application.config.php` y añadimos los módulos al arreglo `modules`:

~~~ php
<?php
/*
 * File: application.config.php
 */
return [
    //...
    'modules' => [
        'DoctrineModule',
        'DoctrineORMModule',
        'ZfcBase',
        'ZfcUser',
        'ZfcUserDoctrineORM',
        'BjyAuthorize',
        'Application',
    ],
    //...
];
~~~ 

Esto activará los módulos recién descargados en nuestra aplicación. El siguiente paso es configurarlos.

## Configuración de Doctrine 2

Con nuestros módulos instalados, vamos a configurar primero Doctrine, el ORM, de manera que nos podamos comunicar con la base de datos. 

Para configurar Doctrine vamos a modificar (o agregar) dos archivos: `config/autoload/doctrine.global.php`y `config/autoload/doctrine.local.php`. En el primero vamos a configurar el nombre de la base de datos, el host y el puerto, mientras que en el segundo vamos a configurar las credenciales de acceso. La idea de tener los dos archivos por separado es que cuando subamos nuestro trabajo a un repositorio como GitHub o Sourceforge, sólo el archivo global se suba.

~~~ php
<?php
/**
 * File: doctrine.global.php.
 */

return [
    'doctrine' => [
        'connection' => [
            'orm_default' => [
                'driverClass' => 'Doctrine\DBAL\Driver\PDOMySql\Driver',
                'params' => [
                    'host' => 'localhost',
                    'port' => '3306'
                ]
            ]
        ]
    ]
];
~~~ 

~~~ php
<?php
/**
 * File: doctrine.local.php.
 */

return [
    'doctrine' => [
        'connection' => [
            'orm_default' => [
                'params' => [
                    'user' => 'application',
                    'password' => 'password',
                    'dbname' => 'application'
                ]
            ]
        ]
    ]
];
~~~ 

En este caso estamos asumiendo que vamos a usar una base de datos MySql en localhost, cuyas credenciales son las dadas (application/password).

Ahora vamos a decirle a Doctrine en dónde se encuentran las entidades para nuestra aplicación. En el archivo de configuración de nuestro módulo (por defecto `module/Application/config/module.config.php`, aunque yo recomiendo crear un módulo aparte), añadimos al arreglo que es retornado:

~~~ php
<?php
/*
 * File: module.config.php
 */
return [
	//...
    'doctrine' => [
        'driver' => [
            'application_entities' => [
                'class' => 'Doctrine\ORM\Mapping\Driver\AnnotationDriver',
                'cache' => 'array',
                'paths' => (__DIR__ . '/../src/Application/Entity')
            ],
            'orm_default' => [
                'drivers' => [
                    'Application\Entity' => 'application_entities'
                ],
            ],
        ],
    ],
    //...
];
~~~ 

Esto le dirá a Doctrine que las entidades de nuestra aplicación se encuentran en el directorio `module/Application/src/Application/Entity`, y que tendrán el namespace `Application\Entity`.

### Generando entidades

Ahora vamos a generar las entidades que vamos a necesitar por ahora. Para los usuarios y roles, afortunadamente el módulo BjyAuthorize viene con entidades de ejemplo que podemos usar directamente para nuestro sistema. En otra entrada veremos cómo extender estas entidades, así como las que tienen que ver con el resto de nuestro sistema.

Por ahora, vamos a copiar los archivos `User.php.dist` y `Role.php.dist` que se encuentran en el directorio `vendor/bjyoungblood/bjy-authorize/data`, de modo que queden en `module/Application/src/Application/Entity`, y con la extensión `php`. 

Al copiarlos, recordemos que tenemos que cambiar el namespace a `Application\Entity`. También tenemos que cambiar las referencias en la propiedad `roles` de la clase `User` y en la propiedad `parent` de la clase `Role`.

Lo siguiente que tenemos que hacer es decirle a ZfcUser que tome nuestra nueva entidad `User` como la entidad de usuario del sistema. Para ello, vamos a crear un archivo llamado `zfcuser.global.php` en `config/autoload/`, y vamos a retornar el siguiente arreglo:

~~~ php
<?php
/**
 * File: zfcuser.global.php.
 */

return [
    'zfcuser' => [
        'user_entity_class' => 'Application\Entity\User',
        'enable_default_entities' => false
    ]
];
~~~

Esto le dirá al módulo ZfcUser que no vamos a usar la clase que viene por defecto con el módulo, sino que vamos a usar nuestra propia clase (que acabamos de copiar de las proveídas por BjyAuthorize). Ahora, validamos las entidades con el siguiente comando:

~~~
$ ./vendor/bin/doctrine-module orm:validate-schema
[Mapping]  OK - The mapping files are correct.
[Database] FAIL - The database schema is not in sync with the current mapping file.
~~~

Y eso es lo que nos debería salir. Para generar la base de datos:

~~~
$ ./vendor/bin/doctrine-module orm:schema-tool:create
ATTENTION: This operation should not be executed in a production environment.

Creating database schema...
Database schema created successfully!
~~~

En este momento, tenemos las tablas en la base de datos creadas. Podemos ver esto corriendo un simple `SHOW TABLES;` en MySQL:

~~~ sql
mysql> SHOW TABLES;
+-----------------------+
| Tables_in_application |
+-----------------------+
| role                  |
| user_role_linker      |
| users                 |
+-----------------------+
3 rows in set (0.00 sec)
~~~

Si vamos a nuestra applicación en el navegador, y accedemos a la ruta `user/register`, podemos ver el formulario de registro de usuarios. Creemos un par de usuarios para hacer pruebas.

## El controlador

Vamos a definir nuestros controladores, para luego configurar el módulo de autorización basados en ellos. La aplicación esqueleto de Zend Framework 2 viene con un controlador por defecto llamado IndexController, que sirve la página de inicio. Dentro del mismo directorio `module/Application/src/Application/Controller` creamos el controlador `TicketController`:

~~~ php
<?php
/**
 * File: TicketController.php
 */

namespace Application\Controller;
use Zend\Mvc\Controller\AbstractActionController;

/**
 * Class TicketController
 * @package Application\Controller
 */
class TicketController extends AbstractActionController
{
    /**
     * Only users with the role "user" can create tickets.
     */
    public function createAction()
    {
        echo "Creating a ticket.";
        return false;
    }

    /**
     * Only users with the role "agent" can change a ticket status.
     */
    public function changeStatusAction()
    {
        echo "Changing a ticket's status.";
        return false;
    }

    /**
     * Only users with the role "agent" can read tickets.
     */
    public function readAction()
    {
        echo "Reading a ticket.";
        return false;
    }
}
~~~

Este contolador tiene las acciones más básicas que usaremos en la entrada de hoy: crear, leer y cambiar el estado de los tickets. Si vamos a las rutas `application/ticket/read`, `application/ticket/create` y `application/ticket/change-status` podremos ver los textos que se muestran. 

## Configurar BjyAuthorize

Es hora de configurar el módulo que nos va a hacer la vida fácil cuando estemos trabajando con roles y permisos. En esta sección vamos a configurar los roles, los permisos, los recursos y las reglas.

Para esto, creamos el archivo de configuración `bjyauthorize.global.php` en `config/autoload/`. En este archivo vamos a retornar el siguente arreglo, que pasaré a describir más adelante:

~~~ php
<?php
/**
 * File: bjyauthorize.global.php
 */

return [
    'bjyauthorize' => [
        'default_role' => 'guest',
        'identity_provider' => 'BjyAuthorize\Provider\Identity\AuthenticationIdentityProvider',

        'role_providers' => [
            'BjyAuthorize\Provider\Role\ObjectRepositoryProvider' => [
                'object_manager' => 'doctrine.entitymanager.orm_default',
                'role_entity_class' => 'Application\Entity\Role',
            ],
            'BjyAuthorize\Provider\Role\Config' => [
                'guest' => []
            ]
        ],

        'resource_providers' => [
            'BjyAuthorize\Provider\Resource\Config' => [
                'Ticket' => [],
                'Home' => []
            ]
        ],

        'rule_providers' => [
            'BjyAuthorize\Provider\Rule\Config' => [
                'allow' => [
                    ['authenticated', 'Home', 'see'],
                    ['user', 'Ticket', 'create'],
                    ['agent', 'Ticket', ['change_status', 'read']]
                ]
            ]
        ],
    ]
];
~~~

En este punto realmente vale la pena descargar y activar el módulo Zend Developer Tools, de modo que podamos visualizar el rol del usuario actual. Si en este momento vamos a la aplicacion sin loguearnos, veremos que tenemos el rol _guest_. Esto, ya que le hemos dicho a BjyAuthorize que el rol por defecto debe ser _guest_, acá: `'default_role' => 'guest'`. Para esto, también le decimos que el rol _guest_ en realidad sí existe. Pero la verdad es que no queremos poner ese rol en la base de datos, puesto que es muy poco probable que cambie; de modo que lo configuramos explícitamente en `'BjyAuthorize\Provider\Role\Config' => ['guest' => []]`.

A continuación, vamos a generar los roles en MySQL. Haremos esto directamente en la base de datos:

~~~ sql
INSERT INTO application.role (id, parent_id, roleId) VALUES (1, null, 'authenticated');
INSERT INTO application.role (id, parent_id, roleId) VALUES (2, 1, 'user');
INSERT INTO application.role (id, parent_id, roleId) VALUES (3, 1, 'agent');
~~~

El rol con id 1 es el rol _authenticated_, que es el rol padre que los usuarios van a tener. Con este rol, un usuario va a poder ver al recurso _Home_, que es la pantalla de inicio. Los roles con ids 2 y 3 son _user_ y _agent_, o usuario y agente. Es con estos roles que jugamos a crear reglas para definir qué puede hacer cada rol sobre el recurso _Ticket_. Como podemos ver, estamos diciendo que el rol _user_ puede realizar la acción _create_ en el recurso _Ticket_, mientras que el rol _agent_ puede realizar las acciones _read_ y _change_status_. Estas reglas las definimos en el arreglo `rule_providers`.

El arreglo `resource_providers` es para definir los recursos. Como podemos ver, cada arreglo `*_providers` tiene como valor a su vez otro arreglo, con un elemento que tiene como llave una cadena que parece como un nombre de clase. Por ejemplo, `BjyAuthorize\Provider\Role\ObjectRepositoryProvider` para el arreglo `role_providers`. Estos no son más que proveedores de roles, recursos y reglas que vienen por defecto en BjyAuthorize. Afortunadamente, el autor del módulo también pone a nuestra disposición una interfaz para cada uno de ellos, de modo que nosotros podemos implementar nuestros propios proveedores si quisiéramos.

## Usando autorización

Ahora vamos a poner esto a funcionar. Para ejemplificar, vamos a modificar las acciones en nuestro `TicketController` de este modo:

~~~ php
<?php
/**
 * File: TicketController.php
 */

namespace Application\Controller;
use BjyAuthorize\Exception\UnAuthorizedException;
use Zend\Mvc\Controller\AbstractActionController;

/**
 * Class TicketController
 * @package Application\Controller
 */
class TicketController extends AbstractActionController
{
    /**
     * Only users with the role "user" can create tickets.
     */
    public function createAction()
    {
        if (!$this->isAllowed('Ticket', 'create')) {
            throw new UnAuthorizedException();
        }
        echo "Creating a ticket.";
        return false;
    }

    /**
     * Only users with the role "agent" can change a ticket status.
     */
    public function changeStatusAction()
    {
        if (!$this->isAllowed('Ticket', 'change_status')) {
            throw new UnAuthorizedException();
        }
        echo "Changing a ticket's status.";
        return false;
    }

    /**
     * Only users with the role "agent" can read tickets.
     */
    public function readAction()
    {
        if (!$this->isAllowed('Ticket', 'read')) {
            throw new UnAuthorizedException();
        }
        echo "Reading a ticket.";
        return false;
    }
}
~~~ 

Y para nuestro `IndexController`:

~~~ php
<?php
/**
 * File: IndexController.php
 */

namespace Application\Controller;

use BjyAuthorize\Exception\UnAuthorizedException;
use Zend\Mvc\Controller\AbstractActionController;
use Zend\View\Model\ViewModel;

class IndexController extends AbstractActionController
{
    public function indexAction()
    {
        if (!$this->isAllowed('Home', 'see')) {
            throw new UnAuthorizedException();
        }
        return new ViewModel();
    }
}
~~~

Ahora, si vamos al home sin estar autenticados, veremos un error `403`. En estos momentos podemos crear entradas en la tabla `user_role_linker` para asignar roles a usuarios específicos, y ver como BjyAuthorize protege nuestras acciones de roles no autorizados. 