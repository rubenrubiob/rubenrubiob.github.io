---
date: 2023-05-16
---

# Testejar l'especificació d'OpenAPI a PHP
{:.no_toc}

* TOC
{:toc}


## Introducció

[OpenAPI](https://www.openapis.org/){:target="_blank"} ha esdevingut l'estàndard _de facto_ d'especificació d'
API: [es tracta un llenguatge d'especificació per a API que defineix una estructura i sintaxi no lligada al llenguatge de programació en què s'escriu l'API](https://www.openapis.org/what-is-openapi){:
target="_blank"}. És a dir, l'especificació és un contracte independent del llenguatge de programació. Així doncs,
aquesta especificació pot ser consultada per qualsevol client de l'API i saber com integrar-se. L'especificació s'escriu
en YAML o JSON.

Ara bé, avui dia no hem d'escriure l'especificació manualment, sinó que hi
ha [interfícies gràfiques que ens permeten fer-ho](https://openapi.tools/#gui-editors){:target="_blank"}. A continuació
es mostren un parell d'imatges [d'Stoplight](https://stoplight.io/) amb exemples de tots els _endpoints_ d'una API i la
definició d'un d'ells:  

![Vista de tots els _endpoints_ a Stoplight](/images/dev/open-api-testing/open-api-testing-1.png)

![Vista d'un _endpoint_ a Stoplight](/images/dev/open-api-testing/open-api-testing-2.png)

Així doncs, si primer dissenyem l'especificació de la nostra API, el que hem de fer és implementar els _endpoints_ de
manera que compleixin aquesta especificació. Ara bé, com podem assegurar-nos que complim l'especificació?

Algú podria argumentar que es tracta d'una tasca simple de dur a terme: només cal seguir el que s'ha especificat. Però,
i si en algun moment fem una refactorització que té efectes col·laterals? En aquest cas, podríem trencar el contracte de
l'API, cosa que pot fer que els nostres clients deixin de funcionar.

La solució consisteix a testejar les respostes de la nostra API contra l'especificació que hem definit. Podem aprofitar
els tests funcionals de la nostra API per a validar les respostes contra l'especificació d'OpenAPI. Si no tenim tests
funcionals, aquest podria ser un bon moment per a afegir-los, encara que l'única cosa que validin sigui l'especificació
d'OpenAPI.

En aquest post veurem com testejar una especificació d'OpenAPI amb utilitats de testing de Symfony.

## Implementació

### Llibreries

Existeix un paquet de The PHP League que permet validar l'especificació d'
OpenAPI: [`league/openapi-psr7-validator`](https://github.com/thephpleague/openapi-psr7-validator){:target="_blank"}.
Aquest paquet valida peticions i respostes
de [l'especificació PSR-7, que defineix interfícies dels missatges HTTP](https://www.php-fig.org/psr/psr-7/){:target="_
blank"}.

Això no obstant, el fluxe de peticions de Symfony empra el
component [HTTP Foundation](https://symfony.com/doc/current/components/http_foundation.html){:target="_blank"}, que no
implementa PSR-7. Per tant, necessitem un paquet extra que permeti convertir les peticions i respostes de Symfony a i de
PSR-7: [`symfony/psr-http-message-bridge`](https://symfony.com/doc/current/components/psr7.html){:target="_blank"}.

Com indica la documentació, aquest paquet només serveix per a la conversió, però ens caldrà també una implementació de
PSR-7 i PSR-17 per a convertir els objectes a i de PSR-7. Podem fer servir la llibreria que recomana la
documentació, [`nyholm/psr7`](https://github.com/Nyholm/psr7){:target="_blank"}, però
n'hi [ha d'altres](https://packagist.org/providers/psr/http-factory-implementation){:target="_blank"}.

Així doncs, instal·lem aquests tres paquets com a dependències de desenvolupament al nostre projecte:

```bash
composer require —dev league/openapi-psr7-validator nyholm/psr7 
symfony/psr-http-message-bridge
```

### `OpenApiResponseAssert`

El que farem serà implementar un servei que, donades una `Response` de Symfony, una ruta i un mètode de petició, validi
la resposta contra l'especificació d'OpenAPI.

Per a simplificar els tests, farem la conversió de `Response` a PSR-7 dins d'aquest servei. Així els tests quedaran nets
i només tindran la lògica del que estan testejant.

La implementació és la següent:

```php
<?php

declare(strict_types=1);

namespace rubenrubiob\Tests\Common\Validation\OpenApi;

use League\OpenAPIValidation\PSR7\Exception\Validation\AddressValidationFailed;
use League\OpenAPIValidation\PSR7\Exception\ValidationFailed;
use League\OpenAPIValidation\PSR7\OperationAddress;
use League\OpenAPIValidation\PSR7\ResponseValidator;
use League\OpenAPIValidation\PSR7\ValidatorBuilder;
use Nyholm\Psr7\Factory\Psr17Factory;
use Symfony\Bridge\PsrHttpMessage\Factory\PsrHttpFactory;
use Symfony\Component\HttpFoundation\Response;

use function strtolower;

final readonly class OpenApiResponseAssert
{
    private PsrHttpFactory $psrHttpFactory;
    private ResponseValidator $validator;

    public function __construct()
    {
        $psr17Factory = new Psr17Factory();
        $this->psrHttpFactory = new PsrHttpFactory($psr17Factory, $psr17Factory, $psr17Factory, $psr17Factory);
        $this->validator = (new ValidatorBuilder())
            ->fromYamlFile(__DIR__ . '/../../../../specs/reference/blog-src.yaml')
            ->getResponseValidator();
    }

    /** @throws ValidationFailed */
    public function __invoke(Response $response, string $route, string $method): void
    {
        $psrResponse = $this->psrHttpFactory->createResponse($response);
        $operation = new OperationAddress($route, strtolower($method));

        try {
            $this->validator->validate($operation, $psrResponse);
        } catch (AddressValidationFailed $e) {
            $class = $e::class;

            throw new $class(
                $e->getVerboseMessage(),
                $e->getCode(),
                $e,
            );
        }
    }
}
```

Al constructor creem el validador d'OpenAPI a partir de la ruta del fitxer d'especificació. No és bona pràctica escriure
lògica dins d'un constructor, però, en aquest cas, és una classe només per a _testing_, no és codi de producció.

Al mètode `__invoke` és on:

- Fem la conversió del `Response` de Symfony a PSR-7.
- Validem aquesta `Response` contra l'especificació d'OpenAPI per a la ruta i el mètode especificats.
- Si no es compleix l'especificació, es llença una excepció.

### Test funcional

Amb el servei de validació, ja podem escriure el test funcional que valida una resposta contra l'especificació d'
OpenAPI.

Anomenem test funcional a realitzar una petició amb tot l'_stack_, incloent-hi la base de dades i qualsevol servei que
necessitem. Farem
servir [`WebTestCase` de Symfony](https://symfony.com/doc/current/testing.html#write-your-first-application-test){:
target="_blank"}, que permet simular un servidor web. Això ens permet fer servir un client per a fer peticions, la
classe `Symfony\Bundle\FrameworkBundle\KernelBrowser`.

Escriurem un test per a la petició d'obtenir un llibre que vam veure
en [un post anterior](/dev/2023-04-controladors-nets-a-symfony-i-gestio-excepcions/){:target="_blank"}:

![Vista de l'_endpoint_ a Stoplight](/images/dev/open-api-testing/open-api-testing-3.png)

La implementació del test és la següent:

```php
<?php

declare(strict_types=1);

namespace rubenrubiob\Tests\Functional\Llibre;

use rubenrubiob\Tests\Common\Validation\OpenApi\OpenApiResponseAssert;
use rubenrubiob\Tests\Functional\FunctionalBaseTestCase;
use Symfony\Bundle\FrameworkBundle\KernelBrowser;
use Symfony\Bundle\FrameworkBundle\Test\WebTestCase;
use Symfony\Component\HttpFoundation\Response;

use function Safe\json_decode;
use function sprintf;

final class GetLlibreTest extends WebTestCase
{
    private const EXISTING_LLIBRE_ID = '080343dc-cb7c-497a-ac4d-a3190c05e323';
    private const NON_EXISTING_LLIBRE_ID = '4bb201e9-2c07-4a3a-b423-eca329d2f081';
    private const INVALID_LLIBRE_ID = 'foo';
    private const EMPTY_LLIBRE_ID = ' ';

    private const REQUEST_METHOD = 'GET';
    
    private readonly KernelBrowser $client;
    private readonly OpenApiResponseAssert $openApiResponseAssert;
    
    protected function setUp(): void
    {
        parent::setUp();

        $this->client = static::createClient();
        $this->openApiResponseAssert = new OpenApiResponseAssert();
    }

    public function test_amb_non_existing_llibre_retorna_404(): void
    {
        $url = $this->url(self::NON_EXISTING_LLIBRE_ID);

        $this->client->request(
            self::REQUEST_METHOD,
            $url,
        );

        $response = $this->client->getResponse();

        self::assertSame(Response::HTTP_NOT_FOUND, $response->getStatusCode());

        $this->openApiResponseAssert->__invoke($response, $url, self::REQUEST_METHOD);
    }

    public function test_amb_llibre_retorna_resposta_valida(): void
    {
        $url = $this->url(self::EXISTING_LLIBRE_ID);

        $this->client->request(
            self::REQUEST_METHOD,
            $url,
        );

        $response = $this->client->getResponse();
        $responseContent = json_decode($this->client->getResponse()->getContent(), true);

        self::assertSame(Response::HTTP_OK, $response->getStatusCode());
        self::assertEquals(
            [
                'id' => self::EXISTING_LLIBRE_ID,
                'titol' => 'Curial e Güelfa',
                'autor' => 'Anònim',
            ],
            $responseContent,
        );

        $this->openApiResponseAssert->__invoke($response, $url, self::REQUEST_METHOD);
    }

    private function url(string $llibreId): string
    {
        return sprintf(
            '/llibres/%s',
            $llibreId
        );
    }
}

```

Veiem que tenim dos tests:
- Un per validar la resposta amb codi HTTP `404`.
- Un altre per a validar la resposta correcta, amb codi HTTP `200`.
- En ambdós casos, validem la resposta contra la validació d'OpenAPI amb la següent línia:

```php
$this->openApiResponseAssert->__invoke($response, $url, self::REQUEST_METHOD);
```

Cal tenir en compte que en aquest exemple caldria testejar també les respostes amb codi HTTP `400`. Tampoc no es mostra
en aquest exemple com persistir les dades per a poder fer-ne la petició.

### `FunctionalBaseTestCase`

Per a simplificar la resta de tests, podem crear un `TestCase` del qual estenguin tots els nostres tests funcionals:

```php
<?php

declare(strict_types=1);

namespace rubenrubiob\Tests\Functional;

use rubenrubiob\Tests\Common\Validation\OpenApi\OpenApiResponseAssert;
use Symfony\Bundle\FrameworkBundle\KernelBrowser;
use Symfony\Bundle\FrameworkBundle\Test\WebTestCase;

abstract class FunctionalBaseTestCase extends WebTestCase
{
    protected readonly KernelBrowser $client;
    protected readonly OpenApiResponseAssert $openApiResponseAssert;

    protected function setUp(): void
    {
        parent::setUp();

        $this->client = static::createClient();
        $this->openApiResponseAssert = new OpenApiResponseAssert();
    }
}

```

## Conclusions

Amb aquest testeig, aconseguirem lligar la nostra implementació a l'especificació. Integrant aquesta validació als
nostres tests funcionals, sempre complirem el contracte definit per a la nostra API.

Recordem que, abans de tot, cal dissenyar l'API. No vol dir fer-ne l'especificació en OpenAPI,
sinó [dissenyar-la per a resoldre els problemes del nostre domini](https://apidesignmatters.substack.com/p/api-design-first-is-not-api-design){:
target="_blank"}.

## Resum

- Hem vist en què consisteix l'estàndard OpenAPI i com escriure'n especificacions.
- Hem revisat la problemàtica que suposa no validar la nostra implementació contra l'especificació de l'API.
- Hem llistat les llibreries que ens permeten testejar una `Response` de Symfony contra l'especificació d'OpenAPI, convertint-la a l'estàndard PSR-7.
- Hem implementat un servei que rep una `Response` de Symfony, la converteix a PSR-7 i la valida contra l'especificació d'OpenAPI.
- Hem emprat aquest servei en els nostres tests funcionals, i n'hem generat una classe base per a fer-la servir a la resta de tests.