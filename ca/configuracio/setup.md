---
lang: ca
lang-ref: configuracion-setup
tags: [setup,php,symfony]
is-content-page: true
layout: content-page
order: 0
pull-request: 1
---

# Setup

* TOC
{:toc}

## 1. Instal·lar Symfony

Per a instal·lar Symfony, el millor és seguir la [guia oficial d'instal·lació](https://symfony.com/doc/current/setup.html).

## 2. Afegir el fitxer `.editorconfig`

Aquest fitxer serveix perquè tothom qui treballi al projecte tingui la mateixa configuració d'espaiat, tabulat, salts de línia i codificació —sempre que l'editor suporti aquest tipus de fitxers. D'aquesta manera, tothom té la mateixa configuració i evitem barrejar estils que en moltes ocasions depenen del sistema operatiu.

El contingut del fitxer està basat en la [guia de Tomas Votruba](https://www.tomasvotruba.com/blog/2019/12/23/5-things-i-improve-when-i-get-to-new-repository/).

```
# .editorconfig
root = true

[*]
charset = utf-8
end_of_line = lf
insert_final_newline = true
trim_trailing_whitespace = true
indent_style = space
indent_size = 4
```

## 3. _Doctrine coding standard_

Al projecte farem servir els [estils de codi de Doctrine](https://www.doctrine-project.org/projects/doctrine-coding-standard/en/8.1/reference/index.html), que són els més estrictes que hi ha a PHP. Per a instal·lar-los al projecte utilitzant [CodeSniffer](https://github.com/squizlabs/PHP_CodeSniffer), n'hi ha prou amb executar:

```bash
composer require --dev doctrine/coding-standard
```

A PHPStorm es configuren automàticament per a Mac i Linux, però es pot seguir [la seva guia](https://www.jetbrains.com/help/phpstorm/using-php-code-sniffer.html) si hi ha cap problema.

## 4. _Apache Pack_

Per a les proves d'aquest projecte s'utilitzarà Apache, de manera que cal instal·lar el [pack d'Apache](https://github.com/symfony/apache-pack) perquè funcionin les rutes de Symfony:

```
composer require apache-pack
```

## 5. Configurar Symfony per utilitzar arquitectura hexagonal

Seguint el que s'explica a la [introducció](https://rubenrubiob.github.io/ca/), farem servir la següent estructura de directoris per implementar arquitectura hexagonal:

```bash
├── src
│   ├── Application
│   ├── Domain
│   ├── Infrastructure
│   └── Ui
```

Així doncs, cal fer alguns canvis de configuració a l'esquema de directoris que per defecte ve amb la instal·lació de Symfony:

1. Eliminar arxius i directoris que no necessitarem:
	- `bin/phpunit`
	- `config/routes/annotations.yaml`
	- `src/Controller/`
	- `src/Entity/`
	- `src/Repository/`
2. Moure el fitxer `src/Kernel.php` a `src/Infrastructure/Kernel/Symfony/Kernel.php`
3. Actualitzar el fitxer `src/Infrastructure/Kernel/Symfony/Kernel.php` perquè quedi:

	```php
	<?php
	
	namespace Infrastructure\Kernel\Symfony;
	
	use Symfony\Bundle\FrameworkBundle\Kernel\MicroKernelTrait;
	use Symfony\Component\DependencyInjection\Loader\Configurator\ContainerConfigurator;
	use Symfony\Component\HttpKernel\Kernel as BaseKernel;
	use Symfony\Component\Routing\Loader\Configurator\RoutingConfigurator;
	
	class Kernel extends BaseKernel
	{
	    use MicroKernelTrait;
	
	    protected function configureContainer(ContainerConfigurator $container): void
	    {
	        $confDir = $this->getProjectDir();
	
	        $container->import($confDir.'/config/{packages}/*.yaml');
	        $container->import($confDir.'/config/{packages}/'.$this->environment.'/*.yaml');
	
	        $container->import($confDir.'/config/{services}.yaml');
	        $container->import($confDir.'/config/{services}_'.$this->environment.'.yaml');
	    }
	
	    protected function configureRoutes(RoutingConfigurator $routes): void
	    {
	        $confDir = $this->getProjectDir();
	
	        $routes->import($confDir.'/config/{routes}/'.$this->environment.'/*.yaml');
	        $routes->import($confDir.'/config/{routes}/*.yaml');
	
	        $routes->import($confDir.'/config/{routes}.yaml');
	    }
	
	    public function getProjectDir(): string
	    {
	        return \dirname(__DIR__).'/../../..';
	    }
	}
	
	```

4. Actualitzar l'_autoloading_ PSR-4 de `composer.json`:

    ```diff
         "autoload": {
             "psr-4": {
    -            "App\\": "src/"
    +            "Infrastructure\\": "src/Infrastructure"
             }
         },
    ```

4. Actualitzar el _namespace_ del _Kernel_ al fitxer `bin/console`:

    ```diff
     #!/usr/bin/env php
     <?php
     
    -use App\Kernel;
    +use Infrastructure\Kernel\Symfony\Kernel;
     use Symfony\Bundle\FrameworkBundle\Console\Application;
    ```

5. Actualitzar el _namespace_ del _Kernel_ al fitxer `public/index.php`:

    ```diff
     <?php
     
    -use App\Kernel;
    +use Infrastructure\Kernel\Symfony\Kernel;
     use Symfony\Component\ErrorHandler\Debug;
     use Symfony\Component\HttpFoundation\Request;
    ```

6. Moure la configuració de serveis de `config/services.yaml` a cada carpeta de l'arquitectura (de moment, només tenim `infrastructure`):

    ```diff
	-# This file is the entry point to configure your own services.
	-# Files in the packages/ subdirectory configure your dependencies.
	-
	-# Put parameters here that don't need to change on each machine where the app is deployed
	-# https://symfony.com/doc/current/best_practices/configuration.html#application-related-configuration
	-parameters:
	-
	-services:
	-    # default configuration for services in *this* file
	-    _defaults:
	-        autowire: true      # Automatically injects dependencies in your services.
	-        autoconfigure: true # Automatically registers your services as commands, event subscribers, etc.
	-
	-    # makes classes in src/ available to be used as services
	-    # this creates a service per class whose id is the fully-qualified class name
	-    App\:
	-        resource: '../src/'
	-        exclude:
	-            - '../src/DependencyInjection/'
	-            - '../src/Entity/'
	-            - '../src/Kernel.php'
	-            - '../src/Tests/'
	-
	-    # controllers are imported separately to make sure services can be injected
	-    # as action arguments even if you don't extend any base controller class
	-    App\Controller\:
	-        resource: '../src/Controller/'
	-        tags: ['controller.service_arguments']
	-
	-    # add more service definitions when explicit configuration is needed
	-    # please note that last definitions always *replace* previous ones
	+imports:
	+    - { resource: 'services/' }
    ```
    
7. Afegir el fitxer `config/services/infrastructure.yaml` amb el contingut següent:

	```php
	services:
	    _defaults:
	        autowire: true
	        autoconfigure: true
	
	    Infrastructure\:
	        resource: '../../src/Infrastructure/**'

	```

8. Eliminar la configuració d'entitats de Doctrine del fitxer `config/packages/doctrine.yaml`:

	```diff
	         auto_generate_proxy_classes: true
	         naming_strategy: doctrine.orm.naming_strategy.underscore_number_aware
	         auto_mapping: true
	-        mappings:
	-            App:
	-                is_bundle: false
	-                type: annotation
	-                dir: '%kernel.project_dir%/src/Entity'
	-                prefix: 'App\Entity'
	-                alias: App
	```

## 6. Llibreries

Pel desenvolupament del projecte utilitzarem algunes llibreries addicionals, com eines de test o analitzadors estàtics de codi.

Al final, afegirem unes comandes de `composer` per a poder executar aquestes eines de manera senzilla.

### 6.1. PHPUnit

Pels nostres tests, farem servir [PHPUnit](https://phpunit.de/):

```bash
composer require --dev phpunit/phpunit ^9
```

### 6.2. Infection

Per saber si els nostres tests funcionen o no, farem servir mutació de tests. A PHP, utilitzarem la llibreria [Infection](https://infection.github.io/):

```
composer require --dev infection/infection
```

### 6.3. Analitzadors estàtics de codi

PHP és un llenguatge feblement tipat, és a dir, les variables no tenen un tipus fixe. Això provoca molts errors a l'hora de programar. Per analitzar el nostre codi amb tipus correctes, podem fer servir llibreries. En aquest projecte, en farem servir dues que són complementàries [Psalm](https://psalm.dev/) i [PHPStan](https://phpstan.org).

#### 6.3.1. Psalm

L'instal·lem amb `composer`:

```bash
composer require vimeo/psalm:"dev-master"
```

I afegim el fitxer de configuració `psalm.xml` a l'arrel del nostre projecte:

```xml
<?xml version="1.0"?>
<psalm
    totallyTyped="false"
    errorLevel="1"
    resolveFromConfigFile="true"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns="https://getpsalm.org/schema/config"
    xsi:schemaLocation="https://getpsalm.org/schema/config vendor/vimeo/psalm/config.xsd"
>
    <projectFiles>
        <directory name="src" />
        <ignoreFiles>
            <directory name="vendor" />
            <directory name="src/Infrastructure/Kernel/Symfony" />
        </ignoreFiles>
    </projectFiles>
</psalm>
```

Per a analitzar el codi, simplement executarem:

```bash
vendor/bin/psalm
```

#### 6.3.2. PHPStan

L'instal·lem amb `composer`:

```bash
composer require phpstan/phpstan
```

I afegim el fitxer de configuració `phpstan.neon.dist` a l'arrel del nostre projecte:

```neon
parameters:
	level: 8
	paths:
		- src
		- tests
```

Per a analitzar el codi, simplement executarem:

```bash
vendor/bin/phpstan analyse
```

### _Scripts_ de `composer`

Al fitxer `composer.json` hi afegim:

```diff
         ],
         "post-update-cmd": [
             "@auto-scripts"
-        ]
+        ],
+        "infection": "vendor/bin/infection",
+        "phpstan": "vendor/bin/phpstan analyse",
+        "phpunit": "vendor/bin/phpunit",
+        "psalm": "vendor/bin/psalm"
     },
     "conflict": {
         "symfony/symfony": "*"
```

D'aquesta manera, podrem llençar des de consola les següents comandes per a executar les diferents llibreries:

```
composer infection
composer phpstan
composer phpunit
composer psalm
```