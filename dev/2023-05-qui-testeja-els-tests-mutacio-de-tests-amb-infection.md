---
date: 2023-05-09
---

# Qui testeja els tests? Mutació de tests amb Infection
{:.no_toc}

* TOC
{:toc}

## Introducció

En enginyeria de _software_, escriure tests és bona pràctica, i molt necessari per a tenir confiança en el nostre codi.
Ara bé, com sabem que els nostres tests comproven correctament el codi? Com podem construir confiança en els nostres tests?

Una de les mètriques més conegudes és la cobertura de codi, que consisteix en el percentatge de línies de codi que
tenen test. Ara bé, amb aquesta mètrica no n'hi prou.

Per exemple, suposem que tenim tests que executen tot el nostre codi. Ara bé, les úniques comprovacions que es fan als
tests són `assertTrue(true)`. En aquest cas, la cobertura del nostre codi és del 100%, però els tests no són fiables.

Quina alternativa tenim, aleshores? El que es coneix com a mutació de tests. Consisteix _grosso modo_ en les següents
passes:

0. Com a pas previ, tots els tests han d'executar-se amb èxit.
1. Es modifica el codi font de la nostra aplicació per a fer-lo fallar. Per exemple, canviar un `>` per un `<` en una
   comparació. Això és el que es coneix com a mutant.
2. Es tornen a executar tots els tests.
3. Aquí tenim dues possibilitats:
   1. Si els tests fallen, vol dir que s'ha aconseguit matar el mutant, i això és positiu. Per a tenir una bona
      _suite_ de tests, tots els mutants haurien de ser eliminats.
   2. En canvi, si els tests continuen executant-se amb èxit, vol dir que el mutant ha sobreviscut. Això pot voler dir dues coses:
   - La línia de codi mutada no està coberta pels tests.
   - Els tests per a aquesta línia no són gaire útils.

