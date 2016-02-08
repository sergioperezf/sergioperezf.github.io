---
layout: post
title:  "Mostrar ventanas de Linux en Windows usando X"
date:   2014-08-11 11:33:00
categories: linux
tags: linux xserver windows
image: /assets/article_images/2014-08-11-ventanas-linux-en-windows/reflections.jpg
comments: true
disqus_id: 6f49a2ce-cbf7-47d4-a997-1af8522ff9a3
---

Uno de los aspectos más interesantes y con menos frecuencia mirados de X, es su naturaleza cliente-servidor. Desde sus inicios, X fue diseñado de manera que pudiera sacar todo el provecho posible de la red. Esto nos permite realizar cosas muy interesantes, como trabajar en varios computadores al tiempo desde una sola máquina o enviar ventanas a otros computadores. Precisamente eso último es lo que voy a describir en esta entrada.

En mi escritorio cuento con dos máquinas en las que normalmente trabajo:

* Un escritorio (viejo portátil convertido) con Ubuntu 12.04.
* Un portátil con el Windows 7 con el que venía.

Me gusta más trabajar en el escritorio cuando se trata de temas de programación, puesto que me parece que Linux es un ambiente más amigable para estos menesteres. Sin embargo, por temas de compatibilidad de aplicaciones y hardware, también suelo estar un buen tiempo en el portátil. Dado que mi computador con Windows solo cuenta con 4 GB de memoria RAM y que Windows no es tan bueno manejando memoria cono Linux, muchas veces tengo que mover la silla y cambiar de máquina varias veces en unos minutos.

Acepto que este puede no ser el mejor motivador para configurar lo que describiré en esta entrada, pero el hecho de que _se puede hacer y es interesante_ también es un motivo válido.

Para este tutorial vamos a mostrar una ventana de Linux en el escritorio de Windows, por medio el protocolo X. Para esto vamos a:

1. Instalar X y SSH en Windows por medio de Cygwin
2. Configurar SSH en Linux (asumo que SSHD ya está instalado)
3. Conectar los dos computadores

Es mejor que los computadores involucrados estén conectados a la red por medio de cable, para mejorar el rendimiento.

### Instalación de Cygwin

Cygwin es un proyecto que pretende dar a usuarios de Windows un entorno similar a Unix para aplicaciones nativas a Windows y para desarrollo. Dentro de las aplcaciones proveídas por Cygwin están el cliente SSH y el servidor X requeridos para lo que vamos a hacer. Para instalar Cygwin vamos a ir a la <a href="https://cygwin.com/install.html">página</a> del proyecto y seguir las instrucciones, según queramos la versión de 32 o 64 bits. Los paquetes que vamos a instalar son los siguientes:

* `xorg-server`, es el servidor X para correr los programas gráficos.
* `xinit`, es requerido para iniciar el servidor X.
* `openssh`, para realizar conesiones seguras a los programas cliente (en mi caso la máquina con Ubuntu).

![Instalación Cygwin](/assets/article_images/2014-08-11-ventanas-linux-en-windows/cygwin-openssh-install-scp-ssh.gif)

Podemos usar la barra de búsqueda para buscar los paquetes. Una vez instalado el entorno en la máquina Windows, vamos a proseguir con el siguiente paso.

### Configurar SSH en Linux

Para este punto voy a asumir que el servidor de ssh ya se encuentra instalado en la máquina con Linux. Si no es así, en Ubuntu esto es fácil de hacer con `apt-get install openssh`. Vamos a configurar ssh para poder conectarnos seguramente de un equipo al otro.

Abrimos el archivo `/etc/ssh/sshd_config` con algún editor de texto, y nos aseguramos que la siguiente opción está activa (no está comentada y existe):

```
X11Forwarding yes
```

Esto habilitará la conexión segura entre el computador que tiene el programa y el que tiene el servidor X. De otra manera tendríamos que abrir el servidor X con la opción `-ac`, lo que deshabilita la autenticación y puede abrir nuestro equipo a ataques de seguridad.

### Conectar las máquinas para usar la funcionalidad

Ahora que todo está configurado, vamos a iniciar un programa cliente en el computador con Linux, y usarlo en el computador con Windows. Primero, iniciamos el servidor X en Windows. Abrimos la terminal de Cygwin y introducimos el siguiente comando:

```
X -multiwindow &
```

Esto inicia el servidor X con la opción `multiwindow`, que significa que por cada ventana que el cliente cree en el computador con Linux, se va a crear una ventana de Windows. El parámetro `&amp;` nos devolverá el control una vez termine de iniciar el servidor, para poder seguir introduciendo comandos. Ahora vamos a iniciar la variable `DISPLAY` para que el reenvío por SSH funcione:

```
export DISPLAY=:0
```

Con esto podemos conectarnos al computador donde residen nuestros programas:

```
ssh -Y usuario@servidor
```

Nos pedirá la contraseña y una vez adentro, ya podemos iniciar la aplicación que queramos, la cual aparecerá en nuestro escritorio de Windows como una ventana mas, por ejemplo:

```
gnome-system-monitor
```

![Una ventana más](/assets/article_images/2014-08-11-ventanas-linux-en-windows/windows.png)

***

Existe otra herramienta para realizar este proceso más fácilmente llamada `xming`, que provee un servidor X para Windows listo para ser usado. Sin embargo en mis pruebas el funcionamiento en general de Cygwin X fue muy superior, de modo que considero que el esfuerzo vale la pena.

Por supuesto, también es posible realizar el mismo proceso al revés. En esta caso la limitante está en que sólo los programas gráficos proveídos por Cygwin serán visibles en el escritorio de Linux usando este método. Es decir, tener Visual Studio, por ejemplo, funcionando en el escritorio de Linux no sería posible. Para lograr esto considero que lo más sensato es usar una aplicación de escritorio remoto como TeamViewer, o virtualizar un entorno Windows. Hay soluciones más óptimas pero requieren mucho más trabajo.