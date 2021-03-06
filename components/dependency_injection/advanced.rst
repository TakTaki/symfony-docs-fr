.. index::
   single: Dependency Injection; Advanced configuration

Configuration Avancée du Conteneur
==================================

Marquer les services comme publics / privés
-------------------------------------------

Lorsque vous définissez des services, vous allez généralement vouloir avoir
accès à ces définitions depuis le code de votre application. Ces services
sont dits ``public``. Par exemple, le service ``doctrine`` enregistré dans
le conteneur lorsque vous utilisez le DoctrineBundle est un service public
auquel vous pouvez accéder via::

   $doctrine = $container->get('doctrine');

Cependant, il y a des cas où vous ne souhaitez pas qu'un service soit public.
Cela arrive souvent quand un service est défini seulement pour être utilisé
comme argument d'un autre service.

.. note::

    Si vous utilisez un service privé comme argument d'un seul autre service,
    cela résultera en une instanciation « instantanée » (par exemple :
    ``new PrivateFooBar()``) à l'intérieur de cet autre service, le rendant publiquement
    indisponible à l'exécution.

Pour faire simple : un service est privé quand vous ne voulez pas y accéder
directement depuis votre code.

Voici un exemple :

.. configuration-block::

    .. code-block:: yaml

        services:
           foo:
             class: Example\Foo
             public: false

    .. code-block:: xml

        <service id="foo" class="Example\Foo" public="false" />

    .. code-block:: php

        $definition = new Definition('Example\Foo');
        $definition->setPublic(false);
        $container->setDefinition('foo', $definition);

Maintenant que le service est privé, vous *ne pouvez pas* l'appeler::

    $container->get('foo');

