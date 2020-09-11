---
layout: page
lang: es
lang-ref: home
---

# Introducción

* TOC
{:toc}

En este proyecto pretendo mostrar el desarrollo completo de un proyecto simple utilizando _Domain Driven Design_ —DDD— con arquitectura hexagonal en PHP. El objetivo principal es explicar cómo implementar las partes más importantes de las diferentes capas que son parte de la arquitectura hexagonal. La intención es que el avance del proyecto sirva como guía de desarrollo, de modo que la teoría emanará de la práctica, y no al revés.

Así pues, desarrollaremos una aplicación web para gestionar un catálogo de libros, con la posibilidad de generar referencias bibliográficas en diferentes formatos: APA, Chicago...

El código del proyecto estará disponible en el repositorio [rubenrubiob/gestor-libros](https://github.com/rubenrubiob/gestor-libros).

## DDD y arquitectura hexagonal

El objetivo de este proyecto no consiste en explicar qué es DDD. Para quien quiera más información, hay abundancia de libros y recursos[^1] donde se explica en detalle. Por tanto, no vamos a preguntarnos «¿qué es DDD?»; más bien, como Heidegger hiciera en «Ser i tiempo», vamos a cambiar la pregunta: en lugar de preguntarnos por el ser de la cosa, nos preguntaremos por el sentido de ser de la cosa.

Es decir, no vamos a preguntarnos «¿qué es DDD?», sino «¿cuál es el sentido de DDD?». Como nos indica el nombre, su sentido es diseñar el proyecto, pensarlo, enfocándolo al _dominio_ o _negocio_. El dominio[^2] dicta cómo debe comportarse nuestra aplicación. Entre otras cosas, contiene:

- Los conceptos del dominio que debemos modelar.
- Las reglas o restricciones que se imponen para nuestros modelos.
- Las acciones o casos de uso de nuestra aplicación.

Es importante hacer notar que, en este punto, no nos preocupamos sobre infraestructura, y esta es una diferenciación crucial: no pensamos cómo vamos a almacenar los datos, ni el _framework_ que vamos a utilizar o las tecnologías del _frontend_. Nos importa cómo debe comportarse nuestra aplicación, independientemente de cómo se implemente, nada más. El sentido de DDD es, pues, pensar el proyecto centrándonos en el dominio.

Haciendo una burda analogía, el dominio sería la partitura de una sinfonía, mientras que la infraestructura una de entre muchas interpretaciones.

Ahora bien, DDD define cómo pensar el proyecto, pero no cómo implementarlo en código, puesto que hay diferentes maneras para hacerlo. En este proyecto utilizaremos arquitectura hexagonal, también conocida como arquitectura de puertos y adaptadores. Este patrón propone una separación por capas: tenemos una capa que es el dominio, a la cual se accede por unos puertos. Estos puertos se conectan con unos adaptadores, que pueden intercambiarse fácilmente. Así pues, se separa el dominio de la infraestructura, de modo que encaja con DDD.

Por ejemplo, un puerto podría ser la interfaz de un repositorio de lectura de un producto. A este puerto podríamos conectar adaptadores, que son las implementaciones concretas: un adaptador podría conectarse a la base de datos para obtener el producto, otro podría leerlo de fichero, otro podría obtenerlo de una API externa... En la capa de dominio no nos importa qué adaptador se use, son intercambiables. Esta explicación puede pecar de ser demasiado teórica, pero veremos ejemplos de uso a lo largo del desarrollo del proyecto.

Concretamente, tendremos las siguientes capas:

![](/images/general/capas-hexagonal.png)

- `Domain` (Dominio): donde residen los modelos que representan el negocio.
- `Application` (Aplicación): contiene los servicios de aplicación, que modelan los casos de uso de nuestra aplicación.
- `Infrastructure` (Infraestructura): contiene las implementaciones concretas de los servicios.
- `Ui` (por _User Interface_): es una capa más de infraestructura, pero que separamos por comodidad. En ella se encuentran los puntos de entrada a la aplicación: por peticiones HTTP, por línea de comandos...

Cada capa sólo puede acceder a los elementos de su capa y a los de las capa anteriores. Así pues, desde `Ui` podemos llamar a la capa `Application`, pero no así desde `Domain`.

En el proyecto también aplicaremos los principios SOLID[^3], que sirven para obtener un _software_ robusto. De igual modo, los iremos viendo a lo largo del desarrollo del proyecto.

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
    - Puede que no tengamos apellido del autor. Podemos utilizar este campo para el toponímico en algunos autores, como para Cristina de Pizán.
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


[^1]: Para a entrar a fondo en DDD, tenemos los [libros de Eric Evans](https://domainlanguage.com/ddd/reference/), que definió por primera vez el concepto, i los [de Vaugh Vernon](https://www.informit.com/authors/bio/5e9dfebf-9550-4a11-800d-35a57c5fa11e). Específico para PHP, tenemos el libro [«Domain-Driven Design in PHP»](https://leanpub.com/ddd-in-php).

[^2]: No me gusta el nombre «negocio», porque tiene la connotación que todo lo que desarrollemos debe tener como objetivo obtener un beneficio, lleva implícito un espíritu capitalista. Quizá alguien debería estudiar cómo el capitalismo está imbuido en el desarrollo de _software_, donde todo lo que se programa ha de ser productivo; si no lo es, se considera inválido. Utilizaremos el nombre «dominio», que, por contra, creo que no es suficientemente claro.

[^3]: Los definió [originalmente Robert C. Martin](http://butunclebob.com/ArticleS.UncleBob.PrinciplesOfOod). Encontramos una buena explicación en castellano sobre ellos en el [blog de Asier Marqués](https://asiermarques.com/2018/principios-solid/) i en el [blog de Fran Iglesias](https://franiglesias.github.io/principios-solid/).