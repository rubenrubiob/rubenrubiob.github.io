---
lang: es
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

## Introducción

Para nuestras entidades necesitaremos un identificador. El funcionamiento tradicional en las aplicaciones web consistía en utilizar como identificador un número entero autoincremental que nos proporcionaba la base de datos. De este modo, cada tabla tenía su propia ristra de identificadores.

Sin embargo, no usaremos identificadores autoincrementales, principalmente por dos motivos:

- Un objeto ha de estar siempre en un estado válido, desde el momento de su creación. Si esperamos a que la base de datos nos proporcione el identificador después de insertarlo en la base de datos, habremos creado una entidad sin identificador, de modo que estaría en un estado inválido —y los test nos deberían fallar.
- Como las entidades están en la capa de Dominio, no podemos depender de la capa Infraestructura. Además, ¿quién nos garantiza que el sistema de persistencia tenga un generador de números autoincrementales? ¿Y si usamos un sistema No-SQL para persistir datos, como Elasticsearch? ¿O si necesitamos identificadores para los ficheros que van a disco y no a la base de datos?

Como alternativa, utilizaremos [UUIDs](https://en.wikipedia.org/wiki/Universally_unique_identifier#Version_4_(random)) —_Universally Unique Identifier_—, de la versión 4. Estos UUIDs son _strings_ aleatorios con una baja probabilidad de colisión, y tienen la forma:

```
e9f7ff88-49d8-4574-aa32-0f0728e3ac20
```

Para nuestra aplicación, por tanto, implementaremos un _Value Object_ que represente un `Id`, y que internamente será un UUIDv4.

## Implementación

La implementación podría ser la siguiente:

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

- Usaremos este _Value Object_ para todos nuestros ids. Alternativamente, podríamos tener un tipo diferente para cada uno que necesitásemos —`AutorId`, `LibroId`...
- Tenemos una _named constructor_ para generar nuevos ids. Para generar los id utilizamos la librería [`ramsey/uuid`](https://github.com/ramsey/uuid).
- Tenemos otro _named constructor_ para reconstruir el id a partir de un _string_, cosa que nos servirá para reconstruirlo desde la capa de persistencia.
- Siguiendo lo visto en la construcción del _Value Object_ `EmailAddress`, añadimos un método para comparar dos instancias de la clase.

## Test

El test que hemos implementado es el siguiente:

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

Lo más importante a destacar en el test es que para la validación del formato hemos usado una librería diferente a la del objeto. Así testeamos el comportamiento del test y no su implementación.