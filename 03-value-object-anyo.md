# _Value Object_: `Anyo`

El siguiente _Value Object_ que vamos a implementar es el que representa un año. Lo utilizaremos con varios fines: el año de nacimiento y muerte de un autor, el año de publicación de un libro...

Podríamos guardar valores temporales más concretos, como por ejemplo el día y més de nacimiento de un autor o de publicación de un libro. Pero son muchas las ocasiones en las que desconocemos las fechas exactas —¿qué día nacieron Platón y Aristóteles?—, así que con la precisión de año tenemos suficiente, según hemos especificado en el Dominio. En todo caso, podríamos ampliar esta precisión en el futuro, si fuere necesario.

Tratándose de tiempo, podríamos pensar en prescindir de un _Value Object_ y utilizar directamente las clases que nos proporciona el lenguaje, `DateTime` y `DateTimeImmutable`. Sin embargo, estaríamos utilizando una precisón extra que podría complicarnos el desarrollo posterior (enlace: http://rosstuck.com/precision-through-imprecision-improving-time-objects ). Y nuestro caso es bien simple: necesitamos un año, que es un número entero, sin más.

Una cosa a destacar antes de pasar a la implementación: el nombre del _Value Object_. Como se explica en la introducción, queremos usar lenguaje ubicuo, de manera que el nombre del _Value Object_ debería ser `Año`. Sin embargo, es preferible usar sólo carácteres ASCII en nombres de variables, clases... (buscar cita)

Así pues, en nuestro caso hemos llamado al _Value Object_ `Anyo`, puesto que el sonido de la `ñ` se escribe con `ny` en idiomas como el catalán —del que soy hablante nativo— o el aragonés. Podríamos haber cogido igualmente la forma portuguesa, _Anho_, o la francesa, _Agno_.

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
    public static function fromValue($value) : self
    {
        if (is_int($value)) {
            return self::fromInt($value);
        }

        if (is_string($value)) {
            return self::fromString($value);
        }

        throw AnyoIsNotValid::becauseTypeIsNotValid($value);
    }

    public function asInt() : int
    {
        return $this->anyo;
    }

    public function equalsTo(Anyo $anotherAnyo) : bool
    {
        return $this->anyo === $anotherAnyo->anyo;
    }

    /**
     * @throws AnyoIsNotValid
     */
    private static function fromInt(int $value) : self
    {
        if ($value === self::ZERO) {
            throw AnyoIsNotValid::becauseAnyoCeroDoesNotExist();
        }

        return new self($value);
    }

    /**
     * @throws AnyoIsNotValid
     */
    private static function fromString(string $value) : self
    {
        $value = str_replace(self::THOUSAND_SEPARATOR, '', $value);

        try {
            Assert::integerish($value);
        } catch (InvalidArgumentException $e) {
            throw AnyoIsNotValid::becauseItDoesNotHaveAValidFormat($value);
        }

        return self::fromInt((int) $value);
    }
}

```

- Tenemos un _named constructor_ que acepta strings de la forma _1.900_ o _1900_ o enteros. Cualquier otro tipo de dato —_floats_, _arrays_, objetos— son marcados como error.
- Siempre acabaremos construyendo el _Value Object_ a partir de un entero —haremos un _cast_ de _string_ a entero—, de modo que en el método correspondiente añadimos la única restricción para los años: el año 0 no existe. Por tanto, aceptaremos cualquier año, ya sea positivo o negativo. Podríamos añadir alguna otra restricción, como no permitir poner un año que fuese menor que la fecha de inicio de la historia —no puede haber libros antes de que exista la escritura—, pero hemos considerado que no era necesario en este momento.
- En el método que construye el año desde un _string_, limpiamos el valor de espacios y separadores, y miramos que el valor recibido sea un entero, y construimos el _Value Object_ con un _cast_ a entero. Lanzamos una excepción en caso de no ser un entero.
- Como lo que guardamos internamente es un entero, tenemos un método `asInt`. Podríamos tener uno que fuese `asString`, pero esa es normalmente una responsabilidad de la capa de visualización, es decir, de Infraestructura, de modo que no lo implementamos a nivel de Dominio.

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

Lo más interesante en este test es que comprobamos el mensaje de la excepción lanzada, para asegurarnos que la excepción se lanza por el motivo que corresponde. Este es un error detectado con _mutation testing_: el _cast_ a entero de `'this is not valid'` es `0`, de manera que en las mutaciones lanzaba la excepción porque no era un año válido, que no era lo correspondiente.

Una estrategia para evitar mirar el mensaje de la excepción podría haber sido lanzar excepciones diferentes en función del error. La contra es que nos añadiría una serie de clases de excepción extra en nuestro Dominio.

