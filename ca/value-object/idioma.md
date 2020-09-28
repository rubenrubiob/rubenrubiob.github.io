---
lang: ca
lang-ref: value-object-idioma
tags: [domain,value-object,codi,idioma,locale]
is-content-page: true
layout: content-page
order: 5
pull-request: 6
---

# _Value Object_: Locale i CodigoIdioma

* TOC
{:toc}

## Introducció

A la nostra aplicació hem de poder treballar amb idiomes: la llengua o llengües de publicació d'una edició d'un llibre, les traduccions dels noms d'autors en diversos idiomes, la mateixa interfície.... Així doncs, hem de desenvolupar _Value Objects_ que representin idiomes.

Això no obstant, com que els mateixos noms dels idiomes són traduïbles, en lloc del nom de l'idioma, el que farem serà més aviat utilitzar codis per a identificar cada idioma. Com és lògic, no reinventarem la roda i crearem un llistat de codis d'idioma propi, sinó que farem servir l'estàndard [ISO-639](https://ca.wikipedia.org/wiki/ISO_639), que s'encarrega de regular la representació de noms d'idioma. Ara bé, l'estàndard té diverses seccions; les que es refereixen al codi d'idioma són les dues primeres:

- [ISO-639-1](https://ca.wikipedia.org/wiki/ISO_639-1): es tracta d'un codi de dues lletres, com per exemple `es` per a castellà, `en` per a anglès o `ca` per a català. Conté idiomes actuals, i és la codificació que acostuma a fer-se servir a pàgines web per a identificar l'idioma, com `es.wikipedia.org` o `ca.wikipedia.org`.
- [ISO-639-2](https://en.wikipedia.org/wiki/ISO_639-2): es tracta d'un codi de tres lletres, com `spa` per a castellà, `eng` per a anglès o `cat` per a català. Va sorgir per a superar la limitació dels codis ISO-639-1, que no permetien representar tants idiomes. Aquesta secció conté també codis per a llengües mortes, com el llatí o el grec clàssic.

A la nostra aplicació necessitarem ambdues codificacions, de manera que implementarem dos _Value Object_: `Locale` i `CodigoIdioma`. A continuació en detallem la seva explicació.

## _Value Object_: Locale

Aquest _Value Object_ servirà per a identificar l'idioma de l'aplicació web. Tal com es va [definir a la introducció](https://rubenrubiob.github.io/ca/#definició), el web només estarà disponible en tres idiomes: català, castellà i anglès.

Algú podria argumentar que l'idioma no pertany a la capa de Domini, sinó a la d'Infraestructura, ja que només afecta a la interfície de visualització del web. I aquesta afirmació seria certa. Això no obstant, es va definir que els models fossin traduïbles, com per exemple el nom d'un autor. Així doncs, també necessitarem aquest _Value Object_ a la capa de Domini.

En resum, utilitzarem el _Value Object_ `Locale` tant a la capa de Domini, per a la traducció dels models, com a la d'Infraestructura, per a la interfície d'usuari i les URL. És així perquè així es defineix des de la part de negoci, és a dir, ho dicta el Domini.

En aquest cas, segons està especificat, limitarem els codis possibles als tres definits, per a català, castellà i anglès, en el format ISO-639-1: `ca`, `es` i `en`, respectivament. Si en algun moment el Domini canviés i volguéssim acceptar algun altre idioma, caldria modificar el _Value Object_.

### _Namespace_

Agruparem tots els _Value Object_ corresponents a idioma, de manera que situarem `Locale` _namespace_ `Domain\ValueObject\Idioma`.

### Implementació

La implementació del _Value Object_ és la següent:

```php
<?php

declare(strict_types=1);

namespace Domain\ValueObject\Idioma;

use Domain\Exception\ValueObject\Idioma\LocaleIsNotValid;
use InvalidArgumentException;
use Webmozart\Assert\Assert;

use function strtolower;
use function trim;

/**
 * @psalm-immutable
 */
final class Locale
{
    public const CA = 'ca';
    public const ES = 'es';
    public const EN = 'en';

    public const LOCALE_MAP = [
        self::CA => self::CA,
        self::ES => self::ES,
        self::EN => self::EN,
    ];

    private string $codigo;

    private function __construct(string $codigo)
    {
        $this->codigo = $codigo;
    }

    public static function fromCodigo(string $codigo): self
    {
        return new self(
            self::parseAndValidateCodigo($codigo)
        );
    }

    public static function ca(): self
    {
        return new self(self::CA);
    }

    public static function es(): self
    {
        return new self(self::ES);
    }

    public static function en(): self
    {
        return new self(self::EN);
    }

    public function asString(): string
    {
        return $this->codigo;
    }

    public function equalsTo(Locale $anotherLocale): bool
    {
        return $this->codigo === $anotherLocale->codigo;
    }

    /**
     * @throws LocaleIsNotValid
     */
    private static function parseAndValidateCodigo(string $codigo): string
    {
        $parsedCodigo = strtolower(
            trim(
                $codigo
            )
        );

        try {
            Assert::keyExists(self::LOCALE_MAP, $parsedCodigo);
        } catch (InvalidArgumentException $e) {
            throw LocaleIsNotValid::withCodigo($codigo);
        }

        return $parsedCodigo;
    }
}

```

- Com que ve definit des del Domini, només acceptarem tres codis d'idioma, `ca`, `es` i `en`, que definim com a constants dins de la classe.
- Per a simplificar la construcció de l'objecte als tests o en casos per defecte, tindrem un _named constructor_ per a cada idioma del Domini.
- Per a la validació, netegem d'espais i convertim a minúscules l'_string_ d'entrada i simplement mirem que el codi sigui un dels possibles. Per fer-ho, comprovem que el codi existeixi en una de les claus de l'_array_ declarat a la constant `LOCALE_MAP`.
- Totes les constants les declarem com a públiques: `LOCALE_MAP` per a fer-la servir a l'excepció `LocaleIsNotValid`, i, la resta, pels tests.

### Test

A causa de la seva longitud, no reproduïm el test aquí. Es pot consultar al _pull request_.

## _Value Object_: CodigoIdioma

A més a més de l'idioma de l'aplicació, necessitarem identificar l'idioma en què es va publicar l'edició d'un llibre. En aquest cas, podem trobar-nos amb llibres publicats en llengües mortes, com el llatí o el grec, de manera que no podem utilitzar l'estàndard ISO-639-1 com hem fet al _Value Oject_ `Locale`, i haurem de fer servir l'estàndard ISO-639-2, que conté codis d'idioma de moltes més llengües, incloent-hi idiomes en desús.

Per a simplificar l'explicació, per ara no tindrem en compte les edicions de llibres en diversos idiomes, com per exemple les obres bilingües o trilingües. És una funcionalitat que podrem afegir al futur.

### _Namespace_

En tant que específic de les edicions d'un llibre, podríem situar el _Value Object_ `CodigoIdioma` al _namespace_ `Domain\ValueObject\Libro\Edicion`. Això no obstant, tal com hem explicat al _Value Object_ `Locale`, agruparem els _Value Object_ relatius a la gestió d'idioma. Així doncs, l'ubicarem al _namespace_ `Domain\ValueObject\Idioma`.

### Implementació

La implementació del _Value Object_ és la següent:

```php
<?php

declare(strict_types=1);

namespace Domain\ValueObject\Idioma;

use Domain\Exception\ValueObject\Idioma\CodigoIdiomaIsNotValid;
use InvalidArgumentException;
use Webmozart\Assert\Assert;

use function strtolower;
use function trim;

/**
 * @psalm-immutable
 */
final class CodigoIdioma
{
    private const REGEX_CODIGO = '/^[a-z]{3}$/';
    private string $codigo;

    private function __construct(string $codigo)
    {
        $this->codigo = $codigo;
    }

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

    public function equalsTo(CodigoIdioma $anotherCodigoIdioma): bool
    {
        return $this->codigo === $anotherCodigoIdioma->codigo;
    }

    /**
     * @throws CodigoIdiomaIsNotValid
     */
    private static function parseAndValidateCodigo(string $codigo): string
    {
        $parsedCodigo = strtolower(
            trim(
                $codigo
            )
        );

        try {
            Assert::regex($parsedCodigo, self::REGEX_CODIGO);
        } catch (InvalidArgumentException $e) {
            throw CodigoIdiomaIsNotValid::withCodigo($codigo);
        }

        return $parsedCodigo;
    }
}

```

- A diferència del _Value Object_ `Locale`, com que contemplem l'ús de qualsevol idioma, no afegim tots els codis possibles al _Value Object_, ja que és massa complex i implicaria manterni-los. En el seu lloc, utilitzarem una interfície per a reconstruir i validar els codis d'idioma, amb l'ús de llibreries dedicades. Aquesta implementació la deixem per més endavant.
- Seguint el punt anterior, al _Value Object_ simplement validem amb una expressió regular que el codi d'entrada, net d'espais i convertit a minúscules, tingui tres lletres.

### Test

A causa de la seva longitud, no reproduïm el test aquí. Es pot consultar al _pull request_.