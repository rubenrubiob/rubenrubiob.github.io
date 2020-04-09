# Value Objects

Este es el primer apartado de desarrollo dentro del proyecto. Debemos empezar de lo más interno a lo más externo, de modo que la primera capa de trabajo es el Dominio. Y, dentro del Dominio, los elementos más básicos a desarrollar son los _Value Objects_.

Un _Value Object_ es un objeto que se define por su valor (+ definiciones). Si no estamos familiarizados con los _Value Objects_, esta definición quizá no nos dice gran cosa, pero veremos su sentido con el primer ejemplo, el de dirección de correo electrónico.

Los _Value Object_ son un concepto potentísimo, especialmente para trabajar con los siempre problemáticos números flotantes (citas).

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

Sabemos que una dirección de correo electrónico es siempre un _string_, pero no todo _string_ es una dirección de correo electrónico:

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

Ya vemos que esto nos implica una validación redundante en cada método. Además, nos añadiría caminos de test que nada tienen que ver con el mètodo que estamos testeando.

La solución es tener un _Value Object_, llamado por ejemplo `EmailAddress`, que podemos pasar como tipo de parámetro de los métodos:

```php
public function notify(EmailAddress $email) : void
{
    // ...
}
```

De este modo, en este método sabemos con toda certeza que `$email` es de tipo `EmailAddress`. Si la construcción del objeto `EmailAddress` fallase en un punto previo de la ejecución, sería responsabidad de esa porción de código tratar la excepción.

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
    public static function becauseItIsNotAValidEmailAddress(string $emailAddress) : self
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

- Siguiendo algunas recomendaciones (enlace), creamos un _named constructor_, para generar la excepción. A mí me gusta que el código se pueda leer en la medida de lo posible, aunque no sea tan breve, de ahí que el nombre sea tan largo. Se puede ver un ejemplo de uso en el siguiente apartado.
- Utilizamos la librería [thecodingmachine/safe](https://github.com/thecodingmachine/safe), que contiene las funciones de PHP pero con una API más usable. En el caso de `sprintf`, si el número de parámetros no es válido, lanza una excepción, en lugar de devolver `false`.

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
    public static function create(string $emailAddress) : self
    {
        self::validate($emailAddress);

        return new self(
            $emailAddress
        );
    }

    public function asString() : string
    {
        return $this->emailAddress;
    }

    public function equalsTo(EmailAddress $anotherEmailAddress) : bool
    {
        return $this->emailAddress === $anotherEmailAddress->emailAddress;
    }

    /**
     * @throws EmailAddressIsNotValid
     */
    private static function validate(string $emailAddress) : void
    {
        try {
            Assert::email($emailAddress);
        } catch (InvalidArgumentException $e) {
            throw EmailAddressIsNotValid::becauseItIsNotAValidEmailAddress($emailAddress);
        }
    }
}
```

- Como seguimos _composition over inheritance_, todas las clases que generemos, salvo algunas excepciones, serán declaradas como `final` (enlace).
- El constructor es privado. De este modo, forzamos a utilizar siempre el _named constructor_, y evitamos casos extraños que permite el lenguaje (enlace).
- Hemos añadido un método `asString`, que permite obtener el valor del objeto como un _string_. No utilizamos el método mágico `__toString` para evitar problemas (enlace). En otros objetos podemos tener varios métodos; por ejemplo, en un importe, podríamos tener `asString`, `asFloat`...
- Hemos añadido un método `equalsTo`, para comparar dos _Value Object_. No es estrictamente necesario, pero es un método muy simple de crear y testear, y ayuda en el proceso de test.
- Un _Value Object_ siempre es inmutable, de modo que hemos añadido la anotación correspondiente para _Psalm_.
- Usamos la librería [`webmozart/assert`](https://github.com/webmozart/assert), ya que simplifica mucho el proceso de validación.


### Test

El test que hemos creado para este objeto es el siguiente:

```
<?php
declare(strict_types=1);

namespace Tests\Domain\ValueObject;

