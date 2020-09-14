---
lang: ca
lang-ref: value-object-biografia
tags: [domain,value-object,biografia,any]
is-content-page: true
layout: content-page
order: 3
pull-request: 4
---

# _Value Object_: Biografia

* TOC
{:toc}

## Introducció

El següent _Value Object_ que implementarem és el que representa la biografia d'un autor, és a dir, en tindrà els anys de naixement i mort. Com s'explica a la [introducció](https://rubenrubiob.github.io/ca/), hi ha diverses restriccions pel que fa a la biografia d'un autor. Les reproduïm a continuació:

- Pot ser que no sapiguem la data exacta de naixement o mort de l’autor, com en el cas d’Homer.
- Per a autors vius, només tindrem la data de naixement.
- En el cas en què coneguem ambdues dates, la data de naixement ha de ser anterior a la de mort.
- Podem tenir dates anteriors a la nostra era (a.C.), tant pel naixement com per la mort.
- L’any 0 no existeix al nostre calendari.

Per ara, amb l'objectiu de simplificar la implementació, no contemplarem la primera restricció, és a dir, no tindrem en compte els casos en què desconeguem algunes de les dates. En un futur podrem afegir una extensió al _Value Object_ per a introduir el segle de naixement i el de mort d'un autor, per exemple.

A diferència dels _Value Objects_ vists fins al moment, que només tenien un atribut, `Biografia` en tindrà dos: l'any de naixement i l'any de mort, podent aquest darrer ser `null` per a autors que encara siguin vius.

Podríem tenir ambdós atributs com a `int` dins del _Value Object_. Això no obstant, a l'entitat `Edicion` també tenim l'any de publicació d'un llibre, de manera que, si optéssim per aquesta opció, acabaríem necessitant les validacions d'any a ambdós _Value Object_. Així doncs, crearem un _Value Object_ que representi un any, i s'emprarà dins de `Biografia`.

## _Value Object_: Anyo

Podríem emmagatzemar valors temporals més concrets, com per exemple el dia i mes de naixement d'un autor o de publicació d'un llibre. Això no obstant, hi ha moltes vegades que desconeixem les dates exactes —quin dia va néixer Plató o Aristòtil?—, de manera que amb la precisió d'un any en tenim prou, segons hem especificat al Domini. En tot cas, podríem ampliar aquesta precisió al futur, si calgués.

En tractar-se d'una dada de temps, podríem pensar a prescindir de fer servir un _Value Object_ i utilitzar directament les classes que ens proporciona el llenguatge, `DateTime` i `DateTimeImmutable`. Això no obstant, estaríem utilitzant una precisió extra que [podria complicar-nos el desenvolupament posterior](http://rosstuck.com/precision-through-imprecision-improving-time-objects). I el nostre cas és ben simple: necessitem un any, que és un nombre enter, res més.

Hi ha un aspecte important a comentar abans de passar a la implementació: el nom del _Value Object_. Com s'explica a la introducció, volem utilitzar llenguatge ubic, així que el nom del _Value Object_ hauria de ser `Año`. Això no obstant, [és preferible utilitzar només caràcters ASCII a noms de variables, classes...](https://stackoverflow.com/a/3422946)

Així doncs, en aquest cas hem anomenat `Anyo` el _Value Object_, donat que el so de la «ñ» s'escriu amb «ny» en idiomes com el català o l'aragonès[^1]. Podríem haver escollit la forma portuguesa del so, _Anho_, o la francesa, _Agno_.

### Implementació

La implementació seria la següent:

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

- Tenim un _named constructor_ que accepta _strings_ de la forma `1.900`  o `1900`, o nombres enters. Qualsevol altre tipus de dada —_floats_, _arrays_, objectes— són marcats com a error.
- Sempre acabarem construïnt el _Value Object_ a partir d'un enter —farem un _cast_ de _string_ a enter—, de manera que al mètode corresponent afegim l'única restricció pels anys: l'any 0 no existeix. Per tant, acceptarem qualsevol any, sigui positiu, sigui negatiu. Podríem afegir alguna altra restricció, com no permetre que un any que sigui anterior de la data d'inici de la història, donat que no pot haver-hi llibres escrits abans no existeixi l'escriptura, però hem considerat que no era necessari en aquest moment.
- Al mètode que construeix l'objecte des d'un _string_, hi netegem el valor rebut d'espais i separadores, mirem que sigui un enter, i construïm el _Value Object_ amb un _cast_ a enter. Llencem una excepció en cas que el valor no sigui un enter.
- Com que el que desem internament és un enter, tenim un mètode `asInt`. Podríem tenir-ne un altre que fos `asString`, però aquesta és normalment una responsabilitat de la capa de visualització, és a dir, d'Infraestructura, de manera que no l'implementem a la capa de Domini.
- Afegim un mètode `lowerThan`, que ens servirà per a comprovar que l'any de naixement d'un autor sigui anterior al de la seva mort al _Value Object_ `Biografia`.

### Test

El test que hem implementat per aquest _Value Object_ és el següent:

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

El més interessant en aquest test és que comprovem el missatge de l'excepció llençada, per assegurar-nos que l'excepció es llença pel motiu corresponent. Aquest és un error detectat amb _mutation testing_: [el _cast_ a enter de l'_string_ `'this is not valid'` és `0`](https://3v4l.org/LZXWd#output), de manera que l'excepció es llençava per un motiu que no corresponia.

Una estratègia per evitar mirar el missatge de l'excepció podria haver estat llençar excepcions diferents en funció de l'error. La contra és que ens afegiria una sèrie de classes d'excepció extra al nostre Domini. Veurem aquesta alternativa al _Value Object_ `Apelativo`.

## _Value Object_: Biografia

### _Namespace_

En tractar-se d'un objecte específic d'autor, situarem el _Value Object_ dins del _namespace_ `Domain\ValueObject\Autor`. Seguirem aquesta regla per a tots els _Value Objects_: si són genèrics per a tota l'aplicació, com l'`Id`, aniran directament al _namespace_ `Domain\ValueObject`; si no, els situarem al _namespace_ equivalent de l'entitat a la qual pertanyen.

Per exemple, si a la nostra aplicació tinguéssim productes amb el seu propi estat associat i comandes amb el seu propi estat associat, podríem tenir els següents _Value Objects_:

- `Domain\ValueObject\Producto\Estado`
- `Domain\ValueObject\Pedido\Estado`

I si es donés el cas que els haguéssim d'utilitzar ambdós en un fragment de codi, podríem importar les classes com a àlies:

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

### Implementació

La implementació és la següent:

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

- Tenim dos _named constructor_ per l'objecte: un a partir d'un valor arbitrari (`string`, `int` o `float`) i un altre a partir del _Value Object_ `Anyo`. A la pràctica sempre es fa servir `Anyo`, per aprofitar el mètode `lowerThan` per a comparar ambdues dades.
- Totes les validacions de format de dades necessàries les fa el _Value Object_ `Anyo`. Deixem passar les possibles excepcions que es llencin.
- En tenir el _Value Object_ dos paràmetres, a més a més podent ser un d'ells `null`, el mètode `equalsTo` és més complex, donat que hi ha més casos a verificar.

### Test

A causa de la seva longitud, no reproduïm el test sencer aquí —es pot consultar al _pull request_. Cal destacar-ne dues coses:

- Testegem totes les combinacions possibles de biografies: autors que van viure a.C., que van néixer a.C. i van morir d.C., que van morir d.C. i que continuïn vius.
- No testegem les validacions del _Value Object_ `Anyo`, perquè d'això ja se n'encarrega el seu mateix test.

[^1]: Així doncs, aquest problema sorgeix pel fet de fer servir el castellà com a idioma de Domini. En cas d'haver fet servir el català, no tindríem el problema amb la «ñ». Sí que no podríem fer servir la c trencada («ç»), però, en ser només un caràcter, és més fàcilment substituïble per una «c».