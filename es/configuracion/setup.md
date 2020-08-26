---
lang: es
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

## 1. Instalar Symfony

Para instalar Symfony, lo mejor es seguir la [guía oficial de instalación](https://symfony.com/doc/current/setup.html).

## 2. Añadir el fichero `.editorconfig`

Este fichero sirve para que todo aquel que trabaje en el proyecto tenga la misma configuración de espaciado, tabulación, saltos de línea y codificación —siempre y cuando el editor soporte este tipo de ficheros. De este modo, todo el mundo tiene la misma configuración evitamos mezclar estilos que dependen en muchas ocasiones del sistema operativo.

El contenido del fichero está basado en la [guía de Tomas Votruba](https://www.tomasvotruba.com/blog/2019/12/23/5-things-i-improve-when-i-get-to-new-repository/).

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

Para el proyecto utilizaremos los [estilos de código de Doctrine](https://www.doctrine-project.org/projects/doctrine-coding-standard/en/8.1/reference/index.html), que son los más estrictos que hay en PHP. Para instalarlos en el proyecto usando [CodeSniffer](https://github.com/squizlabs/PHP_CodeSniffer), basta ejecutar:

```bash
composer require --dev doctrine/coding-standard
```

En PHPStorm se configuran automáticamente para Mac y Linux, pero se puede seguir [su guía](https://www.jetbrains.com/help/phpstorm/using-php-code-sniffer.html) si hay algún problema.

## 4. _Apache Pack_

Como para las pruebas de este proyecto se utilizará Apache, para que funcionen las rutas de Symfony hay que instalar el [pack de Apache](https://github.com/symfony/apache-pack):

```
composer require apache-pack
```

## 5. Configurar Symfony para usar arquitectura hexagonal

Siguiendo lo que se explica en la [introducción](https://rubenrubiob.github.io/es/), usaremos la siguiente estructura de directorios para implementar arquitectura hexagonal:

```bash
├── src
│   ├── Application
│   ├── Domain
│   ├── Infrastructure
│   └── Ui
```

Así pues, hay que hacer algunos cambios de configuración en el esquema de directorios que por defecto viene en la instalación de Symfony.

1. Eliminar archivos y directorios que no necesitaremos:
	- `bin/phpunit`
	- `config/routes/annotations.yaml`
	- `src/Controller/`
	- `src/Entity/`
	- `src/Repository/`
2. Mover el fichero `src/Kernel.php` a `src/Infrastructure/Kernel/Symfony/Kernel.php`
3. Actualizar el fichero `src/Infrastructure/Kernel/Symfony/Kernel.php` para que quede:

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

4. Actualizar el _autoloading_ PSR-4 de `composer.json`:

    ```diff
         "autoload": {
             "psr-4": {
    -            "App\\": "src/"
    +            "Infrastructure\\": "src/Infrastructure"
             }
         },
    ```

4. Actualizar el _namespace_ del _Kernel_ en el fichero `bin/console`:

    ```diff
     #!/usr/bin/env php
     <?php
     
    -use App\Kernel;
    +use Infrastructure\Kernel\Symfony\Kernel;
     use Symfony\Bundle\FrameworkBundle\Console\Application;
    ```

5. Actualizar el _namespace_ del _Kernel_ en el fichero `public/index.php`:

    ```diff
     <?php
     
    -use App\Kernel;
    +use Infrastructure\Kernel\Symfony\Kernel;
     use Symfony\Component\ErrorHandler\Debug;
     use Symfony\Component\HttpFoundation\Request;
    ```

6. Mover la configuración de servicios de `config/services.yaml` a cada carpeta de la arquitectura (de momento, sólo tenemos `infrastructure`):

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
    
7. Añadir el fichero `config/services/infrastructure.yaml` con el siguiente contenido:

	```php
	services:
	    _defaults:
	        autowire: true
	        autoconfigure: true
	
	    Infrastructure\:
	        resource: '../../src/Infrastructure/**'

	```

8. Eliminar la configuración de entidades de Doctrine del fichero `config/packages/doctrine.yaml`:

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

## 6. Librerías

Para el desarrollo del proyecto utilizaremos algunas librerías adicionales, como herramientas de test o analizadores estáticos de código.

Al final, añadiremos unos comandos de `composer` para poder ejecutar estas herramientas fácilmente.

### 6.1. PHPUnit

Para escribir nuestros tests utilizaremos [PHPUnit](https://phpunit.de/):

```bash
composer require --dev phpunit/phpunit ^9
```

### 6.2. Infection

Para saber si nuestros tests funcionan o no usaremos mutación de tests. En PHP, lo haremos con la librería [Infection](https://infection.github.io/):

```
composer require --dev infection/infection
```

### 6.3. Analizadores estáticos de código

PHP es un lenguaje débilmente tipado, es decir, las variables no tienen un tipo fijo. Esto conduce a múltiples errores a la hora de programar. Para analizar nuestro código con los tipos correctos, podemos usar librerías. En este proyecto usaremos dos, que son complementarias: [Psalm](https://psalm.dev/) y [PHPStan](https://phpstan.org).

#### 6.3.1. Psalm

La instalamos con `composer`:

```bash
composer require vimeo/psalm:"dev-master"
```

Y añadimos el fichero de configuración `psalm.xml` a la raíz de nuestro proyecto:

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

Para realizar el análisis, simplemente ejecutaremos:

```bash
vendor/bin/psalm
```

#### 6.3.2. PHPStan

La instalamos con composer:

```bash
composer require phpstan/phpstan
```

Y añadimos el fichero de configuración `phpstan.neon.dist` a la raíz de nuestro proyecto:

```neon
parameters:
	level: 8
	paths:
		- src
		- tests
```

Para realizar el análisis, simplemente ejecutaremos:

```bash
vendor/bin/phpstan analyse
```

### _Scripts_ de `composer`

Al fichero `composer.json` añadimos:

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

Así, podremos lanzar desde consola los siguientes comandos para ejecutar las diferentes librerías:

```
composer infection
composer phpstan
composer phpunit
composer psalm
```