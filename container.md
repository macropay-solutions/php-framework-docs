---
title: Service Container
description: Guide to configuring, binding, and resolving dependencies via the Service Container in PHP Framework.
context: container
---

# Service Container

- [Introduction](#introduction)
  - [Zero Configuration Resolution](#zero-configuration-resolution)
  - [When to Utilize the Container](#when-to-use-the-container)
- [Binding](#binding)
  - [Binding Basics](#binding-basics)
  - [Binding Interfaces to Implementations](#binding-interfaces-to-implementations)
  - [Contextual Injection](#contextual-injection)
  - [Binding Primitives](#binding-primitives)
  - [Binding Typed Variadics](#binding-typed-variadics)
  - [Tagging](#tagging)
  - [Extending Bindings](#extending-bindings)
- [Resolving](#resolving)
  - [The Make Method](#the-make-method)
  - [The MakeWithoutAlias Method](#the-makewithoutalias-method)
  - [Automatic Injection](#automatic-injection)
- [Method Invocation and Injection](#method-invocation-and-injection)
- [Container Events](#container-events)
- [PSR-11](#psr-11)

<a name="introduction"></a>
## Introduction

The Framework service container is a powerful tool for managing class dependencies and performing dependency injection. Dependency injection is a fancy phrase that essentially means this: class dependencies are "injected" into the class via the constructor or, in some cases, "setter" methods.

Let's look at a simple example:

    <?php

    namespace App\Http\Controllers;

    use App\Http\Controllers\Controller;
    use App\Services\OrdersService;
    use MacropaySolutions\Kernel\View\View;

    class OrderController extends Controller
    {
        /**
         * Create a new controller instance.
         */
        public function __construct(
            protected OrdersService $ordersService,
        ) {}

        /**
         * Process the payment for the given order.
         */
        public function process(string $id): View
        {
            $status = $this->ordersService->charge($id);

            return view('order.status', ['status' => $status]);
        }
    }

In this example, the `OrderController` needs to interact with a payment provider. So, we will **inject** a service that handles orders. Since the `OrdersService` is injected, we are able to easily swap it out with another implementation. We are also able to easily "mock", or create a dummy implementation of the `OrdersService` when testing our application.

A deep understanding of the Framework service container is essential to building a powerful, large application, as well as for contributing to the core itself.

<a name="zero-configuration-resolution"></a>
### Zero Configuration Resolution

If a class has no dependencies or only depends on other concrete classes (not interfaces), the container does not need to be instructed on how to resolve that class. For example, if you register a route to a controller:

    \app('router')->get('/', [\App\Http\Controllers\ServiceController::class, 'index']);

And define the controller like so:

    <?php

    namespace App\Http\Controllers;

    use App\Http\Controllers\Controller;
    use App\Services\BillingService;

    class ServiceController extends Controller
    {
        public function index(BillingService $service)
        {
            die($service::class);
        }
    }

In this example, hitting your application's `/` route will automatically resolve the `BillingService` class and inject it into your controller's method. This is game changing. It means you can develop your application and take advantage of dependency injection without worrying about bloated configuration files.

Thankfully, many of the classes you will be writing when building an application automatically receive their dependencies via the container, including controllers, event listeners, middleware, and more. Additionally, you may type-hint dependencies in the `handle` method of queued jobs. Once you taste the power of automatic and zero configuration dependency injection, it feels impossible to develop without it.

<a name="when-to-use-the-container"></a>
### When to Utilize the Container

Thanks to zero configuration resolution, you will often type-hint dependencies on controllers, event listeners, and elsewhere without ever manually interacting with the container. For example, you might type-hint the `MacropaySolutions\Kernel\Http\Request` object on your controller definition so that you can easily access the current request. Even though we never have to interact with the container to write this code, it is managing the injection of these dependencies behind the scenes:

    namespace App\Http\Controllers;

    use MacropaySolutions\Kernel\Http\Request;

    class HomeController
    {
        public function index(Request $request)
        {
            // ...
        }
    }

In many cases, thanks to automatic dependency injection, you can build applications without **ever** manually binding or resolving anything from the container. **So, when would you ever manually interact with the container?** Let's examine two situations.

First, if you write a class that implements an interface and you wish to type-hint that interface on a controller or class constructor, you must [tell the container how to resolve that interface](#binding-interfaces-to-implementations). Secondly, if you are writing a Framework package that you plan to share with other developers, you may need to bind your package's services into the container.

<a name="binding"></a>
## Binding

<a name="binding-basics"></a>
### Binding Basics

<a name="simple-bindings"></a>
#### Simple Bindings

Almost all of your service container bindings will be registered within [service providers](/providers), so most of these examples will demonstrate using the container in that context.

Within a service provider, you always have access to the container via the `$this->app` property. We can register a binding using the `bind` method, passing the class or interface name that we wish to register along with a closure that returns an instance of the class:

    use App\Services\Transistor;
    use App\Services\PodcastParser;
    use MacropaySolutions\Kernel\Contracts\Foundation\Application;

    $this->app->bind(Transistor::class, static function (Application $app) {
        return new Transistor($app->make(PodcastParser::class));
    });

Note that we receive the container itself as an argument to the resolver. We can then use the container to resolve sub-dependencies of the object we are building.

As mentioned, you will typically be interacting with the container within service providers; however, if you would like to interact with the container outside of a service provider, you may do so via the `app()` helper:

    use App\Services\Transistor;
    use MacropaySolutions\Kernel\Contracts\Foundation\Application;

    \app()->bind(Transistor::class, static function (Application $app) {
        // ...
    });

You may use the `bindIf` method to register a container binding only if a binding has not already been registered for the given type:

    $this->app->bindIf(Transistor::class, function (Application $app) {
        return new Transistor($app->make(PodcastParser::class));
    });

> [!TIP]
> **Closure Scoping & Performance Guidelines (`static`)**
> Container factory closures receive the container instance explicitly as an argument (`$app` or `$container`). Therefore, they do not require `$this` scope binding.
>
> **Always declare container closures as `static`** (`static fn(...)` or `static function(...)`). This prevents implicit `$this` context capturing, reduces zval allocations, and prevents cyclic memory references in long-running processes (e.g., Swoole, RoadRunner, queue workers).
>
> [!NOTE]  
> *Do not use `static` closures for Obvious Model accessors/mutators or dynamic Macros, as those features rely on `Closure::bindTo()` to bind `$this` at runtime.*
> There is no need to bind classes into the container if they do not depend on any interfaces. The container does not need to be instructed on how to build these objects, since it can automatically resolve these objects using reflection.

<a name="binding-a-singleton"></a>
#### Binding A Singleton

The `singleton` method binds a class or interface into the container that should only be resolved one time. Once a singleton binding is resolved, the same object instance will be returned on subsequent calls into the container:

    use App\Services\Transistor;
    use App\Services\PodcastParser;
    use MacropaySolutions\Kernel\Contracts\Foundation\Application;

    $this->app->singleton(Transistor::class, function (Application $app) {
        return new Transistor($app->make(PodcastParser::class));
    });

You may use the `singletonIf` method to register a singleton container binding only if a binding has not already been registered for the given type:

    $this->app->singletonIf(Transistor::class, function (Application $app) {
        return new Transistor($app->make(PodcastParser::class));
    });

<a name="binding-scoped"></a>
#### Binding Scoped Singletons

The `scoped` method binds a class or interface into the container that should only be resolved one time within a given Framework request / job lifecycle. While this method is similar to the `singleton` method, instances registered using the `scoped` method will be flushed whenever the Framework application starts a new "lifecycle", such as when a queue worker processes a new job:

    use App\Services\Transistor;
    use App\Services\PodcastParser;
    use MacropaySolutions\Kernel\Contracts\Foundation\Application;

    $this->app->scoped(Transistor::class, function (Application $app) {
        return new Transistor($app->make(PodcastParser::class));
    });

<a name="binding-instances"></a>
#### Binding Instances

You may also bind an existing object instance into the container using the `instance` method. The given instance will always be returned on subsequent calls into the container:

    use App\Services\Transistor;
    use App\Services\PodcastParser;

    $service = new Transistor(new PodcastParser);

    $this->app->instance(Transistor::class, $service);

<a name="binding-interfaces-to-implementations"></a>
### Binding Interfaces to Implementations

A very powerful feature of the service container is its ability to bind an interface to a given implementation. For example, let's assume we have an `EventPusher` interface and a `RedisEventPusher` implementation. Once we have coded our `RedisEventPusher` implementation of this interface, we can register it with the service container like so:

    use App\Contracts\EventPusher;
    use App\Services\RedisEventPusher;

    $this->app->bind(EventPusher::class, RedisEventPusher::class);

This statement tells the container that it should inject the `RedisEventPusher` when a class needs an implementation of `EventPusher`. Now we can type-hint the `EventPusher` interface in the constructor of a class that is resolved by the container. Remember, controllers, event listeners, middleware, and various other types of classes are always resolved using the container:

    use App\Contracts\EventPusher;

    /**
     * Create a new class instance.
     */
    public function __construct(
        protected EventPusher $pusher
    ) {}

<a name="contextual-injection"></a>
### Contextual Injection

Sometimes you may have two classes that utilize the same interface, but you wish to inject different implementations into each class. For example, two controllers may depend on different implementations of the `MacropaySolutions\Kernel\Contracts\Filesystem\Filesystem` [contract](/contracts).

Since PHP-Framework optimizes for absolute maximum execution speed, it does not provide a slower fluent contextual binding builder (like `->when()->needs()->give()`). Instead, you can achieve this with zero overhead using the following methods:

#### 1. The Explicit Bindings Map

The fastest way to bypass contextual lookup is by overriding the `$bindings` array directly inside your `app/Application.php` file using the `registerExplicitBindingsMap` method. This method registers resolution closures directly into the core bindings registry during container bootstrapping, keeping the evaluation path optimized.

    protected function registerExplicitBindingsMap(): void
    {
        $this->bindings = [
            // Explicitly wire PhotoController to use LocalFilesystem
            \App\Http\Controllers\PhotoController::class => [
                'concrete' => static function (
                    \MacropaySolutions\Kernel\Contracts\Container\Container $container,
                    array $parameters = []
                ): \App\Http\Controllers\PhotoController {
                    return new \App\Http\Controllers\PhotoController(
                        $container->resolve(\App\Services\Filesystem\LocalFilesystem::class, $parameters, false)
                    );
                },
                'shared' => false
            ],

            // Explicitly wire VideoController to use S3Filesystem
            \App\Http\Controllers\VideoController::class => [
                'concrete' => static function (
                    \MacropaySolutions\Kernel\Contracts\Container\Container $container,
                    array $parameters = []
                ): \App\Http\Controllers\VideoController {
                    return new \App\Http\Controllers\VideoController(
                        $container->resolve(\App\Services\Filesystem\S3Filesystem::class, $parameters, false)
                    );
                },
                'shared' => false
            ],
        ];
    }

#### 2. Manual Factory Closures in Service Providers

If you prefer to configure your bindings within standard Service Providers, you can avoid contextual evaluations by using a factory closure inside a standard `bind` or `singleton` method. This allows you to manually instantiate the target class and explicitly pass the concrete dependencies it requires.

    use App\Http\Controllers\PhotoController;
    use App\Services\Filesystem\LocalFilesystem;
    use MacropaySolutions\Kernel\Contracts\Foundation\Application;

    $this->app->bind(PhotoController::class, static function (Application $app) {
        return new PhotoController($app->make(LocalFilesystem::class));
    });

#### 3. Type-Hinting Concrete Classes (Zero-Configuration)

Often, the need for contextual binding indicates that an interface is serving too many distinct roles across your application. You can eliminate manual bindings completely by type-hinting unique concrete classes or descriptive sub-interfaces directly in your class constructors. 

By targeting specific concrete dependencies, the container can instantly resolve the object using its highly efficient **Zero Configuration Resolution** pipeline via reflection.

    namespace App\Http\Controllers;

    use App\Http\Controllers\Controller;
    use App\Services\Filesystem\LocalFilesystem;

    class PhotoController extends Controller
    {
        // Fully resolved automatically via Zero-Configuration reflection
        public function __construct(
            protected LocalFilesystem $disk
        ) {}
    }

<a name="binding-primitives"></a>
### Binding Primitives

Sometimes you may have a class that receives some injected classes, but also needs an injected primitive value such as an integer. You can easily inject any value your class may need using a manual factory closure:

    use App\Http\Controllers\UserController;
    use MacropaySolutions\Kernel\Contracts\Foundation\Application;
    
    $this->app->bind(UserController::class, static function (Application $app) {
        return new UserController($app->make(SomeService::class), 'default_value');
    });

Sometimes a class may depend on an array of [tagged](#tagging) instances. Using the `tagged` method within a factory closure, you may easily inject all the container bindings with that tag:

    $this->app->bind(ReportAggregator::class, static function (Application $app) {
        return new ReportAggregator($app->tagged('reports'));
    });

If you need to inject a value from one of your application's configuration files, you may manually resolve the `config` service:

    $this->app->bind(ReportAggregator::class, static function (Application $app) {
        return new ReportAggregator($app->make('config')->get('app.timezone'));
    });

<a name="binding-typed-variadics"></a>
### Binding Typed Variadics

Occasionally, you may have a class that receives an array of typed objects using a variadic constructor argument:

    <?php

    use App\Models\Filter;
    use App\Services\Logger;

    class Firewall
    {
        /**
         * The filter instances.
         *
         * @var array
         */
        protected $filters;

        /**
         * Create a new class instance.
         */
        public function __construct(
            protected Logger $logger,
            Filter ...$filters,
        ) {
            $this->filters = $filters;
        }
    }

You may resolve this dependency by providing the `bind` method with a closure that explicitly injects the resolved `Filter` instances:

    use MacropaySolutions\Kernel\Contracts\Foundation\Application;

    $this->app->bind(Firewall::class, static function (Application $app) {
        return new Firewall(
            $app->make(Logger::class),
            $app->make(NullFilter::class),
            $app->make(ProfanityFilter::class),
            $app->make(TooLongFilter::class),
        );
    });

<a name="variadic-tag-dependencies"></a>
#### Variadic Tag Dependencies

Sometimes a class may have a variadic dependency that is type-hinted as a given class (`Report ...$reports`). Using the spread operator `...` and the `tagged` method within a factory closure, you may easily inject all the container bindings with that [tag](#tagging) for the given dependency:

    $this->app->bind(ReportAggregator::class, static function (Application $app) {
        return new ReportAggregator(...$app->tagged('reports'));
    });

<a name="tagging"></a>
### Tagging

Occasionally, you may need to resolve all of a certain "category" of binding. For example, perhaps you are building a report analyzer that receives an array of many different `Report` interface implementations. After registering the `Report` implementations, you can assign them a tag using the `tag` method:

    $this->app->bind(CpuReport::class, static function () {
        // ...
    });

    $this->app->bind(MemoryReport::class, static function () {
        // ...
    });

    $this->app->tag([CpuReport::class, MemoryReport::class], 'reports');

Once the services have been tagged, you may easily resolve them all via the container's `tagged` method:

    $this->app->bind(ReportAnalyzer::class, static function (Application $app) {
        return new ReportAnalyzer($app->tagged('reports'));
    });

<a name="extending-bindings"></a>
### Extending Bindings

The `extend` method allows the modification of resolved services. For example, when a service is resolved, you may run additional code to decorate or configure the service. The `extend` method accepts two arguments, the service class you're extending and a closure that should return the modified service. The closure receives the service being resolved and the container instance:

    $this->app->extend(Service::class, static function (Service $service, Application $app) {
        return new DecoratedService($service);
    });

<a name="resolving"></a>
## Resolving

<a name="the-make-method"></a>
### The `make` Method

You may use the `make` method to resolve a class instance from the container. The `make` method accepts the name of the class or interface you wish to resolve:

    use App\Services\Transistor;

    $transistor = $this->app->make(Transistor::class);

If some of your class's dependencies are not resolvable via the container, you may inject them by passing them as an associative array (or LIST array with all the needed parameters to avoid runtime reflection) into the `makeWith` method. For example, we may manually pass the `$id` constructor argument required by the `Transistor` service:

    use App\Services\Transistor;

    $transistor = $this->app->makeWith(Transistor::class, ['id' => 1]);
    $transistor = $this->app->make(Transistor::class, [1]);

The `bound` method may be used to determine if a class or interface has been explicitly bound in the container:

    if ($this->app->bound(Transistor::class)) {
        // ...
    }

If you are outside a service provider in a location of your code that does not have access to the `$app` variable, you may use the `app` helper to resolve a class instance from the container:

    use App\Services\Transistor;

    $transistor = \app(Transistor::class);

If you would like to have the Framework container instance itself injected into a class that is being resolved by the container, you may type-hint the `MacropaySolutions\Kernel\Container\Container` class on your class's constructor:

    use MacropaySolutions\Kernel\Container\Container;

    /**
     * Create a new class instance.
     */
    public function __construct(
        protected Container $container
    ) {}

> [!NOTE]
>You can use the `di` helper **exclusively with list parameters** when resolving from the container if you want to make sure it will not introduce issues if called before the application is fully booted.

Because most of the macroable classes are resolved from the container in PHP-Kernel, there are some corner cases like `MacropaySolutions\Kernel\Console\Scheduling\Schedule` which, if resolved from the container, would generate a circular dependency that leads to a segmentation fault or memory exhaustion in `\MacropaySolutions\Kernel\Foundation\Console\Kernel::defineConsoleSchedule`.
This can happen also to developers leading to lost debug time.

To shorten the detection of such cases, we introduced a check, configurable via `app.circular_dependency_memory_limit`:

    /**
     * Limit of extra memory used to detect circular dependencies when resolving an abstract from container (bytes)
     */
    'circular_dependency_memory_limit' => 'production' === \env('APP_ENV') ? 0 : 10485760,

By setting it to 0 bytes you disable the check.
When enabled, an increment is used until 7 and then the memory check kicks in for benchmark speed reasons.

<a name="the-makewithoutalias-method"></a>
### The `makeWithoutAlias` Method

Internally, the Framework container relies heavily on short string aliases (like `'events'` or `'mailer'`) to map core components to their implementations. However, this can introduce a dangerous "Instantiation Loop" if a foundational Service Provider attempts to resolve a core class from the container while that exact class is still in the middle of being booted.

To safely break this chicken-and-egg cycle without sacrificing the flexibility of the alias registry, you may use the `makeWithoutAlias` method. This method forces the container to evaluate the concrete class directly through the reflection and cache pipeline, completely bypassing the forward and reverse alias lookup arrays:

    use MacropaySolutions\Kernel\Mail\Mailer;

    // Triggers the alias factory closure, potentially causing an infinite boot loop...
    $mailer = $this->app->make(Mailer::class, $parameters);

    // Safely resolves the concrete class without triggering alias redirects...
    $mailer = $this->app->makeWithoutAlias(Mailer::class, $parameters);

This method is strictly required when you are authoring custom service providers, foundational decorators, or extending core framework managers (like `MailManager` or `Dispatcher`) that need to resolve framework classes from dependency injection *while* the application is still bootstrapping.

<a name="automatic-injection"></a>
### Automatic Injection

Alternatively, and importantly, you may type-hint the dependency in the constructor of a class that is resolved by the container, including controllers, event listeners, middleware, and more. Additionally, you may type-hint dependencies in the `handle` method of queued jobs. In practice, this is how most of your objects should be resolved by the container.

For example, you may type-hint a service defined by your application in a controller's constructor. The service will automatically be resolved and injected into the class:

    <?php

    namespace App\Http\Controllers;

    use App\Services\BillingService;
    use App\Models\User;

    class UserController extends Controller
    {
        /**
         * Create a new controller instance.
         */
        public function __construct(
            protected BillingService $billing,
        ) {}

        /**
         * Process billing for the given ID.
         */
        public function bill(string $id): User
        {
            $user = $this->billing->processPayment($id);

            return $user;
        }
    }

<a name="method-invocation-and-injection"></a>
## Method Invocation and Injection

Sometimes you may wish to invoke a method on an object instance while allowing the container to automatically inject that method's dependencies. For example, given the following class:

    <?php

    namespace App;

    use App\Services\ReportService;

    class UserReport
    {
        /**
         * Generate a new user report.
         */
        public function generate(ReportService $service): array
        {
            return [
                // ...
            ];
        }
    }

You may invoke the `generate` method via the container like so:

    use App\UserReport;

    $report = \app()->call([new UserReport, 'generate']);

The `call` method accepts any PHP callable. The container's `call` method may even be used to invoke a closure while automatically injecting its dependencies:

    use App\Services\ReportService;

    $result = \app()->call(function (ReportService $service) {
        // ...
    });

<a name="container-events"></a>
## Container Events

The service container fires events at various stages of an object's resolution lifecycle. You may hook into these stages using the `beforeResolving`, `resolving`, and `afterResolving` methods.

> [!WARNING]  
> Use resolution callbacks sparingly. The container engine must compute dynamic cache keys, perform key intersections (`\array_intersect_key`), and flatten callback matrices via `\array_merge` on **every single resolution**. As the number of registered callbacks increases, container execution speed degrades significantly.

#### Before Resolving

The `beforeResolving` method fires right before the container begins looking up or instantiating a target class. It receives the abstract name and any parameters passed to the build sequence:

    use App\Services\Transistor;
    use MacropaySolutions\Kernel\Contracts\Foundation\Application;

    $this->app->beforeResolving(Transistor::class, static function (string $abstract, array $parameters, Application $app) {
        // Executed immediately before the container starts building "Transistor"...
    });

#### Resolving

The `resolving` method fires immediately after the object has been successfully instantiated but before any extender decorators are applied. It passes the fully constructed instance and the container:

    use App\Services\Transistor;
    use MacropaySolutions\Kernel\Contracts\Foundation\Application;

    $this->app->resolving(Transistor::class, static function (Transistor $transistor, Application $app) {
        // Called when the container resolves objects of type "Transistor"...
    });

    $this->app->resolving(static function (mixed $object, Application $app) {
        // Global listener: Called when any object type is resolved...
    });

#### After Resolving

The `afterResolving` method fires at the absolute tail end of the resolution pipeline, right after all class configuration, object instantiation, and factory extensions (`extend()`) have finished processing:

    use App\Services\Transistor;
    use MacropaySolutions\Kernel\Contracts\Foundation\Application;

    $this->app->afterResolving(Transistor::class, static function (Transistor $transistor, Application $app) {
        // Called after the instance is completely built and decorated...
    });

<a name="psr-11"></a>
## PSR-11

Framework's service container implements the [PSR-11](https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-11-container.md) interface. Therefore, you may type-hint the PSR-11 container interface to obtain an instance of the container:

    namespace App\Http\Controllers;

    use App\Services\Transistor;
    use Psr\Container\ContainerInterface;

    class AudioController
    {
        public function index(ContainerInterface $container)
        {
            $service = $container->get(Transistor::class);

            // ...
        }
    }

An exception is thrown if the given identifier can't be resolved. The exception will be an instance of `Psr\Container\NotFoundExceptionInterface` if the identifier was never bound. If the identifier was bound but was unable to be resolved, an instance of `Psr\Container\ContainerExceptionInterface` will be thrown.
