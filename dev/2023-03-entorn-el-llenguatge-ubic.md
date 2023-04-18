---
date: 2023-03-06
---

# Entorn el llenguatge ubic

Un dels elements que trobo més interessant de DDD és el llenguatge ubic. Eric Evans, a «Domain-Driven Design
Reference»[^1] explica així la problemàtica que existeix quan els desenvolupadors i els experts de domini
col·laboren[^2]:

> Els experts de domini empren el seu argot mentre els membres tècnics de l'equip tenen el seu propi llenguatge per a debatre el domini en termes del disseny. La terminologia dels debats diaris està desconnectada de la terminologia present al codi (que, al cap i a la fi, és el producte més important d'un projecte de _software_). Fins i tot la mateixa persona fa servir un llenguatge oral diferent de l'escrit, de manera que les expressions més incisives del domini sorgeixen en una forma transitiva que mai es reflecteixen al codi, i que ni tan sols queden escrites enlloc.

Per a evitar aquestes discrepàncies i aquesta pèrdua d'informació, la solució és el llenguatge ubic:

> Amb el llenguatge ubic, el model no és una mera utilitat de disseny, sinó que esdevé integral per a tot el que els desenvolupadors i els experts de domini fan conjuntament.

Per tant,

> Empra el model com la base del llenguatge. Compromet l'equip a exercitar el llenguatge sense aturador en tota la comunicació dins de l'equip i al codi. Dins d'un _bounded context_, empra el mateix llenguatge en diagrames, escrits i, sobretot, oralment.

> Reconeix que un canvi en el llenguatge és un canvi en el model.

> Elimina les dificultats experimentant amb expressions alternatives, que reflecteixen models alternatius. Aleshores,
_refactoritza_ el codi, reanomenant classes, mètodes i mòduls per a adaptar-se al nou model. Resol qualsevol confusió sobre els termes en converses, de la mateixa manera que ens posem d'acord en el significat de les paraules d'ús comú.

Així doncs, és a partir del diàleg com aconseguim resoldre els problemes dins de l'enginyeria del _software_. El
llenguatge ubic es basa justament en el diàleg: hem de dialogar per a comprendre'ns i revelar les solucions entre tots
els participants en un projecte, siguin els experts de domini, siguin els desenvolupadors. Però, per a fer-ho, hem de
fer servir el mateix llenguatge i els mateixos conceptes dins del mateix context, sota pena de no comprendre'ns.
El llenguatge ubic proposa supeditar el codi al llenguatge, no a la inversa: el codi ha de
reflectir el llenguatge que fem servir per a resoldre el problema. Qualsevol canvi que es doni en el llenguatge obliga a
canviar el codi, tantes vegades com calgui.

Arran d'aquestes idees, hom esperaria trobar el codi en l'idioma del domini a tots els projectes on es diu que s'aplica
DDD. O, si més no, a qualsevol projecte que vulgui ser resolt exitosament, ja que el llenguatge ubic no és exclusiu de
DDD. Això no obstant, molta gent programa el seu domini en anglès, encara que el seu domini sigui en una altra llengua.
Per què?

Una de les respostes que he rebut és que és menys professional: ens sona estrany trobar paraules en català (o castellà)
al codi. És cert que hi ha exemples de webs importants de l'Administració que fallen sovint, on els missatges d'error
acostumen a mostrar-se en castellà o català. Però, més enllà d'aquests exemples, aquest argument sembla més aviat un
prejudici que no pas una raó de pes.

L'altre argument que he trobat tot sovint per a escriure el codi en anglès és que així permet que tothom l'entengui,
perquè «en anglès tothom s'entén». Encara que l'equip sigui local, si el codi es pica en anglès, en el futur es podria
incorporar algú que no parli l'idioma i comprendria el codi immediatament. És un argument vàlid si l'empresa té
projecció internacional. Això no obstant, el codi es fa en anglès encara que l'empresa sigui merament local i no passi
de cinc persones.

Suposem que, així i tot, volem escriure el codi en anglès. Què succeeix si els conceptes del domini són mots que no
tenen traducció a l'anglès? A català en tenim exemples, com ara «seny» o «vespre», però sembla difícil que mai emprem
aquestes paraules en un context tècnic. Ara bé, hi ha casos d'ús que són específicament locals i no tenen equivalent a
altres llengües, com ho poden ser els d'àmbit administratiu o legal. Per exemple, vaig treballar en un projecte que
emmagatzemava licitacions del sistema sanitari espanyol, on hi ha conceptes molt específics i tècnics. Vam fer el codi
en anglès, mentre que el client parlava català i expressava els conceptes en castellà o català. Com a resultat, vam
tenir problemes per a comprendre'ns, perquè havíem de traduir constantment mentre parlàvem amb ell.

El cert és que a la traducció sempre hi ha pèrdua d'informació —la traducció és una fusió d'horitzons, com va dir
Gadamer. Així doncs, el millor és sempre evitar traduir els conceptes del nostre domini per a resoldre el problema, i
plasmar-los en codi tal com en parlem. Si mai tenim algú a l'equip que no parla l'idioma, sempre podem ajudar-lo amb
taules de traducció de conceptes.

Ara bé, si decidim picar el codi en l'idioma del domini, ens sorgiran d'altres problemes. Fins on hem de fer servir
aquest idioma? Només al codi de domini? També a la capa d'infraestructura? I el CSS? I les rutes de l'aplicació? I què
passa si hem de barrejar aquest idioma amb l'anglès?[^3]

Si no tenim cura, el projecte pot créixer malament i esdevenir complex de mantenir. Però, si es fa bé, pot ajudar a
comprendre millor el projecte i a tenir diàlegs més fructífers amb els experts de domini. En contrast amb l'exemple
anterior, vaig treballar a un projecte on vam prendre la decisió de fer servir castellà en el codi, ja que era l'idioma
que emprava l'expert de domini. Es tractava d'un _e-commerce_ B2B que tenia un càlcul de preus complex, ja que el preu
d'un producte depenia dels descomptes associats que pogués tenir per a un client concret. Vam implementar repositoris de
lectura per a aquests casos, amb noms com `PrecioNetoReadRepository` o `PrecioPorGrupoDePrecioDeClienteReadRepository`.
Al principi es feia estrany veure aquesta barreja de castellà i anglès al codi. Això no obstant, ens hi vam acostumar
aviat, i va resultar en una comunicació immediata i senzilla amb l'expert de domini, perquè sempre ens referíem als
mateixos conceptes amb les mateixes paraules.

Amb això no vull dir que s'hagi de fer servir sempre l'idioma del domini, o que sigui millor o pitjor. No hi ha una
solució vàlida per a tots els casos, fer servir un idioma o un altre per al domini dependrà de molts factors, molts dels
quals venen imposats i no es poden canviar. El millor sempre és parlar-ho i acordar-ho amb l'equip. I trobo que,
almenys, cal plantejar la pregunta per a fer reflexionar.

[^1]: Evans, E. (2015). _Domain-Driven Design Reference. Definitions and Pattern Summaries_. [Publicació en línia](https://www.domainlanguage.com/wp-content/uploads/2016/05/DDD_Reference_2015-03.pdf){:target="_blank"}.
[^2]: Totes les traduccions són meves.
[^3]: En [aquest fil de Twitter](https://twitter.com/rubenrubiob/status/1170639007518248960){:target="_blank"} hi ha un debat interessant al respecte.