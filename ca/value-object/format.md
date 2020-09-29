---
lang: ca
lang-ref: value-object-formato
tags: [domain,value-object,format]
is-content-page: true
layout: content-page
order: 6
pull-request: 7
---

# _Value Object_: Formato

* TOC
{:toc}

## Introducció

El darrer[^1] _Value Object_ que explicarem és el que representa el format d'una edició d'un llibre.

Un llibre físic és un objecte que amb prou feines ha variat en cinc segles, així que podem assegurar que la probabilitat d'aparició de nous formats és molt baixa. Si calgués, podríem afegir nous formats fàcilment, amb un impacte mínim sobre el codi: només caldria afegir un nou valor a la llista de formats disponibles de la classe.

## _Namespace_

En tractar-se d'una cosa específica de l'edició d'un llibre, situarem el _Value Object_ al _namespace_ `Domain\ValueObject\Libro\Edicion`.

## Implementació

Una possible implementació seria la següent:

```php
<?php

declare(strict_types=1);

namespace Domain\ValueObject\Libro\Edicion;

use Domain\Exception\ValueObject\Libro\Edicion\FormatoIsNotValid;
use InvalidArgumentException;
use Webmozart\Assert\Assert;

use function strtolower;
use function trim;

/**
 * @psalm-immutable
 */
final class Formato
{
    public const MANUSCRITO  = 'manuscrito';
    public const INCUNABLE   = 'incunable';
    public const TAPA_BLANDA = 'tapa_blanda';
    public const TAPA_DURA   = 'tapa_dura';

    private const VALID_CODIGOS = [
        self::MANUSCRITO  => self::MANUSCRITO,
        self::INCUNABLE   => self::INCUNABLE,
        self::TAPA_BLANDA => self::TAPA_BLANDA,
        self::TAPA_DURA => self::TAPA_DURA,
    ];

    private string $codigo;

    private function __construct(string $codigo)
    {
        $this->codigo = $codigo;
    }

    /**
     * @throws FormatoIsNotValid
     */
    public static function fromCodigo(string $codigo): self
    {
        return new self(
            self::parseAndValidateCodigo($codigo)
        );
    }

    public function asString(): string
    {
        return $this->codigo;
    }

    public function equalsTo(Formato $anotherFormato): bool
    {
        return $this->codigo === $anotherFormato->codigo;
    }

    /**
     * @throws FormatoIsNotValid
     */
    private static function parseAndValidateCodigo(string $input): string
    {
        $codigo = strtolower(
            trim(
                $input
            )
        );

        try {
            Assert::keyExists(self::VALID_CODIGOS, $codigo);
        } catch (InvalidArgumentException $e) {
            throw FormatoIsNotValid::withCodigo($input);
        }

        return $codigo;
    }
}

```

- Només tenim un _named constructor_, que _parseja_ i valida que el codi sigui correcte. D'aquesta manera podem passar el codi independentment de majúscules i minúscules i funcionaria igualment. Tot i això, sempre és preferible fer servir les constants de la classe per a referir-se als diferents formats.
- Per a la validació, tenim un _map_ intern que comprova que el valor d'entrada sigui un dels vàlids.
- Observem que la implementació d'aquest _Value Object_ és similar a la del _Value Object_ `Locale`, i és correcte que sigui així. Però no hem de pensar a abstreure el codi a una classe comuna seguint el principi DRY —_Don't Repeat Yourself_—, perquè [aquest principi s'aplica al coneixement, no pas al codi](https://twitter.com/macerub/status/1309873361233866753?s=20). Cada _Value Object_ representa un coneixement diferent, així que seria una mala decisió abstreure'n el codi.

### Test

Al test simplement comprovem tots els codis disponibles. Pot consultar-se al _pull request_.


[^1]: Hem implementat un _Value Object_ més, per a representar el títol d'un llibre, `Titulo`. Això no obstant, en tractar-se d'un _Value Object_ senzill, la seva explicació no aporta res de nou a tot el que ja hem explicat. El seu codi pot consultar-se al [_pull request_ corresponent](https://github.com/rubenrubiob/gestor-libros/pull/8).