---
date: 2023-05-02
---

# Complir regles d'arquitectura amb Deptrac
{:.no_toc}

* TOC
{:toc}

## Introducció

Quan volem aplicar una arquitectura als nostres projectes, hi ha unes dependències entre capes que hem de complir. Això
succeeix, per exemple, quan volem aplicar arquitectura hexagonal. En el nostre cas, per a aplicar aquesta arquitectura,
fem servir aquestes capes, de més interior a més exterior:

- `Domain`: els objectes que representen conceptes del nostre domini: _Value Objects_, _aggregates_, entitats,
  esdeveniments i serveis de domini... El codi és PHP pur, sense dependències externes.
- `Application`: són els casos d'ús de la nostra aplicació, normalment, _commands_ (que modifiquen el nostre sistema) i
  _queries_ (que consulten dades del nostre sistema). En aquesta capa el codi tampoc no ha de contenir dependències
  externes.
- `Infrastructure`: el codi que s'integra amb serveis externs (connexions a base de dades o a API), llibreries
  concretes (de _CommandBus_)... Aquí hi podem incloure el _framework_ que fem servir, les llibreries (el `vendor`) i la
  capa d'interfície d'usuari, que inclou els controladors.

La dependència entre capes va de dins enfora:

- `Domain` només pot fer servir elements dins d'ella mateixa, no depèn de cap altra, ni pot utilitzar llibreries
  externes.
- `Application` pot emprar elements de la seva capa i de `Domain`. Tampoc no pot fer servir llibreries externes.
- `Infrastructure` pot fer servir elements d'`Application` i `Domain`, a més a més de llibreries externes.

Podem veure les relacions al següent diagrama:

![Dependències entre capes](/images/dev/hexagonal-layers.png)

Ara bé, PHP no ens ofereix cap mecanisme per a respectar aquesta dependència entre capes. És a dir, no hi ha cap manera
en el llenguatge de forçar la dependència entre capes, de manera que hom podria saltar-la i no ens n'adonaríem si no
revisem a fons el codi.

Afortunadament, PHP té un ecosistema molt ric i existeixen utilitats per a verificar les regles de la nostra
arquitectura:

