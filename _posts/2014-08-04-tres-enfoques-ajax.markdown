---
layout: post
title:  "Tres enfoques para manejar AJAX"
date:   2014-08-04 15:24:00
categories: javascript
tags: ajax javascript
image: /assets/article_images/2014-08-04-tres-enfoques-ajax/choice.jpg
comments: true
disqus_id: 2c543f55-74ba-4306-93d0-2de2499cb241
---

En esta entrada comentaré tres métodos de manejar Ajax. Para efectos de la entrada usaré PHP y jQuery, ya que son tecnologías que conozco y que me permiten codificar los ejemplos de manera compacta y rápida, pero otras tecnologías del lado del servidor y del cliente puede ser usada. La idea de la entrada es ejemplificar los conceptos usados.

AJAX es una manera de enviar solicitudes al servidor y actualizar la página con la información obtenida sin realizar una recarga completa. La idea es minimizar el tráfico de red usado, al evitar cargar cosas que ya hemos cargado como imágenes o cabeceras, al tiempo que agilizando la experiencia del usuario con nuestra aplicación. En la entrada usaré jQuery, que es una librería de JavaScript muy popular para, entre otras, ayudarnos con este tipo de tareas.

Por ejemplo, digamos que tenemos un formulario de un campo en el que escribimos un número de identificación, y un espacio de respuesta en el que debe aparecer el nombre de la persona que estamos buscando, así:

~~~ html
<!DOCTYPE html>
<html>
<head>
    <title>Consulta de usuario</title>
</head>
</html>
<body>
<h1>Consultar usuarios</h1>
<form id="form_consulta">
        <label for="documento">Documento: </label>
        <input type="text" id="documento" name="documento" />
<div id="respuesta"></div>
</form>
</body>
</html>
~~~ 
Guardamos el código en un archivo PHP, `consulta.php`. Tenemos así un formulario con id `form_consulta`, que tiene un único campo, `documento`, y un `div` con id `respuesta`. 
Vamos a empezar realizando el proceso sin usar AJAX. Para esto, cambiamos el archivo para que contenga lo siguiente:


~~~ php
<?php
$usuarios = array(
    '123' => 'Pepe Pelotas',
    '535' => 'Juana Juliana',
    '903' => 'Pancho Pecas'
    );
?>
<!DOCTYPE html>
<html>
<head>
    <title>Consulta de usuario</title>
</head>
</html>
<body>
<h1>Consultar usuarios</h1>
<form id="form_consulta" action="handler.php" method="post">
        <label for="documento">Documento: </label>
        <input type="text" id="documento" name="documento" value="<?php echo isset($_POST['documento'])?$_POST['documento']:'';?>" />
        <input type="submit" value="Consultar"></input>
    </form>
<div id="respuesta">
        <?php
        if($_SERVER['REQUEST_METHOD'] == 'POST'){

            if(in_array($_POST['documento'], array_keys($usuarios))){
                echo "Usuario: " . $usuarios[$_POST['documento']];
            }
            else{
                echo "Usuario no encontrado!";
            }
        }
        ?></div>
</body>
</html>
~~~ 
El arreglo `$usuarios` en este caso contiene la información que quedemos obtener. Normalmente este arreglo sería reemplazado por una base de datos, por ejemplo. El siguiente cambio que vemos es en el elemento `input` con id `documento`. Ahora vamos a poner en ese campo lo que sea que el usuario escribió en la solicitud anterior, o nada. 

Dentro del `div` de respuesta viene la lógica del servidor que se va a encargar de realizar la consulta y escribirla en el documento. Sólo ejecutamos el código cuando se trata de una solicitud `POST`, y básicamente pintamos directamente lo que encontremos según la información proveída en la solicitud.

Por supuesto, el problema acá es que estamos recargando totalmente la página cada vez que hacemos una consulta. Estamos descargando la página completa cada vez que realizamos una consulta, y en esa descarga estamos volviendo a transmitir información que ya teníamos, como el título de la página. Eso es lo que queremos evitar.

## Primer enfoque: carga parcial de HTML

En este primer enfoque la idea es cargar parcialmente un trozo de código HTML desde el servidor y reemplazar algún trozo definido de nuestro documento con el HTML cargado. Vamos a devolver únicamente el `div` con la información requerida, y con eso reemplazaremos el `div` que pusimos. Nuestro archivo quedará así:

~~~ php
<?php
$usuarios = array(
    '123' => 'Pepe Pelotas',
    '535' => 'Juana Juliana',
    '903' => 'Pancho Pecas'
    );
