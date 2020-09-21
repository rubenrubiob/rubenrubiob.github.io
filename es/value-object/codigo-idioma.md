---
lang: es
lang-ref: value-object-codigo-idioma
tags: [domain,value-object,codigio,idioma,locale]
is-content-page: true
layout: content-page
order: 5
pull-request: 6
---

# _Value Object_: CodigoIdioma

* TOC
{:toc}

## Introducción

- Diferencia:
    - Libro
    - Aplicación

## _Value Object_: Locale

- Definido para la aplicación: ca, es, en
- Para traducciones de autor/nombre libro
- Para interfaz

### _Namespace_

### Implementación

### Test

## _Value Object_: CodigoIdioma

- Para edición de libro
- Podría ser un collection
- Necesitamos idiomas antiguos

### _Namespace_

### Implementación

### Test

En nuestra aplicación necesitamos poder trabajar con idiomas: para el idioma de publicación de una edición de un libro, para ver las traducciones de nombres de autores en diversos idiomas, para la propia interfaz... Así pues, desarrollaremos un _Value Object_ que permita identificar un idioma.

Sin embargo, como los nombres de los propios idiomas son traducibles, más que el nombre del idioma, lo que haremos será utilizar un código para identificar cada idioma, de modo que llamaremos al _Value Object_ `CodigoIdioma`.

Como es de suponer, no reinventaremos la rueda y crearemos un listado de códigos de idioma propio, sino que utilizaremos el estandar [ISO-639](https://en.wikipedia.org/wiki/ISO_639) se encarga de regular la representación de nombres de idioma. Ahora bien, el estándar tiene diversas secciones; las que se refieren al código de idioma son las dos primeras:

- [ISO-639-1](https://en.wikipedia.org/wiki/ISO_639-1): se trata de códigos de dos letras, como por ejemplo `es` para castellano, `en` para inglés y `ca` para catalán. Este 

- Necesitamos identificar el idioma en que en que se publicó una edición de un libro...
- Servirá como locale de aplicación ??
- Usar códigos ISO: ya hay regulación
- Opciones:
    - ISO-639-1: no suficientes: no tiene antiguos
    - ISO-639-2: es el que usaremos:
        - 3 caracteres, aceptando mayúsculas y minúsculas
- En las entidades, sólo guardaremos el código. Mostraremos el nombre en función de las necesidades
- No añadimos todos los códigos en el ValeuObject (demasiado complejo) --> usaremos interfaz para reconstruir objetos, con librerías

## _Namespace_

- Idioma

El _Value Object_ `Nombre` será reutilizable en más lugares, como por ejemplo el nombre de la ciudad en la entidad `Edicion`. Así pues, será un _Value Object_ genérico, y lo ubicaremos en el _namespace_ `Domain\ValueObject\`.

### Implementación

```php
<?php

declare(strict_types=1);

namespace Domain\ValueObject\Idioma;

use Domain\Exception\ValueObject\Idioma\CodigoIdiomaIsNotValid;
use InvalidArgumentException;
use Webmozart\Assert\Assert;

use function mb_strtolower;
use function trim;

/**
 * @psalm-immutable
 */
final class CodigoIdioma
{
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
            Assert::regex($parsedCodigo, '/^[a-z]{3}$/');
        } catch (InvalidArgumentException $e) {
            throw CodigoIdiomaIsNotValid::withCodigo($codigo);
        }

        return $parsedCodigo;
    }
}

```

- Limpiamos espacios de código
- Expresión regular con exactamente 3 caracteres; validación extra vendrá de interfaz

### Test

```php
<?php

declare(strict_types=1);

namespace Tests\Domain\ValueObject\Idioma;

use Domain\Exception\ValueObject\Idioma\CodigoIdiomaIsNotValid;
use Domain\ValueObject\Idioma\CodigoIdioma;
use PHPUnit\Framework\TestCase;

class CodigoIdiomaTest extends TestCase
{
    /**
     * @test
     * @dataProvider provideForFromCodigo
     */
    public function from_codigo(string $expected, string $codigo): void
    {
        $this->assertSame(
            $expected,
            CodigoIdioma::fromCodigo($codigo)->asString()
        );
    }

    /**
     * @test
     */
    public function from_codigo_with_codigo_vacio_throws_exception(): void
    {
        $this->expectException(CodigoIdiomaIsNotValid::class);

        CodigoIdioma::fromCodigo('  ');
    }

    /**
     * @test
     */
    public function from_codigo_with_codigo_corto_throws_exception(): void
    {
        $this->expectException(CodigoIdiomaIsNotValid::class);

        CodigoIdioma::fromCodigo('es');
    }

    /**
     * @test
     */
    public function from_codigo_with_codigo_largo_throws_exception(): void
    {
        $this->expectException(CodigoIdiomaIsNotValid::class);

        CodigoIdioma::fromCodigo('espa');
    }

    /**
     * @test
     * @dataProvider provideForEqualsTo
     */
    public function equals_to(bool $expected, string $primerCodigo, string $segundoCodigo): void
    {
        $this->assertSame(
            $expected,
            CodigoIdioma::fromCodigo($primerCodigo)->equalsTo(
                CodigoIdioma::fromCodigo($segundoCodigo)
            )
        );
    }

    /**
     * @return string[][]
     */
    public function provideForFromCodigo(): array
    {
        return [
            'codigo sin espacios' => [
                'cat',
                'cat',
            ],
            'codigo con espacios' => [
                'cat',
                ' cat ',
            ],
            'codigo con espacios y mayúsculas' => [
                'cat',
                ' CAT ',
            ],
        ];
    }

    /**
     * @return array[]
     */
    public function provideForEqualsTo(): array
    {
        return [
            'equals' => [
                true,
                'cat',
                'cat',
            ],
            'not equals' => [
                false,
                'eng',
                'cat',
            ],
        ];
    }
}

```