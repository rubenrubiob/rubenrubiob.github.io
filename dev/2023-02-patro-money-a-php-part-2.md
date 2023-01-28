---
date: 2023-02-06
---

# Patró _Money_ a PHP (part 2 de 2)
{:.no_toc}

* TOC
{:toc}

## Introducció

En el post previ (afegir enllaç) vam veure els problemes que suposava treballar amb decimals a causa de la pèrdua de
precisió de la representació dels nombres en punt flotant. Vam explicar que la solució per a treballar amb valors
monetaris consisteix a fer servir el patró moneda, que encapsula una quantitat amb una moneda dins d'un _Value Object_.

A PHP hi ha llibreries de codi obert d'aquest patró. En aquest post veurem una implementació per resoldre l'exemple que
vam mostrar en el post anterior.

## Implementació

En el cas que vam veure, el client establia el PVP i esperava obtenir-ne els següents valors:

| Preu net (€) | IVA (21%) (€) | Total (€) |
|:--|:--|:--|
| 4'55 | 0'95 | 5'50 |
| 4'55 | 0'95 | 5'50 |
| 4'55 | 0'95 | 5'50 |
| 4'55 | 0'95 | 5'50 |
| 4'55 | 0'95 | 5'50 |
| 4'38 | 0'92 | 5'30 |
| 4'38 | 0'92 | 5'30 |
| 4'38 | 0'92 | 5'30 |
| 4'38 | 0'92 | 5'30 |
| 4'38 | 0'92 | 5'30 |
| *44'65* | *9'35* | *54'00* |

El cas de l'import `5'50` és interessant, perquè mostra l'ambigüitat a la qual ens enfrontem.

Per a calcular l'import del 21% d'IVA a partir del preu final, fem la següent
operació: `ImportIva = PreuFinal - PreuFinal/1'21`. En aquest cas, n'obtenim: `0'954` que, arrodonit, és `0,95`.

En canvi, si calculem l'import de l'IVA a partir del preu net, fem la següent operació: `ImportIva = PreuNet·21/100`. En
aquest cas, n'obtenim `0'955` que, arrodonit, és `0'96`.

Així doncs, depenent del valor que fem servir per a calcular els impostos, obtindrem uns preus o uns altres. Per a
l'exemple que tractarem, el client volia que el preu final fos `5'50 €`, de manera que farem els càlculs a partir
d'aquest valor. Però això depèn de cada cas d'ús, de com estigui definit el domini i de quines dades disposem, no hi ha
una solució única. En menesters de diners, el millor és estar assessorat per experts.

A continuació veurem les parts més rellevants de les implementacions. Es poden veure senceres
al [repositori on hi ha tots els exemples](https://github.com/rubenrubiob/blog-src).

### `PercentatgeImpostos`

Per a fer els càlculs, necessitem els impostos. En aquest cas, els hem extret a un `enum` per a simplificar
l'explicació. Però podrien venir de la base de dades, per exemple. De nou, dependrem del cas d'ús de la nostra
aplicació.

```php
<?php

declare(strict_types=1);

namespace rubenrubiob\Domain\ValueObject;

enum PercentatgeImpostos: int
{
    case ES_IVA_21 = 21;
}

```

### `Import`

Podríem tenir un _Value Object_ per a l'import unitari com el següent:

```php
<?php

declare(strict_types=1);

namespace rubenrubiob\Domain\ValueObject;

use Brick\Math\BigDecimal;
use Brick\Math\Exception\NumberFormatException;
use Brick\Math\Exception\RoundingNecessaryException;
use Brick\Math\RoundingMode;
use Brick\Money\Exception\UnknownCurrencyException;
use Brick\Money\Money;
use rubenrubiob\Domain\Exception\ValueObject\ImportIsNotValid;

use function strtoupper;

final readonly class Import
{
    private function __construct(
        private int $preuNet,
        private PercentatgeImpostos $percentatgeImpostos,
        private int $quantitatImpostos,
        private int $preuFinal,
        private string $moneda,
    ) {
    }

    /**
     * @throws ImportIsNotValid
     */
    public static function ambPreuFinalAmbPercentatgeImpostos(
        float|int|string $preuFinal,
        PercentatgeImpostos $percentatgeImpostos,
        string $moneda,
    ): self {
        $preuFinalAsMoney = self::parseAndGetMoney($preuFinal, $moneda);

        $preuNet = $preuFinalAsMoney->dividedBy(
            self::getValorImpostos($percentatgeImpostos),
            RoundingMode::HALF_UP,
        );

        $quantitatImpostos = $preuFinalAsMoney->minus($preuNet);

        return new self(
            $preuNet->getMinorAmount()->toInt(),
            $percentatgeImpostos,
            $quantitatImpostos->getMinorAmount()->toInt(),
            $preuFinalAsMoney->getMinorAmount()->toInt(),
            $preuFinalAsMoney->getCurrency()->getCurrencyCode(),
        );
    }

    /** @throws ImportIsNotValid */
    private static function parseAndGetMoney(float|int|string $quantitat, string $moneda): Money
    {
        try {
            return Money::of($quantitat, strtoupper($moneda), null, RoundingMode::HALF_UP);
        } catch (NumberFormatException | UnknownCurrencyException) {
            throw ImportIsNotValid::ambPreuFinal($quantitat, $moneda);
        }
    }

    private static function getValorImpostos(PercentatgeImpostos $percentatgeImpostos): BigDecimal
    {
        $percentatgeImpososAsBigDecimal = BigDecimal::of($percentatgeImpostos->value);

        return $percentatgeImpososAsBigDecimal
            ->dividedBy(100, RoundingMode::HALF_UP)
            ->plus(1);
    }
}

