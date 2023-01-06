---
date: 2023-01-06
---

# Registrar automàticament tipus de Doctrine a Symfony
{:.no_toc}

* TOC
{:toc}

## Introducció

Com vam veure a [_object calisthenics_](/dev/2022-11-object-calisthenics){:target="_blank"}, i en general, és bona pràctica fer servir _Value Objects_ per a
encapsular les dades del nostre domini. Els tipus de dades que ens ofereix el llenguatge és massa genèric pel nostre
domini: una adreça de correu electrònic és sempre un `string`, però un `string` arbitrari no és una adreça de correu
electrònic.

El nostre domini el desem en un sistema de dades persistent, normalment, però no únicament, una base de dades relacional
com MySQL o PostrgreSQL. Fent servir PHP i Symfony, la llibreria habitual per realitzar la persistència és Doctrine.

### Doctrine _Custom Types_

Ara bé, quan fem servir _Value Objects_ al nostre domini, Doctrine no sap com convertir els nostres tipus de dades
propis als de la base de dades en intentar persistir. Per resoldreo-ho, Doctrine permet
crear [tipus de mapatge propis](https://www.doctrine-project.org/projects/doctrine-dbal/en/current/reference/types.html#custom-mapping-types){:target="_blank"}
. El que fan aquests _custom types_ és convertir en ambdues direccions: de PHP a la base de dades, i viceversa. N'hem de
crear tants tipus com _Value Objects_ tenim, i realitzar la conversió en el tipus.

Per exemple, si tenim el següent _Value Object_ que representa una adreça de correu electrònic[^1]:

```php
<?php

declare(strict_types=1);

namespace rubenrubiob\User\Domain\ValueObject;

final class Email
{
    private function __construct(private readonly string $email)
    {
    }

    public static function create(string $email): self
    {
        return new self($email);
    }

    public function asString(): string
    {
        return $this->email;
    }
}

```

Podríem crear-ne un tipus propi de Doctrine que fos com el següent. És important fer notar que cada _Value Object_ té un
nom que ha de ser únic a tota l'aplicació:

```php
<?php

declare(strict_types=1);

namespace rubenrubiob\User\Infrastructure\Persistence\Doctrine\ValueObject;

use Doctrine\DBAL\Platforms\AbstractPlatform;
use Doctrine\DBAL\Types\StringType;
use rubenrubiob\User\Domain\ValueObject\Email;

final class Email extends StringType
{
    public const NAME = 'email';

    public function convertToDatabaseValue($value, AbstractPlatform $platform): mixed
    {
        if (! $value instanceof Email) {
            return $value;
        }

        return parent::convertToDatabaseValue($value->value(), $platform);
    }

    public function convertToPHPValue($value, AbstractPlatform $platform): ?Email
    {
        $value = parent::convertToPHPValue($value, $platform);

        return $value !== null ? Email::create($value) : null;
    }

    public function requiresSQLCommentHint(AbstractPlatform $platform): bool
    {
        return true;
    }

    public function getName(): string
    {
        return self::NAME;
    }
}

```

### Configuració

Cadascun dels tipus que creem s'ha de registrar. En una aplicació Symfony, això es pot fer
al [fitxer de configuració de Doctrine](https://symfony.com/doc/current/doctrine/dbal.html#registering-custom-mapping-types){:target="_blank"}
. Cal fer servir el nom del tipus com a clau i el _namespace_ del tipus com a valor:

```yml
# config/packages/doctrine.yaml
doctrine:
    dbal:
        types:
            custom_first:  App\Type\CustomFirst
            custom_second: App\Type\CustomSecond
```

Ara bé, si la nostra aplicació creix, tindrem molts _Value Objects_, cadascun amb el seu tipus de Doctrine. Hem de
registrar tots aquests tipus al fitxer de configuració, cosa que pot fer que creixi molt i acabi sent redundant i
difícil de mantenir. A més, és responsabilitat de cada desenvolupador afegir el tipus cada vegada que en crea un, fet
que pot donar peu a oblits.

Igual que amb el [registre de constructors propis de Valinor](/dev/2022-12-valinor-a-symfony-amb-value-objects){:target="_blank"}, podem
implementar un `CompilerPass` que s'encarregui de registrar automàticament tots els tipus propis de Doctrine. Recordem
que aquesta solució no té cap impacte en el rendiment de l'aplicació en entorns de producció, ja que el _kernel_
compilat es _cacheja_ i no es reconstrueix amb cada petició.

## `CompilerPass`

A Symfony existeix un paràmetre del contenidor que conté les definicions de tipus propis de
Doctrine, `doctrine.dbal.connection_factory.types`. Es tracta d'un `array`  associatiu amb
tipus: `array<string, array{class: class-string}>`.

Així doncs, el que hem de fer al `CompilerPass` és recórrer tots els nostres tipus de Doctrine i afegir-los a
aquest `array`. Per fer-ho, podem fer servir la
llibreria [`league/construct-finder`](https://github.com/thephpleague/construct-finder){:target="_blank"}.

Amb això ja podem implementar el `CompilerPass`.

- Recorrem totes les classes del nostre codi dins de `src`.
- Filtrem el _namespace_ dels tipus de Doctrine: tots els _Value Object_ han de ser a un _namespace_ pel qual es pugui
  escriure una expressió regular.
- Per a cada tipus, n'obtenim el nom fent servir _reflection_. En aquest cas, assumim que tots tenen una constant `NAME`
  que conté el nom únic del tipus.
- Afegim cadascun dels tipus a l'`array` i tornem a establir el paràmetre al contenidor.

```php
<?php

declare(strict_types=1);

namespace rubenrubiob\Shared\Infrastructure\Symfony\DependencyInjection;

use League\ConstructFinder\ConstructFinder;
use ReflectionClass;
use ReflectionException;
use Symfony\Component\DependencyInjection\Compiler\CompilerPassInterface;
use Symfony\Component\DependencyInjection\ContainerBuilder;

use function preg_match;
use function sprintf;

final class DoctrineTypeRegisterCompilerPass implements CompilerPassInterface
{
    private const CONTAINER_TYPES_PARAMETER = 'doctrine.dbal.connection_factory.types';

    private const PROJECT_TYPES_PATTERN = '/Infrastructure\\\\Persistence\\\\Doctrine\\\\ValueObject(\\\\(.*))?/i';

    private const TYPE_NAME_CONSTANT_NAME = 'NAME';
    private const SRC_FOLDER_MASK = '%s/src';

    public function __construct(
        private readonly string $projectDir,
    ) {
    }


    public function process(ContainerBuilder $container): void
    {
        if (!$container->hasParameter(self::CONTAINER_TYPES_PARAMETER)) {
            return;
        }

        /** @var array<string, array{class: class-string}> $typeDefinition */
        $typeDefinition = $container->getParameter(self::CONTAINER_TYPES_PARAMETER);

        $types = $this->generateTypes();

        /** @var array{namespace: string, name: string} $type */
        foreach ($types as $type) {
            $name = $type['name'];
            $namespace = $type['namespace'];

            if (isset($typeDefinition[$name])) {
                continue;
            }

            $typeDefinition[$name] = ['class' => $namespace];
        }

        $container->setParameter(self::CONTAINER_TYPES_PARAMETER, $typeDefinition);
    }

    /**
     * @return iterable<array{namespace: string, name: string}>
     */
    private function generateTypes(): iterable
    {
        $srcFolder = sprintf(self::SRC_FOLDER_MASK, $this->projectDir);

        $classNames = ConstructFinder::locatedIn($srcFolder)->findClassNames();

        foreach ($classNames as $className) {
            if (preg_match(self::PROJECT_TYPES_PATTERN, $className) === 0) {
                continue;
            }

            try {
                $reflection = new ReflectionClass($className);
            } catch (ReflectionException) {
                continue;
            }

            yield [
                'namespace' => $reflection->getName(),
                'name' => $reflection->getConstant(self::TYPE_NAME_CONSTANT_NAME),
            ];
        }
    }
}

```

## Registrar el `CompilerPass` al `Kernel` de Symfony

El darrer pas és el d'afegir el `CompilerPass` al `Kernel` de Symfony:

```php
<?php

declare(strict_types=1);

namespace rubenrubiob\Shared\Infrastructure\Symfony;

use rubenrubiob\Shared\Infrastructure\Symfony\DependencyInjection\ DoctrineTypeRegisterCompilerPass;
use Symfony\Bundle\FrameworkBundle\Kernel\MicroKernelTrait;
use Symfony\Component\DependencyInjection\ContainerBuilder;
use Symfony\Component\HttpKernel\Kernel as BaseKernel;

class Kernel extends BaseKernel
{
    use MicroKernelTrait;

    protected function build(ContainerBuilder $container): void
    {
        $container->addCompilerPass(
            new DoctrineTypeRegisterCompilerPass(
                $this->getContainer()->getParameter('kernel.project_dir')
            )
        );
    }
}

```

## Resum

Hem vist com podem crear tipus de Doctrine per a poder fer servir _Value Objects_ de manera transparent a la nostra
aplicació. Hem vist com es poden configurar aquests tipus en una aplicació Symfony. Però, per evitar que es converteixi
en una tasca manual i producte d'errors, hem implementat un `CompilerPass` per a registrar automàticament tots aquests
tipus a Symfony.

[^1]: Aquest _Value Object_ és només un exemple. Caldria afegir-hi validació, normalització... Però això queda fora de l'àmbit d'aquest post.