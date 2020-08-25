---
lang: ca
lang-ref: value-object-email-address
tags: [domain,value-object, email]
is-content-page: true
layout: content-page
order: 1
pull-request: 2
---

# _Value Object_: `EmailAddress`

* TOC
{:toc}

Aquest és el primer apartat de desenvolupament dins del projecte. Hem de començar de la capa més interna a la més externa, de manera que la primera capa que hem de desenvolupar és la de Domini. I, dins del Domini, els elements més bàsics a desenvolupar són els _Value Objects_.

Un _Value Object_ és un objecte que es defineix pel seu valor. Si no estem familiaritzats amb el concepte, potser aquesta definició no ens diu gran cosa, però en veurem el seu sentit amb el primer exemple, el d'adreça de correu electrònic.

Els _Value Object_ són un concepte potentíssim, especialment a l'hora de treballar amb els sempre [problemàtics nombres flotants per a imports monetaris](https://martinfowler.com/eaaCatalog/money.html). Tots els atributs de les nostres entitats seran _Value Objects_, no farem servir en cap cas els tipus primitius del llenguatge.

## Problemàtica

Comencem amb un contraexemple. Imaginem que en un servei qualsevol tenim la següent definició d'un mètode que envia una notificació a una adreça de correu electrònic d'un usuari:

```php
public function notify(string $email) : void
{
    // ...
}
```

O, encara pitjor, sense l'especificació del tipus de dada:

```php
public function notify($email) : void
{
    // ...
}
```

Com sabem que el valor de `$email` és realment un _string_ amb una adreça de correu electrònic vàlida? En el segon cas, a més a més, tampoc no estem segurs que el tipus de `$email` sigui un _string_.

Sabem que una adreça de correu electrònic és sempre un _string_, però no tot _string_ és una adreça de correu electrònic. És a dir, les adreces de correu electrònic són un subconjunt de tots els _string_ existents:

![](/images/value-object/diagrama-venn-email.png){:height="50%" width="50%"}

Així doncs, el primer que hauríem de fer a aquesta funció és validar que l'_string_ que ens arriba és efectivament una adreça de correu electrònic, i llançar una excepció en cas que no ho sigui:

```php
public function notify(string $email) : void
{
    if (filter_var($email, FILTER_VALIDATE_EMAIL) === false) {
        throw new \InvalidArgumentException('Invalid email address');
    }
    
    // ...
}
```

Per a estar segurs, hauríem de tenir aquesta validació a cada mètode que implementem i que esperi rebre un paràmetre que sigui una adreça de correu electrònic. Hauríem de fer el mateix amb qualsevol altre paràmetre que rebem a cada mètode.

Ja veiem que aquesta aproximació ens implica una validació redundant a cada mètode. A més a més, ens afegiria camins de test que no tenen res a veure amb el mètode que estem testejant: en el cas de l'exemple anterior, el que hem de validar és que l'enviament de la notificació funcioni, no la validació del paràmetre.

La solució és tenir un _Value Object_, anomenat per exemple `EmailAddress`, que podem passar com a tipus de paràmetre als mètodes:

```php
public function notify(EmailAddress $email) : void
{
    // ...
}
```

D'aquesta manera, en aquest mètode sabem amb tota certesa que `$email` és de tipus `EmailAddress`. Si la construcció de l'objecte `EmailAddress` fallés en un punt previ de l'execució, seria responsabilitat d'aquella porció de codi tractar l'excepció.

## _Namespace_

El _Value Object_ `EmailAddress` es podrà reutilitzar a diferents parts de l'aplicació: podria servir con a identificador dels usuaris de l'aplicació, com una dada de contacte d'una editorial...[^1]

Així doncs, aquest _Value Object_ el situarem al _namespace_ `Domain\ValueObject`, que dins del projecte correspondrà al directori físic `src/Domain/ValueObject`.

A l'explicació del _Value Object_ del format de l'edició d'un llibre veurem on situar _Value Objects_ que no siguin genèrics, és a dir, que no siguin reutilitzables a tota l'aplicació.

En la explicación del _Value Object_ de formato de una edición de un libro veremos dónde colocar _Value Objects_ que no sean genéricos y reusables para toda la aplicación.

## Implementació

Una possible implementació del _Value Object_ `EmailAddress` podria ser la següent:

```php
<?php

declare(strict_types=1);

namespace Domain\ValueObject;

use Domain\Exception\ValueObject\EmailAddressIsNotValid;
use InvalidArgumentException;
use Webmozart\Assert\Assert;

/*
 * @psalm-immutable
 */
final class EmailAddress
{
    private string $emailAddress;

    private function __construct(string $emailAddress)
    {
        $this->emailAddress = $emailAddress;
    }

    /**
     * @throws EmailAddressIsNotValid
     */
    public static function create(string $emailAddress): self
    {
        self::validate($emailAddress);

        return new self(
            $emailAddress
        );
    }

    public function asString(): string
    {
        return $this->emailAddress;
    }

    public function equalsTo(EmailAddress $anotherEmailAddress): bool
    {
        return $this->emailAddress === $anotherEmailAddress->emailAddress;
    }

    /**
     * @throws EmailAddressIsNotValid
     */
    private static function validate(string $emailAddress): void
    {
        try {
            Assert::email($emailAddress);
        } catch (InvalidArgumentException $e) {
            throw EmailAddressIsNotValid::withFormat($emailAddress);
        }
    }
}

```