- [Deptrac](https://github.com/qossmic/deptrac){:target="_blank"}
- [PHP Architecture Tester](https://github.com/carlosas/phpat){:target="_blank"} (com a _plugin_ de PHPStan)
- [PHPArkitect](https://github.com/phparkitect/arkitect){:target="_blank"}

En aquest post veurem com configurar Deptrac per a fer complir les regles de l'arquitectura hexagonal que hem descrit
més a dalt.

## Deptrac

Deptrac és una aplicació de consola que valida dependències definides en un fitxer de configuració. Si hi ha cap
dependència no permesa, retorna un codi d'error diferent de `0`. Això el fa útil per a integrar aquesta validació de
dependències a la integració contínua del nostre projecte.

### Conceptes

Els [conceptes principals de Deptrac](https://qossmic.github.io/deptrac/concepts/){:target="_blank"} són els següents:

- Capes (_layers_): són agrupacions de _tokens_ (classes, funcions...). Per exemple, totes les classes que formen part de
  la nostra capa de domini. Deptrac ofereix múltiples [col·lectors](https://qossmic.github.io/deptrac/collectors/){:target="_blank"} per a
  seleccionar aquestes capes: per directori, per _namespace_ de la classe, per nom de funció...
- Regles (_rulesets_): són les regles que defineixen les comunicacions permeses entre capes. Per exemple, la capa
  d'aplicació pot accedir a la capa de domini. Per defecte, no hi ha cap dependència permesa entre capes, sempre s'han
  d'explicitar.
- Violacions (_violations_): són els errors de dependències entre capes no permeses. Per exemple, si el nostre domini
  accedeix a la capa d'aplicació, cosa que no és permesa.

### Definició

Amb els conceptes ja revisats, podem escriure el nostre fitxer de configuració, en format YAML:

```yml
deptrac:
    paths:
        - ./src
    layers:
        # Layer definition
        - name: Domain
          collectors:
              - type: directory
                regex: src/Domain

        - name: Application
          collectors:
              - type: directory
                regex: src/Application

        - name: Infrastructure
          collectors:
              - type: directory
                regex: src/Infrastructure

        # Vendor
        - name: DomainVendor
          collectors:
              - type: className
                regex: ^(Brick\\Math|Brick\\Money|Doctrine\\Common\\Collections|Ramsey\\Uuid)\\.*

        - name: Vendor
          collectors:
              - type: className
                regex: ^(Symfony|CuyZ\\Valinor|League\\ConstructFinder|League\\Tactician)\\.*

    ruleset:
        Domain:
            - DomainVendor
        Application:
            - Domain
        Infrastructure:
            - Domain
            - Application
            - Vendor
```

A Deptrac cal especificar quins directoris contenen el codi a analitzar, dins de la clau `paths`. Tot el nostre codi
resideix al directori `src`, així que l'incloem en l'anàlisi.

A continuació definim les capes, dins de la clau `layers`. En tenim una per a cadascuna de les nostres capes, amb
un `collector` de tipus `directory`.

En teoria, el nostre domini només ha de contenir codi PHP pur, és a dir, sense cap dependència externa. Ara bé, hi ha
certes llibreries que volem fer servir al nostre domini, com per exemple la llibreria `ramsey/uuid` per a generar UUID o
les llibreries `brick/math` i `brick/money`, que ens permeten treballar amb nombres i imports monetaris al nostre
domini. Un altre exemple habitual és el
de [`doctrine/collections`](https://www.doctrine-project.org/projects/collections.html){:target="_blank"}: quan fem servir Doctrine com a
ORM, les relacions entre entitats han de ser objectes del tipus `Doctrine\Common\Collection`.

El domini, segons hem explicat, hauria de ser pur, però hi ha vegades en què hem de fer concessions: val la pena
reimplementar un sistema de generació de UUID només per a mantenir pur el nostre domini? La resposta és que no. La major
part de les vegades no cal reinventar la roda. Podem dir no a tenir codi extern al nostre domini i, alhora, dir sí a les
llibreries que necessitem al domini. Sempre ha de ser una llista blanca, és a dir, hem d'escollir quines llibreries
volem al nostre domini, no permetre-les totes per defecte. Hem de prendre aquestes decisions de manera conscient i
crítica.

Per tant, definim una capa anomenada `VendorDomain`, on hi especifiquem les llibreries que permetem al nostre domini. En
aquest cas, fem servir un `collector` per _namespace_ de la classe.

Alhora, a la nostra capa d'infraestructura és on permetem l'accés a llibreries de tercers. Podríem estar temptats
d'incloure tot el `vendor` als `paths` a analitzar. Ara bé, això alentiria l'execució, alhora que analitzaria tot
el `vendor` i les dependències entre si.

En canvi, si solament incloem les llibreries que permetem a la nostra capa d'infraestructura, no només simplifiquem
l'anàlisi i la seva celeritat, sinó que, a més a més, evitarem dependències transitives, és a dir, dependre al nostre
codi d'una llibreria que no hem especificat de manera explícita.

Així doncs, definim una capa més general anomenada `Vendor`, on especifiquem les llibreries que permetem a la nostra
capa d'infraestructura. En aquest cas, també fem servir un `collector` per _namespace_ de la classe.

Amb les capes ja definides, podem finalment definir les regles de dependència (`ruleset`) entre capes:

- `Domain`: pot accedir a `DomainVendor`, com ja hem explicat.
- `Application`: només pot accedir a `Domain`.
- `Infrastructure` pot accedir a `Domain`, `Application` i `Vendor`. És la capa inferior, la que té accés a la resta.

### Execució

Ara ja podem executar Deptrac:

```bash
deptrac analyse --config-file=hexagonal-layers.depfile.yaml --cache-file=.deptrac.hexagonal-layers.cache --report-uncovered --fail-on-uncovered
```

Per defecte, Deptrac fa servir el fitxer de configuració `deptrac.yaml` i el fitxer de caché `.deptrac.cache`. Això no
obstant, a l'hora de fer la crida definim el fitxer de configuració i el fitxer de _caché_, per evitar conflictes amb
els valors per defecte.

Executem Deptrac especificant que reporti els `uncovered` (`--report-uncovered`), és a dir, les dependències que
existeixen i que no hem definit. Volem que falli si n'hi ha (`--report-uncovered`), per a forçar-nos a revisar totes les
dependències, fent més estricta la validació entre capes.

La sortida de l'execució és similar a això:

```
 -------------------- ----- 
  Report                    
 -------------------- ----- 
  Violations           0    
  Skipped violations   0    
  Uncovered            0    
  Allowed              222  
  Warnings             0    
  Errors               0    
 -------------------- ----- 
```

Deptrac ofereix [diversos formatadors](https://qossmic.github.io/deptrac/formatters/){:target="_blank"}, que no cobrirem en aquest post.

## Conclusions

Amb Deptrac configurat, podem assegurar-nos que tothom qui col·labori al nostre projecte compleixi les regles que hem
definit, siguin quines siguin. Per a forçar-ho, el millor és incloure-ho a la nostra integració contínua, de manera que
no es permeti codi que no compleixi les regles.

La configuració de capes que hem vist es pot estendre a l'ús de _Bounded Contexts_ dins del mateix projecte, per a
prevenir que un _Bounded Context_ no n'accedeix a un altre.

## Resum

- Hem vist una possible definició de capes per a aplicar arquitectura hexagonal.
- Hem vist que PHP no ofereix cap mecanisme per a forçar una dependència entre capes.
- Hem llistat utilitats que permeten validar l'arquitectura de la nostra aplicació, entre elles, Deptrac.
- Hem explicat breument els conceptes que fa servir Deptrac internament.
- Hem vist una possible configuració de Deptrac per a respectar les capes de l'arquitectura hexagonal, permetent
  dependències externes a la capa de domini, i evitant dependències transitives a la capa d'infraestructura.
- Hem executat Deptrac de manera estricta per a forçar que tot el nostre codi sigui analitzat.