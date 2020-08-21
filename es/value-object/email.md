---
lang: es
lang-ref: value-object-email-address
tags: [domain,value-object,email]
is-content-page: true
layout: content-page
order: 1
---

# _Value Object_: `EmailAddress`

Este es el primer apartado de desarrollo dentro del proyecto. Debemos empezar de lo más interno a lo más externo, de modo que la primera capa de trabajo es el Dominio. Y, dentro del Dominio, los elementos más básicos a desarrollar son los _Value Objects_.

Un _Value Object_ es un objeto que se define por su valor (+ definiciones). Si no estamos familiarizados con los _Value Objects_, esta definición quizá no nos dice gran cosa, pero veremos su sentido con el primer ejemplo, el de dirección de correo electrónico.

Los _Value Object_ son un concepto potentísimo, especialmente para trabajar con los siempre problemáticos números flotantes (citas). Todos los atributos de nuestras entidades serán _Value Objects_, no usaremos en ningún caso tipos primitivos del lenguaje.

## Email

### Introducción

Empecemos con un contraejemplo. Imaginemos que en un objeto cualquiera tenemos la siguiente definición de un método que envía una notificación a una dirección de correo electrónico de un usuario:

```php
public function notify(string $email) : void
{
    // ...
}
```

O, peor aún, sin la especificación del tipo de dato:

```php
public function notify($email) : void
{
    // ...
}
```

¿Cómo sabemos que el valor de `$email` es realmente un _string_ con una dirección de correo electrónico válida? En el segundo caso, además, tampoco estamos seguros de que el tipo de `$email` sea un _string_.

Sabemos que una dirección de correo electrónico es siempre un _string_, pero no todo _string_ es una dirección de correo electrónico. Es decir, las direcciones de correo electrónico son un subconjunto de todos los _string_ existentes:

(Añadir imagen)

Así pues, lo primero que deberíamos hacer en esta función es validar que el _string_ que nos pasan es efectivamente una dirección de correo electrónico válida, y lanzar una excepción si no lo es:

```php
public function notify(string $email) : void
{
    if (filter_var($email, FILTER_VALIDATE_EMAIL) === false) {
        throw new \InvalidArgumentException('Invalid email address');
    }
    
    // ...
}
```

Para estar seguros, esta validación deberíamos tenerla en cada método que implementemos y que reciba un parámetro que sea una dirección de correo electrónico. Deberíamos hacer lo mismo para cualquier otro parámetro que recibamos en cada función.

Ya vemos que esto nos implica una validación redundante en cada método. Además, nos añadiría caminos de test que nada tienen que ver con el método que estamos testeando.

La solución es tener un _Value Object_, llamado por ejemplo `EmailAddress`, que podemos pasar como tipo de parámetro de los métodos:

```php
public function notify(EmailAddress $email) : void
{
    // ...
}
```

De este modo, en este método sabemos con toda certeza que `$email` es de tipo `EmailAddress`. Si la construcción del objeto `EmailAddress` fallase en un punto previo de la ejecución, sería responsabilidad de esa porción de código tratar la excepción.

### _Namespace_

El _Value Object_ `EmailAddress` es algo que podrá reutilizarse en diferentes partes de la aplicación: podría servir para los usuarios de la aplicación, como dato de contacto de una editorial...[^1]

Así pues, este _Value Object_ lo colocaremos dentro del _namespace_ `Domain\ValueObject`, que corresponderá al directorio físico dentro del proyecto `src/Domain/ValueObject`.

En la explicación del _Value Object_ de formato de una edición de un libro veremos dónde colocar _Value Objects_ que no sean genéricos y reusables para toda la aplicación.

### Implementación

Una posible implementación del _Value Object_ `EmailAddress` podría ser la siguiente:

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

- Como seguimos _composition over inheritance_, todas las clases que generemos, salvo algunas excepciones, [serán declaradas como `final`](https://ocramius.github.io/blog/when-to-declare-classes-final/).
- El constructor es privado. De este modo, forzamos a [utilizar siempre el _named constructor_](https://github.com/ShittySoft/symfony-live-berlin-2018-doctrine-tutorial/pull/3#issuecomment-460614781), y evitamos [casos extraños](https://twitter.com/DaveLiddament/status/1239259573493604352) que permite el lenguaje y que harían que el objeto fuese mutable.
- Hemos añadido un método `asString`, que permite obtener el valor del objeto como un _string_. No utilizamos el método mágico `__toString` [para evitar problemas](https://github.com/ShittySoft/symfony-live-berlin-2018-doctrine-tutorial/pull/3#issuecomment-460441229). En otros objetos podemos tener varios métodos; por ejemplo, en un importe, podríamos tener `asString`, `asFloat`...
- Hemos añadido un método `equalsTo`, para comparar dos _Value Object_. No es estrictamente necesario, pero es un método muy simple de crear y testear, y ayuda en el proceso de test.
- Un _Value Object_ siempre es inmutable, de modo que hemos añadido la anotación correspondiente para _Psalm_.
- Usamos la librería [`webmozart/assert`](https://github.com/webmozart/assert), ya que simplifica mucho el proceso de validación.
- Vemos que la excepción que lanzamos es bastante legible: `throw EmailAddressIsNotValid::withFormat`.

### Excepción

En todos los casos del Dominio en los que la ejecución no sea válida, lanzaremos una excepción específica. Para ello, deberemos crear una clase. Una posible implementación sería:

```
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

- Siguiendo [algunas recomendaciones](https://www.nikolaposa.in.rs/assets/uploads/slides/phpbnl20/handling-exceptional-conditions/#/), creamos un _named constructor_, para generar la excepción, cosa que, además, hace más legible el código.
- Utilizamos la librería [thecodingmachine/safe](https://github.com/thecodingmachine/safe), que contiene las funciones de PHP pero con una API más usable que lanza excepciones en lugar de devolver `false` si la llamada es in´valida. En el caso de `sprintf`, si el número de parámetros no es válido, lanza una excepción.

### Test

El test que hemos creado para este objeto es el siguiente:

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

- Todos los nombres de los métodos de test están con *snake_case*. Es algo que uso solo en los test, puesto que uso nombres muy específicos que acaban siendo largos, y creo que [simplifica bastante la lectura](https://matthiasnoback.nl/2020/06/unit-test-naming-conventions/).
- Testeamos en métodos aparte las excepciones que se pueden generar para el método _create_. En este caso, tenemos dos: que el _string_ que nos pasan o bien no sea una dirección de correo electrónico, o bien que sea un _string_ vacío.
- Aunque no es estríctamente necesario, testeamos la creación correcta del objeto y el uso del método `asString`, para que Infection nos valide correctamente la visibilidad del método.
- Usamos un [_Data provider_](https://phpunit.readthedocs.io/en/9.3/writing-tests-for-phpunit.html#data-providers) para el test de `equalsTo`.


[^1]: Cabe aclarar que este _Value Object_ no es estrictamente necesario para la primera fase de desarrollo, pero seguramente sea el más ilustrativo para la explicación sobre qué es un _Value Object_.