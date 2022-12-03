---
date: 2022-12-03
---

# Valinor a Symfony amb _Value Objects_
{:.no_toc}

* TOC
{:toc}

## Introducció

En el desenvolupament d'aplicacions web ens trobem sovint amb la necessitat de rebre dades en cru i convertir-les al
nostre domini, on totes les primitives haurien d'estar embolcallades en _Value Objects_. Així doncs, necessitem
convertir dades externes i _tipar-les_ en _Value Objects_ al nostre domini.

PHP és un llenguatge feblement tipat, però en les darreres versions ha fet moltes passes per a tenir un sistema de tipus
robust. Això ha permès que apareguessin llibreries i utilitats per a garantir i fer complir els tipus de dades al nostre
codi.

En aquest article, veurem una llibreria que fa ús d'aquestes millores en el sistema de tipus per a convertir dades
externes al nostre domini.

## Llibreria `cuyz/valinor`

Fa poc va aparèixer la versió 1.0.0 de la llibreria [Valinor](https://valinor.cuyz.io/latest/), que permet convertir
dades en cru (_arrays_, JSONs...) a objectes tipats, de manera que sempre tenen un estat vàlid.

Aquesta llibreria fa ús de `__construct` per a construir els objectes. Això no obstant, té diverses opcions de
configuració, una de les quals és la de registrar _named constructors_. Això és útil pels _Value Objects_, ja que
és [bona pràctica fer-ne servir _named
constructors_ per a instanciar-los](https://github.com/ShittySoft/symfony-live-berlin-2018-doctrine-tutorial/pull/3#issuecomment-460614781)
.

A continuació hi ha un exemple d'ús de la llibreria fent servir un _named constructor_:

```php
(new \CuyZ\Valinor\MapperBuilder())
    ->registerConstructor([Color::class, 'fromHex'])
    ->mapper()
    ->map(Color::class, [/* … */]);
```

## Configuració a Symfony

Actualment, no existeix cap _bundle_ de Symfony per a Valinor, de manera que cal configurar la llibreria a mà. Ara bé,
qualsevol aplicació web, per petita que sigui, acostuma a tenir una gran quantitat de _Value Objects_. Per a cadascun
d'aquests _Value Objects_ caldria afegir una crida a `registerConstructor` de la llibreria per a indicar-li el _named
constructor_. Això suposa sobretot dos problemes:

- Genera uns fitxers de configuració enormes, sobretot, si es fa servir YAML.
- Fa que cada desenvolupador hagi de recordar registrar cada _Value Object_ que crea, cosa que pot suposar oblits, ja
  que cal pensar en la infraestructura mentre s'està desenvolupant el domini.

Fóra bo tenir una manera de registrar automàticament els constructors dels _Value Objects_, perquè simplificaria la
configuració de l'aplicació i n'evitaria oblits.

La solució és fer servir
un [`CompilerPass` de Symfony](https://symfony.com/doc/current/service_container/compiler_passes.html) per a registrar
automàticament els _named constructors_ de tots els _Value Objects_ de la nostra aplicació. D'aquesta manera, quan es
compila el _kernel_, es registren automàticament tots els _named constructors_ requerits.

Aquesta solució no té cap impacte en el rendiment de l'aplicació en entorns de producció, ja que el _kernel_ compilat
es _cacheja_ i no es reconstrueix amb cada petició.

A continuació veurem com implementar-ho.

### Interfície `ValueObject`

Abans de res, necessitem una manera d'unificar tots els _Value Object_, per a poder-ne obtenir els seus _named
constructor_ programàticament. La solució és fer servir una interfície que tingui un mètode que retorni el _callable_
del _named constructor_:

```php
<?php

declare(strict_types=1);

namespace rubenrubio\Shared\Domain\ValueObject;

interface ValueObject
{
    public static function defaultNamedConstructor(): callable;
}

```

Aquí tenim un exemple d'implementació a un _Value Object_ d'`Email`:

```php
<?php

declare(strict_types=1);

namespace rubenrubiob\User\Domain\ValueObject;

use rubenrubiob\User\Domain\ValueObject\ValueObject;

final class Email implements ValueObject
{
    private function __construct(private readonly string $email)
    {
    }

    public static function create(string $email): self
    {
        return new self($email);
    }

    public static function defaultNamedConstructor(): callable
    {
        return [self::class, 'create'];
    }
    
    // Others methods
}

```

Fent servir aquesta sintaxi ens assegurem que el nostre IDE indexi el nom del mètode 'create', de manera que permet
autocompletar el codi i refactoritzar-lo sense errors.

### `CompilerPass`

Necessitem una llibreria que ens permeti recórrer el nostre codi per descobrir els _Value Objects_ que implementin la
interfície `ValueObject`. En aquest cas, farem
servir [`league/construct-finder`](https://github.com/thephpleague/construct-finder).

Així doncs, ja podem implementar `ValinorMapperRegisterConstructorCompilerPass`:

- Definim el servei `TreeMapper` amb
  les [definicions de servei de Symfony](https://symfony.com/doc/current/service_container/definitions.html).
- Després, hi afegim les crides al `registerConstructor` de Valinor:
  - Recorrem totes les classes del nostre codi dins de `src`.
  - Filtrem el _namespace_ dels _Value Objects_: tots els _Value Object_ han de ser a un _namespace_ pel qual es pugui
    escriure una expressió regular.
  - Filtrem tots els _Value Object_ que implementin la interfície, i n'obtenim el _callable_ de _named constructor_ fent
    servir _reflection_.
- És important fer notar que cada crida a un mètode de Valinor retorna una nova instància, de manera que a
  cada `addMethodCall` hem de passar-hi el _flag_ `clone` a `true`.
- Addicionalment, podem afegir més crides a `addMethodCall` per a configurar la llibreria.

```php
<?php

declare(strict_types=1);

namespace rubenrubiob\Shared\Infrastructure\Symfony\DependencyInjection;

use CuyZ\Valinor\Mapper\TreeMapper;
use CuyZ\Valinor\MapperBuilder;
use League\ConstructFinder\ConstructFinder;
use ReflectionClass;
use ReflectionException;
use rubenrubiob\Shared\Domain\ValueObject\ValueObject;
use Symfony\Component\DependencyInjection\Compiler\CompilerPassInterface;
use Symfony\Component\DependencyInjection\ContainerBuilder;
use Symfony\Component\DependencyInjection\Definition;

use function array_map;
use function preg_match;
use function sprintf;

final class ValinorMapperRegisterConstructorCompilerPass implements CompilerPassInterface
{
    private const VO_NAMESPACE_PATTERN = '/Domain\\\\ValueObject(\\\\(.*))?/i';

    private const SRC_FOLDER_MASK = '%s/src';

    public function __construct(
        private readonly string $projectDir,
    ) {
    }

    public function process(ContainerBuilder $container): void
    {
        $container->setDefinition(
            TreeMapper::class,
            $this->mapperDefinition(
                $this->mapperBuilderDefinition()
            )
        );
    }

    private function mapperDefinition(Definition $mapperBuilderDefinition): Definition
    {
        $mapperDefinition = new Definition(TreeMapper::class);

        $mapperDefinition->setFactory(
            [
                $mapperBuilderDefinition,
                'mapper'
            ]
        );

        return $mapperDefinition;
    }

    private function mapperBuilderDefinition(): Definition
    {
        $mapperBuilderDefinition = new Definition(MapperBuilder::class);
        $valueObjectsWithNamedConstructor = $this->valueObjectsWithNamedConstructor();
        $valueObjectsConstructorsToRegister = $this->constructorsToRegister($valueObjectsWithNamedConstructor);

        foreach ($valueObjectsConstructorsToRegister as $valueObjectConstructor) {
            $mapperBuilderDefinition->addMethodCall(
                'registerConstructor',
                [$valueObjectConstructor],
                true,
            );
        }

        return $mapperBuilderDefinition;
    }

    /**
     * @return list<ReflectionClass<ValueObject>>
     */
    private function valueObjectsWithNamedConstructor(): array
    {
        $srcFolder = sprintf(self::SRC_FOLDER_MASK, $this->projectDir);

        $classNames = ConstructFinder::locatedIn($srcFolder)->findClassNames();

        $valueObjects = [];

        foreach ($classNames as $className) {
            if (preg_match(self::VO_NAMESPACE_PATTERN, $className) === false) {
                continue;
            }

            try {
                $reflection = new ReflectionClass($className);
            } catch (ReflectionException) {
                continue;
            }

            if ($reflection->isInterface()) {
                continue;
            }

            if (!$reflection->implementsInterface(ValueObject::class)) {
                continue;
            }

            $valueObjects[] = $reflection;
        }

        return $valueObjects;
    }

    /**
     * @return list<callable>
     */
    private function constructorsToRegister(array $valueObjectsWithNamedConstructor): array
    {
        return array_map(
            static function (ReflectionClass $class): callable {
                return $class->getMethod('defaultNamedConstructor')->invoke(null);
            },
            $valueObjectsWithNamedConstructor
        );
    }
}

```


### Registrar el `CompilerPass` al `Kernel` de Symfony

El darrer pas és el d'afegir el `CompilerCass` al `Kernel` de Symfony:

```php
<?php

declare(strict_types=1);

namespace rubenrubiob\Shared\Infrastructure\Symfony;

use rubenrubiob\Shared\Infrastructure\Symfony\DependencyInjection\ValinorMapperRegisterConstructorCompilerPass;
use Symfony\Bundle\FrameworkBundle\Kernel\MicroKernelTrait;
use Symfony\Component\DependencyInjection\ContainerBuilder;
use Symfony\Component\HttpKernel\Kernel as BaseKernel;

class Kernel extends BaseKernel
{
    use MicroKernelTrait;

    protected function build(ContainerBuilder $container): void
    {
        $container->addCompilerPass(
            new ValinorMapperRegisterConstructorCompilerPass(
                $this->getContainer()->getParameter('kernel.project_dir')
            )
        );
    }
}

```

Amb això, la nostra aplicació registrarà automàticament tots els _named constructors_ dels nostres _Value Objects_ a la
llibreria Valinor, que estarà a punt per utilitzar-se amb el servei `TreeMapper`.

## Resum

En el desenvolupament d'aplicacions web necessitem convertir dades en cru a objectes tipats del nostre domini.

Valinor és una solució vàlida a aquest problema, però cal configurar-la per millorar-ne l'experiència de desenvolupament
i evitar possibles errors.

Això ho aconseguim amb els `CompilerPass` de Symfony, que són una eina complexa però alhora potent per a fer
configuracions avançades de llibreries.