- Com que seguim [_composition over inheritance_](https://en.wikipedia.org/wiki/Composition_over_inheritance), totes les classes que implementem, excepte algunes excepcions, [seran declarades com a `final`](https://ocramius.github.io/blog/when-to-declare-classes-final/).
- El constructor és privat. D'aquesta manera, obliguem a [fer servir sempre el _named constructor_](https://github.com/ShittySoft/symfony-live-berlin-2018-doctrine-tutorial/pull/3#issuecomment-460614781) i evitem [casos estranys](https://twitter.com/DaveLiddament/status/1239259573493604352) que permet el llenguatge i que farien que l'objecte fos mutable.
- Hem afegit un mètode `asString`, que permet obtenir el valor de l'objecte com un _string_. No fem servir el mètode màgic `__toString` [per a evitar problemes](https://github.com/ShittySoft/symfony-live-berlin-2018-doctrine-tutorial/pull/3#issuecomment-460441229). A altres objectes podem tenir diversos mètodes diferents per a obtenir el valor del _Value Object_; per exemple, en un import monetari podríem tenir `asString`, `asFloat`, `asInt`...
- Hem afegit un mètode `equalsTo` per a comparar dos _Value Object_ de tipus `EmailAddress`. No és un mètode estríctament necessari, però és un mètode molt senzill de crear i testejar, i ajuda molt al procés de test.
- Un _Value Object_ sempre és immutable, de manera que hem afegit l'anotació corresponent per a `Psalm`.
- Un _Value Object_ siempre es inmutable, de modo que hemos añadido la anotación correspondiente para _Psalm_.
- Fem servir la llibreria [`webmozart/assert`](https://github.com/webmozart/assert), ja que simplifica molt el procés de validació.
- L'excepció que llencem en cas que el valor no sigui correcte és molt llegible: `throw EmailAddressIsNotValid::withFormat`.

## Excepció

En tots els casos del Domini en què l'execució no sigui vàlida, llençarem una excepció específica. N'haurem de crear una classe. Una possible implementació seria la següent:

```php
<?php

declare(strict_types=1);

namespace Domain\Exception\ValueObject;

use Exception;

use function Safe\sprintf;

final class EmailAddressIsNotValid extends Exception
{
    public static function withFormat(string $emailAddress): self
    {
        return new self(
            sprintf(
                '"%s" is not a valid email address',
                $emailAddress
            )
        );
    }
}

```

- Seguint [algunes recomanacions](https://www.nikolaposa.in.rs/assets/uploads/slides/phpbnl20/handling-exceptional-conditions/#/), creem un _named constructor_ per a generar l'excepció, cosa que, a més a més, fa més llegible el codi.
- Fem servir la llibreria [thecodingmachine/safe](https://github.com/thecodingmachine/safe), que conté les funcions de PHP però amb una API més usable, que llença excepcions en lloc de retornar `false` si la crida és invàlida. En el cas de `sprintf`, si el número de paràmetres no és vàlid, `Safe` llença una excepció en comptes de retornar `false`.

## Test

El test que hem creat per a aquest objecte és el següent:

```php
<?php

declare(strict_types=1);

namespace Tests\Domain\ValueObject;

use Domain\Exception\ValueObject\EmailAddressIsNotValid;
use Domain\ValueObject\EmailAddress;
use PHPUnit\Framework\TestCase;

class EmailAddressTest extends TestCase
{
    /**
     * @test
     */
    public function create(): void
    {
        $email = 'valid@email.address';

        $emailAddress = EmailAddress::create($email);

        $this->assertSame(
            $email,
            $emailAddress->asString()
        );
    }

    /**
     * @test
     * @dataProvider provideForEqualsTo
     */
    public function equals_to(string $firstEmailAddress, string $secondEmailAddress, bool $expected): void
    {
        $this->assertSame(
            $expected,
            EmailAddress::create($firstEmailAddress)->equalsTo(
                EmailAddress::create($secondEmailAddress)
            )
        );
    }

    /**
     * @test
     */
    public function create_with_invalid_email_address_throws_exception(): void
    {
        $this->expectException(EmailAddressIsNotValid::class);

        EmailAddress::create('this_is_not_a_valid_email_address@domain');
    }

    /**
     * @test
     */
    public function create_with_empty_email_address_throws_exception(): void
    {
        $this->expectException(EmailAddressIsNotValid::class);

        EmailAddress::create('  ');
    }

    /**
     * @return array[]
     */
    public function provideForEqualsTo(): array
    {
        return [
            'equal email addresses' => [
                'foo@bar.com',
                'foo@bar.com',
                true,
            ],
            'different email addresses' => [
                'foo@bar.com',
                'bar@foo.com',
                false,
            ],
        ];
    }
}

```

- Tots els noms dels mètodes de test estan en format *snake_case*. És una cosa que només faig servir als tests, donat que els noms són molt específics i acaben essent molt llargs, i crec que en [simplifica la lectura](https://matthiasnoback.nl/2020/06/unit-test-naming-conventions/). També existeix [l'opció d'utilitzar caràcters transparents](https://github.com/brefphp/bref/blob/41c634f151d13d30ac323e5d2d78d383bdcc971e/tests/Sam/PhpFpmRuntimeTest.php#L26), però em resulta més complicat escriure el codi.
- Testejem en mètodes separats les excepcions que es poden generar al mètode `create`. En aquest cas, en tenim dues: que l'_string_ que ens passin o bé no sigui una adreça de correu electrònic, o bé que sigui buit.
- Tot i no ser estríctament necessari, testejem la creació correcta de l'objecte i l'ús del mètode `asString`, perquè `Infection` validi correctament la visibilitat del mètode.
- Fem servir un [_Data provider_](https://phpunit.readthedocs.io/en/9.3/writing-tests-for-phpunit.html#data-providers) pel test de `equalsTo`.


[^1]: Cal dir que aquest _Value Object_ no és estríctament necessari per a la primera fase de desenvolupament, però trobo que és el més il·lustratiu per a l'explicació dels _Value Object_.