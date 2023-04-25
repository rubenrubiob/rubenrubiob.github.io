---
date: 2023-04-25
---

# Controladors nets a Symfony (III): gestió de peticions
{:.no_toc}

* TOC
{:toc}

## Introducció

Fins ara, hem vist com simplificar _endpoints_ que representen _queries_, és a dir, que retornen dades. Ara bé, què
passa amb els _endpoints_ que representen _commands_, és a dir, que modifiquen el nostre sistema?

Suposem que tenim el següent _endpoint_ per a crear un llibre:

- Url: `POST /llibres`
- Petició:
    - Contingut:
        ```json
        {
            "titol": "Curial e Güelfa",
            "autor": "Anònim"
        }
        ```
- Resposta correcta
    - Codi HTTP: `204`
    - Sense contingut
- Resposta incorrecta
    - Codi HTTP: `400`
    - Contingut:
        ```json
        {
            "error": "LlibreTitol provided is empty"
        }
        ```

El controlador que gestiona aquesta petició és:

```php
<?php

declare(strict_types=1);

namespace rubenrubiob\Infrastructure\Ui\Http\Controller;

use rubenrubiob\Application\Command\Llibre\CrearLlibreCommand;
use rubenrubiob\Infrastructure\CommandBus\CommandBus;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpKernel\Exception\BadRequestHttpException;

use function array_key_exists;
use function is_string;

final readonly class CreateLlibreController
{
    private const KEY_TITOL = 'titol';
    private const KEY_AUTOR = 'autor';

    public function __construct(private CommandBus $commandBus)
    {
    }

    public function __invoke(Request $request): void
    {
        $requestContent = $this->parseAndGetRequesContent($request);

        $this->commandBus->__invoke(
            new CrearLlibreCommand(
                $requestContent[self::KEY_TITOL],
                $requestContent[self::KEY_AUTOR],
            )
        );
    }

    /**
     * @return array{
     *      titol: string,
     *      autor: string,
     *     ...
     * }
     *
     * @throws BadRequestHttpException
     */
    private function parseAndGetRequesContent(Request $request): array
    {
        $requestContent = $request->toArray();

        if (!array_key_exists(self::KEY_TITOL, $requestContent)) {
            throw new BadRequestHttpException('Missing "titol"');
        }

        if (!is_string($requestContent[self::KEY_TITOL])) {
            throw new BadRequestHttpException('"titol" format is not valid');
        }

        if (!array_key_exists(self::KEY_AUTOR, $requestContent)) {
            throw new BadRequestHttpException('Missing "autor"');
        }

        if (!is_string($requestContent[self::KEY_AUTOR])) {
            throw new BadRequestHttpException('"autor" format is not valid');
        }

        return $requestContent;
    }
}
```

Veiem que al controlador hem de validar el JSON que ens ve a la petició: que vinguin tots els camps, que tinguin el
format correcte...