use Domain\Exception\ValueObject\EmailAddressIsNotValid;
use Domain\ValueObject\EmailAddress;
use PHPUnit\Framework\TestCase;

class EmailTest extends TestCase
{
    /**
     * @test
     */
    public function create() : void
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
    public function equals_to(string $firstEmailAddress, string $secondEmailAddress, bool $expected) : void
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
    public function create_with_invalid_email_address_throws_exception() : void
    {
        $this->expectException(EmailAddressIsNotValid::class);

        EmailAddress::create('this_is_not_a_valid_email_address@domain');
    }

    /**
     * @test
     */
    public function create_with_empty_email_address_throws_exception() : void
    {
        $this->expectException(EmailAddressIsNotValid::class);

        EmailAddress::create('  ');
    }

    public function provideForEqualsTo() : array
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

- Todos los nombres de los métodos de test están con _snake_case_. Es algo que uso solo en los test, puesto que uso nombres muy específicos que acaban siendo largo, y creo que simplifica bastante la lectura.
- Testeamos en métodos aparte las excepciones que se pueden generar para el método _create_. En este caso, tenemos dos: que el _string_ que nos pasan o bien no sea una dirección de correo electrónico, o bien que sea un _string_ vacío.
- Aunque no es estríctamente necesario, testeamos la creación correcta del objeto y el uso del método `asString`, para que Infection nos valide correctamente la visibilidad del método.
- Usamos un _Data provider_ (enlace) para el test de `equalsTo`.

### Salidas Mutation, psalm, testing

## Id

- No usar id autogenerados
	- Un objeto ha de ser siempre válido, desde el momento de su creación
	- No acceso a infrastructura
- Uso de UUID:
	- Colisión baja
	- Aleatorio
- ramsey/uuid: https://github.com/ramsey/uuid
- Id:
	- Podríamos tener instancias por cada tipo de id
	- Generar
	- Desde string
	- Comparar

### Salidas Mutation, psalm, testing

```bash
$ vendor/bin/phpunit
PHPUnit 9.1.1 by Sebastian Bergmann and contributors.

...........                                                       11 / 11 (100%)

Time: 174 ms, Memory: 10.00 MB

OK (11 tests, 11 assertions)
```

```bash
$ composer psalm        
> vendor/bin/psalm
Scanning files...
Analyzing files...

░░░░

------------------------------
No errors found!
------------------------------

Checks took 6.03 seconds and used 138.514MB of memory
Psalm was able to infer types for 100% of the codebase

```

```bash
$ composer infection
> vendor/bin/infection
You are running Infection with Xdebug enabled.

    ____      ____          __  _
   /  _/___  / __/__  _____/ /_(_)___  ____
   / // __ \/ /_/ _ \/ ___/ __/ / __ \/ __ \
 _/ // / / / __/  __/ /__/ /_/ / /_/ / / / /
/___/_/ /_/_/  \___/\___/\__/_/\____/_/ /_/

Infection - PHP Mutation Testing Framework 0.16.2@9a3bca95fa873fba4a4a37a5fd35c6d572c0f23b

Running initial test suite...

PHPUnit version: 9.1.1

    0 [>---------------------------] < 1 sec
    1 [->--------------------------] < 1 sec
    3 [--->------------------------]  1 sec
   10 [----->----------------------]  1 sec
   20 [------->--------------------]  1 secProcessing source code files: 0/4

Generate mutants...


Processing source code files: 2/4
Processing source code files: 4/4
.: killed, M: escaped, S: uncovered, E: fatal error, T: timed out

.................                                    (17 / 17)

17 mutations were generated:
      17 mutants were killed
       0 mutants were not covered by tests
       0 covered mutants were not detected
       0 errors were encountered
       0 time outs were encountered

Metrics:
         Mutation Score Indicator (MSI): 100%
         Mutation Code Coverage: 100%
         Covered Code MSI: 100%

Please note that some mutants will inevitably be harmless (i.e. false positives).

Time: 4s. Memory: 14.00MB

```