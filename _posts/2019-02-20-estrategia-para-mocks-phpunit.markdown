---
layout: post
title:  "Tests unitarios con PHPUnit: mocks."
date:   2019-02-20 16:58:25
categories: php
tags: phpunit tests unittests mocks
image: /assets/article_images/2019-02-20-estrategia-para-mocks-phpunit/plane.jpg
comments: true
disqus_id: B8766D25-70FA-45AC-AC68-90EBCB38AD9B
summary: >
    Cómo escribir mocks de las dependencias de nuestras clases cuando estamos haciendo pruebas unitarias, en PHPUnit.
---

Una de las buenas prácticas mejor aceptadas en el desarrollo es el uso de Tests unitarios para probar que los componentes de nuestro código funcionen como se espera. Sin embargo, cuando tenemos que nuestro proyecto es muy complejo, y los componentes unitarios de nuestro código tienen muchas dependencias en muchas otras partes del código, realizar estos tests suele ser bastante difícil.

Para aliviar esto, PHPUnit provee una manera fácil de crear Mocks. 

## `MockBuilder`

Estando en una clase que implementa `TestCase`, tenemos a nuestra disposición el método `getMockBuilder`. Este método recibe como parametro el tipo del objeto que queremos mockear, y retornará el mock de dicho objeto al llamar `getMock()`.

```php
$userServiceMock = $this->getMockBuilder(UserService::class)->getMock();
```

En el ejemplo, `$userServiceMock` tendrá un objeto que tiene los mismos métodos de UserService, pero estos métodos retornarán siempre `null` cuando sean llamados.

Cuando queremos mockear la respuesta de un método en específico del objeto, podemos usar el médoto `setMethods()` y `method()` en conjunción:

```php
$user = new User();
$user->setId(1);
$user->setName('Bob');
$userServiceMock = $this->getMockBuilder(UserService::class)->getMock();
$userServiceMock->setMethods(['getUsers'])->method('getUsers')
    ->will($this->returnValue([$user]));

// We can also use:
$userServiceMock->setMethods(['getUsers'])->method('getUsers')
    ->willReturn([$user]);
```

En este caso, lo que estamos diciendo es que todas las llamadas al método `getUsers` de que se hagan al objeto `$userServiceMock` van a retornar un arreglo con exactamente un usuario, que definimos arriba. Esto nos permite controlar la respuesta de componentes externos a nuestra clase, de manera que si un el método que estamos probando falla el error está con toda seguridad dentro de la implementación del método que estamos probando.

### Assertions en la llamada de los métodos

Controlar las respuestas de las llamadas de nuestras dependencias externas no sirve de mucho, si el test que estamos escribiendo no tiene en cuenta también la forma en que estos métodos son llamados. Consideremos el siguiente ejemplo:

```php
class UserEnhancer
{
    protected $userService;

    public function __construct(UserService $userService)
    {
        $this->userService = $userService;
    }

    public function enhance(int $id): User
    {
        $user = $this->userService->getUserById($id);

        if ($this->userService->canUserBeEnhanced($user)) {
            $user->power += 1;
        } else {
            $user->power -= 1;
        }

        return $user;
    }
}
```

En el ejemplo estamos usando inyección de dependencias para asegurarnos de que desde fuera de la clase. En el momento de implementación, tenemos control sobre qué dependencias vamos a usar. De este modo, al momento de instanciar la clase en nuestro test, no inyectamos una instancia original de `UserService`, sino el mock que vamos a construír.

Veamos cómo podría ser el mock del _happy path_ de nuestro método `enhance`:

```php
use PHPUnit\Framework\TestCase;

class UserEnhancerTest extends TestCase
{
    public function testDoesEnhanceAUser()
    {
        $user = new User();
        $user->setId(1);
        $user->setName('Bob');
        $user->setPower(4);
        $userServiceMock = $this
            ->getMockBuilder(UserService::class)
            ->getMock();
        $userServiceMock
            ->setMethods(['getUserById', 'canUserBeEnhanced']);

        // Here's where the magic happens
        $userServiceMock
            ->expects($this->exactly(2))
            ->method('getUserById')
            ->with(1)
            ->will($this->returnValue([$user]));

        $userServiceMock
            ->method('canUserBeEnhanced')
            ->with($user)
            ->will($this->returnValue(true));
        
        // Let's use the mock
        $userEnhancer = new UserEnhancer($userServiceMock);
        $userResult = $userEnhancer->enhance(1);

        $this->assertEquals(5, $userResult->power);
    }
}

```

La manera en que este test funciona es mockeando las respuestas de los médotos de `UserService`.

Sin embargo, solo con mockear las respuestas del método no es suficiente. Para que el test se pueda considerar completo, tenemos que controlar también que las llamadas a dichos métodos sean las adecuadas. Esto debido a que queremos asegurarnos de que nuestro componente usa de manera adecuada sus dependencias. Para esto, usamos `with()`, en conjunción con `expects()`. 

### Métodos `with` y `expect`

El método `with` controla que los llamados a los métodos de la dependencia mockeada sean hechos con los argumentos correctos. Cualquier argumento inadecuado hará fallar el test.

Por otro lado, el método `expects` controla la cantidad de veces que *cualquier método de dicho mock* será llamado. En el caso del ejemplo, la implentación únicamente llama a los métodos una vez cada uno, por lo que usamos `$this->exactly(2)`. Cualquier fallo en llamar a los métodos exactamente ese número de veces, resultará así mismo en una falla del test.

Tembién se puede usar `$this->any()`, así como otras opciones que están descritas en la documentación de Phpunit.

## Regla para mockear objetos

Una regla que suelo seguir cuando trato de decidir si mockear un objeto o no, es mirar si el objeto es externo a la clase específica que estoy probando. Si es así, a menos de que sea un objeto muy sencillo, construyo un mock.

En el ejemplo anterior, la única dependencia externa era el servicio `UserService`, pero en projectos de mayor complejidad la distinción entre qué mockear y qué no puede ser más difícil.

Otra cosa a tener en cueta, es que una vez de usa `expects` y `with` en un método mockeado, no es posible volver a definirlos en el mismo mock. Es por eso que es buena práctica definir un mock diferente para casa test, en la función `setup` de nuestra clase de tests, por ejemplo.