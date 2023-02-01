---
date: 2022-11-30
---

# _Object calisthenics_
{:.no_toc}

* TOC
{:toc}

## Introducció

En grec clàssic, cal·listènia, (καλός-σθένος), vol dir, etimològicament, bellesa de la fortalesa. En general, la _cal·listènia_ és un «Exercici físic pensat per a contribuir al desenvolupament de la força i la gràcia en el moviment»[^1].

Ara bé, què és la «cal·listènia d'objectes» en el desenvolupament de _software_?

El sentit[^2] de la «cal·listènia d'objectes» en els llenguatges de programació orientats a objecte és millorar el codi. Concretament, per a incrementar el manteniment, la lectura, la comprensió i la capacitat de testatge del codi mitjançant una sèrie d'exercicis.

La cal·listènia d'objectes consisteix en nou regles. Com diu el seu mateix nom, són una sèrie d'exercicis, no un dogma.

## 1. Només un nivell d'indentació per mètode

Aquesta regla proposa que només hi hauria d'haver un nivell d'indentació per cadascun dels mètodes que tenim a una classe.

Suposem que tenim el següent mètode:

```php
final class ProcessVideoAudioElements
{
    public function execute(array $videoElements, array $audioElements): void
    {
        foreach($videoElements as $videoElement) {
            foreach ($audioElements as $audioElement) {
                $this->api->call($videoElement, $audioElement);
            }
        }
    }
}
```

Aplicant aquesta regla obtindríem el següent codi:

```php
final class ProcessVideoAudioElements
{
    public function execute(array $videos, array $audios): void
    {
         $this->executeApiCalls($videos, $audios);
    }
    
    public function executeApiCalls(array $videos, array $audios): void
    {
        foreach ($videos as $video) {
            $this->handleVideo($video, $audios);
        }
    }
    
    public function handleVideo(Video $video, array $audios): void
    {
        foreach($audios as $audio) {
            $this->api->call($video, $audio);
        }
    }
}
```

## 2. No fer servir `else`

Aquesta regla planteja no fer servir `else` en els condicionals de codi.

En tenim dues alternatives d'aplicació.

### Retorn ràpid (_early return_)

Si tenim el següent fragment de codi:

```php
public function login(string $user, string $password): Response
{
    if ($this->repo->isValid($user, $password)) {
        return new Response('ok', 200);
    } else {
        return new Response('error', 400);
    }
}
```

podríem aplicar la regla i obtenir:

```php
public function login(string $user, string $password): Response
{
    if ($this->repo->isValid($user, $password)) {
        return new Response('ok', 200);
    }
    
    return new Response('error', 400);
}
```

### Retorn parametritzat (_return parametrized_)

Amb el següent fragment de codi:

```php
public function link(string $status): string
{
    if ($status === 'subscribed') {
        $link = 'enabled';
    } else {
        $link = 'not-enabled';
    }
    
    return $link;
}
```

aplicant la regla, podríem obtenir-ne:

```php
public function link(string $status): string
{
    $link = 'not-enabled';

    if ($status === 'subscribed') {
        $link = 'enabled';
    }
    
    return $link;
}
```

## 3. Embolcallar totes les primitives

Aquesta regla ens commina a fer servir _Value Objects_ per a encapsular totes les primitives dins d'objectes amb el seu propi significat.

Per exemple, si tenim el següent codi:

```php
final class Order
{
    private string $id;
    
    private int $numLines;
    
    private float $totalAmount;
}
```

podríem aplicar la regla i obtenir-ne:

```php
final class Order
{
    private Id $id;
    
    private Quantity $numLines;
    
    private Money $totalAmount;
}
```

## 4. Col·leccions de primer nivell

Aquesta regla ens proposa encapsular tots els `array` en una classe pròpia.

Suposem que tenim el següent codi:

```php
final class Order
{
    /** Line[] */
    private array $lines;
    
    private function calculateTotalAmount(): void
    {
        foreach($this->lines as $line) {
            // Line may actually be anything!
        }
    }
}
```

El tipus de `$this->lines`, per molt que hi afegim el comentari, pot ser qualsevol cosa.

En canvi, podríem tenir una classe `LineCollection` que encapsuli aquests elements:

```php
final class LineCollection
{
    /** @var Line[] */
    private array $elements = [];
    private int $count = 0;
    
    public function add(Line $line): void
    {
        $this->elements[] = $line;
        $this->count++;
    }
}
```

La classe principal, doncs, quedaria:

```php
final class Order
{
    private LineCollection $lines;
    
    private function calculateTotalAmount(): void
    {
        foreach($this->lines as $line) {
            // Line can only be of type Line!
        }
    }
}
```

## 5. Només un punt per línia

Aquesta regla també es coneix com a «Llei de Demèter»: no s'ha de parlar amb estranys, perquè trenca l'encapsulació del codi.