```

- Al _Value Object_ hi guardem els cinc components d'un preu unitari:
  - El preu base, en la seva unitat menor.
  - El percentatge d'impostos aplicats al preu base.
  - La quantitat d'impostos, en la seva unitat menor.
  - El preu final, en la seva unitat menor.
  - La moneda.
- Construïm un `Import` amb el _named constructor_ a partir de preu final, perquè així ho indica el domini.
- Fem servir `Money` de `Brick` per a validar tant el valor com la moneda. En cas d'error, llancem una excepció pròpia
  del nostre domini.
- Per a càlcul dels impostos, es fa servir `BigDecimal`, de la llibreria `brick/math`[^1]. Com que a les divisions hi ha
  possible pèrdua de decimals, és necessari no arrodonir en fer els càlculs: un `Money` no ens serveix perquè sempre
  arrodoneix al nombre de decimals de la moneda. Per exemple, per a euros, sempre arrodoniria a dos decimals.

El cas de test més rellevant és el següent:

```php
<?php

declare(strict_types=1);

private const MONEDA_UPPER = 'EUR';
private const MONEDA_LOWER = 'eur';

/** @dataProvider preuNetIImpostProvider */
public function test_import_amb_impost_retorna_valors_esperat(
    int $expectedMinorPreuNet,
    int $expectedMinorQuantitatImpostos,
    int $expectedMinorTotal,
    float|string $preuFinal,
): void {
    $import = Import::ambPreuFinalAmbPercentatgeImpostos(
        $preuFinal,
        PercentatgeImpostos::ES_IVA_21,
        self::MONEDA_LOWER,
    );

    self::assertSame($expectedMinorPreuNet, $import->preuNetMinor());
    self::assertSame($expectedMinorQuantitatImpostos, $import->quantitatImpostosMinor());
    self::assertSame($expectedMinorTotal, $import->preuFinalMinor());
    self::assertSame(21, $import->percentatgeImpostos()->value);
    self::assertSame(self::MONEDA_UPPER, $import->moneda());
}

protected function preuNetIImpostProvider(): array
{
    return [
        '5.50 (float)' => [
            455,
            95,
            550,
            5.50,
        ],
        '5.30 (float)' => [
            438,
            92,
            530,
            5.30,
        ],
    ];
}
```

- Testegem els exemples de què disposem, per assegurar-nos que els càlculs són correctes.
- La validació dels valors la fem amb la unitat menor dels imports monetaris, però es podria fer servir `Money`, per
  exemple.
- Fem servir un _data provider_ construint els valors des de `float`, però al test complet també ho fem amb `string`.

### `LlistaImports`

Per a representar un llistat d'imports —que podria ser una factura— tenim la següent implementació:

```php
<?php

declare(strict_types=1);

namespace rubenrubiob\Domain\Model;

use Brick\Money\Currency;
use Brick\Money\Exception\UnknownCurrencyException;
use Brick\Money\ISOCurrencyProvider;
use Brick\Money\Money;
use rubenrubiob\Domain\Exception\Model\LlistaImportsMonedaNoCoincideix;
use rubenrubiob\Domain\Exception\Model\LlistaImportsMonedaNoValida;
use rubenrubiob\Domain\ValueObject\Import;

use function strtoupper;

final class LlistaImports
{
    /** @var list<Import> */
    private array $imports = [];

    private Money $totalPreusNets;
    private Money $totalImpostos;
    private Money $total;

    private function __construct(
        private readonly Currency $moneda,
    ) {
        $this->totalPreusNets = Money::zero($this->moneda);
        $this->totalImpostos  = Money::zero($this->moneda);
        $this->total          = Money::zero($this->moneda);
    }

    /** @throws LlistaImportsMonedaNoValida */
    public static function ambMoneda(string $moneda): self
    {
        return new self(
            self::parseAndValidateMoneda($moneda),
        );
    }

    /** @throws LlistaImportsMonedaNoCoincideix */
    public function afegirImport(Import $import): void
    {
        $this->validarMonedesCoincideixen($import);

        $this->recalcularImportsTotals($import);

        $this->imports[] = $import;
    }

    /** @throws LlistaImportsMonedaNoValida */
    private static function parseAndValidateMoneda(string $moneda): Currency
    {
        try {
            return ISOCurrencyProvider::getInstance()->getCurrency(strtoupper($moneda));
        } catch (UnknownCurrencyException) {
            throw LlistaImportsMonedaNoValida::perAConstruir($moneda);
        }
    }