Cependant, si un service a été marqué comme privé, vous pouvez toujours
créer un alias de ce dernier (voir ci-dessous) pour y accéder (via l'alias).

.. note::

   Les services sont par défaut publics.

Les services synthétiques
-------------------------

Les services synthétiques sont des services injectés au conteneur de services
au lieu d'être créé par le conteneur.

Par exemple, si vous utilisez le composant :doc:`HttpKernel </components/http_kernel/introduction>`
avec le composant DependencyInjection, et bien le service ``request`` est injecté
dans la méthode :method:`ContainerAwareHttpKernel::handle() <Symfony\\Component\\HttpKernel\\DependencyInjection\\ContainerAwareHttpKernel::handle>`
lorsqu'on entre dans le :doc:`scope </cookbook/service_container/scopes>` de la request.
La classe n'existe pas lorsqu'il n'y a pas de request, donc la request ne peut être incluse
à la configuration du conteneur de services. Aussi, le service devrait être différent
pour chaque sous-requête dans l'application.

Pour créer un service synthétique, mettez ``synthetic`` à ``true`` :

.. configuration-block::

    .. code-block:: yaml

        services:
            request:
                synthetic: true

    .. code-block:: xml

        <service id="request"
            synthetic="true" />

    .. code-block:: php

        use Symfony\Component\DependencyInjection\Definition;

        // ...
        $container->setDefinition('request', new Definition())
            ->setSynthetic(true);

Comme vous pouvez le voir, seule l'option ``synthetic`` est déclarée. toutes
les autres options sont uniquement utilisée pour configurer pour la création
d'un service par le conteneur de services. Comme le service n'est pas créé par
le conteneur, ces options sont ommises.

Désormais, vous pouvez injecter la classe en utilisant
:method:`Container::set <Symfony\\Component\\DependencyInjection\\Container::set>`::

    // ...
    $container->set('request', new MyRequest(...));

Créer un alias
--------------

Parfois, vous pourriez vouloir utiliser des raccourcis pour accéder à certains
de vos services. Vous pouvez faire cela en créant des alias pour ces derniers ;
de plus, vous pouvez même créer des alias pour les services non publics.

.. configuration-block::

    .. code-block:: yaml

        services:
           foo:
             class: Example\Foo
           bar:
             alias: foo

    .. code-block:: xml

        <service id="foo" class="Example\Foo"/>

        <service id="bar" alias="foo" />

    .. code-block:: php

        $definition = new Definition('Example\Foo');
        $container->setDefinition('foo', $definition);

        $containerBuilder->setAlias('bar', 'foo');

Cela signifie que lorsque vous utilisez le conteneur directement, vous
pouvez accéder au service ``foo`` en demandant le service ``bar`` comme
cela::

    $container->get('bar'); // Retourne le service foo

Requérir des fichiers
---------------------

Il pourrait y avoir des cas où vous aurez besoin d'inclure un autre fichier
juste avant que le service lui-même soit chargé. Pour faire cela, vous
pouvez utiliser la directive ``file``.

.. configuration-block::

    .. code-block:: yaml

        services:
           foo:
             class: Example\Foo\Bar
             file: "%kernel.root_dir%/src/path/to/file/foo.php"

    .. code-block:: xml

        <service id="foo" class="Example\Foo\Bar">
            <file>%kernel.root_dir%/src/path/to/file/foo.php</file>
        </service>

    .. code-block:: php

        $definition = new Definition('Example\Foo\Bar');
        $definition->setFile('%kernel.root_dir%/src/path/to/file/foo.php');
        $container->setDefinition('foo', $definition);

Notez que Symfony va appeler en interne la fonction PHP require_once, ce
qui veut dire que votre fichier va être inclus seulement une fois par requête.

Décorer des services
--------------------

.. versionadded:: 2.5
    La décoration de services a été ajoutée dans Symfony 2.5.

Quand vous redéfinissez une définition existante, l'ancien service est perdu :

.. code-block:: php

    $container->register('foo', 'FooService');

    // cela va remplacer l'ancienne définition avec la nouvelle
    // l'ancienne est perdue
    $container->register('foo', 'CustomFooService');

La plupart du temps, c'est exactement le comportement que vous souhaitez. Cependant,
parfois vous souhaitez décorer l'ancien service à la place. Dans ce cas, l'ancien service
devrait être conservé pour être capable de le référencer dans le nouveau. Cette configuration
remplace ``foo`` avec un nouveau, mais garde une référence à l'ancien sous le nom ``bar.inner``:

.. configuration-block::

    .. code-block:: yaml

       bar:
         public: false
         class: stdClass
         decorates: foo
         arguments: ["@bar.inner"]

    .. code-block:: xml

        <service id="bar" class="stdClass" decorates="foo" public="false">
            <argument type="service" id="bar.inner" />
        </service>

    .. code-block:: php

        use Symfony\Component\DependencyInjection\Reference;

        $container->register('bar', 'stdClass')
            ->addArgument(new Reference('bar.inner'))
            ->setPublic(false)
            ->setDecoratedService('foo');

Voici ce qui est réalisé ici : la méthode ``setDecoratedService()`` indique au
conteneur que le service ``bar`` devrait remplacer le service ``foo``, en
renommant ``foo`` en ``bar.inner``.
Par convention, l'ancien service ``foo`` va être renommé ``bar.inner``, pour que
vous puissiez l'injecter dans votre nouveau service.

.. note::
    L'id de service interne généré est basé sur l'id du service décorateur
    (``bar`` dans ce cas), pas sur celui du service décoré (ici ``foo``). C'est
    obligatoire pour permettre plusieurs décorateurs sur le même service (il faut
    que le nom de service interne généré soit différent).

    Dans la plupat des cas, le décorateur devrait être déclaré privée, puisque vous
    n'avez pas besoin de le récupérer sous le nom ``bar`` depuis le conteneur. La
    visibilité du service décoré ``foo`` (qui est un alias pour ``bar``) correspondra
    à la visibilité initiale du service ``foo``.

Vous pouvez changer le nom du service interne si vous le souhaitez :

.. configuration-block::

    .. code-block:: yaml

       bar:
         class: stdClass
         public: false
         decorates: foo
         decoration_inner_name: bar.wooz
         arguments: ["@bar.wooz"]

    .. code-block:: xml

        <service id="bar" class="stdClass" decorates="foo" decoration-inner-name="bar.wooz" public="false">
            <argument type="service" id="bar.wooz" />
        </service>

    .. code-block:: php

        use Symfony\Component\DependencyInjection\Reference;

        $container->register('bar', 'stdClass')
            ->addArgument(new Reference('bar.wooz'))
            ->setPublic(false)
            ->setDecoratedService('foo', 'bar.wooz');