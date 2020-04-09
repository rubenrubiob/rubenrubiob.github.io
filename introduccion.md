# Introducción

Este post es el primero de una serie en la que se pretende mostrar el desarrollo completo de un proyecto utilizando _Domain Driven Design_ —DDD— con arquitectura hexagonal en PHP. El objetivo principal es mostrar cómo implementar las partes más importantes de las diferentes capas que son parte de la aqruitectura hexagonal.

Para tal fin, desarrollaremos un catálogo de libros, para poder generar referencias bibliográficas en diferentes formatos —APA, Chicago...

La intención es que el desarrollo del proyecto sirva como guía de desarrollo, de modo que la teoría emanará de la práctica, y no al revés.

Antes de empezar a programar, debemos definir lo que se denomina _dominio_ o _negocio_. El domino[^1] dicta cómo debe comportarse nuestra aplicación. Entre otras cosas, contiene:

- Los conceptos del dominio que debemos modelar.
- Las reglas que nuestra acciones ha de seguir.
- Las acciones o casos de uso de nuestra aplicación.

Es importante hacer notar que, en este punto, no nos preocupamos sobre infraestructura. No pensamos cómo vamos a almacenar los datos, ni el _framework_ que vamos a utilizar o las tecnologías del _frontend_. Nos importa el dominio, nada más.

El dominio debe venir definido por la gente de producto, ya sean de nuestro equipo, ya sea un cliente. Siguiendo DDD, el lenguaje ha de ser ubicuo, es decir, se deben utilizar los conceptos del dominio en todo el proyecto. Por ejemplo, si la gente de producto llama _grupo_ a lo que comúnmente es una categoría, no debemos cometer el error de «traducirlo» como «categoría», sino que debemos mantener el nombre que utiliza la gente de negocio.

De esto se desprende que tampoco debemos traducir al inglés los conceptos del dominio. Este es un debate interesante (enlace: https://twitter.com/ProjectPolly/status/1169877299337945090 ) y no hay una solución universal aplicable a todos los proyectos.

Como muestra, en este proyecto los conceptos del dominio serán en castellano, y el resto, en inglés. Por ejemplo, tendremos el concepto «Libro», y podríamos tener una interfaz llamada `LibroRepository`. Al principio puede resultar extraño, pero, como todo, es cuestión de acostumbrarse.

La definición del dominio debe realizarse en diálogo continuo con los expertos del dominio. En este proyecto, como no hay ningún experto de dominio, la definición consistirá en una disertación filosófica sobre los libros. Si no es de interés del lector, puede pasar directamente a la segunda parte, la definición del proyecto.

Dominio: partitura de la sinfonía; infraestructura: interpretación

## Libros

El ámbito de este proyecto serán únicamente libros. No tendremos en cuenta otros medios escritos, como diarios o revistas. Y no reflejaremos todas las características que puede tener un libro, como su tamaño o el color del lomo, por ejemplo.

Así pues, para empezar, debemos preguntarnos «¿qué es un libro?» Parece una pregunta sencilla, puesto que en nuestra cultura occidental estamos familiarizados con los libros: existen librerías, bibliotecas, los hay por casa, en la escuela, en el instituto. Sin embargo, cuando profundizamos un poco, vemos que no es tan fácil de responder. ¿Cuál es el libro _original_? ¿El manuscrito del autor? ¿Un ejemplar de la primera edición? ¿Y qué pasa con las traducciones? ¿Es una traducción el mismo libro, aunque el texto sea diferente? No tenemos una respuesta cerrada a estas preguntas.

La pregunta «¿qué es un libro», por tanto, no nos dice mucho. Así pues, debemos cambiar la pregunta. Utilizando una aproximación heideggeriana, en lugar de preguntarnos por el libro, podríamos preguntarnos por el _sentido_ de un libro. La pregunta a hacernos, pues, es _¿cuál es el sentido de un libro?_

Podríamos decir que el sentido de un libro es contener un texto para ser leído. Vemos algo más de claridad aquí. Si es un texto, por un lado debe haber alguien, un _autor_ que lo escribiera; y por el otro, debe estar escrito en algún _idioma_.

Tenemos, por tanto, un _autor_. No vamos a preguntarnos ahora «¿qué es un autor?»[^2], puesto que ya sabemos su _sentido_ para nosotros: escribir un texto que estará contenido en un libro. 

## Conclusión



[^1]: No me gusta el nombre «negocio», porque tiene la connotación que todo lo que desarrollemos debe tener como objetivo obtener un beneficio, lleva implícito un espíritu capitalista. Quizá alguien debería estudiar cómo el capitalismo está imbuído en el desarrollo de _software_. Utilizaremos el nombre «dominio», que, por contra, creo que no es suficientemente claro.

[^2]: Michel Foucault tiene un libro en qué se pregunta precisamente «¿Qué es un autor?»