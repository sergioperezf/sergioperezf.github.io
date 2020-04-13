---
layout: post
title:  "Usando servicios e inyección de dependencias en Zend Framework 2"
date:   2015-03-19 15:41:25
categories: php
tags: zf2 servicios
image: /assets/article_images/2015-03-19-servicios-zf2/restaurant.jpg
comments: true
disqus_id: a9e00b05-57e6-4e0b-845b-4b4fd1f33d06
summary: >
    En la entrada del día de hoy vamos a ver cómo podemos hacer plantear la arquitectura de una aplicación MVC en Zend Framework 2 usando una capa de servicios entre en los controladores y el modelo.
---

En la entrada del día de hoy vamos a ver cómo podemos hacer plantear la arquitectura de una aplicación MVC en Zend Framework 2 usando una capa de servicios entre en los controladores y el modelo. Realmente, muchas aplicaciones MVC hacen uso de este patrón en la vida real, no necesariamente apliaciones hechas con ZF2 o ni siquiera con PHP. Pero en esta entrada me voy a enfocar en demostrar cómo se puede legar a hacer con este framework en específico.

Vamos a continuar la aplicación de la entrada anterior que se refería a cómo implementar una Lista de Control de Acceso. 

## Modelando la entidad Ticket

En la entrada anterior alcanzamos a modelar dos de nuestras entidades, usuario y rol. También alcanzamos a crear un esqueleto de lo que podía ser nuestro controlador de Tickets, `TicketConroller`. Además, restringimos el acceso de los usuarios a ciertas acciones del controlador dependiendo en si tenían el permiso de realizar acciones en el recurso `Ticket`.

Ahora, vamos a modelar nuestra entidad `Ticket`. Para los tiquetes de soporte de nuestro sistema, queremos los siguientes campos:

1. `id`: el identificador únido del tiquete.
2. `title`: título del tiquete.
3. `description`: descripción.
4. `creator`: el usuario que creó el tiquete.
5. `status`: el estado de resolución del tiquete.
6. `assignee`: a qué agente está asignado el tiquete en este momento.
7. `created`: la fecha y hora en que el tiquete fue creado.

Con esto en mente, vamos a modelar nuestra entidad, usando anotaciones de `Doctrine`:

~~~ php
<?php
/**
 * File: Ticket.php.
 */

namespace Application\Entity;

use Doctrine\ORM\Mapping as ORM;

/**
 * Class Ticket
 * @package Application\Entity
 *
 * @ORM\Table(name="ticket")
 *
 * @ORM\Entity
 */
class Ticket
{
    /**
     * @var integer
     *
     * @ORM\Column(name="id", type="integer", nullable=false)
     * @ORM\Id
     * @ORM\GeneratedValue(strategy="IDENTITY")
     */
    private $id;

    /**
     * @var string
     *
     * @ORM\Column(name="title", type="string", length=200, nullable=false)
     */
    private $title;

    /**
     * @var string
     *
     * @ORM\Column(name="description", type="text", nullable=true)
     */
    private $description;

    /**
     * @var User
     *
     * @ORM\ManyToOne(targetEntity="User")
     * @ORM\JoinColumn(name="creator_id", referencedColumnName="id", nullable=false)
     */
    private $creator;

    /**
     * @var string
     *
     * @ORM\Column(name="status", type="string", length=30, nullable=false)
     */
    private $status;

    /**
     * @var User
     *
     * @ORM\ManyToOne(targetEntity="User")
     * @ORM\JoinColumn(name="assignee_id", referencedColumnName="id", nullable=true)
     */
    private $assignee;

    /**
     * @var \DateTime
     *
     * @ORM\Column(name="created", type="datetime", nullable=false)
     */
    private $created;

    /**
     * @return integer
     */
    public function getId()
    {
        return $this->id;
    }

    /**
     * @param integer $id
     */
    public function setId($id)
    {
        $this->id = $id;
    }

    /**
     * @return string
     */
    public function getTitle()
    {
        return $this->title;
    }

    /**
     * @param string $title
     */
    public function setTitle($title)
    {
        $this->title = $title;
    }

    /**
     * @return string
     */
    public function getDescription()
    {
        return $this->description;
    }

    /**
     * @param integer $description
     */
    public function setDescription($description)
    {
        $this->description = $description;
    }

    /**
     * @return User
     */
    public function getCreator()
    {
        return $this->creator;
    }

    /**
     * @param User $creator
     */
    public function setCreator($creator)
    {
        $this->creator = $creator;
    }

    /**
     * @return string
     */
    public function getStatus()
    {
        return $this->status;
    }

    /**
     * @param string $status
     */
    public function setStatus($status)
    {
        $this->status = $status;
    }

    /**
     * @return User
     */
    public function getAssignee()
    {
        return $this->assignee;
    }

