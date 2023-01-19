# Llibreries i eines per a PHP

PHP és un llenguatge amplament utilitzat. Té una comunitat i ecosistema enormes. A continuació llistem algunes
llibreries, eines i comprovacions a fer en integració contínua, algunes específiques per a Symfony.

## Llibreries

- [brick/date-time](https://github.com/brick/date-time): Date and time library for PHP
- [thecodingmachine/safe](https://github.com/thecodingmachine/safe): All PHP functions, rewritten to throw exceptions instead of returning false
- [azjezz/psl](https://github.com/azjezz/psl): PHP Standard Library - a modern, consistent, centralized, well-typed, non-blocking set of APIs for PHP programmers
- [Valinor](https://github.com/CuyZ/Valinor/): PHP library that helps to map any input into a strongly-typed value object structure.
- [league/tactician](https://github.com/thephpleague/tactician): A small, flexible command bus.
- [league/flysystem](https://flysystem.thephpleague.com/docs/): Abstraction for local and remote filesystems
- [Golem-AI/messenger-kit](https://github.com/golem-ai/messenger-kit): This command simulates consumer failures and prints a timeline of the events. It lets you check whether your retry strategy configuration does what you expect it to.
- [swarrot/swarrot](https://github.com/swarrot/swarrot): A lib to consume message from any Broker
- [ronanguilloux/isocodes](https://github.com/ronanguilloux/IsoCodes): PHP library - Validators for standards from ISO, International Finance, Public Administrations, GS1, Manufacturing Industry, Phone numbers & Zipcodes for many countries
- [PHP Units of Measure](https://github.com/PhpUnitsOfMeasure/php-units-of-measure): A library for handling physical quantities and the units of measure in which they're represented.
- [T-Regx](https://github.com/T-Regx/T-Regx): PHP regular expression brought up to modern standards.
- [mpratt/embera](https://github.com/mpratt/Embera): A Oembed consumer library, that gives you information about urls. It helps you replace urls to youtube or vimeo for example, with their html embed code. It has advanced features like offline support, responsive embeds and caching support.
- [spatie/geocoder](https://github.com/spatie/geocoder): Geocode addresses to coordinates
- [flack/ranger](https://github.com/flack/ranger): Formatter for date and time ranges with i18n support
- _Money_:
    - [brick/money](https://github.com/brick/money): A money and currency library for PHP
    - [moneyphp](https://github.com/moneyphp/money): PHP implementation of Fowler's Money pattern

## Eines

- [Deptrac](https://github.com/qossmic/deptrac): Keep your architecture clean.
- [PHP Architecture Tester](https://github.com/carlosas/phpat): Easy to use architectural testing tool for PHP
- [PHPArkitect](https://github.com/phparkitect/arkitect): Put your architectural rules under test!
- [PHP Insights](https://phpinsights.com/): Instant PHP quality checks from your console
- [GrumPHP](https://github.com/phpro/grumphp): A PHP code-quality tool.
- [churn-php](https://github.com/bmitch/churn-php): Discover files in need of refactoring.
- [scheb/tombstone](https://github.com/scheb/tombstone): Dead code detection with tombstones for PHP
- [PHP Magic Number Detector](https://github.com/povils/phpmnd): a tool that aims to help you to detect magic numbers in your PHP code.
- [Robo](https://robo.li/): Modern Task Runner for PHP
- [CaptainHook](https://github.com/captainhookphp/captainhook): Very flexible git hook manager for php developers
- [Psalm](https://psalm.dev/): A static analysis tool for finding errors in PHP applications. Plugins:
    - [boesing/psalm-plugin-stringf](https://github.com/boesing/psalm-plugin-stringf): Psalm plugin to provide more details for sprintf, printf, sscanf and fscanf functions.
    - [hectorj/safe-php-psalm-plugin](https://github.com/hectorj/safe-php-psalm-plugin): vimeo/psalm plugin for thecodingmachine/safe.
    - [marartner/psalm-no-empty](https://github.com/marartner/psalm-no-empty): Psalm plugin to detect usage of empty().
    - [marartner/psalm-strict-equality](https://github.com/marartner/psalm-strict-equality): Psalm plugin to enforce strict equality.
    - [psalm/plugin-phpunit](https://github.com/psalm/psalm-plugin-phpunit): A PHPUnit plugin for Psalm.
    - [psalm/plugin-symfony](https://github.com/psalm/psalm-plugin-symfony): Psalm Plugin for Symfony.
    - [weirdan/doctrine-psalm-plugin](https://github.com/psalm/psalm-plugin-doctrine): Stubs to let Psalm understand Doctrine better.
- [PHPStan](https://phpstan.org/): PHP Static Analysis Tool - discover bugs in your code without running it! Plugins:
    - [ergebnis/phpstan-rules](https://github.com/ergebnis/phpstan-rules): Provides additional rules for phpstan/phpstan.
    - [spaze/phpstan-disallowed-calls](https://github.com/spaze/phpstan-disallowed-calls): PHPStan rules to detect disallowed calls and constant & namespace usages
    - [roave/no-floaters](https://github.com/Roave/no-floaters): static analysis rules to prevent IEEE-754 floating point errors.
    - [More extensions](https://phpstan.org/user-guide/extension-library)

## Test

- [Infection](https://infection.github.io/): PHP Mutation Testing library. Plugins:
    - [roave/infection-static-analysis-plugin](https://github.com/Roave/infection-static-analysis-plugin): Static analysis on top of mutation testing - prevents escaped mutants from being invalid according to static analysis
    - [bitexpert/captainhook-infection](https://github.com/bitExpert/captainhook-infection): Captain Hook Plugin to run InfectionPHP only against the changed files of a commit
- [roave/no-leaks](https://github.com/Roave/no-leaks): PHPUnit Plugin for detecting Memory Leaks in code and tests
- [lulco/populator](https://github.com/lulco/populator): Allows populate fake data to your database.
- [OpenAPI PSR-7 Message (HTTP Request/Response) Validator](https://github.com/thephpleague/openapi-psr7-validator):
It validates PSR-7 messages (HTTP request/response) against OpenAPI specifications.
- [Paratest](https://github.com/paratestphp/paratest): Parallel testing for PHPUnit
- [PHPUnit SpeedTrap](https://github.com/johnkary/phpunit-speedtrap): Reports on slow-running tests in your PHPUnit test suite.

## Eines de Composer

- [ComposerRequireChecker](https://github.com/maglnet/ComposerRequireChecker): A CLI tool to check whether a specific composer package uses imported symbols that aren't part of its direct composer dependencies
- [composer-unused](https://github.com/composer-unused/composer-unused): Show unused composer dependencies by scanning your code
- [composer-normalize](https://github.com/ergebnis/composer-normalize): Provides a composer plugin for normalizing composer.json.

## Seguretat

- [roave/security-advisories](https://github.com/Roave/SecurityAdvisories): Security advisories as a simple composer exclusion list, updated daily
- [roave/backward-compatibility-check](https://github.com/Roave/BackwardCompatibilityCheck): Tool to compare two revisions of a class API to check for BC breaks
- [Local PHP Security Checker](https://github.com/fabpot/local-php-security-checker): PHP security vulnerabilities checker

## Comprovacions de CI

- Lints (per a Symfony):
    - PHP: `find src public bin -name "*.php" -print0 | xargs -0 -n1 php -l`
    - Container: `bin/console lint:container`
    - YAML: `bin/console lint:yaml config src`
    - Twig: `bin/console lint:twig src`
- Symfony + Doctrine:
    - Deprecations: `bin/console debug:container --deprecations` (si falla, no retorna un _exit code_ diferent de 0)
    - Doctrine schema: `bin/console doctrine:schema:validate --skip-sync`
- Composer
    - Audit (The audit command checks for security vulnerability advisories for installed packages.): `composer audit`
    - Outdated (The `outdated` command shows a list of installed packages that have updates available, including their current and latest versions): `composer outdated --minor-only --direct --strict`
    - Validate (It will check if your `composer.json` is valid): `composer validate --strict`

## Fonts

Aquesta llista està basada en gran manera en:

- [https://twitter.com/ramsey/status/1396592906102722561](https://twitter.com/ramsey/status/1396592906102722561)
- [https://twitter.com/lulco/status/1397813303037079553](https://twitter.com/lulco/status/1397813303037079553)
- [https://twitter.com/ArkadiuszKondas/status/1338485275002068993](https://twitter.com/ArkadiuszKondas/status/1338485275002068993)
- [https://twitter.com/ArkadiuszKondas/status/1596232242627366912](https://twitter.com/ArkadiuszKondas/status/1596232242627366912)
- [Aggressive PHP Quality Assurance in 2019 \| Marco Pivetta](https://youtu.be/8rdTSYljts4)