    /** @throws LlistaImportsMonedaNoCoincideix */
    private function validarMonedesCoincideixen(Import $import): void
    {
        if ($import->moneda() !== $this->moneda->getCurrencyCode()) {
            throw LlistaImportsMonedaNoCoincideix::perALlistaAmbMoneda(
                $this->moneda->getCurrencyCode(),
                $import->moneda(),
            );
        }
    }

    private function recalcularImportsTotals(Import $import): void
    {
        $this->totalPreusNets = $this->totalPreusNets->plus(
            $import->preuNetAsMoney(),
        );

        $this->totalImpostos = $this->totalImpostos->plus(
            $import->quantitatImpostosAsMoney(),
        );

        $this->total = $this->total->plus(
            $import->preuFinalAsMoney(),
        );
    }
}

```

- Situem `LlistatImports` dins de `Model`. No pot ser un _Value Object_ perquè no és immutable. L'hem implementat així
  per estalviar memòria.
- Tenim un mètode per a inicialitzar el llistat amb la moneda. Tots els components seran un `Money` inicialitzat a 0: el
  sumatori de preus net, el sumatori total d'impostos i l'import total.
- Afegim els `Import` un a un:
  - Primer es valida que les monedes coincideixin.
  - Se suma cada import, fent servir `Money`.

Com que tenim un exemple real que ens donava problemes, podem escriure un test per plasmar aquest cas i validar que la
implementació sigui correcta:

```php
<?php

private const MONEDA_LOWER = 'eur';

public function test_amb_imports_valids_calculs_correctes(): void
{
    $primerImport = Import::ambPreuFinalAmbPercentatgeImpostos(
        5.50,
        PercentatgeImpostos::ES_IVA_21,
        self::MONEDA_LOWER,
    );

    $segonImport = Import::ambPreuFinalAmbPercentatgeImpostos(
        5.30,
        PercentatgeImpostos::ES_IVA_21,
        self::MONEDA_LOWER,
    );

    $llistaImports = LlistaImports::ambMoneda(self::MONEDA_LOWER);

    $llistaImports->afegirImport($primerImport);
    $llistaImports->afegirImport($primerImport);
    $llistaImports->afegirImport($primerImport);
    $llistaImports->afegirImport($primerImport);
    $llistaImports->afegirImport($primerImport);

    $llistaImports->afegirImport($segonImport);
    $llistaImports->afegirImport($segonImport);
    $llistaImports->afegirImport($segonImport);
    $llistaImports->afegirImport($segonImport);
    $llistaImports->afegirImport($segonImport);

    self::assertSame(4465, $llistaImports->totalPreusNetsMinor());
    self::assertSame(935, $llistaImports->totalImpostosMinor());
    self::assertSame(5400, $llistaImports->totalMinor());
    self::assertCount(10, $llistaImports->imports());
}
```

## Persistència

Com que totes les monedes tenen una escala oficial definida per l'estàndard ISO 4127, es poden persistir tots els
components de `Import` com a valors escalars, i després reconstruir el _Value Object_ a partir d'ells.

Depenent del motor de base de dades que es faci servir, hi ha diferents estratègies per a persistir un `Import`. En una
base de dades No-SQL es poden emmagatzemar els components en un JSON. Aquesta també és una opció vàlida per a una base
de dades relacional, on també és possible desar cada component en una columna separada. Per a aquest últim cas, si es fa
servir Doctrine ORM, es pot fer servir un _embeddable_ com el següent:

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<doctrine-mapping xmlns="https://doctrine-project.org/schemas/orm/doctrine-mapping"
                  xmlns:xsi="https://www.w3.org/2001/XMLSchema-instance"
                  xsi:schemaLocation="https://doctrine-project.org/schemas/orm/doctrine-mapping
                          https://www.doctrine-project.org/schemas/orm/doctrine-mapping.xsd">
    <embeddable name="rubenrubiob\Domain\ValueObject\Import">
        <field name="preuNet" type="integer" column="preu_net" />
        <field name="percentatgeImpostos" type="integer" enumType="rubenrubiob\Domain\ValueObject\PercentatgeImpostos" column="percentatge_impostos" />
        <field name="quantitatImpostos" type="integer" column="quantitat_impostos" />
        <field name="preuFinal" type="integer" column="preu_final" />
        <field name="currency" type="string" column="currency" length="3" />
    </embeddable>
</doctrine-mapping>

```

## Conclusions

Seguint l'exemple que vam veure en el post anterior, hem vist els càlculs que cal fer per a obtenir el preu net i la
quantitat d'impostos a partir del preu final, tal com ens indica el domini.

Hem vist una implementació per a resoldre el problema: amb un `Import` unitari que conté tots els components i
un `LlistatImports`, que representa un conjunt d'imports unitaris. Gràcies als test, hem pogut validar la nostra
implementació amb l'exemple real.

Per últim, hem vist una possible implementació d'un _Value Object_ que representa un import, amb la seva capa de
persistència amb Doctrine ORM.

[^1]: `brick/money` requereix el paquet `brick/math`, però sempre és millor
evitar [dependències transitives](https://github.com/maglnet/ComposerRequireChecker).