    /**
     * @param User $assignee
     */
    public function setAssignee($assignee)
    {
        $this->assignee = $assignee;
    }

    /**
     * @return \DateTime
     */
    public function getCreated()
    {
        return $this->created;
    }

    /**
     * @param \DateTime $created
     */
    public function setCreated($created)
    {
        $this->created = $created;
    }
}
~~~

El contenido arriba expuesto lo vamos a copiar en el archivo `module/Application/src/Application/Entity/Ticket.php`. Ahora, para regenerar la base de datos sin perder nuestros usuarios y roles existentes, corremos desde una terminal:

~~~
$ ./vendor/bin/doctrine-module orm:schema-tool:update --force
Updating database schema...
Database schema updated successfully! "3" queries were executed
~~~

En este momento veremos que tenemos una nueva tabla llamada `ticket` en nuestra base de datos.

## Interactuando con la Base de Datos

Para darnos una idea de cómo interactuar con la base de datos desde nuestra aplicación vamos a añadir a nuestro controlador la acción `listAction`: 

~~~ php
<?php
// ...
class TicketController extends AbstractActionController
{
    // ...
    /**
     * Only users that can read tickets can list them.
     */
    public function listAction()
    {
        if (!$this->isAllowed('Ticket', 'read')) {
            throw new UnAuthorizedException();
        }
        /** @var EntityManager $em */
        $em = $this->getServiceLocator()->get('Doctrine\ORM\EntityManager');
        /** @var Ticket[] $tickets */
        $tickets = $em->getRepository('Application\Entity\Ticket')->findAll();

        foreach ($tickets as $ticket) {
            echo $ticket->getTitle() . ', by ' . $ticket->getCreator()->getEmail() . '<br/>';
        }
        return false;
    }
}
~~~

Podemos ver en `$em = $this->getServiceLocator()->get('Doctrine\ORM\EntityManager')`, que estamos usando una funcionalidad de Zend Framework llamada `ServiceManager`. El `ServiceManager` es un componente disponible en los controladores que se encarga de instanciar o traer servicios, de modo que podamos usarlos dentro de nuestro código. Pero primero, ¿qué es un servicio?

Un servicio es, en pocas palabras, un conjunto de funcionalidades que pueden ser reusadas. En este contexto, un servicio es una clase que contiene funcionalidad relacionada con otros componentes de la aplicación. Para poder hacer uso de un servicio, debemos indicarle al `ServiceManager` dónde está nuestro servicio, así como una clave que vamos a usar cuando queramos traerlo. ¿Dónde está esto en el caso del servicio `Doctrine\ORM\EntityManager`? La respuesta está en el código fuente del módulo `doctrine-orm-module`, que instalamos en la entrada pasada para manejar la base de datos con Doctrine.

## Servicio de tiquetes

Vamos a crear nuestro propio servicio, que vamos a usar para realizar todas la operaciones con los tiquetes de nuestro sistema. Inicialmente, crearemos una clase llamada `TicketService` bajo `module/Application/src/Application/Service/`:

~~~ php
<?php
/**
 * File: TicketService.php.
 */

namespace Application\Service;

use Application\Entity\Ticket;

/**
 * Class TicketService
 * @package Application\Service
 */
class TicketService
{
    /**
     * @param Ticket $ticket
     *
     * @return Ticket
     */
    public function saveTicket(Ticket $ticket)
    {

    }

    /**
     * @return Ticket[]
     */
    public function getAllTickets()
    {

    }

    /**
     * @param integer $id
     *
     * @return Ticket
     */
    public function getTicketById($id)
    {

    }
}
~~~

Esta clase será nuestro servicio de tiquetes. Aún no hemos implementado ningún método en ella, pero por ahora veamos lo que hará cada método:

- `saveTicket(Ticket $ticket)`: este método se encargará se guardar a base de datos un `Ticket`, retornándolo inmediatamente.
- `getAllTickets()`: se encarga de traer todos los tiquetes de la base de datos y retornarlos en un arreglo.
- `getTicketById($id)`: se encarga de buscar un tiquete por su id, y retornarlo.

La manera en que vamos a implementar cada uno de esos métodos es, como no, usando el `EntityManager` de doctrine. Sin embargo, esto posa un problema, puesto que no tenemos cómo acceder al `ServiceLocator` desde nuestro servicio. Vamos, por ahora, a inventarnos una manera muy sencilla de tener acceso al `EntityManager` en nuestro servicio: por medio del constructor.

~~~ php
<?php
// ...
use Doctrine\ORM\EntityManager; // añadimos este use

// ...
class TicketService
{
    /**
     * @var EntityManager
     */
    private $entityManager;

    /**
     * @param EntityManager $entityManager
     */
    public function __construct(EntityManager $entityManager)
    {
        $this->entityManager = $entityManager;
    }
    
