---
date: 2023-04-11
---

# Controladors nets a Symfony (I): gestió d'excepcions
{:.no_toc}

* TOC
{:toc}

## Introducció

Suposem que tenim el següent _endpoint_ d'una API:

Url: `GET /llibres/{uuid}`

- Resposta correcta
    - Codi HTTP: 200
    - Contingut:
        ```json
        {
            "id": "c59620eb-c0ab-4a0c-8354-5a20faf537e5",
            "titol": "Curial e Güelfa",
            "autor": "Anònim"
        }
        ```

- Resposta incorrecta: UUID amb un format incorrecte:
    - Codi HTTP: `400`
    - Contingut:
        ```json
        {
            "error": "LlibreId provided format \"c59620eb-c0ab-4a0c-8354-5a20faf537e5\" is not a valid UUID"
        }
        ```

- Resposta incorrecta: llibre inexistent:
    - Codi HTTP: `404`
    - Contingut:
        ```json
        {
            "error": "LlibreDTO with LlibreId \"c59620eb-c0ab-4a0c-8354-5a20faf537e5\" not found"
        }
        ```

A Symfony, si fem servir el patró _query bus_, podríem tenir el següent controlador per a implementar l'_endpoint_
especificat:

```php
<?php

declare(strict_types=1);

namespace rubenrubiob\Infrastructure\Ui\Http\Controller;

use rubenrubiob\Application\Query\Llibre\GetLlibreDTOByIdQuery;
use rubenrubiob\Domain\DTO\Llibre\LlibreDTO;
use rubenrubiob\Domain\Exception\Repository\Llibre\LlibreDTONotFound;
use rubenrubiob\Domain\Exception\ValueObject\Llibre\LlibreIdFormatIsNotValid;
use rubenrubiob\Domain\Exception\ValueObject\Llibre\LlibreIdIsEmpty;
use rubenrubiob\Infrastructure\QueryBus\QueryBus;
use Symfony\Component\HttpFoundation\JsonResponse;
use Symfony\Component\HttpFoundation\Response;

final readonly class GetLlibreController
{
    public function __construct(private QueryBus $queryBus)
    {
    }

    public function __invoke(string $llibreId): Response
    {
        try {
            /** @var LlibreDTO $llibreDTO */
            $llibreDTO = $this->queryBus->__invoke(
                new GetLlibreDTOByIdQuery(
                    $llibreId
                )
            );
        } catch (LlibreIdFormatIsNotValid|LlibreIdIsEmpty $e) {
            return new JsonResponse(
                [
                    'error' => $e->getMessage(),
                ],
                Response::HTTP_BAD_REQUEST,
            );
        } catch (LlibreDTONotFound $e) {
            return new JsonResponse(
                [
                    'error' => $e->getMessage(),
                ],
                Response::HTTP_NOT_FOUND,
            );
        }

        return new JsonResponse(
            [
                'id' => $llibreDTO->llibreId->toString(),
                'titol' => $llibreDTO->llibreTitol->toString(),
                'autor' => $llibreDTO->autorNom->toString(),
            ],
            Response::HTTP_OK,
        );
    }
}

```

Amb aquesta implementació complim l'especificació, però veiem que el controlador té molta lògica: la gestió del format
de la resposta i dels codis HTTP d'error.

En aquest post, veurem com podem delegar la gestió de les excepcions al _framework_, per així simplificar els nostres
controladors i centralitzar el formatatge dels errors.

## Esdeveniments del _Kernel_ de Symfony

