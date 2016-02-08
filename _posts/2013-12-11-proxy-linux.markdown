---
layout: post
title:  "Proxy en Linux"
date:   2013-12-11 18:00:00
categories: linux
tags: linux consola proxy
image: /assets/article_images/2013-12-11-proxy-linux/network.jpg
comments: true
disqus_id: cb47548b-fab5-459e-bed6-02721321b22c
---

En algunas organizaciones existe una red que requiere de autenticación por proxy para salir a Internet. Normalmente la configuración del proxy de la red es bastante sencilla y gráfica cuando estamos en un computador con Windows o Macintosh, incluso cuando no vamos a hacer otra cosa más que navegar en una máquina con Linux y alguno de los escritorios más populares. Muchas aplicaciones no toman en cuenta la configuración del proxy del sistema operativo y proveen una interfaz propia para estos menesteres. Es el caso de Dropbox o Synaptic, por ejemplo.
Esto se da aún con más frecuencia cuando estamos intentando acceder a Internet desde aplicaciones en modo de texto en Linux. Dado que estas aplicaciones no dependen de un entorno de escritorio, no pueden esperar que instalemos Gnome o KDE (por ejemplo) para configurar el proxy y usar esa configuración. Están, por así decirlo, una capa por debajo. 

## Configuración básica
Aún más por debajo de las aplicaciones en modo de texto se encuentran las variables de entorno del sistema. Las variables de entorno `http_proxy`, `https_proxy`, `ftp_proxy`, `rsync_proxy` y `no_proxy` son usadas por ciertos programas (como `wget` y `lynx`) para extraer la configuración del proxy. Tienen la forma `protocolo_proxy`. Por ejemplo:

~~~ bash
export http_proxy = http://10.203.0.1:5187/
export https_proxy = $http_proxy
~~~ 

## Automatizando
Alan Pope, según [este artículo](https://wiki.archlinux.org/index.php/proxy_settings) de ArchLinux en el que me basé, dio con una manera de automatizar la levantada del proxy:

~~~ bash
function proxy(){
  echo -n "Usuario:"
  read -e username
  echo -n "Clave:"
  read -es password
  export http_proxy="http://$username:$password@proxyserver:8080/"
  export https_proxy=$http_proxy
  export ftp_proxy=$http_proxy
  export rsync_proxy=$http_proxy
  export no_proxy="localhost,127.0.0.1,localaddress,.localdomain.com"
  echo -e "\Proxy configurado."
}
function proxyoff(){
  unset HTTP_PROXY
  unset http_proxy
  unset HTTPS_PROXY
  unset https_proxy
  unset FTP_PROXY
  unset ftp_proxy
  unset RSYNC_PROXY
  unset rsync_proxy
  echo -e "\nProxy removido."
} 
~~~ 

La idea es añadir esas funciones a nuestro archivo `.bashrc`. De ese modo, siempre que tengamos que usar el proxy desde la consola, primero llamamos la función `proxy`:

~~~ bash
$ proxy
Usuario:sergio
Clave:
Proxy configurado.
$ wget www.google.com
...
$ proxyoff
Proxy removido.
$
~~~ 

## ¿Y sudo?
Como para ahora se supondrá, la configuración del proxy no perdura cuando estamos haciendo tareas que involucran `sudo`, como instalar paquetes por medio de `apt`. Para esto, accedemos a la configuración de `sudo` por medio del comanto `visudo` y añadimos la siguiente línea:

~~~ bash
Defaults env_keep += "http_proxy https_proxy ftp_proxy"
~~~ 

Esto mantendrá las variables de entorno dadas del usuario que está llamando a `sudo`, en este caso, las correspondientes al proxy.

## _Disclaimer_
Esta entrada es básicamente una traducción al español de la documentación de ArchLinux encontrada en el artículo anterioemente mencionado. La escribo para poner la información a disposición de aquellos que puedan leerla, así como a manera de bitácora. 
La red de la Universidad Nacional me obligó a usar esta información. Que la red no haya funcionado y al final me haya dado la misma es otro cuento.