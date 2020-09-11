---
lang: es
lang-ref: value-object-apelativo
tags: [domain,value-object,nombre,apelativo]
is-content-page: true
layout: content-page
order: 4
pull-request: 5
---

# _Value Object_: Apelativo

## Introducción

A continuación implementaremos el _Value Object_ que nos servirá para representar el nombre con el que se conoce a un autor. Recordemos las restricciones que teníamos según la [definición de la introducción](https://rubenrubiob.github.io/es/#autor):

- Ha de ser traducible. Safo tiene el mismo nombre en catalán y en castellano, pero no así en inglés, idioma en el que se escribe _Sappho_.
- Puede que no tengamos apellido del autor. Podemos utilizar este campo para el toponímico en algunos autores, como para Cristina de Pizán.
- El pseudónimo es el nombre por el cual se conoce al autor. Así por ejemplo, el seudónimo de Aristócles de Atenas es Platón.

Se puede hacer una crítica a estas restricciones diciendo que son eurocentristas, porque pensamos en utilizar nombre y apellido, que no existen en todas las culturas; y la crítica sería acertada. Sin embargo, la edición de libros tal como la conocemos hoy en día nació en Europa y, por tanto, las referencias bibliográficas se basan justamente en este modelo cultural, ya que los formatos requieren especificar nombre y apellidos del autor por separado.

Así pues, como debemos dar cabida tanto a autores propiamente occidentales como de otras culturas, tendremos un _Value Object_ llamado `Apelativo` que, igual que el _Value Object_ `Biografia`, contendrá diversos atributos, tres en este caso:

- `Alias`: es como se conoce popularmente al autor. Con este campo, cubrimos todos los casos:
    - Los casos en que desconocemos nombre y apellidos del autor, como Homero.
    - Los casos en que se conoce al autor por su toponímico, como Cristina de Pizán.
    - Los casos en que el autor utiliza un pseudónimo diferente a su nombre. Tenemos los casos de Caterina Albert, que publicaba como Víctor Català, o de Aristocles de Atenas, que era conocido como Platón.
- `Nombre`: es el nombre del autor. Como puede ser que lo desconozcamos, es un campo que podrá ser `null`.
- `Apellido`: es el apellido o apellidos del autor. Como puede ser que lo desconozcamos, es un campo que podrá ser `null`.

Cada uno de estos atributos será un _Value Object_ de tipo `Nombre`.

En este momento no nos importa la primera restricción, que el nombre de un autor ha de ser traducible. Esa es una responsabilidad de la entidad `Autor`. Un _Value Object_ `Apelativo` representará el apelativo de un autor en cualquier idioma.

## _Value Object_: `Nombre`

### _Namespace_

El _Value Object_ `Nombre` será reutilizable en más lugares, como por ejemplo el nombre de la ciudad en la entidad `Edicion`. Así pues, será un _Value Object_ genérico, y lo ubicaremos en el _namespace_ `Domain\ValueObject\`.

### Implementación

```php
<?php

declare(strict_types=1);

namespace Domain\ValueObject;

use Domain\Exception\ValueObject\NombreIsNotValid;
use Domain\Exception\ValueObject\NombreIsVacio;
use InvalidArgumentException;
use Webmozart\Assert\Assert;

use function trim;

/**
 * @psalm-immutable
 */
final class Nombre
{
    public const MAX_LONGITUD = 255;

    private string $nombre;

    private function __construct(string $nombre)
    {
        $this->nombre = $nombre;
    }

    /**
     * @throws NombreIsNotValid
     * @throws NombreIsVacio
     */
    public static function fromString(string $nombre): self
    {
        return new self(
            self::parseAndValidate($nombre)
        );
    }

    public function asString(): string
    {
        return $this->nombre;
    }

    public function equalsTo(Nombre $anotherNombre): bool
    {
        return $this->nombre === $anotherNombre->nombre;
    }

    /**
     * @throws NombreIsNotValid
     * @throws NombreIsVacio
     */
    private static function parseAndValidate(string $nombre): string
    {
        $nombre = trim($nombre);

        try {
            Assert::stringNotEmpty($nombre);
        } catch (InvalidArgumentException $e) {
            throw NombreIsVacio::crear();
        }

        try {
            Assert::maxLength($nombre, self::MAX_LONGITUD);
        } catch (InvalidArgumentException $e) {
            throw NombreIsNotValid::withNombreDemasiadoLargo($nombre);
        }

        return $nombre;
    }
}

```

- En este caso sólo realizamos dos validaciones en el _Value Object_:
    - Que no esté vacío.
    - Que la longitud no sea superior a 255 caracteres. Este valor de 255 es arbitrario; con esta longitud, debería ser suficiente para almacenar cualquier nombre.
- A diferencia del _Value Object_ `Anyo`, lanzamos excepciones diferentes para cada una de las validaciones erróneas. Esto nos será útil en el _Value Object_ `Apelativo`, como veremos a continuación.
- Limpiamos los espacios en blanco al principio y al final del valor de entrada.

### Test

```php
<?php

declare(strict_types=1);

namespace Tests\Domain\ValueObject;

use Domain\Exception\ValueObject\NombreIsVacio;
use Domain\Exception\ValueObject\NombreIsNotValid;
use Domain\ValueObject\Nombre;
use PHPUnit\Framework\TestCase;

use function str_repeat;

class NombreTest extends TestCase
{
    /**
     * @test
     * @dataProvider provideForFromString
     */
    public function from_string(string $expectedNombre, string $nombre): void
    {
        $this->assertSame(
            $expectedNombre,
            Nombre::fromString($nombre)->asString()
        );
    }

    /**
     * @test
     */
    public function from_string_with_nombre_vacio_throws_exception(): void
    {
        $this->expectException(NombreIsVacio::class);

        Nombre::fromString('  ');
    }

    /**
     * @test
     */
    public function from_string_with_nombre_demasiado_largo_throws_exception(): void
    {
        $this->expectException(NombreIsNotValid::class);

        Nombre::fromString(
            str_repeat(
                'a',
                Nombre::MAX_LONGITUD + 1
            )
        );
    }

    /**
     * @test
     * @dataProvider provideForEqualsTo
     */
    public function equals_to(bool $expected, string $primerNombre, string $segundoNombre): void
    {
        $this->assertSame(
            $expected,
            Nombre::fromString($primerNombre)->equalsTo(
                Nombre::fromString($segundoNombre)
            )
        );
    }

    /**
     * @return string[][]
     */
    public function provideForFromString(): array
    {
        return [
            'sin espacios' => [
                'Platón',
                'Platón',
            ],
            'con espacio al principio' => [
                'Platón',
                ' Platón',
            ],
            'con espacio al final' => [
                'Platón',
                'Platón ',
            ],
            'con espacio al principio y al final' => [
                'Platón',
                ' Platón ',
            ],
        ];
    }

    /**
     * @return array[]
     */
    public function provideForEqualsTo(): array
    {
        return [
            'iguales' => [
                true,
                'Platón',
                'Platón',
            ],
            'iguales con espacio' => [
                true,
                'Platón ',
                '  Platón  ',
            ],
            'diferentes' => [
                false,
                'Platón',
                'Aristóteles',
            ],
        ];
    }
}

```

- Como lanzamos dos excepciones diferentes, debemos testear ambas en métodos separados, pero no es necesario mirar el mensaje de error para diferenciarlas.
- Comprobamos que la limpieza de espacios al principio y al final del valor funciona correctamente, tanto a la hora de crear el _Value Object_ como a la hora de compararlo.

## _Value Object_: `Apelativo`

### _Namespace_

Tal como lo hemos definido, `Apelativo` es un _Value Object_ específico de `Autor`. En consecuencia, lo situaremos en el _namespace_ `Domain\ValueObject\Autor`.

### Implementación

```php
<?php

declare(strict_types=1);

namespace Domain\ValueObject\Autor;

use Domain\Exception\ValueObject\Autor\ApelativoIsNotValid;
use Domain\Exception\ValueObject\NombreIsNotValid;
use Domain\Exception\ValueObject\NombreIsVacio;
use Domain\ValueObject\Nombre;

/**
 * @psalm-immutable
 */
final class Apelativo
{
    private Nombre $alias;
    private ?Nombre $nombre;
    private ?Nombre $apellidos;

    private function __construct(Nombre $alias, ?Nombre $nombre, ?Nombre $apellidos)
    {
        $this->alias     = $alias;
        $this->nombre    = $nombre;
        $this->apellidos = $apellidos;
    }

    /**
     * @throws ApelativoIsNotValid
     * @throws NombreIsNotValid
     */
    public static function fromParts(?string $alias, ?string $nombre, ?string $apellidos): self
    {
        return self::fromComponents(
            self::parseAndValidateAlias($alias),
            self::parseAndValidateComponent($nombre),
            self::parseAndValidateComponent($apellidos)
        );
    }

    public static function fromComponents(Nombre $alias, ?Nombre $nombre, ?Nombre $apellidos): self
    {
        return new self(
            $alias,
            $nombre,
            $apellidos
        );
    }

    public function alias(): Nombre
    {
        return $this->alias;
    }

    public function nombre(): ?Nombre
    {
        return $this->nombre;
    }

    public function apellidos(): ?Nombre
    {
        return $this->apellidos;
    }

    /**
     * @throws ApelativoIsNotValid
     * @throws NombreIsNotValid
     */
    private static function parseAndValidateAlias(?string $alias): Nombre
    {
        if ($alias === null) {
            throw ApelativoIsNotValid::withAliasVacio();
        }

        try {
            return Nombre::fromString($alias);
        } catch (NombreIsVacio $e) {
            throw ApelativoIsNotValid::withAliasVacio();
        }
    }

    /**
     * @throws NombreIsNotValid
     */
    private static function parseAndValidateComponent(?string $component): ?Nombre
    {
        if ($component === null) {
            return null;
        }

        try {
            return Nombre::fromString($component);
        } catch (NombreIsVacio $e) {
        }

        return null;
    }
}

```

- Siempre construimos el _Value Object_ `Apelativo` a partir de sus componentes, con el _named constructor_ `fromComponents`. La validación del formato de datos cuando el valor de entrada es un `string` se delega al _Value Object_ `Nombre`.
- La validación que realizamos es diferente para el alias que para nombre y apellido, puesto que el alias es siempre obligatorio. Así pues, cuando el _Value Object_ `Nombre` valida el valor de entrada para el alias, puede lanzar dos excepciones:
    - Que el valor esté vacío. En este caso, como el alias es obligatorio para el _Value Object_ `Apelativo`, lanzamos una excepción propia de este _Value Object_.
    - Que la longitud sea superior a la permitida. En este caso, dejamos pasar la excepción que lanza `Nombre`.
- De manera análoga, tenemos la validación para el nombre y apellido, que puede lanzar dos excepciones:
    - Que el valor esté vacío. En este caso, como ni el nombre ni el apellido son obligatorios para el _Value Object_ `Apelativo`, capturamos la excepción que lanza `Nombre` y devolvemos un valor `null`. Aquí es donde cobra sentido tener dos excepciones diferentes para la validación de `Nombre`.
    - Que la longitud sea superior a la permitida. En este caso, dejamos pasar la excepción que lanza `Nombre`.


### Test

Debido a su longitud, no reproducimos el test aquí. Puede consultarse en el _pull request_ correspondiente.

Cabe destacar únicamente que no se testean las excepciones que puede lanzar `Nombre`, pues ya las comprueba el propio test de `Nombre`.