El flux d'execució web consisteix a rebre una petició i retornar una resposta. El component
[`HttpKernel`](https://symfony.com/doc/current/components/http_kernel.html) de Symfony és prou flexible per a gestionar
aquest flux per a _frameworks_ sencers, _microframeworks_ o un CMS com Drupal.

El _Kernel_ de Symfony és enfocat a esdeveniments, de manera que tota la lògica de gestió de peticions està abstreta,
alhora que proporciona flexibilitat. Això permet al desenvolupador executar codi propi en punts concrets del flux de
gestió d'una petició. Aquests punts es poden veure a la imatge següent[^1]:

![Font: documentació de Symfony](/images/dev/symfony-kernel-events.png)

Així doncs, podem delegar la gestió de les excepcions a un _Listener_ o _Subscriber_ del _Kernel_ que gestioni les
excepcions. En el nostre cas, optem per un _Subscriber_. Per a decidir-nos per un o
altre, [a la documentació de Symfony hi tenen una comparativa](https://symfony.com/doc/current/event_dispatcher.html#listeners-or-subscribers).

## `ExceptionResponseSubscriber`: primera versió

Podem afegir `EventSubscriber` que escolti l'esdeveniment `kernel.exception`, que és el punt en què Symfony gestiona les
excepcions no capturades. La implementació és la següent:

```php
<?php

declare(strict_types=1);

namespace rubenrubiob\Infrastructure\Symfony\Subscriber;

use rubenrubiob\Domain\Exception\Repository\Llibre\LlibreDTONotFound;
use rubenrubiob\Domain\Exception\ValueObject\Llibre\LlibreIdFormatIsNotValid;
use rubenrubiob\Domain\Exception\ValueObject\Llibre\LlibreIdIsEmpty;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Symfony\Component\HttpFoundation\JsonResponse;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\HttpKernel\Event\ExceptionEvent;
use Symfony\Component\HttpKernel\KernelEvents;
use Throwable;

use function array_key_exists;

final readonly class ExceptionResponseSubscriber implements EventSubscriberInterface
{
    private const EXCEPTION_RESPONSE_HTTP_CODE_MAP = [
        LlibreDTONotFound::class => Response::HTTP_NOT_FOUND,
        LlibreIdFormatIsNotValid::class => Response::HTTP_BAD_REQUEST,
        LlibreIdIsEmpty::class => Response::HTTP_BAD_REQUEST,
    ];

    public static function getSubscribedEvents(): array
    {
        return [KernelEvents::EXCEPTION => ['__invoke']];
    }

    public function __invoke(ExceptionEvent $event): void
    {
        $throwable = $event->getThrowable();

        $response = new JsonResponse(
            [
                'error' => $throwable->getMessage(),
            ],
            $this->httpCode($throwable),
            [
                'Content-Type' => 'application/json',
            ]
        );

        $event->setResponse($response);
    }

    private function httpCode(Throwable $throwable): int
    {
        $throwableClass = $throwable::class;
        if (array_key_exists($throwableClass, self:: EXCEPTION_RESPONSE_HTTP_CODE_MAP)) {
            return self:: EXCEPTION_RESPONSE_HTTP_CODE_MAP[$throwableClass];
        }

        return Response::HTTP_INTERNAL_SERVER_ERROR;
    }
}

```

Per a cadascuna de les excepcions del nostre codi, l'afegim a la constant `EXCEPTION_RESPONSE_HTTP_CODE_MAP`, on
_mapegem_ l'excepció al codi HTTP que ha de retornar. Si no tenim l'excepció controlada, retornem un error `500`.

En el _Subscriber_ també hi formatem la resposta d'error, que és molt bàsica: simplement mostrem el missatge de
l'excepció[^2].

En funció del format de la resposta, caldria afegir més lògica al _Subscriber_, per exemple si fem
servir [Problem Details](https://www.rfc-editor.org/rfc/rfc7807) o [JSON:API](https://jsonapi.org//format/#errors)[^3].

Si fem servir l'autoconfiguració per a definir els serveis de Symfony, el nostre _Subscriber_ ja estarà automàticament
configurat a l'aplicació.

## Controlador simplificat

Amb l'ús de `ExceptionResponseSubscriber`, el controlador el podem simplificar esborrant tota la gestió d'errors:

```php
<?php

declare(strict_types=1);

namespace rubenrubiob\Infrastructure\Ui\Http\Controller;

use rubenrubiob\Application\Query\Llibre\GetLlibreDTOByIdQuery;
use rubenrubiob\Domain\DTO\Llibre\LlibreDTO;
use rubenrubiob\Infrastructure\QueryBus\QueryBus;
use Symfony\Component\HttpFoundation\JsonResponse;
use Symfony\Component\HttpFoundation\Response;

final readonly class GetLlibreController
{
    public function __construct(private QueryBus $queryBus)
    {
    }

    public function __invoke(string $llibreId): Response
    {
        /** @var LlibreDTO $llibreDTO */
        $llibreDTO = $this->queryBus->__invoke(
            new GetLlibreDTOByIdQuery(
                $llibreId
            )
        );

        return new JsonResponse(
            [
                'id' => $llibreDTO->llibreId->toString(),
                'titol' => $llibreDTO->llibreTitol->toString(),
                'autor' => $llibreDTO->autorNom->toString(),
            ],
            Response::HTTP_OK,
        );
    }
}

```

Delegant la gestió d'errors a un _Subscriber_, aconseguim simplificar molt la lògica del controlador: només fem la crida
per a obtenir les dades i les formatem.

Ara bé, amb aquesta implementació del `Subscriber`, per a cada nova excepció que afegim al nostre codi i que vulguem que
tingui un codi HTTP concret, hem d'afegir-la a la constant `EXCEPTION_RESPONSE_HTTP_CODE_MAP`. Aquesta opció no creix
bé, ja que ens pot quedar un `array` enorme. A més a més, implica que cada desenvolupador ha de recordar afegir
l'excepció.

## Interfícies d'excepcions

Per a resoldre aquest problema, hi ha una alternativa: fer servir interfícies a les nostres excepcions de domini.

Per exemple, per a aquest cas, podem tenir dues interfícies diferents, una per a totes les excepcions de _Value Objects_
incorrectes, i una altra per als `NotFound`:

```php
<?php

declare(strict_types=1);

namespace rubenrubiob\Domain\Exception\ValueObject;

interface InvalidValueObject
{
}

```

```php
<?php

declare(strict_types=1);

namespace rubenrubiob\Domain\Exception\Repository;

interface NotFound
{
}

```

Implementar aquestes excepcions al nostre domini no implica res més que afegir un `implements`, ja que no tenen cos.
Així, doncs, podem veure com quedaria l'excepció `LlibreDTONotFound`:

```php
<?php

declare(strict_types=1);

namespace rubenrubiob\Domain\Exception\Repository\Llibre;

use Exception;
use rubenrubiob\Domain\Exception\Repository\NotFound;
use rubenrubiob\Domain\ValueObject\Llibre\LlibreId;

use function sprintf;

final class LlibreDTONotFound extends Exception implements NotFound
{
    public static function withLlibreId(LlibreId $llibreId): self
    {
        return new self(
            sprintf(
                'LlibreDTO with LlibreId "%s" not found',
                $llibreId->toString(),
            )
        );
    }
}

```

## `ExceptionResponseSubscriber`: segona versió

Amb les interfícies, ja podem refactoritzar el `ExceptionResponseSubscriber`:

```php
<?php

declare(strict_types=1);

namespace rubenrubiob\Infrastructure\Symfony\Subscriber;

use rubenrubiob\Domain\Exception\Repository\NotFound;
use rubenrubiob\Domain\Exception\ValueObject\InvalidValueObject;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Symfony\Component\HttpFoundation\JsonResponse;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\HttpKernel\Event\ExceptionEvent;
use Symfony\Component\HttpKernel\KernelEvents;
use Throwable;

use function array_key_exists;
use function Safe\class_implements;

final readonly class ExceptionResponseSubscriber implements EventSubscriberInterface
{
    private const EXCEPTION_RESPONSE_HTTP_CODE_MAP = [
        NotFound::class => Response::HTTP_NOT_FOUND,
        InvalidValueObject::class => Response::HTTP_BAD_REQUEST,
    ];

    public static function getSubscribedEvents(): array
    {
        return [KernelEvents::EXCEPTION => '__invoke'];
    }

    public function __invoke(ExceptionEvent $event): void
    {
        $throwable = $event->getThrowable();

        $response = new JsonResponse(
            [
                'error' => $throwable->getMessage(),
            ],
            $this->httpCode($throwable),
            [
                'Content-Type' => 'application/json',
            ]
        );

        $event->setResponse($response);
    }

    private function httpCode(Throwable $throwable): int
    {
        /** @var class-string[] $interfaces */
        $interfaces = class_implements($throwable);

        foreach ($interfaces as $interface) {
            if (array_key_exists($interface, self::RESPONSE_CODE_MAP)) {
                return self::EXCEPTION_RESPONSE_HTTP_CODE_MAP[$interface];
            }
        }

        return Response::HTTP_INTERNAL_SERVER_ERROR;
    }
}

```

Amb només les dues interfícies, ens estalviem haver d'afegir cadascuna de les excepcions que implementem al nostre codi. Qualsevol excepció que sorgeixi d'un _Value Object_ invàlid o d'un `NotFound` retornarà un codi de resposta HTTP adequat.

## Conclusions

Fer servir les interfícies no és una solució definitiva, ja que pot haver-hi situacions en què vulguem tenir excepcions
concretes. En qualsevol cas, es poden combinar ambdues opcions, tot i que, en aquest cas, segurament el millor seria
extreure-ho a un servei que fes aquesta gestió.

Com sempre, el millor és anar evolucionant el codi en funció de les necessitats que ens hi anem trobant.

## Resum

- Hem vist el potencial de l'orientació a esdeveniments del _Kernel_ de Symfony per a simplificar les nostres
  aplicacions.
- Hem aconseguit refactoritzar un controlador per a extreure'n el control d'errors i delegar-lo a un `Subscriber`.
- Hem vist com crear un `Subscriber` per a l'esdeveniment `kernel.exception` per a gestionar les excepcions d'una API de
  manera centralitzada.


[^1]: Es pot consultar la taula de tots els esdeveniments i els paràmetres que reben a la [documentació oficial de Symfony](https://symfony.com/doc/current/components/http_kernel.html#component-http-kernel-event-table).

[^2]: Mostrar directament el missatge de l'excepció no és una bona opció, perquè acostumen a ser missatges escrits per i per a desenvolupadors. A més a més, poden contenir dades o informació sensible que mai no hauríem d'exposar públicament.

[^3]: Per a veure una bona descripció de possibles alternatives, consulteu [aquest article d'_APIs you won't hate_](https://apisyouwonthate.com/blog/useful-api-errors-for-rest-graphql-and-grpc/).