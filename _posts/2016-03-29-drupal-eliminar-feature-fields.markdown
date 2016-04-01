---
layout: post
title:  "Eliminando campos de un tipo de contenido con el módulo Features de Drupal 7"
date:   2015-03-29 16:58:25
categories: php
tags: drupal drupal7 features
image: /assets/article_images/2016-03-29-drupal-eliminar-feature-fields/trash.jpg
comments: true
disqus_id: BD172294-3B52-41B7-A766-F260DBDBC3EC
---

El módulo **Features** de Drupal es uno de los módulos más útiles cuando queremos manejar en código configuraciones como variables de entorno, vocabularios de taxonomías o servicios web. Uno de los usos más extensos que se da de este módulo es el manejo de tipos de contenido. Gracias este módulo, podemos exportar nuestros tipos de contenido personalizados a un módulo _featurizado_ en código, que podemos instalar en cualquier instancia de Drupal para replicarlos fácilmente. 

Al cambiar el tipo de contenido que tenemos _featurizado_, por ejemplo, al ser añadido un campo, simplemente podemos recrear el módulo, o _feature_, y subir los cambios generados en nuestro código como un nuevo _commit_.

Por ejemplo, digamos que tenemos un tipo de contenido llamado _Producto_. Si queremos _featurizar_ este tipo de contenido, podemos hacerlo desde la interfaz gráfica en `Structure`->`Features`->`Create feature`, seleccionar el tipo de contenido y descargar el código generado. Este código lo podemos extraer en `/sites/all/modules` y habilitarlo como un módulo cualquiera en otra instalación de Drupal, y nuestro tipo de contenido _Producto_ estará disponible para su uso.

## Drush

[**Drush**](http://www.drush.org/en/master/install/) (Drupal Shell) es la mejor manera de interactuar con Drupal cuando tenemos a disposición la línea de comando del servidor. Cuando se tiene un ambiente de despliegue, por ejemplo, se suele desplegar y configurar Drupal haciendo uso de Drush. Para exportar un tipo de contenido, teniendo por sentado que el módulo Features ya está instalado, corremos el siguiente comando:

~~~
drush fe nombre_del_modulo node:producto
~~~ 

Esto va a generar y poner el código en el directorio adecuado, para que lo podamos añadir a nuestro sistema de versionamiento.

## Añadiendo campos

Cuando el tipo de contenido ha evolucionado, y hay nuevos campos que queremos incluir en nuestro módulo _featurizado_, simplemente corremos:

~~~
drush fu nombre_del_modulo
~~~

Y tendremos el nuevo código listo en nuestro módulo. Para aplicar los cambios de nuestro código en nuestra otra instancia de Drupal (que puede ser, por ejemplo, la instancia de pruebas o la instancia de producción), añadimos el comando:

~~~
drush features-revert nombre_del_modulo -y --force
~~~

al script que está realizando el despliegue. Hasta ahora todo tiene sentido, excepto por una cosa: ¿por qué necesitamos la bandera `--force`? Ya lo veremos más adelante.

## Eliminando campos

Ahora viene lo bueno. Por la manera en que el módulo Features, y los módulos _featurizados_ funcionan, al eliminar un campo de nuestra instancia local y re-generar el módulo, y luego aplicar los cambios en una instancia de Drupal que ya tenga la versión anterior instalada, el campo eliminado no se elimina de nuestra segunda instancia.

En pocas palabras, Features usa una lista de todos los componentes que están exportados en un dado módulo _featurizado_. Si un componente no está en dicha lista, simplemente lo ignora, y no lo toca. Ahora, los campos de un tipo de contenido son a su vez un tipo de componente, de modo que si eliminamos ese campo de la lista de componentes que están exportados en nuestro módulo, Features no va a hacer nada con los componentes ya existentes, por más que ya no estén en nuestra lista. Simplemente los va a ignorar.

Para eliminar efectivamente un campo de un módulo _featurizado_, esto es lo que yo hago:

### Eliminar los campos por código

Primero, implemento en el archivo `.module` del módulo _featurizado_ `hook_post_features_revert`, así como una función de ayuda para mantener las cosas _DRY_:

~~~ php
<?php

/**
 * Implements hook_post_features_revert()
 */
function nombre_del_modulo_post_features_revert($component) {
  $fields = array(
    'field_campo_a_eliminar'
  );
  
  // Let's delete those fields!
  foreach ($fields as $field) {
    _nombre_del_modulo_remove_field_instance('node', 'producto', $field);
  }
}

/**
 * This function deletes an instance of a field if it exists.
 *
 * @param string $entity_type The type of the entity, for example 'node'.
 * @param string $bundle_name The machine name of the bundle.
 * @param string $field_name The machine name of the field.
 */
function _nombre_del_modulo_remove_field_instance($entity_type, $bundle_name, $field_name) {
  // First, check that what we want to delete actually exists.
  if ($instance = field_info_instance($entity_type, $field_name, $bundle_name)) {
    // Invoke this hook to do all migration-related work.
    module_invoke_all('pre_delete_featurized_field', $instance);
    // Perform actual deletion.
    field_delete_instance($instance);
  }
}

~~~

Cuando deseo eliminar un campo, simplemente lo añado al arreglo `$fields`. 

En la función `_nombre_del_modulo_remove_field_instance` estoy invocando un _hook_ personalizado, `pre_delete_featurized_field`. Esto con el objeto de, en la implementación de dicho _hook_, realizar todos los movimientos de información y otras operaciones que tengamos que hacer antes de eliminar el campo definitivamente. Recordemos que al eliminar una instancia de un campo, todos los datos guardados allí también se eliminan. Muchas veces un requerimiento puede ser cambiar el tipo de un campo, de modo que para el usuario final sea transparente. Esto por lo general involucra crear un nuevo campo, migrar los datos, y eliminar el anterior campo. Se podría hacer tocando directamente la base de datos, pero a mi me gusta mantenerme alejado de la base de datos cuando trabajo con Drupal. 

A continuación, en mi máquina local, revierto el módulo _featurizado_: `drush features-revert nombre_del_modulo -y --force`. Esta es la primera vez que vemos el sentido en usar `--force`: Si no lo usamos, Features nos dirá que el módulo ya se encuentra aplicado en su totalidad, y que no es necesario revertirlo. Pero lo que queremos es forzar la ejecución de `hook_post_features_revert`, de modo que le decimos que lo revierta de todas maneras. Ahora, si vamos a la interfaz de configuación de nuestro tipo de contenido, veremos que el campo ya no existe.

### Regenerar el módulo _featurizado_

Ahora nos queda correr el comando para actualizar nuestro módulo:

~~~
drush fu nombre_del_modulo
~~~

Esto eliminará de la lista de componentes (y del código del módulo) los campos que ya no existen. Con esto, podemos correr `drush features-revert nombre_del_modulo -y --force` en nuestra otra instancia de Drupal, y no tendremos que preocuparnos por que los campos que eliminamos no queden realmente eliminados. Features va a omitir los campos que no aparecen en la lista, pero la implementación de `hook_post_features_revert` se va a encargar de eliminar los campos que no deseamos.

Como nota extra, acá nos percatamos por segunda vez de la necesidad de usar la bandera `--force`: si bien la lista de componentes cambió, es posible que los componentes mismos sigan en el mismo estado, por lo que Features nos dirá que el módulo ya se encuentra aplicado. Es por eso que necesitamos forzar la ejecución de la reversión.