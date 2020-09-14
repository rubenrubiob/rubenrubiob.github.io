---
lang: ca
lang-ref: value-object-email-id
tags: [domain,value-object,id,uuid]
is-content-page: true
layout: content-page
order: 2
pull-request: 3
---

# _Value Object_: Id

* TOC
{:toc}

## Introducció

A les nostres entitats necessitarem un identificador. El funcionament tradicional a les aplicacions web consistia a fer servir com a identificador un número enter autoincremental que ens proporcionava la base de dades. D'aquesta manera, cada taula tenia el seu propi rast d'identificadors.

Això no obstant, no utilitzarem identificadors autoincrementals, principalment per dos motius:

- Un objecte ha de ser sempre en un estat vàlid, des del moment de la seva creació. Si esperem que la base de dades ens proporcioni l'identificador després d'inserir-lo, haurem creat una entitat sense identificador, de manera que tindrem una entitat en un estat invàlid —i els tests haurien de fallar.
- Com que les entitats són a la capa de Domini, no podem dependre de la capa d'Infraestructura. A més a més, com podem saber que el sistema de persistència tingui un generador de números autoincrementals? I si féssim servir un sistema No-SQL per a persistir dades, com Elasticsearch? O si necessitem identificadors per a fitxers, que van a disc i no a la base de dades?

Com a alternativa, farem servir [UUIDs](https://en.wikipedia.org/wiki/Universally_unique_identifier#Version_4_(random)) —_Unversally Unique Identifier— de la versió 4. Aquests UUIDs són _strings_ aleatoris amb una baixa probabilitat de col·lisió, amb la forma:

```
e9f7ff88-49d8-4574-aa32-0f0728e3ac20
```

Per tant, a la nostra aplicació implementarem un _Value Object_ que representi un `Id` i que, internament, serà un UUIDv4.

## Implementació

La implementació podria ser la següent:

```php
<?php

declare(strict_types=1);

namespace Domain\ValueObject;

use Domain\Exception\ValueObject\IdIsNotValid;
use InvalidArgumentException;
use Ramsey\Uuid\Uuid;
use Webmozart\Assert\Assert;

/**
 * @psalm-immutable
 */
final class Id
{
    private string $id;

    private function __construct(string $id)
    {
        $this->id = $id;
    }

    public static function generate(): self
    {
        return new self(
            Uuid::uuid4()->toString()
        );
    }

    /**
     * @throws IdIsNotValid
     */
    public static function fromString(string $id): self
    {
        self::validate($id);

        return new self($id);
    }

    public function asString(): string
    {
        return $this->id;
    }

    public function equalsTo(Id $anotherId): bool
    {
        return $this->id === $anotherId->id;
    }

    /**
     * @throws IdIsNotValid
     */
    private static function validate(string $id): void
    {
        try {
            Assert::uuid($id);
        } catch (InvalidArgumentException $e) {
            throw IdIsNotValid::withFormat($id);
        }
    }
}

```

- Utilitzarem aquest _Value Object_ per a tots els nostres ids. Alternativament, podríem tenir un tipus diferent per cadascun que necessitéssim —`AutorId`, `LibroId`...
- Tenim un _named constructor_ per a generar nous ids. Per aquesta generació farem servir la llibreria [`ramsey/uuid`](https://github.com/ramsey/uuid).
- Tenim un altre _named constructor_ per a reconstruir l'id a partir d'un _string_. Ens servirà per a reconstruir-lo des de la capa de persistència.
- Seguint el mateix que al _Value Object_ `EmailAddress`, afegim un mètode per a comparar dues instàncies de la classe.

## Test

El test que hem implementat és el següent:

```php
<?php

declare(strict_types=1);

namespace Tests\Domain\ValueObject;

use Domain\Exception\ValueObject\IdIsNotValid;
use Domain\ValueObject\Id;
use PHPUnit\Framework\TestCase;
use Ramsey\Uuid\Rfc4122\Validator;

class IdTest extends TestCase
{
    /**
     * @test
     */
    public function generate(): void
    {
        $this->assertTrue(
            (new Validator())->validate(
                Id::generate()->asString()
            )
        );
    }

    /**
     * @test
     */
    public function from_string(): void
    {
        $idAsString = 'accbfed6-2a47-486e-b5e8-89e9616cb445';

        $id = Id::fromString($idAsString);

        $this->assertSame(
            $idAsString,
            $id->asString()
        );
    }

    /**
     * @test
     */
    public function from_string_with_invalid_id_throws_exception(): void
    {
        $this->expectException(IdIsNotValid::class);

        Id::fromString('this-is-not-a-valid-id');
    }

    /**
     * @test
     */
    public function from_string_with_empty_id_throws_exception(): void
    {
        $this->expectException(IdIsNotValid::class);

        Id::fromString('  ');
    }

    /**
     * @test
     * @dataProvider provideForEqualsTo
     */
    public function equals_to(string $firstId, string $secondId, bool $expected): void
    {
        $this->assertSame(
            $expected,
            Id::fromString($firstId)->equalsTo(
                Id::fromString($secondId)
            )
        );
    }

    /**
     * @return array[]
     */
    public function provideForEqualsTo(): array
    {
        return [
            'equals' => [
                'c12327ef-9af5-4da6-b4a7-d9c434239100',
                'c12327ef-9af5-4da6-b4a7-d9c434239100',
                true,
            ],
            'not equals' => [
                'c12327ef-9af5-4da6-b4a7-d9c434239100',
                '44be2055-768b-4d79-9149-18b806cf7f94',
                false,
            ],
        ];
    }
}

```

El més important a destacar del test és que per a la validació del format hem fet servir una llibreria diferent de la de l'objecte. D'aquesta manera testegem el comportament del test i no la seva implementació.