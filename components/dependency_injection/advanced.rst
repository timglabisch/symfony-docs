.. index::
   single: DependencyInjection; Advanced configuration

Advanced Container Configuration
================================

Marking Services as public / private
------------------------------------

When defining services, you'll usually want to be able to access these definitions
within your application code. These services are called ``public``. For example,
the ``doctrine`` service registered with the container when using the DoctrineBundle
is a public service as you can access it via::

   $doctrine = $container->get('doctrine');

However, there are use-cases when you don't want a service to be public. This
is common when a service is only defined because it could be used as an
argument for another service.

.. _inlined-private-services:

.. note::

    If you use a private service as an argument to only one other service,
    this will result in an inlined instantiation (e.g. ``new PrivateFooBar()``)
    inside this other service, making it publicly unavailable at runtime.

Simply said: A service will be private when you do not want to access it
directly from your code.

Here is an example:

.. configuration-block::

    .. code-block:: yaml

        services:
           foo:
             class: Example\Foo
             public: false

    .. code-block:: xml

        <?xml version="1.0" encoding="UTF-8" ?>
        <container xmlns="http://symfony.com/schema/dic/services"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
            xsi:schemaLocation="http://symfony.com/schema/dic/services http://symfony.com/schema/dic/services/services-1.0.xsd">

            <services>
                <service id="foo" class="Example\Foo" public="false" />
            </services>
        </container>

    .. code-block:: php

        use Symfony\Component\DependencyInjection\Definition;

        $definition = new Definition('Example\Foo');
        $definition->setPublic(false);
        $container->setDefinition('foo', $definition);

Now that the service is private, you *cannot* call::

    $container->get('foo');

However, if a service has been marked as private, you can still alias it (see
below) to access this service (via the alias).

.. note::

   Services are by default public.

Synthetic Services
------------------

Synthetic services are services that are injected into the container instead
of being created by the container.

For example, if you're using the :doc:`HttpKernel </components/http_kernel/introduction>`
component with the DependencyInjection component, then the ``request``
service is injected in the
:method:`ContainerAwareHttpKernel::handle() <Symfony\\Component\\HttpKernel\\DependencyInjection\\ContainerAwareHttpKernel::handle>`
method when entering the request :doc:`scope </cookbook/service_container/scopes>`.
The class does not exist when there is no request, so it can't be included in
the container configuration. Also, the service should be different for every
subrequest in the application.

To create a synthetic service, set ``synthetic`` to ``true``:

.. configuration-block::

    .. code-block:: yaml

        services:
            request:
                synthetic: true

    .. code-block:: xml

        <?xml version="1.0" encoding="UTF-8" ?>
        <container xmlns="http://symfony.com/schema/dic/services"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
            xsi:schemaLocation="http://symfony.com/schema/dic/services http://symfony.com/schema/dic/services/services-1.0.xsd">

            <services>
                <service id="request" synthetic="true" />
            </services>
        </container>

    .. code-block:: php

        use Symfony\Component\DependencyInjection\Definition;

        $container
            ->setDefinition('request', new Definition())
            ->setSynthetic(true);

As you see, only the ``synthetic`` option is set. All other options are only used
to configure how a service is created by the container. As the service isn't
created by the container, these options are omitted.

Now, you can inject the class by using
:method:`Container::set <Symfony\\Component\\DependencyInjection\\Container::set>`::

    // ...
    $container->set('request', new MyRequest(...));

Aliasing
--------

You may sometimes want to use shortcuts to access some services. You can
do so by aliasing them and, furthermore, you can even alias non-public
services.

.. configuration-block::

    .. code-block:: yaml

        services:
           foo:
             class: Example\Foo
           bar:
             alias: foo

    .. code-block:: xml

        <?xml version="1.0" encoding="UTF-8" ?>
        <container xmlns="http://symfony.com/schema/dic/services"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
            xsi:schemaLocation="http://symfony.com/schema/dic/services http://symfony.com/schema/dic/services/services-1.0.xsd">

            <services>
                <service id="foo" class="Example\Foo" />

                <service id="bar" alias="foo" />
            </services>
        </container>

    .. code-block:: php

        use Symfony\Component\DependencyInjection\Definition;

        $container->setDefinition('foo', new Definition('Example\Foo'));

        $containerBuilder->setAlias('bar', 'foo');

This means that when using the container directly, you can access the ``foo``
service by asking for the ``bar`` service like this::

    $container->get('bar'); // Would return the foo service

.. tip::

    In YAML, you can also use a shortcut to alias a service:

    .. code-block:: yaml

        services:
           foo:
             class: Example\Foo
           bar: "@foo"


Requiring Files
---------------

There might be use cases when you need to include another file just before
the service itself gets loaded. To do so, you can use the ``file`` directive.

