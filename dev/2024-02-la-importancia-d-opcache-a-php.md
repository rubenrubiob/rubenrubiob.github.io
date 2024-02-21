---
date: 2024-02-21
---

# La importància d'OPcache a PHP

<p style="font-size: 18px; margin: 0 0 -15px 0; font-style: italic">Dimecres, 20 de desembre de 2023, 10.08 h</p>
__Client__: «La nostra aplicació en PHP i Symfony té problemes de rendiment. El web és lent i triga molt a carregar algunes pàgines. De vegades, es congela i no es pot fer servir. Estem rebent queixes dels nostres usuaris: necessiten el web per a treballar, però no poden fer-ho.»

<p style="font-size: 18px; margin: 0 0 -15px 0; font-style: italic">Dimecres, 20 de desembre de 2023, 16.57 h</p>
__Desenvolupador__: «Hola! Ho puc revisar i resoldre, sense cap problema. Això no obstant, estic de vacances les dues pròximes setmanes, ho revisaré quan torni. Bon Nadal i bon any nou!»

<p style="font-size: 18px; margin: 0 0 -15px 0; font-style: italic">Dilluns, 8 de gener de 2024, 07.43 h</p>
__Client__: «Com va l'optimització? Els meus usuaris no van poder tenir vacances: van haver de compensar hores per tot el temps perdut treballant amb el nostre web. S'estan començant a enfadar. He rebut notes amb amenaces a la meva bústia.»

<p style="font-size: 18px; margin: 0 0 -15px 0; font-style: italic">Dilluns, 8 de gener de 2024, 13.19 h</p>
__Desenvolupador__: «Abans de res, bon any nou! Comprenc els teus usuaris: és normal posar-se nerviós quan no tens vacances. Haurien de prendre's la seva feina menys seriosament. Al cap i a la fi, no heretaran pas l'empresa. Per cert, encara mires la bústia? Qui envia cartes avui dia?»

«Deixa'm donar un cop d'ull a la vostra aplicació. Primer, necessitem mètriques per a saber què està succeint. Hem d'observar quines pàgines van lentes, quines són les que més es fan servir... Sense això, estarem guiant-nos per sensacions, no pas per dades. Acabo d'instal·lar New Relic per a analitzar el sistema. Haurem de tenir prou dades per a extreure'n conclusions en una setmana.»

<p style="font-size: 18px; margin: 0 0 -15px 0; font-style: italic">Dilluns, 15 de gener de 2024, 08.56 h</p>
__Client__: «Ja ha passat una setmana. Has pogut veure res que es pugui millorar amb les mètriques? Els usuaris estan perdent la paciència, i m'estic començant a esporuguir. Ahir vaig trobar trencades les finestres del meu cotxe, amb una nota a dins: els usuaris exigeixen temps de resposta de menys d'1 segon. En cas contrari, tot esdevindrà més seriós.»

<p style="font-size: 18px; margin: 0 0 -15px 0; font-style: italic">Dilluns, 15 de gener de 2024, 10.25 h</p>
__Desenvolupador__: «He de dir que és una petició raonable. Qualsevol web hauria de tenir temps de resposta de menys d'1 segon de mitjana. Potser podries considerar aquesta situació com un punt  d'inflexió a la teva vida i deixar de fer servir el cotxe? Caminar és més sa, i et sentiràs millor.»

«Veig que hi ha dues pàgines que suposen el 50% de la càrrega del sistema. Ambdues són de consulta, és a dir, es limiten a mostrar dades als usuaris. Veig que fan servir Doctrine, un ORM, així que ens trobem davant del famós problema N+1. Però no pateixis! Lluitaré i venceré: hauria de ser possible _cachejar_ les dades a Redis, de manera que evitem consultar la base de dades cada vegada per a obtenir les mateixes dades, que ja estaran calculades en memòria. Aguanta!»

<p style="font-size: 18px; margin: 0 0 -15px 0; font-style: italic">Divendres, 19 de gener de 2024, 02.05 h</p>
__Client__: «Com va la millora del rendiment? La meva família i jo estem "passant un temps" amb alguns dels usuaris ~~més enfadats~~ ~~més penjats~~ de la nostra aplicació. Saps què és el pitjor de tot? Que no paren de mostrar-me com de lenta i inutilitzable és. Empatitzo amb ells. Fes-me saber com va la millora com més aviat millor.»

<p style="font-size: 18px; margin: 0 0 -15px 0; font-style: italic">Divendres, 19 de gener de 2024, 12.46 h</p>
__Desenvolupador__: «M'agrada saber que estàs gaudint d'alguns dies de descans. T'ho mereixes! Mentrestant, he optimitzat les dues pàgines més lentes: ara fan servir projeccions, i tots els models que necessiten estan _cachejats_ a Redis. Això no obstant, no hi veig gaire millora, els temps de resposta continuen per sobre del segon. Però bé, és divendres migdia, ja ho miraré dilluns a primera hora. Bon cap de setmana!»

<p style="font-size: 18px; margin: 0 0 -15px 0; font-style: italic">Dissabte, 20 de gener de 2024, 03.23 h</p>
__Client__: «S'HAN ENDUT LA MEVA FILLA. M'HAN FET ACOMIADAR-ME D'ELLA. JA NO SÉ QUÈ ESTÀ PASSANT.»

<p style="font-size: 18px; margin: 0 0 -15px 0; font-style: italic">Dilluns, 22 de gener de 2024, 12.28 h</p>
__Desenvolupador__: «Creixen rapidíssim, ni te n'adones! És bo deixar-los espai i que cometin els seus propis errors. Bé, ja ho he trobat: sembla que vam oblidar activar OPcache al servidor. L'he activat a les 11.45, mira com es redueixen els temps de resposta:»

![Temps de resposta](/images/dev/opcache/temps-de-resposta.png)

«Aquí està el gràfic amb la càrrega del sistema després d'activar l'OPcache:»

![Temps de transaccions](/images/dev/opcache/transaccions.png)

«I aquí el gràfic amb l'AppIndex:»

![Appindex](/images/dev/opcache/appindex.png)

«Problema resolt!»

<p style="font-size: 18px; margin: 0 0 -15px 0; font-style: italic">Dilluns, 22 de gener de 2024, 12.29 h</p>
__Client__: QUÈ COI ÉS OPCACHE?

<p style="font-size: 18px; margin: 0 0 -15px 0; font-style: italic">Dilluns, 22 de gener de 2024, 15.07 h</p>
__Desenvolupador__: «PHP és un llenguatge interpretat. Això vol dir que cada vegada que un _script_ és executat en una petició, s'ha de compilar a _bytecode_, a codi màquina, és a dir, al codi que l'ordinador comprèn. Però a producció el codi només canvia cada cop que es fa un desplegament, de manera que aquesta compilació es pot _cachejar_ per a totes les peticions. Això és el que fa OPcache. Per cert, hauries de revisar la teva tecla de bloqueig de majúscules, sembla que està encallada.»

<p style="font-size: 18px; margin: 0 0 -15px 0; font-style: italic">Dilluns, 29 de gener de 2024, 11.50 h</p>
__Desenvolupador__: «Aquí tens la factura. Almenys podries haver-me agraït la feina feta, no m'hi he estat pas poc temps.»