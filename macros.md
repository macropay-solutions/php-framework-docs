---
title: Macros & Extending Core Classes
description: Guide to avoiding dynamic macros, extending the Request object natively, and utilizing Dependency Injection in PHP-Framework.
context: macros
---

# Macros & Extending Core Classes

- [Introduction](#introduction)
- [Extending the Request Object](#extending-the-request-object)
- [Avoiding Macros via Dependency Injection](#avoiding-macros-via-dependency-injection)

<a name="introduction"></a>
## Introduction

PHP-Framework is a DI-oriented, high-performance PHP framework. It is built to enforce strict architectural boundaries by actively preventing implicit magic, serialized closures, and dynamic static proxies.

The `Macroable` trait dynamically injects methods into core classes at runtime. This approach introduces overhead, breaks IDE autocompletion and static analysis.

Instead, PHP-Framework advocates for strict, native class extension and Dependency Injection (DI) to maintain absolute type safety and zero-overhead execution.

> **NOTE**
> **mixin** is not available.
> **Classes bound in the container with their FQN that use the Macroable trait CAN NOT be replaced with a child class!** They will trigger circular dependency exception.
> Example 1: `\MacropaySolutions\Kernel\Bus\Dispatcher`

        $this->app->singleton(Dispatcher::class, function ($app) {
            return new Dispatcher($app, function ($connection = null) use ($app) {
                return $app[QueueFactoryContract::class]->connection($connection);
            });
        });
> In this case you should replace the BussServiceProvider by overriding in your `\App\Application`:

     /**
     * Register container bindings for the application.
     *
     * @return void
     */
    protected function registerBusBindings()
    {
        $this->register(BusServiceProvider::class); // replace here with ChildBusServiceProvider
    }

> Example 2: `\MacropaySolutions\Kernel\Console\Scheduling\Schedule`

    protected function defineConsoleSchedule()
    {
        $this->app->instance(
            Schedule::class,
            $schedule = new Schedule()
        );

        $this->schedule($schedule);
    }

> In this case you should replace in `\App\Application`:

    \MacropaySolutions\Kernel\Contracts\Console\Kernel::class => [
        'concrete' => fn($app): \App\Console\Kernel => new \App\Console\Kernel($app), // replace with new ChildKernel($app)
        'shared' => true
    ],

> and in that child class override the `defineConsoleSchedule` method to instantiate the child class.

> `\MacropaySolutions\Kernel\Database\Obvious\Builder` has its own macroable implementation without using the Macroable trait.

> Use `\di(Class::class)` for Macroable classes to support advanced dependency injection.

> Using a macro method on a class is 1:1 with creating a child class but if that class needs multiple macros, then the macro path becomes slower! 

<a name="extending-the-request-object"></a>
## Extending the Request Object

To maximize performance, the `Macroable` trait has been entirely removed from the core HTTP Request lifecycle. You can no longer use `Request::macro()` in your service providers.

If your application requires custom helper methods on the Request object, you must define them natively:

1.  **Modify the Base Class Directly:** Open `App/RequestTrait.php` and add your strictly-typed method directly to the class body.
2.  **IDE autocomplete:** To enable autocomplete, add these new methods in your `App\Request` docblock via `@method Request newMethod(array $data)`. This is needed because `\MacropaySolutions\Kernel\Http\Request` is the key that resolves the global request singleton but in fact it is an instance of `\App\Request`.

By forcing developers to physically define the methods in the class, you get guaranteed autocompletion, strict type hinting, and better performance by eliminating the macro closure-binding pipeline.

<a name="avoiding-macros-via-dependency-injection"></a>
## Avoiding Macros via Dependency Injection

For all other services, business logic, and third-party integrations, you should avoid macroing existing core services and instead rely on the framework's Service Container.

### 1. Pure Constructor Injection
*   You must use pure Constructor Injection, the `\app()` helper, or container resolution.
*   Rather than attaching a macro to an existing class, create a child class that extends it and register it inside your `app/Application.php` file using the `registerExplicitBindingsMap` method.

### 2. Zero-Overhead Container Bindings
*   To achieve maximum performance during application boot, IoC bindings should be configured directly on the application instance rather than wrapped in Service Provider classes.
*   Map container bindings directly within `App\Application::registerExplicitBindingsMap()` or `App\Application::$availableBindings`.
*   Because these callbacks are evaluated on-demand only when a service is explicitly requested from the container, all bindings become implicitly deferred with zero class loading cost during boot.