---
lang: es
lang-ref: value-object-formato
tags: [domain,value-object,formato,estado]
is-content-page: true
layout: content-page
order: 4
---

# _Value Object_: `Formato`

El último _Value Object_ que vamos a explicar es el que representa el formato de una edición de un libro.

Un libro físico es un objeto que apenas ha variado en cinco siglos, así que podemos asegurar que la aparición de nuevos formatos es muy reducida. Como en toda nuestra aplicación, usaremos este _Value Object_ para tratar con el formato de una edición de un libro, si hiciese falta podríamos añadir nuevos formatos fácilmente, con un impacto mínimo en el código: sólo habría que añadir un nuevo valor a la lista de formatos disponibles de la clase.

## _Namespace_

Al tratarse de algo específico de los libros, pertenece al _namespace_ `Domain\ValueObject\Libro\Edicion`. Seguiremos esta regla para todos los _Value Object_: si son genéricos para toda la aplicación, como el `Id`, irán directamente en el _namespace_ `Domain\ValueObject`; si no, irán al _namespace_ equivalente de la entidad a la que pertenecen.

Así, por ejemplo, si en nuestra aplicación tuviéramos productos con su propio estado asociado y pedidos con su propio estado asociado, podríamos tener las siguientes clases:

- `Domain\ValueObject\Producto\Estado`
- `Domain\ValueObject\Pedido\Estado`

Y si se diese el caso de que tuviéramos que usar ambas clases en una porción del código, podríamos importar los _namespace_ como alias:

```php
<?php
// ...

use Domain\ValueObject\Pedido\Estado as EstadoPedido;
use Domain\ValueObject\Producto\Estado as EstadoProducto;

// ...

public function doSomethingWithPedido(EstadoPedido $estado)
{
    // ...
}

public function doSomethingWithProducto(EstadoProducto $estado)
{
    // ...
}
```


- Valores cerrados -> similar a estado -> pueden tener comportamientos diferentes (ebook puede tener enlace)

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
- Para validar, tenemos un _map_ interno para comprobar si el valor que nos han pasado corresponde a alguno de los válidos.