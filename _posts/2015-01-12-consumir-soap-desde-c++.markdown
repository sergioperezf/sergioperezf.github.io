---
layout: post
title:  "Consumir un servicio SOAP desde C++ usando gSOAP"
date:   2015-01-12 23:23:00
categories: c++
tags: soap webservices c++
image: /assets/article_images/2015-01-12-consumir-soap-desde-c++/machine.jpg
comments: true
disqus_id: 0b258a2b-802b-47c8-b365-3a30cebf39a0
---

Es posible consumir servicios web SOAP usando C++, usando una librería llamada gSOAP. Esta librería funciona también para crear servicios usando C++ que pueden ser desplegados con CGI, pero ese es tema de otra entrada. En esta entrada, me enfocaré en qué hacer para consumir un servicio web desde una aplicación C++, gracias a la librería mencionada, gSOAP.

GSOAP es una librería de código abierto para interactuar con servicios web SOAP desde programas en C y C++. El programa resultante de esta entrada consumirá el servicio de calculadora que se encuentra en http://ws1.parasoft.com/glue/calculator. Este servicio se encuentra listado en algunos directorios públicos, y tiene las operaciones <code>add</code>, <code>divide</code>, <code>multiply</code> y <code>substract</code>, para cada una de las operaciones aritméticas básicas. 

### Obtener gSOAP

Si estamos trabajando en un ambiente Linux, es muy posible que el paquete esté incluido en los paquetes oficiales de la distribución. En mi caso, estoy trabajando en Ubuntu 14.04, y tuve que instalar los paquetes `gsoap`, `libgsoap4` y `libgsoap-dev`. GSOAP contiene dos ejecutables que usaremos en el proceso, `wsdl2h` y `soapcpp2`. El primero genera las cabeceras de nuestro programa a partir de un documento WSDL que le demos, mientras que el segundo genera las implementaciones y las estructuras de datos.

En caso de no tener gSOAP en el repositorio oficial de nuestra distribución, o de estar trabajando con otro sistema operativo, tendremos que ajar gSOAP.GSOAP se puede obtener desde su página web oficial, http://www.cs.fsu.edu/~engelen/soap.html. 

Antes de empezar a generar el código usando la herramienta, tenemos que tener clara la ubicación de los siguientes archivos:

- `typemap.dat`
- `stlvector.h`

Estos se encuentran en la carpeta que descargamos de la página oficial, o en mi caso, en `/usr/share/gsoap`.

### Generar código a partir de WSDL

Primero, ejecutamos `wsdl2h` sobre el WSDL del servicio que queremos consumir.

`$ wsdl2h -o calculator.h -I/usr/share/gsoap/WS http://soaptest.parasoft.com/calculator.wsdl`

Con la opción `-o`, especificamos el nombre del archivo de salida. Con la opción `-I`, le decimos a `wsdl2h` el directorio donde puede encontrar ciertos archivos que necesita, como `typemap.dat`. Como último parámetro pasamos la dirección del WSDL. Ahora tenemos un archivo en nuestro directorio: `calculator.h`.

A continuación vamos a usar la herramienta `soapcpp2` para generar el código que vamos a usar para escribir nuestro cliente.

`$ soapcpp2 -j -C -I/usr/share/gsoap/import/ calculator.h`

Con la opción `-j`, generamos objetos y proxies en C++, que vamos a usar más adelante desde nuestro código. Con la opción `-C`, le decimos a la herramienta que sólo genere código del lado del cliente. Como dije al principio de esta entrada, crear servicios usando C++ y gSOAP es un tema para otro día. Al igual que con el comando anterior, la opción `-I` le dice a la herramienta en qué directorios puede encontrar archivos necesarios, como `stlvector.h`.

Al ejecutar este comando tendremos en el directorio de trabajo todos los archivos necesarios para empezar a escribir un pequeño programa de prueba de cliente.

### Crear proyecto y escribir el cliente

A continuación vamos a nuestro IDE favorito y creamos un proyecto con los siguientes archivos:

- ICalculator.nsmap
- soapC.cpp
- soapH.h
- soapCalculatorProxy.cpp
- soapCalculatorProxy.h
- soapStub.h

Si estamos trabajando con la versión que bajamos desde la página oficial, también añadimos `stdsoap2.cpp` y `stdsoap2.h`.

Creamos un archivo `main.cpp`, y pegamos el siguiente código:

~~~ cpp
#include "soapICalculatorProxy.h"
#include "ICalculator.nsmap"

int main() {
    // define the objects
    ICalculatorProxy service;
    soap *theSoap = soap_new();
    SOAP_ENV__Header theHeader;

    _ns1__multiply theMultiplication;
    theMultiplication.x = 22.0;
    theMultiplication.y = 59.0;
    _ns1__multiplyResponse theMultiplicationResponse;

    // connect up the objects
    theSoap->header = &theHeader;
    service.soap = theSoap;

    printf("Calling Web Service...\n");
    // run the multiplication
    if (service.multiply(&theMultiplication, &theMultiplicationResponse) == SOAP_OK) {
        // if we got results, print them out
        printf("Result: %f\n", theMultiplicationResponse.Result);
    } 

    // delete the service object and free the memory
    service.destroy();

    return 0;
}
~~~

El anterior es el código de prueba en el que vamos a llamar a los proxies generados por gSOAP, para hacer uso del servicio web. Como podemos ver, creamos una variable tipo `_ns1__multiply`, la cual llenamos con los parámetros que queremos pasar al servicio. El llamado al proxy, cuando es exitoso, retorna `SOAP_OK`, y guarda el resultado en una variable tipo `_ns1__multiplyResponse`. La salida de el programa anterior es:

~~~
Calling Web Service...
Result: 1298.000000
~~~