    // ...
}
~~~

Ahora ya tenemos acceso al `EntityManager` en nuestro servicio. Lo siguiente que haremos es implementar la funcion de listar tiquetes, similarmente a como lo implementamos en el controlador:

~~~ php
<?php
    // ...
    public function getAllTickets()
    {
        return $this->entityManager->getRepository('Application\Entity\Ticket')->findAll();
    }
    // ...
~~~

Y llamar al servicio desde `TicketController`:

~~~ php
<?php
// ...
use Application\Service\TicketService; // añadimos este use

    //...
    /**
     * Only users that can read tickets can list them.
     */
    public function listAction()
    {
        if (!$this->isAllowed('Ticket', 'read')) {
            throw new UnAuthorizedException();
        }
        /** @var EntityManager $em */
        $em = $this->getServiceLocator()->get('Doctrine\ORM\EntityManager');

        $service = new TicketService($em);
        $tickets = $service->getAllTickets();

        foreach ($tickets as $ticket) {
            echo $ticket->getTitle() . ', by ' . $ticket->getCreator()->getEmail() . '<br/>';
        }
        return false;
    }
    // ...
~~~

Por ahora sólo vamos a implementar ese método de nuestro servicio. 

## Registrando el servicio e inyección de dependencias

Ahora vamos a registrar nuestro servicio en el `ServiceManager` de Zend Framework, de modo que podamos hacer uso de él simplemente llamando al `SeviceLocator`. Para esto, vamos a modificar el archivo `module\Application\config\module.config.php` de nuestro módulo, y añadimos al arreglo con clave `service_manager['factories']` la siguiente entrada:

~~~
'Application\Service\Ticket' => 'Application\Factory\TicketServiceFactory'
~~~

Esto le va a decir a Zend Framework que existe una clase factoría que retorna nuestro servicio. La razón por la que necesitamos una factoría en este caso, es porque nuestro servicio tiene una dependencia, el `EntityManager`, que estamos inyectando a en el constructor. A esto se refiere la inyección de dependencias, a pasar las dependencias de una clase por medio del constructor o de métodos `setters`, en vez de instanciarlas en la misma clase. 

Nuestra factoría entonces va a ser una muy sencilla. La creamos en `module/Application/src/Application/Factory/TicketServiceFactory.php`:

~~~ php
<?php
/**
 * File: TicketServiceFactory.php.
 */

namespace Application\Factory;
use Application\Service\TicketService;
use Doctrine\ORM\EntityManager;
use Zend\ServiceManager\FactoryInterface;
use Zend\ServiceManager\ServiceLocatorInterface;

/**
 * Class TicketServiceFactory
 * @package Application\Factory
 */
class TicketServiceFactory implements FactoryInterface
{
    /**
     * Create service
     *
     * @param ServiceLocatorInterface $serviceLocator
     * @return TicketService
     */
    public function createService(ServiceLocatorInterface $serviceLocator)
    {
        /** @var EntityManager $entityManager */
        $entityManager = $serviceLocator->get('Doctrine\ORM\EntityManager');
        return new TicketService($entityManager);
    }
}
~~~

Con esta factoría registrada en nuestra aplicación, podemos cambiar la acción de nuestro controlador para que use nuestro servicio recién registrado:


~~~ php
<?php
// ...

    //...
    /**
     * Only users that can read tickets can list them.
     */
    public function listAction()
    {
        if (!$this->isAllowed('Ticket', 'read')) {
            throw new UnAuthorizedException();
        }

        /** @var TicketService $service */
        $service = $this->getServiceLocator()->get('Application\Service\Ticket');
        $tickets = $service->getAllTickets();

        foreach ($tickets as $ticket) {
            echo $ticket->getTitle() . ', by ' . $ticket->getCreator()->getEmail() . '<br/>';
        }
        return false;
    }
    // ...
~~~

Si vamos a `/application/ticket/list`, veremos que nuestro sitio sigue funcionando como siempre. Lo único que ha cambiado es que ahora, si por alguna razón nuestro `TicketService` tiene nuevas dependencias, podemos añadirlas fácilmente en la factoría, y tener el servicio funcional en todas las ocasiones en que lo llamemos usando el `ServiceLocator`.

Hay que recordar que en nuestro ejemplo, cuando hacemos el llamado a `$this->getServiceLocator()->get('Application\Service\Ticket');`, `'Application\Service\Ticket'` no es más que el nombre que le dimos a nuestro servicio cuando lo registramos en el Service Manager. En Zend Framework 2, la convención parece ser que los nombres de los servicios son parecidos a los nombres canónicos de las clases, sin la palabra `Service` al final. Pero bien hubiéramos podido registrar nuestro servicio con cualquier otro nombre, y llamarlo así desde el `ServiceLocator`.
 