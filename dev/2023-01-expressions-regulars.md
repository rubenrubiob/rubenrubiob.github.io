---
date: 2023-01-13
---

# Expressions regulars
{:.no_toc}

* TOC
{:toc}

## Introducció

Les expressions regulars són una eina per a processar i analitzar text. Consisteixen en una seqüència de caràcters que
coincideixen amb un patró regular.

Les expressions regulars són una eina complexa de fer servir, ja que no són gaire llegibles; però, alhora, són
potentíssimes.

![](https://imgs.xkcd.com/comics/regular_expressions.png){: width="1000" }

El format de les expressions regulars POSIX, el més estès, és el següent:

`/regex/modifiers`

Les barres (`/`) son el delimitador de l'expressió regular, no en formen part. Es pot fer servir algun altre caràcter,
com `#`. Si es vol fer una coincidència del delimitador dins de l'expressió regular, cal escapar el caràcter.

En aquest post veurem les parts bàsiques que formen part de les expressions regulars. A més proposarem uns exercicis per
a practicar.

![](/images/dev/regular-expression/feelings-of-power.png){: width="400" }

## Components de les expressions regulars

### Àncores

- `^`: coincideix amb l'inici de la línia o el text.
- `$`: coincideix amb el final de la línia o el text.

### Context per defecte

En el context per defecte, hi ha una `i` (`and`) implícita entre els seus elements.

- `.`: coincideix amb qualsevol caràcter.
- `\d`: coincideix amb qualsevol dígit.
- `\D`: coincideix amb qualsevol no dígit.
- `\s`: coincideix amb qualsevol caràcter d'espai en blanc (`␣`, `\t`, `\r`, `\n`).
- `\S`: coincideix amb qualsevol caràcter que no sigui d'espai en blanc (`␣`, `\t`, `\r`, `\n`).
- `\w`: coincideix amb qualsevol caràcter o caràcters que sigui una paraula.
- `\W`: coincideix amb qualsevol caràcter o caràcters que no sigui una paraula.
- `\`: fa literal qualsevol caràcter.
- `-`: coincideix amb un guió.

### Classes de caràcters `[]`

Les classes de caràcters es construeixen amb claudàtors (`[]`). Hi ha una `o` (`or`) implícita entre els seus elements.

- `[^...]`: classe de caràcters negada: coincideix amb un caràcter no llistat a la classe.
- `[a-z]`: coincideix amb un caràcter en el rang a-z.
- `[^a-z]`: coincideix amb un caràcter no present en el rang a-z.
- `[0-9]`: coincideix amb un caràcter en el rang 0-9.
- `[^0-9]`: coincideix amb un caràcter no present en el rang 0-9.
- `-`: al principi o final d'una classe de caràcters, coincideix amb un guió literal; en qualsevol altra posició,
  serveix per a construir rangs.

### Constructors de grup `()`

Els constructors de grup es construeixen amb parèntesis (`()`). Hi ha una `i` (`and`) implícita entre els seus
elements.

- `(a|b)`: coincideix amb `a` o `b`. És equivalent a `[ab]`.
- `(?:...)`: grup no capturant.
- `(?'name'...)`: grup de captura amb nom
- `(?=...)`: _positive lookahead_, mira coincidències endavant del grup, sense consumir caràcters.
- `(?!...)`: _negative lookahead_, mira coincidències negatives endavant del grup, sense consumir caràcters.
- `(?<=...)`: _positive lookbehind_, mira coincidències enrere del grup, sense consumir caràcters.
- `(?<!...)`: _negative lookbehind_, mira coincidències negatives enrere del grup, sense consumir caràcters.

### Quantificadors

Afecten el caràcter, classe o grup previ.

- `a?`: 0 o 1 coincidències de a.
- `a*`: 0 o més coincidències de a, sense límit.
- `a+`: 1 o més coincidències de a, sense límit.
- `a{3}`: exactament 3 coincidències de a.
- `a{3,}`: 3 o més coincidències de a, sense límit.
- `a{3,7}`: entre 3 i 7 coincidències de a.

### Modificadors

Afecten el comportament de tota l'expressió regular.

- `g`: evita que el motor d'expressions regulars acabi l'anàlisi després de la primera coincidència.
- `m`: les àncores `^` i `$` coincideixen amb el principi i el final de cada línia, no amb el principi i el final del
  text.
- `i`: fa les coincidències independents de majúscules i minúscules.
- `x`: fa que el motor ignori tots els espais en blanc, i permet tenir comentaris començant per `#`.

## Exercicis

Es pot fer servir l'eina [Regex101](regex101.com) per a verificar les respostes.

### Exercici 1

Coincidir els espais en blanc al final d'una línia.

Text a analitzar:

```
dos_espais_en_blanc_al_final  
un_espai_en_blanc_al_final 
```

Coincidències:
- Línia 1: `[␣␣]`
- Línia 2: `[␣]`

### Exercici 2

Coincideix el número i guió del principi de cadascuna de les línies.

Text a analitzar:

```
1.company.user.2022
2.company.videos.2022
10.company.categories.2022
20-company.lists.2022
```

Coincidències:
- Línia 1: `1.`
- Línia 2: `2.`
- Línia 3: `10.`
- Línia 4: no coincideix

### Exercici 3

Coincideix una data en el format `dd/mm/yyyy` o `dd/mm/yy` dins d'un text. La data pot estar a qualsevol posició. No cal
validar que la data sigui correcta. Pista: fer servir constructors de grup.

Text a analitzar:

```
Avui és 01/01/2022.
Demà será 02/01/22, segon dia de l'any.
```

Coincidències (fent servir grups):
- Línia 1: `dia: 01, mes: 01, any: 2022`
- Línia 2: `dia: 01, mes: 01, any: 22`

### Exercici 4

Coincideix un URI de la forma `/productes/{categoria}/{subcategoria}/`. El text entre claus (`{}`) és opcional. La barra
del final, també. L'URI només pot contenir lletres en minúscula, números i barres.

Text a analitzar:

```
/productes/
/productes/categoria-1/
/productes/categoria-1/subcategoria-1/
/productes/categoria-2
/productes/categoria-2/subcategoria-3
```

Coincidències
- Línia 1: `/productes/`
- Línia 2: `/productes/categoria-1/`
- Línia 3: `/productes/categoria-1/subcategoria-1/`
- Línia 4: `/productes/categoria-2`
- Línia 5: `/productes/categoria-2/subcategoria-3`

### Exercici 5

Coincideix un URI de la forma `/productes/{categoria}/`, que no coincideix amb un URI de la
forma `/productos/favoritos/`. El text entre claus (`{}`) és opcional. La barra del final, també. L'URI només pot
contenir lletres en minúscula, números i barres.

Text a analitzar:

```
/productes/
/productes/categoria-1/
/productes/categoria-2
/productes/favorits/
```

Coincidències:

- Línia 1: `/productes/`
- Línia 2: `/productes/categoria-1/`
- Línia 3: `/productes/categoria-2`
- Línia 4: no coincideix.

### Exercici 6

Coincideix la part prèvia a `http://` o `https://` dels URL següents.

Text a analitzar:

```
/local/url/path/http://www.example.com/path/
/local/url/path/https://www.example.com/path/
/local/url/path/ftp://ftp.example.com/path/
```

Coincidències:

- Línia 1: `/local/url/path/`
- Línia 2: `/local/url/path/`
- Línia 3: no coincideix.

## Referències

- [_Mastering regular expressions_, de Jeffrey Friedl](http://regex.info/book.html)
- [_Writing better regular expressions in PHP_](https://php.watch/articles/php-regex-readability)
- [Autoregex](https://www.autoregex.xyz/): _Effortless conversions from English <-> Regex_
- [Regexr](https://regexr.com/): _RegExr is an online tool to learn, build, & test Regular Expressions (RegEx / RegExp)._
- [iHateRegex](https://ihateregex.io/).