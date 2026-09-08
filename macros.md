---
title: Macros & Extending Core Classes
description: Guide to avoiding dynamic macros, extending the Request object natively, and utilizing Dependency Injection in PHP-Framework.
context: macros
---

# Macros & Extending Core Classes

- [Introduction](#introduction)
- [Dedicated Trait Architecture & AOT Compilation](#dedicated-trait-architecture--aot-compilation)
- [Package Commands & Traitables Architecture](#package-commands--traitables-architecture)
- [Deferred Macros](#deferred-macros)
- [Extending the Request Object](#extending-the-request-object)
- [Avoiding Macros via Dependency Injection](#avoiding-macros-via-dependency-injection)

<a name="introduction"></a>
## Introduction

PHP-Framework is a DI-oriented, high-performance PHP framework. It is built to enforce strict architectural boundaries by actively preventing implicit magic, serialized closures, and dynamic static proxies.

The `Macroable` trait is `@internal` to the framework engine. Direct usage of `use MacropaySolutions\Kernel\Support\Traits\Macroable;` in application or third-party code is strictly prohibited.

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

> Macros are still around to solve the situation where 2 packages want to add functionalities into the same macroable class.

<a name="dedicated-trait-architecture--aot-compilation"></a>
## Dedicated Trait Architecture & AOT Compilation

To support Ahead-of-Time (AOT) compilation in production while preserving dynamic runtime macros in local development, macroable classes do not import the `@internal` `Macroable` trait directly. Instead, every macroable class and subclass implements `\MacropaySolutions\Kernel\Macroable\Contracts\Macroable` and imports its own unique, flattened Fully Qualified Name (FQN) trait:

```php
namespace MacropaySolutions\Kernel\Database;

use MacropaySolutions\Kernel\Macroable\Contracts\Macroable;

class Connection implements ConnectionInterface, Macroable
{
    use \MacropaySolutions\Framework\Traitables\MacropaySolutionsKernelDatabaseConnection; // this
}
```

### Local Development vs. Production Execution

* **Local Development (`src/Traitables/` from php-kernel):** Composer resolves the trait import to a fallback trait in `src/Traitables/`. This fallback internally imports the `@internal` `Macroable` trait, maintaining dynamic `__call` magic method dispatch for seamless local development.
* **Production (`bootstrap/cache/traitables/`):** Running `php run macro:cache` inspects all registered deferred macros and compiles concrete, native PHP methods directly into class-specific trait files saved in `bootstrap/cache/traitables/`. These compiled traits utilize `CompiledMacroable`, completely bypassing `__call` at runtime for maximum execution speed.

### Class Inheritance & Scope Scenarios

> **NOTE:** This applies for php-kernel not for external packages!
 
Understanding why **every single class requires its own trait import** comes down to maintaining strict class isolation in both development and production:

#### Scenario 1: Only the Parent Has the Trait ❌ (Forbidden)
If `ChildClass` extends `ParentClass` but forgets to import its own dedicated trait:
* **In Development:** `ChildClass` shares the exact same static memory array as `ParentClass`.
  * Adding a macro to `ChildClass` registers it directly on `ParentClass`, leaking it to the parent and all sibling subclasses.
  * Calling a parent macro on `ChildClass` works, but only because both classes share the same underlying memory array.
* **In Production (`macro:cache`):** **Hierarchy Leakage.** `macro:cache` sees the macro inside `ParentClass::$macros` and compiles it into `ParentClass`'s trait. The macro leaks to `ParentClass` and all siblings in production as well.

#### Scenario 2: Both Parent & Child Have Dedicated Traits ✅ (Mandatory)
When `ChildClass` imports its own unique FQN trait:
* **In Development:** PHP gives `ChildClass` its own private memory array.
  * Macros added to `ChildClass` stay strictly inside `ChildClass`.
  * Macros added to `ParentClass` are resolved on `ChildClass` via inheritance tree walking (`resolveMacro`), keeping memory usage minimal without copying static state.
* **In Production (`macro:cache`):** **Clean OOP Inheritance.** `macro:cache` compiles macros into their respective class traits. `ChildClass` inherits parent methods cleanly via standard PHP class inheritance (`ChildClass extends ParentClass`) at full engine speed.

| Scenario / Action | Single Trait (Parent Only) ❌ | Dedicated Traits (Parent & Child) ✅ |
| :--- | :--- | :--- |
| **Adding macro to `ChildClass` (Dev)** | Leaks to Parent & all sibling classes | Stays isolated strictly to `ChildClass` |
| **Calling Parent macro on Child (Dev)** | Works by accident (shared memory) | Resolves dynamically via inheritance tree traversal (`resolveMacro`) |
| **Adding macro to `ChildClass` (Prod)** | Compiles into Parent (leaks to siblings) | Compiles strictly into `ChildClass` trait |

> [!CRITICAL]
> **Mandatory Subclass Trait Injection**
>
> Because PHP static properties (`static::$macros`) are shared across class inheritance trees, **every subclass extending a `Macroable` parent MUST import its own dedicated FQN trait.**
>
> If a newly created subclass inherits its parent's trait instead of declaring its own:
> 1. Macros registered on the subclass will leak upwards into the parent class's static state during development.
> 2. The `macro:cache` compiler will not bind native methods to the subclass body, but instead it will bind to the parent's body.
>
> Whenever you create a new child class extending a macroable framework class or implementing the Macroable contract, you must manually add a corresponding trait for it.
>
> This ensures child macros remain strictly isolated in development, while parent macros resolve dynamically via inheritance tree traversal without duplicating static state.

Example:
```php
<?php

namespace App\Providers;

use MacropaySolutions\Kernel\Database\Obvious\Collection;
use MacropaySolutions\Kernel\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    /**
     * Register any application services.
     */
    public function register(): void
    {
        Collection::deferredMacro('testMacro', [$this::class, 'getTestMacro']);
        Collection::deferredMacro('testMacroStatic', [$this::class, 'testMacroStatic']);
    }

    public static function getTestMacro()
    {
        return fn(string $param): string => 'not static ' . $param;
    }

    public static function testMacroStatic()
    {
        return static fn(string $param): string => 'static ' . $param;
    }

    /**
     * Bootstrap any application services.
     */
    public function boot(): void
    {
        if (\str_starts_with(\config('app.url'), 'https://')) {
            \app('url')->forceScheme('https');
        }
    }
}
```
```bash
php run macro:cache
```
Generates:
```php
<?php

namespace MacropaySolutions\Framework\Traitables;

trait MacropaySolutionsKernelDatabaseObviousCollection
{
    use \MacropaySolutions\Kernel\Support\Traits\CompiledMacroable;

    public function testMacro(string $param): string
    {
        return (array(
            0 => 'App\\Providers\\AppServiceProvider',
            1 => 'getTestMacro',
        ))()->call($this, $param);
    }

    public static function testMacroStatic(string $param): string
    {
        return (array(
            0 => 'App\\Providers\\AppServiceProvider',
            1 => 'testMacroStatic',
        ))()($param);
    }
}

```

<a name="package-commands--traitables-architecture"></a>
## Package Commands & Traitables Architecture

Third-party packages extending framework components (such as `MacropaySolutions\Kernel\Console\Command`) **do not write Traitables, do not configure PSR-4 fallback paths, and must not reference the `@internal` `Macroable` trait.**

### Extending Framework Base Classes in Packages
Package commands simply extend the core framework base class directly:

```php
namespace MyVendor\MyPackage\Console;

use MacropaySolutions\Kernel\Console\Command;
use Symfony\Component\Console\Attribute\AsCommand;

#[AsCommand(name: 'package:custom')]
class CustomPackageCommand extends Command
{
    protected $name = 'package:custom';
}
```

* **In Development:** Calling a macro on `CustomPackageCommand` traverses the inheritance chain up to `Command` via `resolveMacro()`, locating macros registered on `Command` without needing a package-level trait.
* **In Production (`macro:cache`):** `macro:cache` compiles native methods into `Command`'s trait (`MacropaySolutionsKernelConsoleCommand`). `CustomPackageCommand` inherits all compiled macro methods directly through standard PHP class extension (`extends Command`).

<a name="deferred-macros"></a>
## Deferred Macros

To completely eliminate boot-time performance penalties, standard eager macros (`Class::macro()`) have been strictly disabled. If you must use macros (for example, to allow multiple packages to hook into the same class), you **must** use **Deferred Macros**.

Standard macros used to require allocating closures and loading referenced classes during the framework's boot phase, even if the macro was never called during the request lifecycle. To enforce zero-overhead, `Macroable` classes now only support `deferredMacro`:

```php
use MacropaySolutions\Kernel\Support\Collection;

// The macro closure will only be resolved if 'customFilter' is actually called
Collection::deferredMacro('customFilter', [\App\Macros\CollectionMacroFactory::class, 'getClosure']);
```
> [!CRITICAL]
> Boot-Time Only Registration
> All macros must be registered strictly during the application boot phase (inside Service Provider register). Registering macros after the application has booted is strictly forbidden. Dynamic runtime macro registration during HTTP request handling or console command execution breaks AOT compilation guarantees and is not supported.

> [!WARNING]
> Instance macro closures must not be declared static, because they are bound to the target object using Closure::call(). Static macro closures may be declared static, because they are invoked without object binding.

> **WARNING**
> The second argument of `deferredMacro` **must** be an array callable in `[Class::class, 'method']` format (using a class FQN string, not an instantiated object) that resolves to a static method and returns the macro callable. The closure will be bound to the target class on execution.
> 
> Passing an inline closure directly is strictly prevented (it will throw a `\RuntimeException`), as it would allocate memory immediately and defeat the purpose of deferring the macro resolution.
> 
> This applies also to the [Obvious Builder](/obvious#query-macros).

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