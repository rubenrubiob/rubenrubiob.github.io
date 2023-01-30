---
date: 2023-01-30
---

# Patró _Money_ a PHP (part 1 de 2)
{:.no_toc}

* TOC
{:toc}

## Introducció

Quan treballem amb nombres, podem trobar-nos operacions en les quals perdem precisió, o bé perquè el nombre és
immensament gran, o bé perquè té decimals infinits. El problema és representar nombres infinits en un sistema finit: per
molta memòria que tinguem disponible, sempre serà finita. Aquesta representació es coneix com
a [punt flotant (IEEE 754)](https://en.wikipedia.org/wiki/IEEE_754).

Depèn del cas, el problema pot ser més o menys greu. Per exemple, en un comerç electrònic, un error de precisió pot fer
que cobrem de menys al client, fent que el comerç perdi diners; o que li cobrem de més, provocant un possible problema
legal.

Un problema d'aquest estil em vaig trobar en un projecte de comerç electrònic, on teníem fins i tot un Excel anomenat
_pitote_ descrivint-ne exemples. Per il·lustrar-ho, suposem dos dels productes del comerç: un amb un PVP de 5'50 €, i un
altre, de 5'30 €. Si un client compra cinc unitats de cada producte, esperaríem una factura desglossada com la que es
mostra a la següent taula:

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
| **44'65** | **9'35** | **54'00** |

Com que cada producte havia de tenir el preu desglossat per a generar les factures, el que feia el _software_ era
permetre afegir el preu net i, en funció del percentatge d'IVA associat al producte, en calculava els impostos i el PVP
del producte. Per a intentar pal·liar els problemes amb els decimals, es desaven fins a tres posicions decimals pel preu
net.

Ara bé, els imports de cada producte no es desaven ja calculats, sinó que es recalculaven allà on calia mostrar-los. I
aquest càlcul no era el mateix a tot arreu, sinó que en algunes parts s'arrodonia primer el preu net, en d'altres,
s'arrodonia el total...

Així doncs, ens podem trobar que el resum que veia el client abans de pagar, i el que pagava, era el següent:

| Preu net (€) | IVA (21%) (€) | Total (€) |
|:--|:--|:--|
| 4'545 | 0'9545 | 5'4995 |
| 4'545 | 0'9545 | 5'4995 |
| 4'545 | 0'9545 | 5'4995 |
| 4'545 | 0'9545 | 5'4995 |
| 4'545 | 0'9545 | 5'4995 |
| 4'380 | 0'9198 | 5'2998 |
| 4'380 | 0'9198 | 5'2998 |
| 4'380 | 0'9198 | 5'2998 |
| 4'380 | 0'9198 | 5'2998 |
| 4'380 | 0'9198 | 5'2998 |
| **44'625** | **9'372** | **53'997** |

I, en canvi, la factura que rebia el client era la següent:

| Preu net (€) | IVA (21%) (€) | Total (€) |
|:--|:--|:--|
| 4'550 | 0'9555 | 5'5055 |
| 4'550 | 0'9555 | 5'5055 |
| 4'550 | 0'9555 | 5'5055 |
| 4'550 | 0'9555 | 5'5055 |
| 4'550 | 0'9555 | 5'5055 |
| 4'380 | 0'9198 | 5'2998 |
| 4'380 | 0'9198 | 5'2998 |
| 4'380 | 0'9198 | 5'2998 |
| 4'380 | 0'9198 | 5'2998 |
| 4'380 | 0'9198 | 5'2998 |
| **44'65** | **9'38** | **54'03** |

Més enllà de les inconsistències en la forma de calcular, fruit de la manca de coneixement de bones pràctiques de programació, el problema real era que en cap cas s'obtenia l'import esperat. Com podríem haver realitzat els càlculs per a obtenir l'import correcte? On caldria haver arrodonit?

## Arrodoniment a PHP

A PHP tenim diverses funcions pròpies del llenguatge per a arrodonir nombres.

- `floor($amount)`: _Returns the next lowest integer value (as float) by rounding down value if necessary._
- `ceil($value)`: _Returns the next highest integer value by rounding up value if necessary._
- `round($amount, $precision, $mode)`: _Returns the rounded value of val to specified precision (number of digits after the decimal point). Precision can also be negative or zero (default)._
- `number_format($amount, $decimals)`: _Formats a number with grouped thousands and optionally decimal digits._

Així doncs, quina d'aquestes funcions caldria fer servir per a obtenir imports correctes? I com? La resposta és cap
d'elles.

En qualsevol d'aquests casos acabaríem perdent precisió, i els càlculs serien erronis. 

## Patró moneda (_Money pattern_)

El cert és que no estem enfocant correctament el problema. A més a més d'una quantitat, un import o preu sempre té una
moneda. Quan diem que un producte costa 10, a què ens referim? 10 €? 10 $? 10 ¥? Un import pot variar molt si no tenim
en compte la moneda. Al cap i a la fi, hi ha una gran diferència entre 100 € i 100 pessetes.

Per a negocis petits que només operen a un país o regió on només hi ha una moneda, pot tenir sentit no tenir-la en
compte. Si el negoci no s'expandeix a una regió amb una altra moneda, aquesta assumpció serà vàlida. Ara bé, mai no
podem estar segurs al cent per cent que això no succeeixi al futur.

Per resoldre el problema de l'arrodoniment d'imports, aleshores, podem fer servir el [patró moneda (_Money
pattern_)](https://www.martinfowler.com/eaaCatalog/money.html), que consisteix a fer servir un _Value Object_ amb dos
atributs: la moneda i la quantitat en la menor unitat de la moneda. És a dir, la quantitat `89'99` es desaria com
a `8999`.

## Llibreries

A PHP existeixen llibreries de codi obert que implementen el patró moneda, resolent així els dos problemes: treballar
amb números molt grans (amb un límit), i tenir imports amb quantitat i moneda.

Les dues llibreries més importants són:

- [brick/money](https://github.com/brick/money)
- [moneyphp/money](https://github.com/moneyphp/money)

Internament, fan servir l'extensió [`BCMath`](https://www.php.net/manual/en/book.bc.php) de PHP per a fer els càlculs,
sigui representant els nombres amb `string`, sigui amb `int`, de manera que no hi ha pèrdua de decimals.

[Existeix una comparativa entre les dues llibreries](https://github.com/brick/money/issues/28), però qualsevol de les
dues és una bona opció i permet fer càlculs monetaris de manera segura.

## Conclusions

Hem vist els problemes de pèrdues de precisió que suposa treballar amb nombres de punt flotant, que poden arribar a
suposar pèrdues monetàries a un negoci. Cap de les funcions natives de PHP són una solució.

Per a fer servir imports, la solució consisteix a emprar el patró moneda. A PHP hi ha llibreries de codi obert que
permeten treballar amb imports de manera segura fent servir l'extensió BCMath, sense pèrdua de precisió (dins d'uns
límits).

En el proper post veurem una implementació que resol correctament l'exemple que hem vist a la introducció, obtenint-ne
els imports esperats.
