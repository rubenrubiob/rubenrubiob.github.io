---
date: 2023-05-30
---

# _Type hint_ de _Query Bus_ a PHP
{:.no_toc}

* TOC
{:toc}

## Introducció

Si fem servir un _Query Bus_[^1], possiblement tenim una interfície que defineix una `Query`:

```php
<?php

declare(strict_types=1);

namespace rubenrubiob\Application;

interface Query
{
}
```

I una altra interfície que defineix el `QueryBus`:

```php
<?php

declare(strict_types=1);

namespace rubenrubiob\Infrastructure\QueryBus;

use rubenrubiob\Application\Query;
use Throwable;

interface QueryBus
{
    public function __invoke(Query $query): mixed;
}
```

Veient un cas d'ús concret, tindríem per exemple aquesta `Query`, que retorna un llibre:

```php
<?php

declare(strict_types=1);

namespace rubenrubiob\Application\Query\Llibre;

use rubenrubiob\Application\Query;
use rubenrubiob\Domain\DTO\Llibre\LlibreDTO;

final readonly class GetLlibreDTOByIdQuery implements Query
{
    public function __construct(
        public string $id,
    ){
    }
}

```

Aquesta `Query` es cridaria des del controlador de la següent manera:

```php
<?php

declare(strict_types=1);

namespace rubenrubiob\Infrastructure\Ui\Http\Controller;

use rubenrubiob\Application\Query\Llibre\GetLlibreDTOByIdQuery;
use rubenrubiob\Domain\DTO\Llibre\LlibreDTO;
use rubenrubiob\Infrastructure\QueryBus\QueryBus;

final readonly class GetLlibreController
{
    public function __construct(private QueryBus $queryBus)
    {
    }

    public function __invoke(string $llibreId): LlibreDTO
    {
        /** @var LlibreDTO $llibre */
        $llibre = $this->queryBus->__invoke(
            new GetLlibreDTOByIdQuery(
                $llibreId,
            )
        );

        return $llibre;
    }
}


```

El problema que tenim és que no sabem quin tipus de dada retorna el `QueryBus`, perquè el tipus de retorn és `mixed`,
que pot ser qualsevol cosa: retorna un objecte? Un `array`? No retorna res?

L'única manera de saber-ho és forçant un _type hint_ a la variable dins del controlador. Això, a part del problema que
l'anotació quedi obsoleta en algun moment, també genera problemes amb els analitzadors de codi estàtic com Psalm o
PHPStan.

Ara bé, precisament amb la introducció d'aquests analitzadors, una cosa similar
als [genèrics](https://stitcher.io/blog/generics-in-php-1){:target="_blank"} es van començar a poder fer servir a
PHP. [També PHPStorm dona suport als genèrics, i en va introduint suport gradualment](https://blog.jetbrains.com/phpstorm/tag/generics/){:target="_blank"}.

En aquest post veurem com podem emprar genèrics al nostre `QueryBus` per a saber-ne el tipus de retorn.

## Implementació

### `Query`

El que farem és connectar la `Query` amb el valor de resposta del `QueryHandler` que la gestiona. Ho farem amb genèrics,
que a PHP es descriuen com a `template`.

A la interfície `Query` el que fem és afegir una anotació per indicar que és un `template`:

```php
<?php

declare(strict_types=1);

namespace rubenrubiob\Application;

/** @template T */
interface Query
{
}

```

Un `template` o genèric el que indica és que tindrem un tipus variable, igual que tenim variables normal i corrents.

En aquest cas, tenim un tipus variable que anomenem `T`. El nom és arbitrari. Així, indicarem que totes les classes que
implementin la interfície `Query`, implementaran un tipus genèric `T`.

### `QueryBus`

Ara podem tipar el nostre `QueryBus` així:

```php
<?php

declare(strict_types=1);

namespace rubenrubiob\Infrastructure\QueryBus;

use rubenrubiob\Application\Query;
use Throwable;

interface QueryBus
{
    /**
     * @template T
     *
     * @param Query<T> $query
     *
     * @return T
     *
     * @throws Throwable
     */
    public function __invoke(Query $query): mixed;
}

```

Com a argument, rebem una `Query` que incorpora el tipus `T`. Recordem que `T` és un tipus variable, i que el nom és
arbitrari. El valor de retorn del `QueryBus` será aquest tipus variable que incorpora la `Query`, `T`.

Si ens fixem en els tipus, la descripció del `QueryBus` és:

```
f(Query<T>) => T
```

### `GetLlibreDTOByIdQuery`

Si ara actualitzem la `Query` concreta `GetLlibreDTOByIdQuery`, ens queda així:

```php
<?php

declare(strict_types=1);

namespace rubenrubiob\Application\Query\Llibre;

use rubenrubiob\Application\Query;
use rubenrubiob\Domain\DTO\Llibre\LlibreDTO;

/** @implements Query<LlibreDTO> */
final readonly class GetLlibreDTOByIdQuery implements Query
{
    public function __construct(
        public string $id,
    ){
    }
}

```

Tornant a mirar la descripció dels tipus, si substituïm el tipus `T` pel nostre cas concret, `LlibreDTO`, tenim la
següent descripció:

```
f(Query<T>) => T
f(Query<LlibreDTO>) => LlibreDTO
```

Veiem que PHPStorm també detecta correctament el tipus de dada:

![Variable tipada a PHPStorm](/images/dev/type-hint-query-bus/typed-controller-variable.png)

## Conclusions

Emprant genèrics aconseguim tipar el nostre `QueryBus`: les implementacions de `Query` indiquen el tipus de resposta, i
aquest és l'enllaç necessari que necessitem.

## Resum

- Hem vist el problema de tipat que suposa no tipar un `QueryBus`.
- Hem explicat molt resumidament què són els genèrics.
- Hem implementat una descripció de tipus que permet tipar correctament el nostre `QueryBus`.


[^1]: Cal plantejar-nos si realment necessitem un `QueryBus`. Per què ens cal? Un `CommandBus` aporta tots els _middleware_ que poden, per exemple, obrir i tancar una transacció a la base de dades. Però, quin _middleware_ fem servir a un `QueryBus`? Potser podem substituir el `QueryBus` pel `QueryHandler` que crida; o, si som encara més radicals, potser ni tan sols necessitem els `Query` i `QueryHandler`, i podem cridar directament al servei de domini que ens retorna la dada que necessitem. Per a veure'n un debat més extens, [vegeu el següen article](https://matthiasnoback.nl/2019/06/you-may-not-need-a-query-bus/){:target="_blank"} i els comentaris.
