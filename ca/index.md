---
layout: page
lang: ca
lang-ref: home
---

# Introducció

* TOC
{:toc}

En aquest projecte intentaré mostrar el desenvolupament complet d'un projecte simple fent servir _Domain Driven Design_ —DDD— amb arquitectura hexagonal a PHP. L'objectiu principal és explicar com implementar les parts més importants de les diferents capes que formen part de l'arquitectura hexagonal. La intenció és que el desenvolupament del projecte serveixi com a guia de desenvolupament, de manera que la teoria sorgirà de la pràctica, i no a l'inrevés.

Així doncs, desenvoluparem una aplicació web per a gestionar un catàleg de llibres, amb la possibilitat de generar referències bibliogràfiques en diferents formats: APA, Chicago...

El codi del projecte estarà disponible al repositori [rubenrubiob/gestor-libros](https://github.com/rubenrubiob/gestor-libros).

## DDD i arquitectura hexagonal

L'objectiu d'aquest projecte no consisteix a explicar què és DDD. Qui vulgui més informació, hi ha una gran quantitat de llibres i recursos on s'explica en detall[^1]. Per tant, no ens preguntarem «què és DDD?»; més aviat, com Heidegger va fer a «Ésser i temps», canviarem la pregunta: en lloc de preguntar-nos pel ser de la cosa, ens preguntarem pel sentit de ser de la cosa.

És a dir, no ens preguntarem «què és DDD?», sinó «quin és el sentit de DDD?». Com el seu nom indica, el sentit és dissenyar el projecte, pensar-lo, enfocant-lo al _domini_ o _negoci_. El domini[^2] dicta com ha de comportar-se la nostra aplicació. Entre altres coses, conté:

- Els conceptes del domini que hem de modelar.
- Les regles o restriccions que s'imposen als nostres models.
- Les accions o casos d'ús de la nostra aplicació.

Cal remarcar que, en aquest punt, no ens preocupem sobre infraestructura, i això és important: no pensem com emmagatzemarem les dades, ni el _framework_ que farem servir o quines tecnologies del _frontend_ utilitzar. El que ens importa és com s'ha de comportar la nostra aplicació, independentment de com s'implementi, res més. Aquí veiem l'origen del nom: Domain-Driven Design, disseny enfocat al domini.

Fent una simple analogia, el domini seria la partitura d'una simfonia, mentre que la infraestructura és una d'entre les moltes interpretacions que n'hi poden haver.

Ara bé, DDD defineix com pensar el projecte, però no com implementar-lo en codi, donat que hi ha diferents maneres de fer-ho. A aquest projecte farem servir arquitectura hexagonal, també coneguda com a arquitectura de ports i adaptadors. Aquest patró proposa una separació per capes: tindrem una capa que és el domini, a la qual s'accedeix amb uns adaptadors, que poden intercanviar-se fàcilment. Així doncs, se separa el domini de la infraestructura, de manera que encaixa amb DDD.

Per exemple, un port podria ser la interfície d'un repositori de lectura d'un producte. A aquest port hi podríem connectar diferents adaptadors, que són les implementacions concretes: un adaptador podria connectar-se a la base de dades per a obtenir el producte, un altre el podria llegir des d'un fitxer, un altre el podria obtenir d'una API externa... A la capa de domini no ens importa quin adaptador es faci servir, són intercanviables. Aquesta explicació pot ser una mica teòrica, però veurem exemples d'ús al llarg del desenvolupament del projecte.

Per a aquest projecte, tindrem les següents capes:

![](/images/general/capas-hexagonal.png)

- `Domain` (Domini): on resideixen els models que representen el negoci.
- `Application` (Aplicació): conté els serveis d'aplicació, que modelen els casos d'ús de la nostra aplicació.
- `Infrastructure` (Infraestructura): conté les implementacions concretes dels serveis.
- `Ui` (per _User Interface_): és una capa d'infraestructura, que separem per comoditat. S'hi troben els punts d'entrada de l'aplicació: per peticions HTTP, per línea d'ordres...

Cada capa només pot accedir als elements de la seva capa i als de les capes anteriors. Així doncs, des de `Ui` podem cridar a la capa `Application`, però no ho podem fer des de `Domain`.

Al projecte també hi aplicarem els principis SOLID[^3], que serveixen per a obtenir un _software_ robust. També els anirem veient al llarg del desenvolupament del projecte.

## Llenguatge ubic i traducció

El domini ha de venir definit pels experts de producte, siguin del nostre equip, sigui un client extern. Seguint DDD, el llenguatge ha de ser ubic, és a dir, els conceptes del domini s'han de fer servir a tot arreu del projecte. Per exemple, si els experts de negoci anomenen «grup» al que es coneix com a «categoria», no hem de cometre l'error de «traduir-lo» com a «categoria», sinó que hem de mantenir el nom que fan servir els experts, en aquest cas, «grup», donat que ells són els que coneixen el negoci. Com explica Gadamer a «Veritat i mètode», a la traducció sempre hi ha una pèrdua del sentit original del text.

D'això es desprèn, com a conseqüència, que tampoc hem de traduir a l'anglès els conceptes del domini que fem servir al nostre codi font. Aquest és un [debat interessant](https://twitter.com/ProjectPolly/status/1169877299337945090) i no hi ha una solució universal aplicable a tots els projectes. Depèn del projecte en si, de l'equip que el desenvoluparà, de l'àmbit en què es desenvolupa...

Com a exemple, en aquest projecte els conceptes del domini seran en castellà[^4], i la resta, en anglès. Per exemple, tindrem el concepte «Libro», i podríem tenir una interfície anomenada `LibroRepository`. Al principi ens pot resultar estrany però, com tot, només ens hi hem d'acostumar.

## Definició

La nostra aplicació web serà multiidioma, és a dir, estarà disponible en més d'un idioma: castellà, català i anglès. Per tant, hem de tenir en compte no només els literals estàtics del web, sinó que els models que farem servir també hauran de ser traduïbles.

En aquest projecte, primer crearem la funcionalitat per a emmagatzemar la informació que farem servir després per a generar la bibliografia: llibres i autors. Com que l'objectiu del projecte és didàctic, no implementarem un sistema complex que contingui totes les casuístiques del funcionament bibliogràfic; ens perdríem en detalls que només enfosquirien l'explicació.

### Autor

Així doncs, tindrem autors, dels quals n'emmagatzemarem la informació bàsica:

- Biografia: any de naixement i mort.
- Apel·latiu: nom, cognoms i pseudònim de l'autor.

Per a aquestes dades, tindrem les següents restriccions:

- Biografia:
    - Pot ser que no sapiguem la data exacta de naixement o mort de l'autor, com en el cas d'Homer.
    - Per a autors vius, només tindrem la data de naixement.
    - En el cas en què coneguem ambdues dates, la data de naixement ha de ser anterior a la de mort.
    - Podem tenir dates anteriors a la nostra era (a.C.), tant pel naixement com per la mort.
    - L'any 0 no existeix al nostre calendari.
- Apel·latiu:
    - Ha de ser traduïble. Safo té el mateix nom en català i en castellà, però no en anglès, on s'escriu _Sappho_.
    - Pot ser que no tinguem cognom de l'autor. Podem fer servir aquest camp pel denominatiu d'alguns autors, com Cristina de Pizan.
    - El pseudònim és el nom pel qual es coneix l'autor. Per exemple, el pseudònim d'Arístocles d'Atenes és Plató.

### Llibre

Per a la nostra aplicació, un llibre és sempre d'un autor. No contemplarem, per ara, els llibres escrits per diversos autors, ni llibres amb autor anònim. Tampoc no contemplarem l'opció que un llibre pugui estar recollit en altres edicions, com succeeix sovint amb obres antigues. Aquestes casuístiques podrien ser afegides al futur.

Els llibres sempre tenen edicions, de manera que desarem per una banda la informació bàsica del llibre com a obra, i per l'altra la informació d'una edició. Així doncs, tindrem la següent informació:

- Llibre:
    - El títol i el subtítol —que pot no existir— seran traduïbles, igual que succeïa amb el nom de l'autor.
    - L'any de publicació del llibre pot ser qualsevol, no cal que coincideixi amb la vida de l'autor, donat que podem trobar obres pòstumes.
- Edició:
    - D'una edició d'un llibre en desarem la ciutat, l'any de publicació, l'editorial, l'idioma del llibre i el format.
    - Tindrem diferents formats d'edició preestablerts: llibres de tapa dura, de butxaca...

### Pantalles de l'aplicació

A la nostra aplicació tindrem un gestor —o _backoffice_— des d'on un usuari podrà afegir tant autors com llibres.

La interfície de gestió d'un autor és la següent:

![](https://raw.githubusercontent.com/rubenrubiob/book-manager/master/docs/wireframe/backend-autor.png)

Per a afegir traduccions primer cal crear l'autor, la informació del qual s'emmagatzemarà en l'idioma que l'usuari estigui visualitzant en aquell moment.

La interfície de gestió d'un llibre és la següent:

![](https://raw.githubusercontent.com/rubenrubiob/book-manager/master/docs/wireframe/backend-libro.png)

Igualment, tant les traduccions com les edicions s'afegeixen al llibre un cop aquest ha estat creat.

## Eines

Aquest projecte el desenvoluparem amb el següent entorn:

- PHP 7.4
- Symfony 5.1
- MySQL 5.7
- ElasticSearch 6.8
- Apache 2.4

[^1]: Per a entrar a fons en DDD, tenim els [llibres d'Eric Evans](https://domainlanguage.com/ddd/reference/), qui va definir per primer cop el concepte, i els [de Vaugh Vernon](https://www.informit.com/authors/bio/5e9dfebf-9550-4a11-800d-35a57c5fa11e). Específic per a PHP, tenim el llibre [«Domain-Driven Design in PHP»](https://leanpub.com/ddd-in-php).

[^2]: No m'agrada el nom «negoci», perquè té la connotació que tot el que desenvolupem ha de tenir com a objectiu un benefici, duu implícit un esperit capitalista. Potser algú hauria d'estudiar com el capitalisme subjau al desenvolupament de _software_, on tot el que es programa ha de ser productiu; si no ho és, es considera invàlid. Farem servir el nom «domini» que, per contra, crec que no és prou clar.

[^3]: Els va definir [originalment Robert C. Martin](http://butunclebob.com/ArticleS.UncleBob.PrinciplesOfOod). En trobem una bona explicació en castellà  al [blog d'Asier Marqués](https://asiermarques.com/2018/principios-solid/) i al [blog de Fran Iglesias](https://franiglesias.github.io/principios-solid/).

[^4]: Com que aquest projecte s'explica tant en castellà com en català, es fan servir els noms en castellà per evitar haver de mantenir dos repositoris amb el mateix codi, on només hi canviessin els noms de les classes.