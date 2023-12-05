---
date: 2023-12-06
---

# Optimitzant una aplicació web (I): veure
{:.no_toc}

* TOC
{:toc}

## Introducció

Aquest any, un client em va contactar perquè estaven tenint problemes de rendiment en una aplicació: respostes lentes, caigudes del servidor...

Aquesta aplicació és estacional: s'utilitza únicament durant l'estiu, els pics de càrrega succeeixen els caps de setmana. Es tracta d'una aplicació per a gestionar l'ocupació d'edificis en horaris concrets: permet comprar tiquets, validar-los per a entrar i sortir, veure l'ocupació en temps real...

L'aplicació consisteix en una aplicació mòbil per a usuaris finals, una altra per a administradors, i els seus equivalents web, a part d'un _backoffice_. El _backend_ està escrit en PHP emprant Symfony, amb una arquitectura hexagonal, i sense cap sistema d'esdeveniments.

Com a infraestructura, tenen un servidor per a la base de dades, una instància de MariaDB; i un altre per al servidor web: PHP, Apache i Nginx. Ambdós són servidors dedicats.

Aquest article és el primer d'una sèrie de tres sobre com detectar els colls d'ampolla a aquesta aplicació i millorar-ne el rendiment.

## Monitoratge

Per a millorar el rendiment, necessitem saber on són els problemes. Necessitem veure què succeeix de manera objectiva, amb mètriques mesurables, especialment per dues raons.

La primera d'elles és que necessitem saber quins camins de l'aplicació són lents, per a atacar i prioritzar què millorar. No té gaire sentit optimitzar camins de l'aplicació que no són importants per a l'usuari final. Per exemple, en aquest cas, pel client és acceptable que el _backoffice_ no sigui tan ràpid com la resta de l'aplicació.

La segona raó és que necessitem mètriques per a mesurar si les millores que apliquem són efectives, i quant ho són. Al final, presentarem aquestes mètriques al client per mostrar quant hem millorat el rendiment.

En resum, necessitem observabilitat, hem de monitorar la nostra aplicació. Hi ha diferents serveis amb aquesta finalitat: Datadog, New Relic, Tideways... Són serveis que cal limitar correctament, perquè poden esdevenir cars:

![Datadog meme](/images/dev/optimitzacio-aplicacio-web/datadog-meme.png)

Vam escollir New Relic perquè té un pla gratuït de 100 GB al mes, que és suficient per a aquest cas d'ús. A més a més, hi ha un [_bundle_ per a Symfony](https://github.com/ekino/EkinoNewRelicBundle) que permet tenir les mètriques més integrades, com per exemple tenir els noms de ruta de Symfony a les transaccions de New Relic.

Vam haver d'instal·lar l'APM de New Relic[^1] al servidor i configurar-lo, seguint la documentació oficial. Un cop fet això, New Relic va començar a col·lectar dades, de manera que només vam haver d'esperar per a tenir prou mètriques per a analitzar.

## Anàlisi

Com que el client ens va comentar que els pics d'ús succeïen durant els caps de setmana, vam esperar a un dilluns per a analitzar les mètriques, de manera que fossin prou significatives.

Podem, doncs, passar a analitzar alguns dels gràfics que New Relic posa a la nostra disposició en el panell per defecte de PHP.

### Càrrega general

Aquesta gràfica ens mostra la càrrega del sistema al llarg del dia, segmentada per l'ús del servei: PHP, MySQL...

![Gràfic Web transactions time](/images/dev/optimitzacio-aplicacio-web/grafic-web-transactions-time.png)

- Hi ha una càrrega alta durant el dia i baixa a la nit, que encaixa amb el que esperem.
- La base de dades té una càrrega alta, de manera que pot ser un possible punt d'optimització.
- Hi ha un pic de càrrega inusual el 25 de juliol, que, com que no es repeteix, pot ser degut a una càrrega temporal.

### Transaccions

El gràfic de transaccions ens mostra les 5 transaccions més lentes agrupades per la ruta de Symfony (gràcies a `ekino/newrelic-bundle`).

![Gràfic Transactions](/images/dev/optimitzacio-aplicacio-web/grafic-transactions.png)

- Hi ha algunes rutes que tenen una mitjana de resposta alta, de gairebé 2 segons.
- Algunes rutes tenen traces de més de 24 segons, que és massa.
- Aquestes rutes haurien de ser les primeres a ser optimitzades.

### Top 20 operacions de base de dades

Aquesta gràfica agrupa les consultes a la taula principal en una consulta, mostrar el percentatge que suposa per a l'aplicació.

![Gràfic Top 20 database operations](/images/dev/optimitzacio-aplicacio-web/grafic-top-20-database-operations.png)

- Hi ha una consulta que suposa el 43 % de temps de consulta de l'aplicació.
- Unes altres dues consultes suposen el 19 % i l'11 % del temps total.
- Entre aquestes tres consultes suposen el 73 % de càrrega de la base de dades.

## Conclusions

Després de veure aquestes mètriques, podem suposar que la base de dades pot ser un coll d'ampolla. Com que de vegades és senzill millorar el rendiment de la base de dades creant índexs o reescrivint les consultes, aquest serà el primer pas a optimitzar. Suposarà també una victòria ràpida.

## Resum

- Hem revisat la necessitat d'un servei de monitoratge quan optimitzem aplicacions.
- Hem llistat diferents serveis de monitoratge que encaixarien en el nostre cas d'ús, i hem donat els motius per a escollir New Relic.
- Hem analitzat les mètriques de New Relic per a trobar els colls d'ampolla de l'aplicació i tenir alguns punts on poder començar a optimitzar.

[^1]: Gestió de rendiment de l’aplicació, en anglès, _Application Performance Management_.