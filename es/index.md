---
layout: page
lang: es
lang-ref: home
---

# Introducción

En este proyecto pretendo mostrar el desarrollo completo de un proyecto simple utilizando _Domain Driven Design_ —DDD— con arquitectura hexagonal en PHP. El objetivo principal es explicar cómo implementar las partes más importantes de las diferentes capas que son parte de la arquitectura hexagonal. La intención es que el avance del proyecto sirva como guía de desarrollo, de modo que la teoría emanará de la práctica, y no al revés.

Para tal fin, desarrollaremos un catálogo de libros con el objetivo de poder generar referencias bibliográficas en diferentes formatos: APA, Chicago...

## Dominio e infraestructura

Sin embargo, antes de empezar a programar, debemos definir lo que se denomina _dominio_ o _negocio_. El dominio[^1] dicta cómo debe comportarse nuestra aplicación. Entre otras cosas, contiene:

- Los conceptos del dominio que debemos modelar.
- Las reglas o restricciones que se imponen para nuestros modelos.
- Las acciones o casos de uso de nuestra aplicación.

Es importante hacer notar que, en este punto, no nos preocupamos sobre infraestructura, y esta es una diferenciación crucial: no pensamos cómo vamos a almacenar los datos, ni el _framework_ que vamos a utilizar o las tecnologías del _frontend_. Nos importa cómo debe comportarse nuestra aplicación, independientemente de cómo se implemente, nada más. El propio nombre ya lo dice: Domain-Driven Design, diseño enfocado al dominio.

Haciendo una burda analogía, el dominio sería la partitura de una sinfonía, mientras que la infraestructura una de entre muchas interpretaciones.

TODO: añadir capas y deptrac

## Lenguaje ubicuo y traducción

El dominio debe venir definido por los expertos de producto, ya sean de nuestro equipo, ya sea un cliente externo. Siguiendo DDD, el lenguaje ha de ser ubicuo, es decir, se deben utilizar los conceptos del dominio en todo el proyecto. Por ejemplo, si los expertos de negocio llaman «grupo» a lo que comúnmente es una categoría, no debemos cometer el error de «traducirlo» como «categoría», sino que debemos mantener el nombre que utilizan los expertos, «grupo» en este caso, puesto que ellos son los conocedores del negocio. Como explica Gadamer en «Verdad y método», en la traducción siempre hay una pérdida del sentido original del texto.

En consecuencia, se desprende que tampoco debemos traducir al inglés los conceptos del dominio. Este es un [debate interesante](https://twitter.com/ProjectPolly/status/1169877299337945090) y no hay una solución universal aplicable a todos los proyectos. Depende del proyecto en sí, del equipo que lo compone, del ámbito en el que se desarrolla...

A modo de ejemplo, en este proyecto los conceptos del dominio serán en castellano, y el resto, en inglés. Por ejemplo, tendremos el concepto «Libro», y podríamos tener una interfaz llamada `LibroRepository`. Al principio puede resultar extraño, pero, como todo, es cuestión de acostumbrarse.

## Definición

Nuestra aplicación será multiidioma, es decir, estará disponible en más de un idioma: castellano, catalán e inglés. Por tanto, necesitaremos tener en cuenta que no sólo los literales de la web, sino también los modelos deberán ser traducibles.

Para este proyecto, crearemos primero la funcionalidad para almacenar la información que utilizaremos posteriormente para generar la bibliografía: libros y autores. Como el objetivo del proyecto es didáctico, no implementaremos un sistema completo que contenga todas las casuísticas del funcionamiento bibliográfico; nos perderíamos en detalles que oscurecerían la explicación.

### Autor

Así pues, tendremos autores, de los cuales almacenaremos la información básica:
    - Biografía: año de nacimiento y muerte.
    - Apelativo: nombre, apellidos y seudónimo del autor.

Tendremos las siguientes restricciones para dicha información:

- Biografía:
    - Puede que no sepamos la fecha exacta de nacimiento o de muerte del autor, como en el caso de Homero.
    - Para autores vivos, sólo tendremos la fecha de nacimiento.
    - En el caso en que sepamos ambas fechas, la fecha de nacimiento debe ser anterior a la de fallecimiento.
    - Podemos tener fechas anteriores antes de nuestra era (a.C.), tanto para el nacimiento como para la muerte.
    - El año 0 no existe en nuestro calendario.
- Apelativo:
    - Ha de ser traducible. Safo tiene el mismo nombre en catalán y en castellano, pero no así en inglés, idioma en el que se escribe _Sappho_.
    - Puede que no tengamos apellido del autor. Podemos utilizar este campo para el denominativo en algunos autores, como para Cristina de Pizán.
    - El pseudónimo es el nombre por el cual se conoce al autor. Así por ejemplo, el seudónimo de Aristócles de Atenas es Platón.

### Libro

Para nuestra aplicación, un libro es siempre de un autor. No contemplamos, por ahora, ni libros escritos por varios autores, ni libros con autor anónimo. Tampoco contemplamos la opción de que un libro pueda estar recogido en otras ediciones, como sucede a menudo en obras antiguas. Estas casuísticas podrían ser añadidas en el futuro.

Los libros siempre tienen ediciones, de modo que guardaremos por un lado la información básica del libro como obra, y por otro, la información de una edición. Así pues, tendremos la siguiente información:

- Libro:
    - El título y el subtítulo —que puede no existir— serán traducibles, igual que sucedía con el nombre del autor.
    - El año de publicación del libro puede ser cualquiera, no hace falta que coincida con la vida del autor, puesto que podemos encontrar obras póstumas.
- Edición:
    - De una edición de un libro guardaremos la ciudad, el año de publicación, la editorial, el idioma del libro y el formato.
    - Tendremos varios formatos de edición preestablecidos: libro de tapa dura, libro de bolsillo...

### Pantallas de la aplicación

Para nuestra aplicación tendremos un gestor —o _backoffice_— desde el que un usuario podrá añadir tanto autores como libros.

La interfaz de gestión de autor es la siguiente:

![](https://raw.githubusercontent.com/rubenrubiob/book-manager/master/docs/wireframe/backend-autor.png)

Para añadir traducciones primero hay que crear el autor, cuya información se almacenará en el idioma en que el usuario esté visualizando la web en ese momento.

La interfaz de gestión de libros es la siguiente:

![](https://raw.githubusercontent.com/rubenrubiob/book-manager/master/docs/wireframe/backend-libro.png)

De igual manera, tanto las traducciones como las ediciones se añaden una vez el libro ya ha sido creado.

## Herramientas

Este proyecto lo desarrollaremos con el siguiente entorno:

- PHP 7.4
- Symfony 5.1
- MySQL 5.7
- ElasticSearch 6.8
- Apache 2.4


[^1]: No me gusta el nombre «negocio», porque tiene la connotación que todo lo que desarrollemos debe tener como objetivo obtener un beneficio, lleva implícito un espíritu capitalista. Quizá alguien debería estudiar cómo el capitalismo está imbuido en el desarrollo de _software_, donde todo lo que se programa ha de ser productivo; si no lo es, se considera inválido. Utilizaremos el nombre «dominio», que, por contra, creo que no es suficientemente claro.