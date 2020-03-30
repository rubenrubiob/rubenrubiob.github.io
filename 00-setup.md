# 0. Setup

## Install Symfony

https://symfony.com/doc/current/setup.html

## Add `.editorconfig` file

Based on: https://www.tomasvotruba.com/blog/2019/12/23/5-things-i-improve-when-i-get-to-new-repository/

```.editorconfig
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

## Change Symfony's kernel path

1. Move file `src/Kernel.php` -> `src/Infrastructure/Kernel/Symfony/Kernel.php`
2. Update `src/Infrastructure/Kernel/Symfony/Kernel.php` file:

	```diff
	-namespace App;
	+namespace Infrastructure\Kernel\Symfony;
	```

	```diff
	     public function getProjectDir(): string
	     {
	-        return \dirname(__DIR__);
	+        return \dirname(__DIR__).'/../../..';
	     }
	```

3. Update `composer.json` PSR-4 autoloading:

    ```diff
         "autoload": {
             "psr-4": {
    -            "App\\": "src/"
    +            "Infrastructure\\": "src/Infrastructure"
             }
         },
    ```

4. Update `bin/console` file namespace of `Kernel`:

    ```diff
     #!/usr/bin/env php
     <?php
     
    -use App\Kernel;
    +use Infrastructure\Kernel\Symfony\Kernel;
     use Symfony\Bundle\FrameworkBundle\Console\Application;
    ```

5. Update `public/index.php` file namespace of `Kernel`:

    ```diff
     <?php
     
    -use App\Kernel;
    +use Infrastructure\Kernel\Symfony\Kernel;
     use Symfony\Component\ErrorHandler\Debug;
     use Symfony\Component\HttpFoundation\Request;
    ```

6. Remove configuration of services' autowiring in `config/services.yaml`:

    ```diff
         _defaults:
             autowire: true      # Automatically injects dependencies in your services.
             autoconfigure: true # Automatically registers your services as commands, event subscribers, etc.
    -
    -    # makes classes in src/ available to be used as services
    -    # this creates a service per class whose id is the fully-qualified class name
    -    App\:
    -        resource: '../src/*'
    -        exclude: '../src/{DependencyInjection,Entity,Migrations,Tests,Kernel.php}'
    -
    -    # controllers are imported separately to make sure services can be injected
    -    # as action arguments even if you don't extend any base controller class
    -    App\Controller\:
    -        resource: '../src/Controller'
    -        tags: ['controller.service_arguments']
    -
    -    # add more service definitions when explicit configuration is needed
    -    # please note that last definitions always *replace* previous ones
    ```

# Add composer libraries as helpers

PHPUnit: https://phpunit.de/

```bash
composer require --dev phpunit/phpunit ^9
```

Infection: https://infection.github.io/

```bash
composer require --dev infection/infection
```

Psalm

```bash
composer require --dev vimeo/psalm
```

Coding Standard: Doctrine ( https://www.doctrine-project.org/projects/doctrine-coding-standard/en/6.0/reference/index.html )