---
lang: es
lang-ref: value-object-biografia
tags: [domain,value-object,biografia,año]
is-content-page: true
layout: content-page
order: 3
pull-request: 4
---

# _Value Object_: Biografía

## Introducción

El siguiente _Value Object_ que implementaremos es el que representa la biografía de un autor, es decir, contendrá los años de nacimiento y de fallecimiento de un autor. Como se explica en la [introducción](https://rubenrubiob.github.io/es/), hay varias restricciones respecto a la biografía de un autor. Las reproducimos a continuación:

- Puede que no sepamos la fecha exacta de nacimiento o de muerte del autor, como en el caso de Homero.
- Para autores vivos, sólo tendremos la fecha de nacimiento.
- En el caso en que sepamos ambas fechas, la fecha de nacimiento debe ser anterior a la de fallecimiento.
- Podemos tener fechas anteriores antes de nuestra era (a.C.), tanto para el nacimiento como para la muerte.
- El año 0 no existe en nuestro calendario.

Por ahora, con el fin de simplificar la implementación, no contemplaremos la primera restricción, es decir, no tendremos en cuenta los casos en los que desconozcamos algunas de las fechas. En el futuro podremos añadir una extensión del _Value Object_ que permita introducir el siglo de nacimiento y el de muerte del autor, por ejemplo.

A diferencia de los _Value Objects_ vistos hasta ahora, que sólo tenían un atributo, `Biografia` tendrá un par: el año de nacimiento y el año de muerte, pudiendo este último ser `null` para autores en vida.

Podríamos tener ambos atributos como `int` dentro del _Value Object_. Sin embargo, en la entidad `Edicion` también tenemos el año de publicación de un libro, de modo que, si optáramos por esta opción, acabaríamos necesitando las validaciones de año en ambos _Value Object_. Así pues, crearemos un _Value Object_ que represente un año, y que se utilizará dentro de `Biografia`.

## _Value Object_: `Anyo`

Podríamos guardar valores temporales más concretos, como por ejemplo el día y mes de nacimiento de un autor o de publicación de un libro. Pero son muchas las ocasiones en las que desconocemos las fechas exactas —¿qué día nacieron Platón o Aristóteles?—, así que con la precisión de año tenemos suficiente, según hemos especificado en el Dominio. En todo caso, podríamos ampliar esta precisión en el futuro, si fuere necesario.

Tratándose de un dato de tiempo, podríamos pensar en prescindir del uso de un _Value Object_ y utilizar directamente las clases que nos proporciona el lenguaje, `DateTime` y `DateTimeImmutable`. Sin embargo, estaríamos utilizando una precisión extra que [podría complicarnos el desarrollo posterior](http://rosstuck.com/precision-through-imprecision-improving-time-objects). Y nuestro caso es bien simple: necesitamos un año, que es un número entero, sin más.

Hay un aspecto importante a comentar antes de pasar a la implementación: el nombre del _Value Object_. Como se explica en la introducción, queremos usar lenguaje ubicuo, de manera que el nombre del _Value Object_ debería ser `Año`. Sin embargo, [es preferible usar sólo caracteres ASCII en nombres de variables, clases...](https://stackoverflow.com/a/3422946)

Así pues, en nuestro caso hemos llamado al _Value Object_ `Anyo`, puesto que el sonido de la «ñ» se escribe con «ny» en idiomas como el catalán o el aragonés. Podríamos haber escogido igualmente la forma portuguesa del sonido, _Anho_, o la francesa, _Agno_.

### Implementación

La implementación sería la siguiente:

```php
<?php

declare(strict_types=1);

namespace Domain\ValueObject;

use Domain\Exception\ValueObject\AnyoIsNotValid;
use InvalidArgumentException;
use Webmozart\Assert\Assert;

use function is_int;
use function is_string;
use function str_replace;

/**
 * @psalm-immutable
 */
final class Anyo
{
    private const ZERO = 0;

    private const THOUSAND_SEPARATOR = [',', '.'];

    private int $anyo;

    private function __construct(int $anyo)
    {
        $this->anyo = $anyo;
    }

    /**
     * @param mixed $value
     *
     * @return Anyo
     *
     * @throws AnyoIsNotValid
     */
    public static function fromValue($value): self
    {
        if (is_int($value)) {
            return self::fromInt($value);
        }

        if (is_string($value)) {
            return self::fromString($value);
        }

        throw AnyoIsNotValid::withType($value);
    }

    public function asInt(): int
    {
        return $this->anyo;
    }

    public function equalsTo(Anyo $anotherAnyo): bool
    {
        return $this->anyo === $anotherAnyo->anyo;
    }

    public function lowerThan(Anyo $anotherAnyo): bool
    {
        return $this->anyo < $anotherAnyo->anyo;
    }

    /**
     * @throws AnyoIsNotValid
     */
    private static function fromInt(int $value): self
    {
        if ($value === self::ZERO) {
            throw AnyoIsNotValid::withAnyoCero();
        }

        return new self($value);
    }

    /**
     * @throws AnyoIsNotValid
     */
    private static function fromString(string $value): self
    {
        $value = str_replace(self::THOUSAND_SEPARATOR, '', $value);

        try {
            Assert::integerish($value);
        } catch (InvalidArgumentException $e) {
            throw AnyoIsNotValid::withFormat($value);
        }

        return self::fromInt((int) $value);
    }
}

```

- Tenemos un _named constructor_ que acepta _strings_ de la forma `1.900` o `1900` o enteros. Cualquier otro tipo de dato —_floats_, _arrays_, objetos— son marcados como error.
- Siempre acabaremos construyendo el _Value Object_ a partir de un entero —haremos un _cast_ de _string_ a entero—, de modo que en el método correspondiente añadimos la única restricción para los años: el año 0 no existe. Por tanto, aceptaremos cualquier año, ya sea positivo o negativo. Podríamos añadir alguna otra restricción, como no permitir un año que fuese anterior de la fecha de inicio de la historia, puesto que no puede haber libros antes de que exista la escritura, pero hemos considerado que no era necesario en este momento.
- En el método que construye el objeto desde un _string_, limpiamos el valor de espacios y separadores, miramos que sea un entero, y construimos el _Value Object_ con un _cast_ a entero. Lanzamos una excepción en caso de no ser un entero.
- Como lo que guardamos internamente es un entero, tenemos un método `asInt`. Podríamos tener uno que fuese `asString`, pero esa es normalmente una responsabilidad de la capa de visualización, es decir, de Infraestructura, de modo que no lo implementamos en Dominio.
- Añadimos un método `lowerThan`, que nos servirá para comprobar que el año de nacimiento de un autor sea anterior al de su muerte en el _Value Object_ `Biografia`.

### Test

El test que hemos implementado para este _Value Object_ es el siguiente:

```php
<?php
declare(strict_types=1);

namespace Tests\Domain\ValueObject;

use Domain\Exception\ValueObject\AnyoIsNotValid;
use Domain\ValueObject\Anyo;
use PHPUnit\Framework\TestCase;

class AnyoTest extends TestCase
{
    /**
     * @param int|string $value
     *
     * @test
     * @dataProvider provideForFromValue
     */
    public function from_value($value, int $expectedAsInt) : void
    {
        $anyo = Anyo::fromValue($value);

        $this->assertSame(
            $anyo->asInt(),
            $expectedAsInt
        );
    }

    /**
     * @test
     */
    public function from_value_with_cero_as_int() : void
    {
        $this->expectException(AnyoIsNotValid::class);
        $this->expectExceptionMessage('El año "0" no existe');

        Anyo::fromValue(0);
    }

    /**
     * @test
     */
    public function from_value_with_cero_as_string() : void
    {
        $this->expectException(AnyoIsNotValid::class);
        $this->expectExceptionMessage('El año "0" no existe');

        Anyo::fromValue('0');
    }

    /**
     * @test
     */
    public function from_value_with_invalid_string_format() : void
    {
        $this->expectException(AnyoIsNotValid::class);
        $this->expectExceptionMessage('"this is not valid" no tiene un formato de año válido');

        Anyo::fromValue('this is not valid');
    }

    /**
     * @test
     */
    public function from_value_with_invalid_data_type() : void
    {
        $this->expectException(AnyoIsNotValid::class);
        $this->expectExceptionMessage('El tipo "array" no es válido');

        Anyo::fromValue([]);
    }

    /**
     * @test
     * @dataProvider provideForEqualsTo
     */
    public function equals_to(int $firstAnyo, int $secondAnyo, bool $expected) : void
    {
        $this->assertSame(
            $expected,
            Anyo::fromValue($firstAnyo)->equalsTo(
                Anyo::fromValue($secondAnyo)
            )
        );
    }

    /**
     * @return array<string,array<int|string,int>>
     */
    public function provideForFromValue() : array
    {
        return [
            'int positive' => [
                1900,
                1900,
            ],
            'int negative' => [
                -1900,
                -1900,
            ],
            'string positive' => [
                '1900',
                1900,
            ],
            'string negative' => [
                '-1900',
                -1900,
            ],
            'string with dot as separator positive' => [
                '1.900',
                1900,
            ],
            'string with dot as separator negative' => [
                '-1.900',
                -1900,
            ],
            'string with comma as separator positive' => [
                '1,900',
                1900,
            ],
            'string with comma as separator negative' => [
                '-1,900',
                -1900,
            ],
        ];
    }

    public function provideForEqualsTo() : array
    {
        return [
            'equal anyo' => [
                1900,
                1900,
                true,
            ],
            'different anyo' => [
                1900,
                -1900,
                false,
            ],
        ];
    }
}

```

Lo más interesante en este test es que comprobamos el mensaje de la excepción lanzada, para asegurarnos que la excepción se lanza por el motivo que corresponde. Este es un error detectado con _mutation testing_: [el _cast_ a entero del _string_ `'this is not valid'` es `0`](https://3v4l.org/LZXWd#output), de manera que se lanzaba la excepción por un motivo que no correspondía.

Una estrategia para evitar mirar el mensaje de la excepción podría haber sido lanzar excepciones diferentes en función del error. La contra es que nos añadiría una serie de clases de excepción extra en nuestro Dominio. Veremos esta alternativa en el _Value Object_ de `Apelativo`.

## _Value Object_: `Biografia`

### _Namespace_

Al tratarse de un objeto específico de autor, situaremos el _ValueObject_ dentro del _namespace_ `Domain\ValueObject\Autor`. Seguiremos esta regla para todos los _Value Object_: si son genéricos para toda la aplicación, como el `Id`, irán directamente en el _namespace_ `Domain\ValueObject`; si no, los situaremos en el _namespace_ equivalente de la entidad a la que pertenecen.

Así, por ejemplo, si en nuestra aplicación tuviéramos productos con su propio estado asociado y pedidos con su propio estado asociado, podríamos tener los siguientes _Value Objects_:

- `Domain\ValueObject\Producto\Estado`
- `Domain\ValueObject\Pedido\Estado`

Y si se diese el caso de que tuviéramos que usar ambos en una porción del código, podríamos importar las clases como alias:

```php
<?php
// ...

use Domain\ValueObject\Pedido\Estado as EstadoPedido;
use Domain\ValueObject\Producto\Estado as EstadoProducto;

// ...

public function doSomethingWithPedido(EstadoPedido $estado)
{
    // ...
}

public function doSomethingWithProducto(EstadoProducto $estado)
{
    // ...
}
```

### Implementación

La implementación es la siguiente:

```php
<?php

declare(strict_types=1);

namespace Domain\ValueObject\Autor;

use Domain\Exception\ValueObject\AnyoIsNotValid;
use Domain\Exception\ValueObject\Autor\BiografiaIsNotValid;
use Domain\ValueObject\Anyo;

use function is_string;
use function trim;

/**
 * @psalm-immutable
 */
final class Biografia
{
    private Anyo $anyoNacimiento;

    private ?Anyo $anyoMuerte;

    private function __construct(Anyo $anyoNacimiento, ?Anyo $anyoMuerte)
    {
        $this->anyoNacimiento = $anyoNacimiento;
        $this->anyoMuerte     = $anyoMuerte;
    }

    /**
     * @param string|int|null $anyoNacimiento
     * @param string|int|null $anyoMuerte
     *
     * @throws BiografiaIsNotValid
     * @throws AnyoIsNotValid
     */
    public static function fromAnyoNacimientoYMuerte($anyoNacimiento, $anyoMuerte): self
    {
        return self::fromAnyos(
            Anyo::fromValue($anyoNacimiento),
            self::parseAnyoMuerte($anyoMuerte)
        );
    }

    public static function fromAnyos(Anyo $anyoNacimiento, ?Anyo $anyoMuerte): self
    {
        if ($anyoMuerte === null) {
            return new self($anyoNacimiento, $anyoMuerte);
        }

        if ($anyoMuerte->lowerThan($anyoNacimiento)) {
            throw BiografiaIsNotValid::withAnyoMuerteAnteriorAAnyoNacimiento($anyoNacimiento, $anyoMuerte);
        }

        return new self($anyoNacimiento, $anyoMuerte);
    }

    public function equalsTo(Biografia $anotherBiografia): bool
    {
        if ($this->anyoMuerte !== null && $anotherBiografia->anyoMuerte !== null) {
            return $this->anyoNacimiento->equalsTo($anotherBiografia->anyoNacimiento)
                   && $this->anyoMuerte->equalsTo($anotherBiografia->anyoMuerte);
        }

        if ($this->anyoMuerte === null && $anotherBiografia->anyoMuerte !== null) {
            return false;
        }

        if ($this->anyoMuerte !== null && $anotherBiografia->anyoMuerte === null) {
            return false;
        }

        return $this->anyoNacimiento->equalsTo($anotherBiografia->anyoNacimiento);
    }

    public function anyoNacimiento(): Anyo
    {
        return $this->anyoNacimiento;
    }

    public function anyoMuerte(): ?Anyo
    {
        return $this->anyoMuerte;
    }

    /**
     * @param string|int|null $anyoMuerte
     *
     * @throws AnyoIsNotValid
     */
    private static function parseAnyoMuerte($anyoMuerte): ?Anyo
    {
        if (is_string($anyoMuerte)) {
            $anyoMuerte = trim($anyoMuerte);
        }

        if ($anyoMuerte === null || $anyoMuerte === '') {
            return null;
        }

        return Anyo::fromValue($anyoMuerte);
    }
}

```

- Tenemos dos _named constructor_ para el objeto: uno a partir de un valor arbitrario (`string`, `int` o `float`) y otro a partir del _Value Object_ `Anyo`. A la práctica siempre se utiliza que utiliza `Anyo`, para utilizar su método `lowerThan` para comparar ambos valores.
- Todas las validaciones de formato de datos necesarias las realiza el _Value Object_ `Anyo`. Dejamos pasar las posibles excepciones que se lancen.
- Al tener el _Value Object_ dos parámetros, pudiendo además ser `null` uno de ellos, el método `equalsTo es más complejo, puesto que hay más casos a verificar.

### Test

Debido a su longitud, no reproducimos el test entero aquí —puede consultarse en el _pull request_. Cabe destacar dos cosas:

- Testeamos todas las combinaciones posibles de biografías: autores que vivieron a.C., que nacieron a.C. y murieron d.C., que murieran d.C. y que sigan vivos.
- No testeamos las validaciones del _Value Object_ `Anyo`, porque de eso se encarga el test propio de ese _Value Object_.