---
lang: es
lang-ref: value-object-idioma
tags: [domain,value-object,codigo,idioma,locale]
is-content-page: true
layout: content-page
order: 5
pull-request: 6
---

# _Value Object_: Locale y CodigoIdioma

* TOC
{:toc}

## Introducción

En nuestra aplicación necesitamos poder trabajar con idiomas: la lengua o lenguas de publicación de una edición de un libro, las traducciones de nombres de autores en diversos idiomas, la propia interfaz... Así pues, necesitamos desarrollar _Value Objects_ que representen idiomas.

Sin embargo, como los propios nombres de los idiomas son traducibles, más que el nombre del idioma, lo que haremos será utilizar códigos para identificar cada idioma. Como es de suponer, no reinventaremos la rueda y crearemos un listado de códigos de idioma propio, sino que utilizaremos el estándar [ISO-639](https://en.wikipedia.org/wiki/ISO_639), que se encarga de regular la representación de nombres de idioma. Ahora bien, el estándar tiene diversas secciones; las que se refieren al código de idioma son las dos primeras:

- [ISO-639-1](https://en.wikipedia.org/wiki/ISO_639-1): se trata de un código de dos letras, como por ejemplo `es` para castellano, `en` para inglés o `ca` para catalán. Contiene idiomas actuales, y es la codificación que suele utilizarse en páginas web para identificar el idioma, como `es.wikipedia.org` o `ca.wikipedia.org`.
- [ISO-639-2](https://en.wikipedia.org/wiki/ISO_639-2): se trata de un código compuesto de tres letras, como `spa` para castellano, `eng` para inglés o `cat` para catalán. Surgió para superar la limitación de los códigos ISO-639-1, que no permitían representar tantos idiomas. Esta sección contiene también códigos para lenguas muertas, como latín o griego clásico.

En nuestra aplicación necesitaremos ambas codificaciones, de modo que implementaremos dos _Value Object_: `Locale` y `CodigoIdioma`. Su explicación se detalla a continuación.

## _Value Object_: Locale

Este _Value Object_ servirá para identificar el idioma de la aplicación web. Tal como se [definió en la introducción](https://rubenrubiob.github.io/es/#definición), la web sólo estará disponible en tres idiomas: catalán, castellano e inglés.

Alguien podría argumentar que el idioma no pertenece a la capa de Dominio sino a la de Infraestructura, pues es algo que atañe a la interfaz de visualización de la aplicación. Y esta afirmación sería cierta. Sin embargo, se definió que los modelos fuesen traducibles, como por ejemplo el nombre de un autor. Así pues, también necesitamos este _Value Object_ en la capa de Dominio.

En resumen, utilizaremos el _Value Object_ `Locale` tanto en la capa de Dominio, para la traducción de modelos, como en la de Infraestructura, para la interfaz de usuario y las URL. Es así porque se define desde la parte de negocio, es decir, lo dicta el Dominio.

En este caso, según está especificado, limitaremos los códigos posibles a los tres definidos, para catalán, castellano e inglés, en el formato ISO-639-1: `ca`, `es` y `en`, respectivamente. Si en algún momento el Dominio cambiase y quisiéramos aceptar algún otro idioma, deberíamos modificar el _Value Object_.

### _Namespace_

Agruparemos los _Value Object_ correspondientes a idioma, de modo que situaremos `Locale` en el _namespace_ Domain\ValueObject\Idioma`.

### Implementación

La implementación del _Value Object_ es la siguiente:

```php
<?php

declare(strict_types=1);

namespace Domain\ValueObject\Idioma;

use Domain\Exception\ValueObject\Idioma\LocaleIsNotValid;
use InvalidArgumentException;
use Webmozart\Assert\Assert;

use function strtolower;
use function trim;

/**
 * @psalm-immutable
 */
final class Locale
{
    public const CA = 'ca';
    public const ES = 'es';
    public const EN = 'en';

    public const LOCALE_MAP = [
        self::CA => self::CA,
        self::ES => self::ES,
        self::EN => self::EN,
    ];

    private string $codigo;

    private function __construct(string $codigo)
    {
        $this->codigo = $codigo;
    }

    public static function fromCodigo(string $codigo): self
    {
        return new self(
            self::parseAndValidateCodigo($codigo)
        );
    }

    public static function ca(): self
    {
        return new self(self::CA);
    }

    public static function es(): self
    {
        return new self(self::ES);
    }

    public static function en(): self
    {
        return new self(self::EN);
    }

    public function asString(): string
    {
        return $this->codigo;
    }

    public function equalsTo(Locale $anotherLocale): bool
    {
        return $this->codigo === $anotherLocale->codigo;
    }

    /**
     * @throws LocaleIsNotValid
     */
    private static function parseAndValidateCodigo(string $codigo): string
    {
        $parsedCodigo = strtolower(
            trim(
                $codigo
            )
        );

        try {
            Assert::keyExists(self::LOCALE_MAP, $parsedCodigo);
        } catch (InvalidArgumentException $e) {
            throw LocaleIsNotValid::withCodigo($codigo);
        }

        return $parsedCodigo;
    }
}

```

- Como viene definido desde el Dominio, sólo aceptaremos tres códigos de idioma, `ca`, `es` y `en`, que definimos con constantes dentro de la clase.
- Para simplificar la construcción del objeto en los tests o en casos por defecto, tendremos un _named constructor_ por cada idioma del Dominio.
- Para la validación, limpiamos de espacios y convertimos a minúsculas el _string_ de entrada y simplemente miramos que el código sea uno de los posibles. Para ello, comprobamos que el código exista en una de las claves del _array_ declarado en la constante `LOCALE_MAP`.
- Todas las constantes las declaramos como públicas: `LOCALE_MAP` para utilizarla en la excepción `LocaleIsNotValid`, y el resto, para los tests.

### Test

Debido a su longitud, no reproducimos el test aquí. Puede consultarse en el _pull request_.

## _Value Object_: CodigoIdioma

Además del idioma de la aplicación, necesitaremos identificar el idioma en que se publicó una edición de un libro. En este caso, podemos encontrarnos libros publicados en lenguas muertas, como el latín o el griego, de modo que no podemos utilizar el estándar ISO-639-1 como hemos hecho en el _Value Object_ `Locale`, y deberemos utilizar el estándar ISO-639-2, que contiene códigos de idioma de muchas más lenguas, incluyendo idiomas en desuso.

Para simplificar la explicación, por ahora no contemplaremos las ediciones de libros en varios idiomas, como por ejemplo las obras bilingües o trilingües. Es una funcionalidad que podremos añadir en el futuro.

### _Namespace_

Siendo el _Value Object_ `CodigoIdioma` específico de las ediciones de libro, lo podríamos situar en el _namespace_ `Domain\ValueObject\Libro\Edicion`. Sin embargo, tal como hemos explicado en el _Value Object_ `Locale`, agruparemos los _Value Object_ relativos a gestión de idioma. Así pues, lo ubicaremos en el _namespace_ `Domain\ValueObject\Idioma`.

### Implementación

La implementación es la que sigue:

```php
<?php

declare(strict_types=1);

namespace Domain\ValueObject\Idioma;

use Domain\Exception\ValueObject\Idioma\CodigoIdiomaIsNotValid;
use InvalidArgumentException;
use Webmozart\Assert\Assert;

use function strtolower;
use function trim;

/**
 * @psalm-immutable
 */
final class CodigoIdioma
{
    private const REGEX_CODIGO = '/^[a-z]{3}$/';
    private string $codigo;

    private function __construct(string $codigo)
    {
        $this->codigo = $codigo;
    }

    public static function fromCodigo(string $codigo): self
    {
        return new self(
            self::parseAndValidateCodigo($codigo)
        );
    }

    public function asString(): string
    {
        return $this->codigo;
    }

    public function equalsTo(CodigoIdioma $anotherCodigoIdioma): bool
    {
        return $this->codigo === $anotherCodigoIdioma->codigo;
    }

    /**
     * @throws CodigoIdiomaIsNotValid
     */
    private static function parseAndValidateCodigo(string $codigo): string
    {
        $parsedCodigo = strtolower(
            trim(
                $codigo
            )
        );

        try {
            Assert::regex($parsedCodigo, self::REGEX_CODIGO);
        } catch (InvalidArgumentException $e) {
            throw CodigoIdiomaIsNotValid::withCodigo($codigo);
        }

        return $parsedCodigo;
    }
}

```

- A diferencia del _Value Object_ `Locale`, como contemplamos el uso de cualquier idioma, no añadimos todos los códigos posibles en el _Value Object_, pues es demasiado complejo y nos implicaría mantenerlos. En su lugar, utilizaremos una interfaz para reconstruir y validar los códigos de idioma, con el uso librerías dedicadas. Haremos esta implementación más adelante.
- Siguiendo el punto anterior, en el _Value Object_ simplemente validamos con una expresión regular que el código de entrada, limpio de espacios y convertido a minúsculas, tenga tres letras.

### Test

Debido a su longitud, no reproducimos el test aquí. Puede consultarse en el _pull request_.