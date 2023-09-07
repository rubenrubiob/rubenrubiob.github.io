---
date: 2023-09-07
---

# Servir un _stream_ de fitxer a Symfony
{:.no_toc}

* TOC
{:toc}

## Introducció

Tal com s'explica a [The Twelve-Factor App](https://12factor.net/backing-services), és bona pràctica tractar tots els
servis de suport com a recursos enllaçats a la nostra aplicació. Això inclou també els _assets_ binaris, és a dir, els
fitxers que els nostres usuaris pugen a la nostra aplicació. Entre altres beneficis, seguir aquesta regla permet escalar
horitzontalment la nostra aplicació.

PHP té un ecosistema molt ric, de manera que existeixen llibreries que ens permeten treballar amb sistema
d'emmagatzematge de fitxers de manera transparent. La més popular
és [Flysystem](https://flysystem.thephpleague.com/docs/), que té integració amb diferents serveis d'emmagatzematge com
Amazon S3, Google Cloud Storage, Azure Blob Storage o SFTP.

Això no obstant, podem trobar-nos amb problemes quan treballem amb _assets_ privats i hem de servir-los a alguns usuaris
concrets. Com el nostre emmagatzematge és públic, no podem servir els _assets_ directament, ni tampoc no podem donar
accés directe al nostre servei d'emmagatzematge.

Una opció és
emprar [URL presignats](https://docs.aws.amazon.com/AmazonS3/latest/userguide/ShareObjectPreSignedURL.html). Aquests URL
són accessibles únicament durant un període de temps, després del qual deixen de
funcionar. [Flysystem permet generar aquests URL per a alguns adaptadors](https://flysystem.thephpleague.com/docs/usage/temporary-urls/).

Ara bé, no tots els adaptadors permeten generar aquests URL presignats, de manera que necessitem una alternativa. Una
possibilitat seria, per exemple, descarregar el fitxer remot en local, per a servir-lo posteriorment com
un `BinaryFileResponse`, passant-hi el directori sencer dins del servidor web. Aquesta opció, emperò, no és òptima ni en
temps ni en memòria, ja que el fitxer s'ha de descarregar dos cops: un al servidor, i un altre al client.

Com a alternativa, podem fer
servir [`StreamedResponse`](https://symfony.com/doc/current/components/http_foundation.html#streaming-a-response) de
Symfony.

## `StreamedResponse`

Symfony ofereix un tipus de `Response` especial, `StreamedResponse`, que permet servir blocs de _bytes_ als clients.
Necessita una funció interna que itera sobre el fluxe de dades i el serveix.

Per al nostre cas, podem obtenir un _stream_ de dades del fitxer que es troba al servei d'emmagatzematge, i servir-lo
directament dins d'un `StreamedResponse`, de manera que la nostra aplicació només actuarà de _proxy_.

Un _stream_ és un tipus `resource` a PHP.
L'[API de Flysystem ens proporciona el mètode `readStream`](https://flysystem.thephpleague.com/docs/usage/filesystem-api/)
per a obtenir l'_stream_ per a un fitxer concret.

Un cop tenim l'_stream_ del fitxer, podem servir-lo dins d'un `StreamedResponse`. Hi ha diverses opcions per a fer-ho,
totes semblants en consum de memòria i de temps.

### Opció 1: `fpasstrhu`

L'opció més simple és emprar `fpasstrhu`, una funció del llenguatge que emet totes les dades pendents de llegir en un
_stream_.

Una implementació podria ser:

```php
$stream = getStreamSomehow(); // For example, with Flysystem::readStream();

return new StreamedResponse(
    function () use ($stream) {
        fpassthru($stream);
    },
    Response::HTTP_OK,
    [
        'Content-Transfer-Encoding', 'binary',
        'Content-Type' => 'image/jpeg',
        'Content-Disposition' => 'attachment; filename="attachment.jpg"',
        'Content-Length' => fstat($stream)['size'],
    ]
);

```

### Opció 2: `stream_copy_to_stream`

La segona opció consisteix a fer servir la funció `stream_copy_to_stream`, que fa exactament el que el seu nom indica:
copiar el contingut d'un _stream_ a un altre.

En aquest cas, però, necessitem un _stream_ de destí on copiar les dades. Com que estem servint una resposta, podem fer
servir l'_stream_ `php://output`.

Per tant, podríem tenir la següent implementació:

```php
$stream = getStreamSomehow(); // For example, with Flysystem::readStream();

return new StreamedResponse(
    function () use ($stream) {
        $outputStream = fopen('php://output', 'wb');
        
        stream_copy_to_stream($stream, $outputStream);
    },
    Response::HTTP_OK,
    [
        'Content-Transfer-Encoding', 'binary',
        'Content-Type' => 'image/jpeg',
        'Content-Disposition' => 'attachment; filename="attachment.jpg"',
        'Content-Length' => fstat($stream)['size'],
    ]
);

```

### Opció 3: `fread`

La darrera opció empra la funció `fread` per a emetre dades des d'un _stream_.

Una implementació podria ser la següent, llegint una mida arbitrària de 1024 _bytes_ per a cada iteració:

```php
$stream = getStreamSomehow(); // For example, with Flysystem::readStream();

return new StreamedResponse(
    function () use ($stream) {
        while (! feof($stream)) {
            echo fread($stream, 1024);
        }
        fclose($stream);
    },
    Response::HTTP_OK,
    [
        'Content-Transfer-Encoding', 'binary',
        'Content-Type' => 'image/jpeg',
        'Content-Disposition' => 'attachment; filename="attachment.jpg"',
        'Content-Length' => fstat($stream)['size'],
    ]
);

```

## Resum

- Hem llistat els motius i beneficis de tractar el nostre sistema d'emmagatzematge de fitxers com a recursos enllaçats a la nostra aplicació.
- Hem vist el problema de servir fitxers privats des d'un sistema d'emmagatzematge.
- Hem explicat l'alternativa de generar URL temporals, que no funcionen a tots els casos.
- Finalment, hem implementat tres alternatives per a servir fitxers des d'un sistema de fitxers aprofitant `StreamedResponse` de Symfony.