.. configuration-block::

    .. code-block:: yaml

        services:
           foo:
             class: Example\Foo\Bar
             file: "%kernel.root_dir%/src/path/to/file/foo.php"

    .. code-block:: xml

        <?xml version="1.0" encoding="UTF-8" ?>
        <container xmlns="http://symfony.com/schema/dic/services"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
            xsi:schemaLocation="http://symfony.com/schema/dic/services http://symfony.com/schema/dic/services/services-1.0.xsd">

            <services>
                <service id="foo" class="Example\Foo\Bar">
                    <file>%kernel.root_dir%/src/path/to/file/foo.php</file>
                </service>
            </services>
        </container>

    .. code-block:: php

        use Symfony\Component\DependencyInjection\Definition;

        $definition = new Definition('Example\Foo\Bar');
        $definition->setFile('%kernel.root_dir%/src/path/to/file/foo.php');
        $container->setDefinition('foo', $definition);

Notice that Symfony will internally call the PHP statement ``require_once``,
which means that your file will be included only once per request.

Decorating Services
-------------------

.. versionadded:: 2.5
    Decorated services were introduced in Symfony 2.5.

When overriding an existing definition, the old service is lost:

.. code-block:: php

    $container->register('foo', 'FooService');

    // this is going to replace the old definition with the new one
    // old definition is lost
    $container->register('foo', 'CustomFooService');

Most of the time, that's exactly what you want to do. But sometimes,
you might want to decorate the old one instead. In this case, the
old service should be kept around to be able to reference it in the
new one. This configuration replaces ``foo`` with a new one, but keeps
a reference of the old one  as ``bar.inner``:

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

Here is what's going on here: the ``setDecoratedService()` method tells
the container that the ``bar`` service should replace the ``foo`` service,
renaming ``foo`` to ``bar.inner``.
By convention, the old ``foo`` service is going to be renamed ``bar.inner``,
so you can inject it into your new service.

.. note::
    The generated inner id is based on the id of the decorator service
    (``bar`` here), not of the decorated service (``foo`` here).  This is
    mandatory to allow several decorators on the same service (they need to have
    different generated inner ids).

    Most of the time, the decorator should be declared private, as you will not
    need to retrieve it as ``bar`` from the container. The visibility of the
    decorated ``foo`` service (which is an alias for ``bar``) will still be the
    same as the original ``foo`` visibility.

You can change the inner service name if you want to:

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

Software Layers
--------

Dependency Injection is awesome to write complex and flexible systems.

It is so easy to stick any kind of service in any kind of service.
Often people think that this is one of the benefits using dependency injection.
That's totally fine for small projects but the truth is that you will end up in a huge mess
if you're working on projects with a massive amount of services
and don't care about which kind of service is allowed to be injected in which kind of services.

Consider you have an infrastructure and a domain layer.
You may want to inject services from your infrastructure layer to services that lives in the domain layer.
But your infrastructure layer shouldn't depend on any kind of domain services.

The Dependency Injection Component allows you to define rules that enforces software layers.

.. code-block:: php

    class SomeBundle extends Bundle {

        public function build(ContainerBuilder $container)
        {
            parent::build($container);

            $container->getLayerRuleBuilder()
                ->layerCanDependOn('controller', 'domain')
                ->layerCanDependOn('domain', 'infrastructure')
            ;
        }
    }

The layer a service will live in is defined in the Configuration file:

.. configuration-block::

    .. code-block:: xml

       <service id="some_service_id" class="DomainServiceA">
            <layer name="domain" />
       </service>

    .. code-block:: yaml

        some_service_id:
            layer:
                - domain

    .. code-block:: php

        $container->register('some_service_id')
            ->setLayers(array('domain'))


By default any service definition will live in a software layer defined as "default" (ContainerInterface::LAYER_DEFAULT).
A service can live in a bunch of different layers, you should use this carefully.
Normal times you just should use one layer for one service.

If you specify a layer, the service won't automatically live in the "default" layer anymore.
Consider you want to inject the logger service to a service that lives in the controller layer:

    <service id="tg_layer_pr_abundle.controller.example" class="Tg\LayerPr\ABundle\Controller\ExampleController">
        <layer name="controller" />
        <argument type="service" id="logger" />
    </service>

By default you'll get an "InvalidLayerException" because the logger service lives in the default layer.
The Container comes with the default rule

.. code-block:: php

    $container->getLayerRuleBuilder()
        ->layerCanDependOn('default', 'default')
    ;

So it would be possible to move the controller to the default and the controller layer.
A better approach would be allowing the controller layer to depend on the default layer.

.. code-block:: php

    $container->getLayerRuleBuilder()
        ->layerCanDependOn('controller', 'default')
    ;

Defining Software Layers for every Bundle
--------

Defining a bunch of global rules to enforce service layers could end up in a huge mess.
Think about a bundle that requires different rules than your bundle.

So it's not recommended to define global rules, instead add rules based on conditions.
A condition can be everything you could do with the expression language and the list of defined layers.

.. code-block:: php

    public function build(ContainerBuilder $container)
    {
        parent::build($container);

        $container->getLayerRuleBuilder()
            ->child('SomeExampleBundle') // child name is optional.
                ->when('"SomeExampleBundle" in layers') // any expression
                ->layerCanDependOn('controller', 'domain')
                ->layerCanDependOn('domain', 'infrastructure')
            ->end()
        ;
    }

In this case the rules are just enforced by services that lives in the "SomeExampleBundle" layer.
So you can sandbox any rules for any bundle by using conditions.

This also allows you to enforce more complex cross bundle rules, for example if you're building a bridge between
Symfony2 and a library.














