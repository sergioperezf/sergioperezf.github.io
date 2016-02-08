---
layout: post
title:  "Implementando usuarios y permisos en Zend Framework 2"
date:   2015-05-15 15:41:25
categories: php
tags: zf2 usuarios permisos acl
image: /assets/article_images/2014-08-29-welcome-to-jekyll/desktop.jpg
comments: true
disqus_id: 9b1888cd-58d4-433b-b50f-8fa94aa96369
---

El día de hoy vamos a crear un sistema funcional que implementa usuarios y permisos en Zend Framework 2, usando dos módulos muy populares para ello:

- `Zfc-user`
- `BjyAuthorize`

Además, lo haremos usando Doctrine para la comunicación con la base de datos.

### La aplicación

La aplicación que vamos a desarrollar es la forma más básica de un sistema de _tickets_ de soporte. Tendremos dos roles, `usuario` y `agente`. Los usuarios pueden crear tickets, así como modificar los tickets que ellos mismos hayan creado. Los agentes pueden leer y cambiar de estado a los tickets, pero no pueden modificar nada más. Las personas se pueden registrar en el sitio, pero sólo bajo el rol de `usuario`.

Eso es todo lo que hará esta pequeña aplicación, la cual espero sirva de manera ilustrativa de los módulos involucrados.

## Módulos necesarios

Para empezar, vamos a crear nuestra aplicación con base en la aplicación esqueleto de Zend Framework 2. Para obtenerla, vamos a la página oficial y seguimos las instrucciones: https://github.com/zendframework/ZendSkeletonApplication.

Una vez tengamos la aplicación instalada, es hora de añadir los módulos que vamos a usar. Estos se añaden a el archovp `composer.json`, de modo que debe quedar más o menos así:

```json
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
        "php": ">=5.3.3",
        "zendframework/zendframework": "2.4.*",
        "doctrine/doctrine-orm-module": "0.*",
        "doctrine/doctrine-module": "0.*",
        "zf-commons/zfc-user-doctrine-orm": "1.0.*",
        "bjyoungblood/bjy-authorize": "1.4.*"
    }
}
```

Guardamos el archivo, y corremos `php composer.phar update`, para actualizar e instalar los módulos necesarios.

### Activar los módulos en nuestra aplicación

Ahora que hemos añadido y descargado los módulos necesarios, vamos a tomar un vistazo a lo que hemos instalado:

- `doctrine/doctrine-module` y `doctrine/doctrine-orm-module`: estos son los módulos que proveen la integración con el ORM Doctrine a nuestra aplicación en Zend Framework 2.
- `zf-commons/zfc-user-doctrine-orm`: este módulo provee la funcionalidad de `zfcuser`integrada con Doctrine. Al instalar este módulo, el sistema de resolución de dependencias de Composer instalará también los módulos `zfc-base`y `zfc-user`. Más acerca de `zfcuser` más adelante.
- `bjyoungblood/bjy-authorize`: es el módulo que usaremos para proveer la funcionalidad de permisos. Este módulo es básicamente una manera más fácil de usar  la ACL (Access Control List) proveída por Zend Framework 2 por defecto. Ya veremos cómo se configura.

Para activar dichos módulos en nuestra aplicación, vamos al archivo `config/application.config.php` y añadimos los módulos al arreglo `modules`:

```php
<?php
return array(
    //...
    'modules' => array(
        'DoctrineModule',
        'DoctrineORMModule',
        'ZfcBase',
        'ZfcUser',
        'ZfcUserDoctrineORM',
        'BjyAuthorize',
        'Application',
    ),
    //...
);
```

Esto activará los módulos recién descargados en nuestra aplicación. El siguiente paso es configurarlos.

## Configuración de Doctrine 2

Con nuestros módulos instalados, vamos a configurar primero Doctrine, el ORM, de manera que nos podamos comunicar con la base de datos. Antes que nada, este será el modelo de datos que servirá para nuestros requerimientos:

![Modelo de datos](modelo.png)

Como podemos ver, es un modelo muy sencillo.

Para configurar Doctrine vamos a modificar (o agregar) dos archivos: `config/autoload/doctrine.global.php`y `config/autoload/doctrine.local.php`. En el primero vamos a configurar el nombre de la base de datos, el host y el puerto, mientras que en el segundo vamos a configurar las credenciales de acceso. La idea de tener los dos archivos por separado es que cuando subamos nuestro trabajo a un repositorio como GitHub o Sourceforge, sólo el archivo global se suba.

```php
<?php
//config/autoload/doctrine.global.php
return array(
    'doctrine' => array(
        'connection' => array(
            'orm_default' => array(
                'driverClass' => 'Doctrine\DBAL\Driver\PDOMySql\Driver',
                    'params' => array(
                        'host' => 'localhost',
                        'port' => '3306',
                        'dbname' => 'edulers',
                ),
            ),
        )
));
```

```php
<?php
//config/autoload/doctrine.global.php
return array(
    'doctrine' => array(
        'connection' => array(
            'orm_default' => array(
                'driverClass' => 'Doctrine\DBAL\Driver\PDOMySql\Driver',
                    'params' => array(
                        'user' => 'appication',
                        'password' => 'password',
                ),
            ),
        )
));
```

En este caso estamos asumiendo que vamos a usar una base de datos MySql en localhost, cuyas credenciales son las dadas (application/password).

Ahora vamos a decirle a Doctrine en dónde se encuentran las entidades para nuestra aplicación. En el archivo de configuración de nuestro módulo (por defecto `module/Application/config/module.config.php`, aunque yo recomiendo crear un módulo aparte), añadimos al arreglo que es retornado:

```php
<?php
return array(
	//...
    'doctrine' => array(
        'driver' => array(
            'application_entities' => array(
                'class' => 'Doctrine\ORM\Mapping\Driver\AnnotationDriver',
                'cache' => 'array',
                'paths' => array(__DIR__ . '/../src/Application/Entity')
            ),
            'orm_default' => array(
                'drivers' => array(
                    'Application\Entity' => 'application_entities'
                ),
            ),
        ),
    ),
    //...
);
```

Esto le dirá a Doctrine que las entidades de nuestra aplicación se encuentran en el directorio `module/Application/src/Application/Entity`, y que tendrán el namespace `Application\Entity`.