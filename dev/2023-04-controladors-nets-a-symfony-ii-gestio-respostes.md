---
date: 2023-04-17
---

# Controladors nets a Symfony (II): gestió de respostes
{:.no_toc}

* TOC
{:toc}

## Introducció

Suposem que tenim el mateix _endpoint_ [que en el post anterior](/dev/2023-04-controladors-nets-a-symfony-i-gestio-excepcions/){:target="_blank"} (per simplicitat, només en mostrem la resposta correcta):

- Url: `GET /llibres/{uuid}`
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

El controlador, havent refactoritzat la gestió d'excepcions, quedava així:

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

Suposem ara que tenim un altre _endpoint_ que retorna un llistat de llibres, descrit com el següent:

- Url: `GET /llibres`
- Resposta correcta
    - Codi HTTP: 200
    - Contingut:
        ```json
        [
            {
                "id": "c59620eb-c0ab-4a0c-8354-5a20faf537e5",
                "titol": "Curial e Güelfa",
                "autor": "Anònim"
            },
            {
                "id": "4f9d75b7-5dd4-4d19-8a31-8876d54cddee",
                "titol": "Tirant Lo Blanc",
                "autor": "Joanot Martorell"
            },
        ]
        ```

El controlador que gestiona aquest _endpoint_ és el següent:

```php
<?php

declare(strict_types=1);

namespace rubenrubiob\Infrastructure\Ui\Http\Controller;

use rubenrubiob\Application\Query\Llibre\FindLlibreDTOsQuery;
use rubenrubiob\Domain\DTO\Llibre\LlibreDTO;
use rubenrubiob\Infrastructure\QueryBus\QueryBus;
use Symfony\Component\HttpFoundation\JsonResponse;
use Symfony\Component\HttpFoundation\Response;

use function array_map;

final readonly class FindLlibresController
{
    public function __construct(private QueryBus $queryBus)
    {
    }

    public function __invoke(): Response
    {
        /** @var list<LlibreDTO> $llibres */
        $llibres = $this->queryBus->__invoke(
            new FindLlibreDTOsQuery()
        );

        $formattedResponse = array_map(
            fn(LlibreDTO $llibreDTO): array =>
            [
                'id' => $llibreDTO->llibreId->toString(),
                'titol' => $llibreDTO->llibreTitol->toString(),
                'autor' => $llibreDTO->autorNom->toString(),
            ],
            $llibres,
        );

        return new JsonResponse(
            $formattedResponse,
            Response::HTTP_OK,
        );
    }
}
```

Veiem que, en ambdós casos, el que fem és formatar un `LlibreDTO`, en el primer cas, directament, i, en el segon, dins
d'un `array`.

Ara bé, suposem que hem d'afegir un nou camp al `LlibreDTO` que hem de retornar a totes les respostes on s'empra. Quin
problema ens trobem? Que l'hem d'afegir a diversos cops, tants com controladors on es faci servir. Què passa si oblidem
afegir el camp a un _endpoint_? Que la nostra API retornarà respostes diferents per al mateix element.

Un bon disseny d'API requereix donar respostes uniformes, és a dir, que els elements es representin de la mateixa manera
arreu. Per a aconseguir-ho, cal treballar molt el projecte amb gent que el conegui, veure les dades existents i els
requisits del domini. És una feina de paciència. Un estàndard com OpenAPI ens pot ajudar en aquesta tasca de
definició[^1].

