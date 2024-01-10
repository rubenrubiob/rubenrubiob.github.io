---
date: 2024-01-10
---

# Optimitzant una aplicació web (III): projectar
{:.no_toc}

* TOC
{:toc}

## Introducció

Amb aquest post acabem la sèrie d’articles sobre l’optimització d’una aplicació web. Com a recordatori, aquesta aplicació fa servir Symfony per a gestionar l’entrada i sortida a edificis, validant tiquets comprats prèviament.

A l’article anterior, vam explicar les optimitzacions realitzades, que consistien a afegir índexs a la base de dades. Vam millorar el rendiment en un 57 %. Això no obstant, els temps de resposta absoluts eren de 300 ms, que considerem massa elevats. En aquest post, revisarem què fa que l’aplicació trigui tant a respondre i ho resoldrem.

## Problema

Igual que en el cas anterior, vam fer servir New Relic per a identificar el problema. Després de revisar les mètriques tant dels _endpoints_ com de la base de dades, vam trobar una consulta que causava aquesta lentitud. L’aplicació ofereix algunes dades en temps real als seus usuaris: el nombre total de persones dins d’un edifici, i els seus noms —només als administradors. Per a servir aquesta informació, l’aplicació fa servir taules que tenen els logs de totes les entrades i sortides dels usuaris.

Així doncs, per a calcular el nombre total de persones que hi ha ara mateix a un edifici, l’aplicació compta el total d’entrades i de sortides des de la seva existència. És a dir, consulta dades d’anys anteriors per a donar informació del present. La consulta que executa és:

```sql
SELECT COUNT(event_checkin.id)
FROM event_checkin
WHERE (event_checkin.building_id = :building_id)
  AND (DATE(event_checkin.created_on) = DATE(:date)
  /*...*/
```

Aquesta consulta és complexa i s’empra a diferents llocs de l’aplicació, de manera que afecta regles de negoci. En conseqüència, vam decidir no canviar la consulta i buscar alternatives.

## Solució

Volem millorar el rendiment per a usuaris finals i administradors, ja que són ells els qui fan servir l’aplicació per a accedir i sortir de l’edifici. Com més ràpid sigui aquest procés, millor. La resta de camins de l’aplicació no són crítics en termes de temps de resposta.

Realment, només volem dues dades per a aquests camins crítics:

- El nombre total de persones dins d’un edifici.
- Els noms i algunes dades extres, com l’adreça de correu electrònic, de les persones dins d’un edifici —només per a administradors.

El que podem fer es tenir aquestes dades precalculades: podem actuar quan l’usuari valida el seu tiquet d’entrada i sortida. Farem servir esdeveniments per a denormalitzar les dades que necessitem i generar-ne projeccions: desarem conjuntament totes les dades que necessitem consultar alhora, de manera que només ens caldrà executar consultes senzilles per a obtenir-les.

L’esquema per a la validació d’entrada en un edifici serà:

![Esquema validació](/images/dev/optimitzacio-aplicacio-web/esquema-checkin-checkout.jpeg)

Quan un usuari valida el seu tiquet, s’envia un esdeveniment a un sistema de cues. Després dos consumidors processen aquest esdeveniment i executen l’acció necessària: actualitzar el total de persones dins d’un edifici, o generar la projecció de les dades de l’usuari. En el cas de sortida, l’esquema serà anàleg.

Només aplicarem els canvis necessaris als camins crítics de l’aplicació, sense tocar la resta, per a no afectar la lògica de domini.

## Implementació

L’aplicació està implementada amb arquitectura hexagonal, així que les dues comandes necessàries, per a validar el tiquet d’entrada i de sortida, estan aïllades i ben identificades al codi.

Vam haver de realitzar els següents canvis:

- Llençar els esdeveniments des de les entitats de domini quan es valida un tiquet.
- Consumir aquests esdeveniments per a generar o actualitzar les projeccions:
    - Actualitzar el nombre total de persones dins d’un edifici, incrementant o reduïnt un valor, depenent de si la validació és d’entrada o de sortida, respectivament.
    - Inserir o eliminar una fila amb les dades de l’usuari dins de l’edifici, depenent de si la validació és d’entrada o de sortida, respectivament.
- Canviar el camí des de la capa externa —els controladors— fins a la capa de domini, de manera que llegeixi les projeccions en lloc d’accedir a la base de dades.

Podríem haver fet servir Redis o un altre sistema de persistència, però vam decidir emprar MySQL, per a no fer més canvis al projecte. Com que el projecte segueix arquitectura hexagonal, és senzill canviar la implementació concreta en el futur, si cal. Vam fer servir Symfony Messenger amb RabbitMQ com a _broker_ per a l’intercanvi de missatges. Aquesta configuració queda fora de l’àmbit d’aquest post.

Al final, les consultes que vam haver d’executar eren simples i ràpides. Per a obtenir el nombre total de persones dins d’un edifici, la consulta esdevé:

```sql
SELECT total_in
FROM view_building_data
WHERE building_id = :building_id
```

I la consulta per a obtenir les dades dels usuaris dins d’un edifici és:

```sql
SELECT *
FROM view_users_inside_building
WHERE building_id = :building_id
```

Ambdues consultes empren les claus primàries, de manera que són òptimes.

## Resultats

Per desgràcia, l’aplicació només es fa servir a l’estiu, i vam desplegar aquests canvis a la tardor. Per tant, no podem validar els resultats fins a l’estiu vinent.

Això no obstant, vam realitzar algunes comprovacions manuals. El temps de resposta dels _endpoints_, abans de desplegar els canvis, era d’1,1 segons —amb la possibilitat que augmentessin en el futur, en haver-hi més dades a les taules que consultaven. Després de desplegar els canvis, el temps de resposta es va reduir fins als 150 ms. Com que les dues consultes eren simples i feien servir claus primàries, tenim la certesa que aquest temps de resposta serà sempre similar.

## Conclusions

Presumptament, podem afirmar que hem millorat el temps de resposta. Ho sabrem del cert aquest estiu —hi haurà un post amb els resultats.

Com a millores futures, podríem deixar d’utilitzar les taules de _logs_ a tot arreu on es fan servir, i analitzar les dades de manera diferent. Serà una tasca a fer si fos necessària.

## Resum

- De nou, fent servir New Relic hem identificat problemes de rendiment en les consultes de l’aplicació.
- Hem proposat una solució basada en esdeveniments per a desacoblar l’aplicació i projectar les dades que necessitem.
- Hem implementat aquesta solució canviant només els camins de l’aplicació que són més importants pels nostres usuaris.
- No hem pogut assegurar el rendiment dels canvis realitzats, perquè l’aplicació és estacional.