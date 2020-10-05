---
lang: es
lang-ref: value-object-formato
tags: [domain,value-object,formato]
is-content-page: true
layout: content-page
order: 6
pull-request: 7
---

# _Value Object_: Formato

* TOC
{:toc}

## Introducción

El último[^1] _Value Object_ que explicaremos es el que representa el formato de una edición de un libro.

Un libro físico es un objeto que apenas ha variado en cinco siglos, así que podemos asegurar que la probabilidad de aparición de nuevos formatos es muy baja. Si hiciese falta podríamos añadir nuevos formatos fácilmente, con un impacto mínimo en el código: sólo habría que añadir un nuevo valor a la lista de formatos disponibles de la clase.

## _Namespace_

Al tratarse de algo específico de la edición de un libro, situaremos el _Value Object_ en el _namespace_ `Domain\ValueObject\Libro\Edicion`.

## Implementación

Una posible implementación para el _Value Object_ sería la siguiente:

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

- Sólo tenemos un _named constructor_, que _parsea_ y válida que el código sea correcto. Así, podemos pasar el código independientemente en mayúsculas y minúsculas y funcionaría igualmente. Aunque siempre es preferible usar las constantes de la clase para referirse a los diferentes formatos.
- Para la validación, tenemos un _map_ interno que comprueba que el valor de entrada sea uno de los válidos.
- Vemos que la implementación de este _Value Object_ es similar a la del _Value Object_ `Locale`, y es correcto que sea así. Pero no hemos de pensar en abstraer el código a una clase común siguiendo el principio DRY —_Don't Repeat Yourself_—, porque [este principio se aplica a conocimiento, no a código](https://twitter.com/macerub/status/1309873361233866753?s=20). Cada _Value Object_ representa un conocimiento diferente, así que sería una mala decisión asbtraer su código.

### Test

En el test simplemente comprobamos todos los códigos disponibles. Puede consultarse en el _pull request_.


[^1]: Hemos implementado un _Value Object_ más, para representar el título de un libro, `Titulo`. Sin embargo, al tratarse de un _Value Object_ sencillo, su explicación no aporta nada nuevo a todo lo que ya hemos explicado. Su código puede consultarse en el [_pull request_ correspondiente](https://github.com/rubenrubiob/gestor-libros/pull/8).