Pel que fa a la implementació de l'_endpoint_, igual que vam fer amb les excepcions
al [post anterior](https://rubenrubiob.github.io/dev/2023-04-controladors-nets-a-symfony-i-gestio-excepcions/){:target="_blank"}, podem
delegar el formatat de les respostes al _framework_.

## `kernel.view` esdeveniment

Tal com vam explicar, el `Kernel` de Symfony fa servir esdeveniments en què el desenvolupador pot fer accions:

![Font: documentació de Symfony](/images/dev/symfony-kernel-events.png)

Veiem que hi ha dos punts que s'executen després del controlador:
- El punt 5, si el controlador retorna un objecte `Response`.
- El punt 6, si el controlador no retorna un objecte `Response`. S'executa l'esdeveniment `view`.

Consultant
la [documentació de Symfony](https://symfony.com/doc/current/components/http_kernel.html#6-the-kernel-view-event){:target="_blank"}, veiem que aquest esdeveniment ens permet processar el valor de resposta d'un controlador per a obtenir
un objecte `Response`:

> _If the controller doesn't return a `Response` object, then the kernel dispatches another event - `kernel.view`. The job of a listener to this event is to use the return value of the controller (e.g. an array of data or an object) to create a `Response`._

Així doncs, igual que vam fer amb les excepcions, podem delegar i centralitzar la presentació de les dades que retorna
el controlador a aquest esdeveniment.

## Implementació

Per a convertir les dades que retorna el controlador farem servir un serialitzador. El serialitzador s'encarregarà de
cridar un servei que anomenarem `Presenter`, que serà el que formatarà els nostres objectes.

### `LlibreDTOPresenter`

Aquesta classe es limita a rebre un `LlibreDTO` i a formatar-lo. És el servei que centralitza la presentació del nostre
objecte de domini.

Per a cada element del nostre domini que presentem, haurem de crear un nou `Presenter`.

En aquest cas, cal tenir en compte que aquest `Presenter` és per a JSON. Si volguéssim presentar l'element en algun
altre format, necessitaríem un altre `Presenter`.

```php
<?php

declare(strict_types=1);

namespace rubenrubiob\Infrastructure\Ui\Http\Response\Presenter;

use rubenrubiob\Domain\DTO\Llibre\LlibreDTO;

final readonly class LlibreDTOPresenter
{
    /** @return array{
     *     id: non-empty-string,
     *     titol: non-empty-string,
     *     autor: non-empty-string
     * }
     */
    public function __invoke(LlibreDTO $llibreDTO): array
    {
        return [
            'id' => $llibreDTO->llibreId->toString(),
            'titol' => $llibreDTO->llibreTitol->toString(),
            'autor' => $llibreDTO->autorNom->toString(),
        ];
    }
}

```

### Symfony Serializer

Per a gestionar la presentació de dades a l'esdeveniment `kernel.view` emprarem
el [Serializer de Symfony](https://symfony.com/doc/current/components/serializer.html){:target="_blank"}. Aquest serialitzador fa
servir [normalitzadors](https://symfony.com/doc/current/components/serializer.html#normalizers){:target="_blank"} per a convertir objectes
a `array`, i viceversa. El serialitzador permet afegir nous normalitzadors.

Podem aprofitar aquest fet per a afegir un normalitzador per a passar el nostre `LlibreDTO` a JSON. Seria com aquest[^2]:

```php
<?php

declare(strict_types=1);

namespace rubenrubiob\Infrastructure\Ui\Http\Response\Serializer;

use rubenrubiob\Domain\DTO\Llibre\LlibreDTO;
use rubenrubiob\Infrastructure\Ui\Http\Response\Presenter\LlibreDTOPresenter;
use Symfony\Component\Serializer\Normalizer\NormalizerInterface;

use function assert;

final readonly class LlibreDTOJsonNormalizer implements NormalizerInterface
{
    private const FORMAT = 'json';

    public function __construct(
        private LlibreDTOPresenter $presenter
    ) {
    }

    /** @param array<array-key, mixed> $context */
    public function supportsNormalization(mixed $data, string $format = null, array $context = []): bool
    {
        return $format === self::FORMAT && $data instanceof LlibreDTO;
    }

    /**
     * @param array<array-key, mixed> $context
     *
     * @return array{
     *     'id': non-empty-string,
     *     'titol': non-empty-string,
     *     'autor': non-empty-string
     * }
     */
    public function normalize(mixed $object, string $format = null, array $context = []): array
    {
        assert($object instanceof LlibreDTO);

        return $this->presenter->__invoke($object);
    }
}


```

Per a cada objecte que vulguem presentar ens caldrà afegir un normalitzador. Per a evitar això, podríem implementar un
_factory_ que contingui tots els `Presenter` i fer-lo servir als dos mètodes del normalitzador.

Si fem servir l’autoconfiguració per a definir els serveis de Symfony, el nostre normalitzador ja estarà automàticament
afegit al serialitzador per defecte que incorpora el _framework_.

### `ViewResponseSubscriber`

Amb el `Presenter` i el serialitzador ja configurats, podem implementar el `Subscriber` que gestiona
l'esdeveniment `kernel.view`:

```php
<?php

declare(strict_types=1);

namespace rubenrubiob\Infrastructure\Symfony\Subscriber;

use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\HttpKernel\Event\ViewEvent;
use Symfony\Component\HttpKernel\KernelEvents;
use Symfony\Component\Serializer\SerializerInterface;

final readonly class ViewResponseSubscriber implements EventSubscriberInterface
{
    private const HEADERS = ['Content-Type' => 'application/json'];
    private const FORMAT = 'json';

    public function __construct(
        private SerializerInterface $serializer
    ) {
    }

    public static function getSubscribedEvents(): array
    {
        return [KernelEvents::VIEW => '__invoke'];
    }

    public function __invoke(ViewEvent $event): void
    {
        $controllerResult = $event->getControllerResult();

        if ($controllerResult === null) {
            $event->setResponse(
                new Response(
                    null,
                    Response::HTTP_NO_CONTENT,
                    self::HEADERS,
                )
            );

            return;
        }

        $response = new Response(
            $this->serializer->serialize(
                $event->getControllerResult(),
                self::FORMAT,
            ),
            Response::HTTP_OK,
            self::HEADERS,
        );

        $event->setResponse($response);
    }
}

```

El que fem és establir com a contingut de la resposta el valor serialitzat que retorna el controlador.

Cal tenir en compte que és una implementació un xic simplista: sempre s'estableix el codi de resposta HTTP 200 si el
controlador retorna un valor diferent de `null`; si no, retorna un 204, de `No Content`. Si calgués retornar altres
codis, com el 201 de `Created`, caldria afegir lògica per a controlar-ho.

També cal tenir en compte que el format de resposta és sempre JSON. En el futur veurem com fer flexible també aquest
format de resposta.

Si fem servir l’autoconfiguració per a definir els serveis de Symfony, el nostre `Subscriber` ja estarà automàticament
configurat a l’aplicació.

## Controladors simplificats

Amb el `Subscriber` complet i funcionant, ja podem simplificar els nostres controladors.

El controlador que retorna un únic llibre queda així:

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
        return $this->queryBus->__invoke(
            new GetLlibreDTOByIdQuery(
                $llibreId
            )
        );
    }
}
```

El controlador que retorna un llistat de llibres queda així:

```php
<?php

declare(strict_types=1);

namespace rubenrubiob\Infrastructure\Ui\Http\Controller;

use rubenrubiob\Application\Query\Llibre\FindLlibreDTOsQuery;
use rubenrubiob\Domain\DTO\Llibre\LlibreDTO;
use rubenrubiob\Infrastructure\QueryBus\QueryBus;

final readonly class FindLlibresController
{
    public function __construct(private QueryBus $queryBus)
    {
    }

    /** @return list<LlibreDTO> */
    public function __invoke(): array
    {
        return $this->queryBus->__invoke(
            new FindLlibreDTOsQuery()
        );
    }
}
```

Veiem que als controladors es limiten a cridar i retornar el _QueryBus_, donat que la gestió tant dels errors com del
formatat de la resposta s'han delegat a esdeveniments del _framework_.

## Conclusions

Delegant la gestió de les respostes al _framwork_ aconseguim centralitzar el formatat de respostes. Si cal afegir un
camp nou, amb afegir-lo en un únic lloc, el `Presenter`, n'hi haurà prou perquè s'actualitzi a tots els controladors.
Cal tenir en compte, això sí, que caldrà actualitzar els tests funcionals que hi hagi per tenir en compte aquest canvi.

A més a més, juntament amb la gestió d'excepcions que vam veure al post anterior, hem simplificat els nostres
controladors perquè tinguin la mínima lògica possible: rebre una petició i retornar una resposta, res més.

Aquesta solució no impossibilita retornar una resposta directament des del controlador, ja que
l'esdeveniment `kernel.view` només s'executa quan el controlador no retorna un objecte `Response`.

## Resum

- Hem vist el problema que suposa no tenir el formatat de resposta centralitzat.
- Hem introduït els `Presenters` per a formatar els nostres objectes de domini.
- Hem configurat el serialitzador de Syfmony amb un normalitzador per a presentar els nostres objectes.
- Hem creat un `Subscriber` per a l’esdeveniment `kernel.view` per a gestionar les respostes d’una API de manera centralitzada.
- Hem simplificat els nostres controladors perquè tinguin la mínima lògica possible.

[^1]: Per a bones pràctiques en el disseny d'API, vegeu per exemple [aquest article de Swagger](https://swagger.io/resources/articles/best-practices-in-api-design/){:target="_blank"} o [aquest altre d'Stack Overfow](https://stackoverflow.blog/2020/03/02/best-practices-for-rest-api-design/){:target="_blank"}.

[^2]: A Symfony 5.4 la interfície `NormalizerInterface` varia una mica. Es pot fer servir en el seu lloc `ContextAwareNormalizer`. 