A Symfony existeix el [component Form](https://symfony.com/doc/current/forms.html), que permet validar les dades que
vinguin a una petició. Ara bé, el component Form té sentit per a aplicacions tradicionals, en què PHP renderitza el
_backend_ i el _frontend_ d'una aplicació, no tant per a una API. A més a més, és complex de configurar en aquest cas
d'ús.

Ara bé, internament, el component Form empra el [component Validator](https://symfony.com/doc/current/validation.html)
per a validar els camps. Aquest és el component que conté tota la potència de validació del component Form.

Per a simplificar el nostre controlador, aprofitarem el component Validator fent servir els esdeveniments del `Kernel`
de Symfony.

## Esdeveniment `Resolve arguments`

Tal com vam explicar, el `Kernel` de Symfony fa servir esdeveniments en què el desenvolupador pot fer accions:

![Font: documentació de Symfony](/images/dev/symfony-kernel-events.png)

Veiem que hi ha un punt que s'executa immediatament abans del controlador, el 4,
que [resol els seus arguments](https://symfony.com/doc/current/components/http_kernel.html#4-getting-the-controller-arguments).
El que fa el `Kernel` de Symfony és executar un controlador, que és un `callable`, passant-li un `array` d'arguments.
Per a cadascun d'aquests arguments, Symfony en calcula el valor emprant serveis que implementen la
interfície [`ValueResolverInterface`](https://github.com/symfony/symfony/blob/6.2/src/Symfony/Component/HttpKernel/Controller/ValueResolverInterface.php)[^1].

Per exemple, Symfony incorpora una implementació de `ValueResolver` que mira si un dels arguments del controlador
és de tipus `Request`; si ho és, n'injecta la petició actual.

El que farem és aprofitar aquesta resolució d'arguments per a poder injectar objectes que representin les nostres
peticions, de manera que ja vinguin validats amb el component Validator.

## Implementació

### `APIRequestBody`

Tindrem una interfície que representi el contingut de totes les peticions de tipus _command_:

```php
<?php

declare(strict_types=1);

namespace rubenrubiob\Infrastructure\Symfony\Http\Request;

interface APIRequestBody
{
}
```

### `CreateLlibreRequestBody`

Per a aquest cas concret, tindrem un objecte que representa la petició de crear un llibre, que
implementa `APIRequestBody`:

```php
<?php

declare(strict_types=1);

namespace rubenrubiob\Infrastructure\Ui\Http\Request;

use rubenrubiob\Infrastructure\Symfony\Http\Request\APIRequestBody;
use Symfony\Component\Validator\Constraints as Assert;

final readonly class CreateLlibreRequestBody implements APIRequestBody
{
    public function __construct(
        #[Assert\NotBlank(normalizer: 'trim')]
        public string $titol,
        #[Assert\NotBlank(normalizer: 'trim')]
        public string $autor
    ) {
    }
}
```

Els atributs fan servir el validador de Symfony, en aquest cas, ambdós han de ser `NotBlank`. Hi apliquem `trim`, per a
normalitzar el contingut.

Cal notar que tots els atributs són públics, ja que aquesta classe és un DTO.

### `APIRequestResolver`

Amb tot això, ja podem implementar el nostre `ValueResolverInterface`, que transformarà una petició en un
objecte de tipus `APIRequestBody`.

Fem servir la llibreria [cuyz/valinor](https://valinor.cuyz.io/latest/) que
ja [vam configurar al `Kernel`](/dev/2022-12-valinor-a-symfony-amb-value-objects/). També cal instal·lar
el [component Validator de Symfony](https://symfony.com/doc/current/validation.html).

La implementació és la següent:

```php
<?php

declare(strict_types=1);

namespace rubenrubiob\Infrastructure\Symfony\Http\Request;

use CuyZ\Valinor\Mapper\MappingError;
use CuyZ\Valinor\Mapper\Source\Exception\InvalidSource;
use CuyZ\Valinor\Mapper\Source\Source;
use CuyZ\Valinor\Mapper\TreeMapper;
use ReflectionClass;
use ReflectionException;
use rubenrubiob\Infrastructure\Symfony\Http\Exception\InvalidRequest;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpKernel\Controller\ValueResolverInterface;
use Symfony\Component\HttpKernel\ControllerMetadata\ArgumentMetadata;
use Symfony\Component\Validator\Validator\ValidatorInterface;

use function count;

final readonly class APIRequestResolver implements ValueResolverInterface
{
    public function __construct(
        private TreeMapper $treeMapper,
        private ValidatorInterface $validator,
    ) {
    }

    /**
     * @return iterable<APIRequestBody>|iterable<null>
     *
     * @throws InvalidRequest
     */
    public function resolve(Request $request, ArgumentMetadata $argument): iterable
    {
        /** @var class-string|null $class */
        $class = $argument->getType();

        if (! $this->supports($class)) {
            return [null];
        }

        try {
            $request = $this->treeMapper->map(
                $class,
                Source::json($request->getContent())->camelCaseKeys(),
            );
        } catch (MappingError|InvalidSource) {
            throw InvalidRequest::createFromBadMapping();
        }

        $errors = $this->validator->validate($request);

        if (count($errors) > 0) {
            throw InvalidRequest::fromConstraintViolationList($errors);
        }

        yield $request;
    }

    /**
     * @param class-string|null $class
     *
     * @psalm-assert-if-true class-string<APIRequestBody> $class
     * @phpstan-assert-if-true class-string<APIRequestBody> $class
     */
    private function supports(?string $class): bool
    {
        if ($class === null) {
            return false;
        }

        try {
            $reflection = new ReflectionClass($class);

            if ($reflection->implementsInterface(APIRequestBody::class)) {
                return true;
            }
        } catch (ReflectionException) {
        }

        return false;
    }
}

```

El mètode `supports` és el que indica si cal emprar aquest `ArgumentValueResolver` per a aquest argument o no. En el
nostre cas, seran tots objectes que implementin la interfície `APIRequestBody`.

En el mètode `resolve` és on fem la conversió i validació:

- La crida a `$this->treeMapper->map` és la que converteix una font en JSON —la petició— al nostre objecte. Si la conversió falla, llancem una excepció.
- Amb `$this->validator->validate` executem la validació de l'objecte amb el validador de Symfony. Si la validació falla, llancem una excepció.

Si fem servir l’autoconfiguració per a definir els serveis de Symfony, el nostre `ArgumentValueResolver` ja estarà
automàticament afegit al _framework_.

## Controlador simplificat

Amb el nostre `ArgumentValueResolverInterface` complet i configurat, ja podem simplificar el controlador, que queda així:

```php
<?php

declare(strict_types=1);

namespace rubenrubiob\Infrastructure\Ui\Http\Controller;

use rubenrubiob\Application\Command\Llibre\CrearLlibreCommand;
use rubenrubiob\Infrastructure\CommandBus\CommandBus;
use rubenrubiob\Infrastructure\Ui\Http\Request\CreateLlibreRequestBody;

final readonly class CreateLlibreController
{
    public function __construct(private CommandBus $commandBus)
    {
    }

    public function __invoke(CreateLlibreRequestBody $createLlibreRequestBody): void
    {
        $this->commandBus->__invoke(
            new CrearLlibreCommand(
                $createLlibreRequestBody->titol,
                $createLlibreRequestBody->autor,
            )
        );
    }
}
```

En _tipejar_ com a argument el `CreateLlibreRequestBody`, ja ens arriba directament aquest objecte, que podem fer
servir. I com que a l'`ArgumentValueResolverInterface` llancem algun error si la petició no es _mapeja_ correctament o
si no passa alguna de les regles de validació, al controlador l'objecte `CreateLlibreRequestBody` ja ens arriba vàlid. 

## Conclusions

Igual que vam fer amb la gestió d'excepcions i de respostes, deleguem la gestió de la conversió de peticions _command_
al _framework_. A més a més, aprofitem el component Validator de Symfony per a validar el contingut de les peticions.

D'aquesta manera, al controlador només gestionem peticions semànticament correctes —tot i que això no evita que hi hagi
algun error de domini durant l'execució del `Command`.

## Resum

- Hem vist el problema que suposa haver de validar les dades a totes les peticions.
- Hem introduït els `APIRequestBody` per a representar les nostres peticions amb contingut.
- Hem creat una implementació d'`APIRequestBody` que incorpora validació dels atributs.
- Hem creat un `ArgumentValueResolver` que converteix peticions JSON a un objecte de tipus `APIRequestBody` i en valida els atributs.
- Hem simplificat els nostres controladors perquè tinguin la mínima lògica possible.

[^1]: A Symfony 5.4 cal fer implementar la interfície `ArgumentValueResolverInterface`.