Evidentment, no podem executar tots aquests canvis manualment. És comú que tots els llenguatges de programació tinguin
una utilitat de mutació. Per a PHP existeix [Infection](https://infection.github.io/){:target="_blank"}, una aplicació de consola que
executa mutacions per al nostre codi i en dona unes mètriques.

## Infection

### Mètriques

[Infection fa servir les següents mètriques](https://infection.github.io/guide/#Metrics-Mutation-Score-Indicator-MSI){:target="_blank"}:

- _Mutation Score Indicator (MSI)_: és el percentatge de mutants detectats (eliminats) del total generat pel nostre codi
  font. Com més alt és aquest percentatge, més robust seran els nostres tests.
- _Mutation Code Coverage (MCC)_: és el percentatge de codi cobert pels mutants. Acostuma a ser igual que la
  cobertura del codi.
- _Covered Code Mutation Score Indicator_: és el MSI per al codi que realment és cobert pels nostres tests.

La mètrica que es fa servir normalment per a mesurar la qualitat dels nostres tests és el MSI.

### Exemple

Suposem que tenim la següent classe que comprova si un nombre és positiu o no (per simplicitat, considerem `0` com un
nombre positiu, [tot i que no és del tot correcte](https://math.stackexchange.com/a/26708){:target="_blank"}):

```php

final readonly class NumberChecker
{
    public static function isPositive(int $number): bool
    {
        return $number >= 0;
    }
}
```

I que tenim el seu test així:

```php
final class NumberCheckerTest extends TestCase
{
    public function test_isPositive(): void
    {
        self::assertTrue(
            NumberChecker::isPositive(10),
        );
        
        self::assertFalse(
            NumberChecker::isPositive(-10),
        );
    }
}

```

El test passa correctament. Ara bé, quin MSI tenim? Ho podem veure executant Infection per a la nostra classe:

```bash
infection --threads=max --filter=NumberChecker.php --show-mutations
```

Les mètriques que obtenim són les següents:

```
Metrics:
         Mutation Score Indicator (MSI): 66%
         Mutation Code Coverage: 100%
         Covered Code MSI: 66%
```

Veiem que els nostres tests no tenen un MSI gaire alt, tot i tenir un 100% de cobertura de codi. A la sortida de
l'execució prèvia, podem veure també els mutants que han escapat:

```diff
Escaped mutants:
================


1) NumberChecker.php:11    [M] GreaterThanOrEqualTo

--- Original
+++ New
@@ @@
 {
     public static function isPositive(int $number) : bool
     {
-        return $number >= 0;
+        return $number > 0;
     }
 }
```

Veiem que no estem comprovant correctament el límit de la comparativa. Quan escrivim tests per a intervals, hem de
testejar sempre els límits. Realment, a partir del nombre `0` i pujant, tots els nombres són positius, tant és comprovar
10 com 98.764.343: el valor important és el `0`.

I viceversa, tot i que no és estrictament necessari en aquest cas, però és bona pràctica: a partir del nombre `-1` i
baixant, tots els nombres són negatius. El valor realment rellevant és `-1`.

Així doncs, per a intervals, hem de testejar sempre els límits, que és on hi ha els valors crítics.

Si reescrivim els tests així:

```php
final class NumberCheckerTest extends TestCase
{
    public function test_isPositive(): void
    {
        self::assertTrue(
            NumberChecker::isPositive(0),
        );
        
        self::assertFalse(
            NumberChecker::isPositive(-1),
        );
    }
}

```

I tornem a executar Infection, veiem que ara sí que obtenim un MSI del 100%:

```
Metrics:
         Mutation Score Indicator (MSI): 100%
         Mutation Code Coverage: 100%
         Covered Code MSI: 100%
```

Aquest és només un exemple, però Infection genera molts més
mutants. [Els podeu consultar a la documentació](https://infection.github.io/guide/mutators.html){:target="_blank"}.

### Configuració i execució

Normalment, voldrem executar Infection per a tot el nostre projecte. Ara bé, si fem servir arquitectura hexagonal, és
probable que només vulguem executar Infection per als tests unitaris del nostre domini, per dos motius.

El primer d'ells és que, si generem mutació de tests pels quals no són unitaris —d'integració, acceptació o funcionals—,
és probable que ens trobem amb problemes de _timeout_: com Infection executa tots els tests per a cada mutant, sovint
alguna execució s'endarrereix per alguna connexió amb algun servei extern del nostre sistema.

El segon motiu és que si executem mutació de tests per a la capa d'infraestructura, sovint estarem generant mutants per
a codi que s'integra amb tercers que no controlem, i invertirem esforços en una part que no és rellevant pel domini.

Així doncs, per a un projecte amb Symfony que implementi arquitectura hexagonal, podríem tenir la següent configuració
d'Infection al fitxer `infection.json.dist` de l'arrel del nostre projecte:

```json
{
    "$schema": "https://raw.githubusercontent.com/infection/infection/master/resources/schema.json",
    "source": {
        "directories": [
            "src"
        ],
        "excludes": [
            "{Infrastructure/.*}",
            "{Domain/Exception/.*}"
        ]
    },
    "logs": {
        "html": "var/log/infection/infection.html",
        "text": "var/log/infection/infection.log",
        "summary": "var/log/infection/infection-summary.log",
        "debug": "var/log/infection/infection-debug.log",
        "perMutator": "var/log/infection/infection-permutator.log"
    },
    "mutators": {
        "@default": true
    }
}
```

El que fem és:

- Configurar que s'analitzi tot el directori `src`.
- Excloure tant les excepcions de domini, perquè és codi que sovint no ens cal testejar, com tota la capa
  d'infraestructura.
- Desar tots els _logs_ generats en el mateix directori en què Symfony hi desa els seus, per evitar haver d'excloure més
  fitxers al `.gitignore`.

Així doncs, suposant que tenim la següent configuració de PHPUnit, en què tenim separades les _suites_ unitària i
funcional:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!-- https://phpunit.readthedocs.io/en/latest/configuration.html -->
<phpunit xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:noNamespaceSchemaLocation="https://schema.phpunit.de/10.0/phpunit.xsd"
         backupGlobals="false"
         colors="true"
         bootstrap="tests/bootstrap.php"
         executionOrder="random"
         resolveDependencies="true"
         cacheDirectory=".phpunit.cache">
  <php>
    <ini name="display_errors" value="1"/>
    <ini name="error_reporting" value="-1"/>
    <server name="APP_ENV" value="test" force="true"/>
    <server name="SHELL_VERBOSITY" value="-1"/>
    <env name="KERNEL_CLASS" value="rubenrubiob\Infrastructure\Symfony\Kernel" />
  </php>
  <testsuites>
    <testsuite name="Unit">
      <directory>tests/Unit</directory>
    </testsuite>
    <testsuite name="Functional">
      <directory>tests/Functional</directory>
    </testsuite>
  </testsuites>
</phpunit>
```

Podem executar Infection només per als tests unitaris de la següent manera:

```bash
infection --threads=max --min-msi=100 --test-framework-options=\"--testsuite=Unit\"
```

El que fem en aquesta comanda és:

- Indiquem que empri els màxims _threads_ disponibles del sistema operatiu, per a accelerar-ne l'execució.
- Forcem un codi de retorn diferent de `0` si no hi ha un MSI mínim del 100%. Això és útil sobretot a _pipelines_.
- Executem només els tests unitaris passant l'opció pel _framework_ de test, en aquest cas, PHPUnit.

Tal com l'hem configurat, Infection genera un _log_ HTML que és molt útil per a visualitzar tots els mutants que genera
i quins fallen. Podem veure el resum per a l'exemple de la secció anterior:

![Infection: resum general](/images/dev/mutation-testing/infection-1.png)

I el cas concret del mutant que no s'aconseguia eliminar.

![Infection: mutant escapat](/images/dev/mutation-testing/infection-2.png)

## Conclusió

Amb aquesta configuració, ja podem incloure Infection als nostres projectes per a augmentar la qualitat de la nostra
_suite_ de tests. Cal tenir en compte que no a tots els projectes es pot tenir un MSI del 100%. I que sovint és útil
deixar un _marge_, per si cap dia hem de fer un _deploy_ amb un _hotfix_. Com sempre, cal adaptar la configuració a les
necessitats del projecte.

## Resum

- Hem vist la utilitat de la mutació de tests com a mètrica de qualitat alternativa a la cobertura de tests.
- Hem explicat el funcionament general de la mutació de tests.
- Hem mostrat els conceptes generals que fa servir Infection.
- Hem revisat un exemple concret de com canviar els nostres tests perquè aconsegueixin matar mutants i augmentar l'MSI.
- Hem mostrat una configuració d'Infection per a projectes amb Symfony i arquitectura hexagonal.