if($_SERVER['REQUEST_METHOD'] == 'GET'){
    ?>
    <!DOCTYPE html>
    <html>
    <head>
        <title>Consulta de usuario</title>
        <script src="//ajax.googleapis.com/ajax/libs/jquery/1.11.1/jquery.min.js"></script>
    </head>
    </html>
    <body>
        <h1>Consultar usuarios</h1>
        <form id="form_consulta" action="handler.php" method="post">
            <label for="documento">Documento: </label>
            <input type="text" id="documento" name="documento" />
            <input type="button" id="enviar" value="Consultar"></input>
        </form>
        <div id="respuesta">
        </div>
        <script type="text/javascript">
            $(function(){
                $("#enviar").click(function(){
                    $.ajax({
                        type: "POST",
                        url: $("#form_consulta").attr("action"),
                        data: $("#form_consulta").serialize()
                    }).success(function(data){
                        $("#respuesta").replaceWith(data)
                    }).error(function(){
                        alert("error!")
                    })
                })
            })
        </script>
    </body>
    </html>
<?php
}
if($_SERVER['REQUEST_METHOD'] == 'POST'){
    if(in_array($_POST['documento'], array_keys($usuarios))){
        echo '<div id="respuesta">';
        echo "Usuario: " . $usuarios[$_POST['documento']];
        echo '</div>';
    }
    else{
        echo '<div id="respuesta">';
        echo "Usuario no encontrado!";
        echo '</div>';
    }
}
~~~ 

Tomemos un vistazo al código. Lo que más salta a la vista es que ahora estamos dividiendo claramente el cógido entre `POST` y `GET`. Cuando el método es `GET`, mostramos la misma página de consulta que veníamos mostrando, más un código en Javascript que miraremos en detalle más adelante. Si el método es `POST`, por otro lado, sólo mostramos un `div`, con el resultado del procesamiento en el servidor.

En el código Javascript hacemos uso de jQuery (que previamente cargamos desde Google) para hacer el llamado AJAX a nuestra página con el método `POST` y los datos del formulario. Una vez tengamos el `div` retornado, reemplazamos el `div` en el documento con el que obtuvimos con la función `replaceWith`.

Este enfoque nos permite reducir considerablemente el tráfico necesario para realizar ua consulta, obteniendo los mismos resultados de manera más rápida y fluida. Sin embargo, es posible reducir aún más el uso de la red, sólo extrayendo del servidor los datos necesarios.

## Segundo enfoque: carga de JSON

En el anterior enfoque cargamos desde el servidor únicamente el HTML que necesitamos actualizar, no toda la página. Como ejemplo, la respuesta del servidor en el caso de una consulta con el documento 535, se vería así:

~~~ html
<div id="respuesta">Usuario: Juana Juliana</div>
~~~ 

Sin embargo, el tamaño de la respuesta se puede disminuir aún más. Podemos devolver *únicamente* los datos que necesitamos, en este caso el nombre del usuario. Elementos concernientes con el HTML no son concernientes al servidor, de modo que vamos a omitirlos en la respuesta. Para tal efecto, devolveremos los datos que nos interesan, junto a un código de error que vamos a utilizar para mostrar la respuesta según el caso. Todo esto irá en formato JSON. Para que la nueva solución funcione, tenemos que modificar el código Javascript y del lado del servidor (PHP).

Javascript:

~~~ javascript 
$(function(){
    $("#enviar").click(function(){
        $.ajax({
            dataType: "json",
            type: "POST",
            url: $("#form_consulta").attr("action"),
            data: $("#form_consulta").serialize()
        }).success(function(data){
            data = $.parseJSON(data);
            $("#respuesta").html("");
            switch(data.code){
                case "NF":
                $("#respuesta").html("Usuario no encontrado!");
                break;
                case "OK":
                $("#respuesta").html("Usuario: " + data.response);
                break;
            }
        }).error(function(){
            alert("error!")
        })
    })
})
~~~ 

PHP:

~~~ php
if($_SERVER['REQUEST_METHOD'] == 'POST'){
    $respuesta = array();
    if(in_array($_POST['documento'], array_keys($usuarios))){
        $respuesta['code'] = 'OK';
        $respuesta['response'] = $usuarios[$_POST['documento']];
    }
    else{
        $respuesta['code'] = 'NF';
    }
    echo json_encode(json_encode($respuesta));
}
~~~ 

Si bien en este ejemplo no se ve claramente el ahorro en ancho de banda que se obtiene con este enfoque, con una mayor cantidad de datos el cambio es importante.

## Tercer enfoque: HTML Templates

Cuando tenemos una gran cantidad de datos que obtenemos del servidor, podemos usar el enfoque de HTML Templates para aliviar un poco el trabajo de actualizar los datos en la página. La idea en este enfoque es crear una plantilla de la información a mostrar, poniendo marcadores en las posiciones en que los datos del servidor irán. Una manera de lograr esto es creando una etiqueta `script` con un descriptor de lenguaje que sepamos que el navegador no soporta, y escogiendo un patrón de caracteres fácilmente identificable, para reemplazar posteriormente con los datos provenientes del servidor:

~~~ html
<script id="responseTemplate" type="html/template">
    <div id="respuesta">{{response}}</div>
</script>
~~~ 

Nuevo script:

~~~ javascript
switch(data.code){
    case "NF":
    $("#respuesta").html("Usuario no encontrado!");
    break;
    case "OK":
    var template = $("#responseTemplate").html()
    for(key in data){
        template = template.replace("{{" + key + "}}", data[key])
    }
    $("#respuesta").replaceWith(template);
    break;
}
~~~ 

Podemos ver que en el bucle `for` estamos iterando los datos provenientes del servidor, y reemplazándolos en los marcadores que previamente preparamos en la plantilla. Finalmente, ponemos la plantilla ya llena en la página para mostrar. 
