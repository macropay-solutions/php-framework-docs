---
title: Service Providers & Container Bindings
description: Guide to configuring container bindings, performance optimizations, and writing optional service providers in PHP Framework.
context: providers
---

# Service Providers & Container Bindings

- [Introduction](#introduction)
- [Zero-Overhead Container Bindings (Recommended)](#zero-overhead-container-bindings)
- [Writing Service Providers](#writing-service-providers)
  - [The Register Method](#the-register-method)
  - [The Boot Method](#the-boot-method)
  - [Deferred Providers](#deferred-providers)
- [Registering Providers](#registering-providers)

<a name="introduction"></a>
## Introduction

In this Framework, **Service Providers are completely optional** and stripped of legacy architectural weight. Dynamic asset registration helpers (`loadViewsFrom`, `loadTranslationsFrom`, `loadJsonTranslationsFrom`, `loadMigrationsFrom`, and `publishes`) **do NOT exist** in the `ServiceProvider` base class to eliminate runtime container closure bindings and per-request filesystem scans.

Because instantiating provider classes and executing `register()` methods on every request adds unnecessary filesystem and boot overhead, the framework natively defers container resolution by default at the application level.

Instead of writing providers for bindings, you can map container bindings directly within the `App\Application::$bindings` property or `App\Application::$availableBindings`. When a provider is needed, only [deferred providers](#deferred-providers) are recommended.

Only the providers passed to `$app->boot(...)` in `bootstrap/app.php` are booted. Registering a provider, with `$app->register()` or from a deferred binding, never calls its `boot()` method.

<a name="zero-overhead-container-bindings"></a>
## Zero-Overhead Container Bindings (Recommended)

To achieve maximum performance during application boot, IoC bindings should be configured directly on the application instance rather than wrapped in Service Provider classes.

### 1. Available Bindings & Aliases

For core services and autoloaded singletons, define your available bindings and class aliases directly within your application class (`App\Application`):

- `$availableBindings`: Maps abstract keys or interfaces to their concrete resolvers.
- `$aliases`: Registers container aliases without instantiating provider objects.

### 2. Explicit Binding Maps

If you have custom service bindings that would traditionally sit inside a provider's `register()` method, move them to the `App\Application::$bindings` property as an array callable.

Because these callbacks are evaluated on-demand only when a service is explicitly requested from the container, **all bindings become implicitly deferred with zero class loading cost during boot**.

<a name="writing-service-providers"></a>
## Writing Service Providers

If your application requires custom bootstrapping logic that must execute during startup (such as registering event listeners or queue failure handlers), you can create a Service Provider.

All service providers extend `MacropaySolutions\Kernel\Support\ServiceProvider`.

The Run CLI can generate a new provider via the `make:provider` command:

```shell
php run make:provider RiakServiceProvider
```

<a name="the-register-method"></a>
### The Register Method

Within the `register` method, you should **only bind things into the [service container](/container)**:

```php
<?php

namespace App\Providers;

use App\Services\Riak\Connection;
use MacropaySolutions\Kernel\Support\ServiceProvider;

class RiakServiceProvider extends ServiceProvider
{
    /**
     * Register any application services.
     */
    public function register(): void
    {
        $this->app->singleton(Connection::class, [\App\Factories\RiakFactory::class, 'createConnection']);
    }
}
```

> [!TIP]  
> If a Service Provider *only* contains container bindings inside `register()`, delete the Service Provider entirely and move its bindings into the `App\Application::$bindings` property to eliminate class instantiation overhead during boot.

<a name="the-boot-method"></a>
### The Boot Method

The `boot` method runs only for the providers passed to `$app->boot(...)`, after all container bindings are available, and it runs on every request. Prefer a [deferred provider](#deferred-providers) that does this work in `register()` behind `afterResolving()`, so it runs only when the service is used. Use `boot()` only for logic that must run on every request. For example, a queue failure listener belongs in a deferred provider:

```php
<?php

namespace App\Providers;

use App\Listeners\QueueFailureListener;
use MacropaySolutions\Kernel\Container\Container;
use MacropaySolutions\Kernel\Contracts\Support\DeferrableProvider;
use MacropaySolutions\Kernel\Queue\QueueManager;
use MacropaySolutions\Kernel\Support\ServiceProvider;

class QueueServiceProvider extends ServiceProvider implements DeferrableProvider
{
    public function register(): void
    {
        // instances/singletons run their resolving events only once
        $this->app->afterResolving('queue', [$this, 'configureQueue']);
    }

    public function configureQueue(QueueManager $queue, Container $app): void
    {
        $queue->failing([QueueFailureListener::class, 'handle']);
    }
}
```

Register it from the `App\Application` method that registers the queue, so it is loaded only when the queue is first resolved:

```php
// App\Application
protected function registerQueueBindings()
{
    parent::registerQueueBindings();
    $this->register(\App\Providers\QueueServiceProvider::class);
}
```

Calling `\app('queue')->failing(...)` from `boot()` instead would resolve the queue on every request, even on requests that never dispatch a job.

<a name="deferred-providers"></a>
### Deferred Providers

Deferred providers are the recommended kind. If your provider is registering bindings in the service container, configuring a service when it is created, or performing component registration (such as calling `loadViewComponentsAs()`), you should defer its registration until one of the provided services is actually needed. Deferring the loading of such a provider will improve the performance of your application because it is not loaded from the filesystem on every request.

To defer a provider you must follow the logic of `MailServiceProvider` from your `\App\Application` file and DO NOT register it in bootstrap/app.php via $app->register(...).
Implement the `MacropaySolutions\Kernel\Contracts\Support\DeferrableProvider` interface and define a `provides(): array` method. Deferred providers are skipped by `Application::boot()` and will never have their `boot()` method called automatically. If a deferred provider requires JIT setup logic upon resolution, register an `$this->app->afterResolving()` array callable inside its `register()` method, like the template's `App\Providers\AuthServiceProvider`, which `App\Application::registerAuthBindings()` registers when auth is first resolved:

```php
public function register(): void
{
    // instances/singletons run their resolving events only once
    $this->app->afterResolving('auth', [$this, 'configureAuthGuard']);
}
```

<a name="registering-providers"></a>
## Registering Providers

Prefer deferred providers (see above). If your provider **is non-deferrable**, register it directly on the `$app` instance inside `bootstrap/app.php`:

```php
$app->register(\App\Providers\AppServiceProvider::class);
```

Registering does not boot a provider. If your provider **has a `boot` method defined**, execute it by passing its class name directly into the variadic `$app->boot(...)` method inside `bootstrap/app.php`; only the providers listed there are booted:

```php
return $app->boot(
    \App\Providers\AppServiceProvider::class,
);
```

If a Service Provider's `register()` logic has been refactored into the `App\Application::$bindings` array, its `$app->register()` line in `bootstrap/app.php` should be commented out or removed entirely.

> [!TIP]  
> If a Service Provider *only* contains a `boot()` method and no `register()` logic, do not call `$app->register()`. Simply pass the class name directly to `$app->boot(...)`.
> When passing any kind of providers to `$app->boot(...)` the framework will Just-In-Time (JIT) instantiate it (if not loaded already), execute the boot method, and if not loaded already, discard the instance without consuming permanent memory in the loaded providers state.

> [!CRITICAL]
> Providers that are omitted from `$app->boot(...)` **will NOT be booted** during application startup.
> **Only providers explicitly listed in** `$app->boot(...)` inside `bootstrap/app.php` will be booted during application startup. Providers MUST declare an explicit `boot()` method if listed in the boot arguments. This `boot` method is not included in autowiring, so be sure to leave its definition without parameters.
> 
> Not allowing providers to be booted automatically, increases the visibility on the boot logic and prevents devs from adding overhead logic on each request when not needed.