Suposem que tenim les següents classes:

```php
final class Language
{
    private readonly Code $code;
    
    public function code(): Code
    {
        return $this->code;
    }
}

final class Audio
{
    private readonly Language $language;
    
    public function language(): Language
    {
        return $this->language;
    }
}

final class Response
{
    private Audio $audio;
    
    public function audio(): Audio
    {
        return $this->audio;
    }
}
```

Aleshores, és possible executar el següent codi:

```php
echo $response->audio()->language()->code();
```

Això no obstant, què passaria si alguns del mètode canviés el seu valor de retorn, per a retornar un `null`, per exemple? Es trencaria l'encapsulació i el codi deixaria de funcionar.

Aplicant la regla, podríem, doncs, canviar les classes `Audio` i `Response` perquè siguin:

```php
final class Audio
{
    private readonly Language $language;
    
    public function languageCode(): Code
    {
        return $this->language->code();
    }
}

final class Response
{
    private readonly Audio $audio;
    
    public function audioLanguageCode(): Code
    {
        return $this->audio->languageCode();
    }
}
```

Aleshores, podríem tenir la següent línia de codi que no trencaria l'encapsulació:

```php
echo $response->audioLanguageCode();
```

## 6. No abreviar

La regla parla per si mateixa[^3].

Suposem que tenim la següent classe:

```php
final class AudioStrResCol
{
    // ...
}
```

Què vol dir `AudioStrResCol`?
- `AudioStringResourceColumn`?
- `AudioStringResultColumn`?
- `AudioStringResultCollation`?
- `AudioStrongResultColumn`?

Realment, el nom de la classe hauria de ser:

```php
final class AudioStreamResponseCollection
{
    // ...
}
```

## 7. Mantenir totes les entitats petites

Els fitxers amb moltes línies de codi són difícils de llegir, de manera que aquesta regla proposa mantenir les classes entre 50 i 150 línies.

Això no obstant, aquesta regla no sempre es pot complir, donat que depèn molt del llenguatge de programació.

## 8. Cap classe amb més de dos atributs

Aquesta regla, igual que l'anterior, no sempre és fàcil de complir.

Suposem que tenim la següent classe:

```php
final class Customer
{
    private Id $id;
    
    private Email $email;
    private PasswordHash $passwordHash;
    
    private FirstName $firstName;
    private LastName $lastName;
    
    private Name $address;
    private City $city;
    private PostalCode $postalCode;
}
```

Aplicant la regla, podríem canviar el codi perquè quedés:

```php
final class Customer
{
    private Id $id;
    private UserData $userData;
}

final class UserData
{
    private AuthData $authData;
    private PersonalData $personalData;
}

final class AuthData
{
    private Email $email;
    private PasswordHash $passwordHash;
}

final class PersonalData
{
    private NameData $nameData;
    private AddressData $addressData;
}

final class NameData
{
    private FirstName $firstName;
    private LastName $lastName;
}

final class AddressData
{
    private Name $address;
    private CityData $cityData;
}

final class CityData
{
    private City $city;
    private PostalCode $postalCode;
}
```

## 9. No _getters_, _setters_, _properties_

Aquesta regla també es coneix com a «fes, no preguntis» (_tell, don't ask_). Es tracta de no preguntar (_get_) per a fer una acció (_set_), sinó encapsular el comportament en un objecte.

Per exemple, si tenim el següent codi:

```php
if ($cart->getAmount() >= 100) {
    $cart->setDiscount(20);
}
```

Podríem canviar-lo perquè fos:

```php
final class Cart
{
    private const DISCOUNT_THRESHOLD = 100;
    private const DISCOUNT_AMOUNT = 20;

    public function applyDiscount(): void
    {
        if ($this->totalAmount < self::DISCOUNT_THRESHOLD) {
            return; // Throw exception
        }
        
        $this->discount = self::DISCOUNT_AMOUNT;
    }
}
```

Aleshores, es podria cridar amb:

```php
$cart->applyDiscount();
```

## Resum

La «cal·listènia d'objectes» consisteix en una sèrie d'exercicis per a millorar el manteniment, la lectura, la comprensió i la capacitat de testatge del nostre codi.

No totes les regles són sempre aplicables, però fora bo tenir-les present com una guia a seguir.


[^1]: [Termcat: cal·listènia](https://www.termcat.cat/ca/diccionaris-en-linia/30/fitxa/NjQ3NTg5){:target="_blank"}

[^2]: Seguint a Heidegger, no ens hem de preguntar mai per la cosa, sinó pel sentit de la cosa.

[^3]: En general, no abrevieu mai, ni quan parleu ni quan escriviu. Què en fareu, dels 30 segons que guanyareu per abreviar durant tota la vostra vida?