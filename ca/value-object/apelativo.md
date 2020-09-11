---
lang: ca
lang-ref: value-object-apelativo
tags: [domain,value-object,nombre,apelativo]
is-content-page: true
layout: content-page
order: 4
pull-request: 5
---

# _Value Object_: Apelativo

## Introducció

A continuació implementarem el _Value Object_ que ens servirà per a representar el nom amb què es coneix un autor. Recordem les restriccions que teníem segons la [definició de la introducció](https://rubenrubiob.github.io/ca/#autor):

- Ha de ser traduïble. Safo té el mateix nom en català i en castellà, però no en anglès, on s’escriu _Sappho_.
- Pot ser que no tinguem cognom de l’autor. Podem fer servir aquest camp pel denominatiu d’alguns autors, com Cristina de Pizan.
- El pseudònim és el nom pel qual es coneix l’autor. Per exemple, el pseudònim d’Arístocles d’Atenes és Plató.

És possible fer una crítica a aquestes restriccions dient que són eurocentristes, perquè pensem a fer servir nom i cognoms, que no existeixen a totes les cultures; i la crítica seria encertada. Això no obstant, l'edició de llibres tal com la coneixem actualment va néixer a Europa i, per tant, les referències bibliogràfiques es basen justament en aquest model cultural, ja que els formats requereixen especificar nom i cognoms de l'autor per separat.

Així doncs, com que hem de donar cabuda tant a autors pròpiament occidentals com d'altres cultures, tindrem un _Value Object_ anomenat `Apelativo` que, igual que el _Value Object_ `Biografia`, contindrà diversos atributs, tres en aquest cas:

- `Alias`: és com es coneix popularment l'autor. Amb aquest camp, cobrim tots els casos:
    - Els casos en què desconeixem nom i cognoms de l'autor, com Homer.
    - Els casos en què es coneix l'autor pel seu denominatiu, com Cristina de Pizan.
    - Els casos en què l'autor utilitza un pseudònim diferent del seu nom. Tenim els casos de Caterina Albert, que publicava com a Víctor Català, o d'Arístocles d'Atenes, que era conegut com a Plató.
- `Nombre`: és el nom de l'autor. Com pot ser desconegut, és un camp que podrà ser `null`.
- `Apellido`: és el cognom o cognoms de l'autor. Com pot ser desconegut, és un camp que podrà ser `null`.

Cadascun d'aquests atributs serà un _Value Object_ de tipus `Nombre`.

En aquest moment no ens importa la primera restricció, que el nom d'un autor ha de ser traduïble. Aquesta és una responsabilitat de l'entitat `Autor`. Un _Value Object_ `Apelativo` representarà l'apel·latiu d'un autor en qualsevol idioma.

## _Value Object_: `Nombre`

### _Namespace_

El _Value Object_ `Nombre` serà reutilitzable a més llocs, com per exemple el nom de la ciutat a l'entitat `Edicion`. Així doncs, serà un _Value Object_ genèric, i el situarem al _namespace_ `Domain\ValueObject\`.

### Implementació

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

- En aquest cas només realitzem dues validacions al _Value Object_:
    - Que no estigui buit.
    - Que la longitud no sigui superior a 255 caràcters. Aquest valor 255 és arbitrari; amb aquesta longitud, hauríem de tenir-ne prou per emmagatzemar qualsevol nom.
- A diferència del _Value Object_ `Anyo`, llancem excepcions diferents per a cadascuna de les validacions errònies. Això ens serà molt útil al _Value Object_ `Apelativo`, com veurem a continuació.
- Netegem els espais en blanc al principi i al final del valor d'entrada.

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

- Com que llancem dues excepcions diferents, hem de testejar cadascuna d'elles en mètodes separats, però no és necessari mirar el missatge d'error per a diferenciar-les.
- Comprovem que la neteja d'espais al principi i al final del valor funciona correctament, tant a l'hora de crear el _Value Object_ com a l'hora de comparar-lo.

## _Value Object_: `Apelativo`

### _Namespace_

Segons l'hem definit, `Apelativo` és un _Value Object_ específic d'`Autor`. En conseqüència, el situarem al _namespace_ `Domain\ValueObject\Autor`.

### Implementació

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

- Sempre construïm el _Value Object_ `Apelativo` a partir dels seus components, amb el _named constructor_ `fromComponents`. La validació del format de dades quan el valor d'entrada és un `string` es delega al _Value Object_ `Nombre`.
- La validació que realitzem per a l'àlies és diferent que la que realitzem per a nom i cognoms, donat que l'àlies és sempre obligatori. Així doncs, quan el _Value Object_ `Nombre` valida el valor d'entrada per a l'àlies, pot llençar dues excepcions:
    - Que el valor estigui buit. En aquest cas, com que l'àlies és obligatori pel _Value Object_ `Apelativo`, llencem una excepció pròpia d'aquest _Value Object_.
    - Que la longitud sigui superior a la permesa. En aquest cas, deixem passar l'excepció que llença `Nombre`.
- De manera anàloga, tenim la validació per a nom i cognom, que pot llençar dues excepcions:
    - Que el valor estigui buit. En aquest cas, com que ni nom ni cognoms són obligatoris pel _Value Object_ `Apelativo`, capturem l'excepció que llença `Nombre` i retornem un valor `null`. Aquí és on cobra sentit tenir dues excepcions diferents per a la validació de `Nombre`.
    - Que la longitud sigui superior a la permesa. En aquest cas, deixem passar l'excepció que llença `Nombre`.

### Test

A causa de la seva longitud, no reproduïm aquí el test. Pot consultar-se al _pull request_ corresponent.

L'única cosa a destacar és que no es testegen les excepcions que pugui llençar `Nombre`, ja que aquestes ja són comprovades al mateix test de `Nombre`.