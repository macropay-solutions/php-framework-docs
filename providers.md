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

Instead of writing boilerplate `DeferrableProvider` classes, you can map container bindings directly within `App\Application::registerExplicitBindingsMap()` or `App\Application::$availableBindings`. Service providers are only required when explicit `boot()` lifecycle hooks are needed.

<a name="zero-overhead-container-bindings"></a>
## Zero-Overhead Container Bindings (Recommended)

To achieve maximum performance during application boot, IoC bindings should be configured directly on the application instance rather than wrapped in Service Provider classes.

### 1. Available Bindings & Aliases

For core services and autoloaded singletons, define your available bindings and class aliases directly within your application class (`App\Application`):

- `$availableBindings`: Maps abstract keys or interfaces to their concrete resolvers.
- `registerContainerAliases()`: Registers container aliases without instantiating provider objects.

### 2. Explicit Binding Maps

If you have custom service bindings that would traditionally sit inside a provider's `register()` method, move them to `App\Application::registerExplicitBindingsMap()`.

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
        $this->app->singleton(Connection::class, function () {
            return new Connection(\app('config')->get('riak'));
        });
    }
}
```

> [!TIP]  
> If a Service Provider *only* contains container bindings inside `register()`, delete the Service Provider entirely and move its bindings into `App\Application::registerExplicitBindingsMap()` to eliminate class instantiation overhead during boot.

<a name="the-boot-method"></a>
### The Boot Method

The `boot` method is called after all container bindings are available. This is where cross-cutting concerns and event listeners should be registered:

```php
<?php

namespace App\Providers;

use App\Listeners\QueueFailureListener;
use MacropaySolutions\Kernel\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    /**
     * Bootstrap any application services.
     */
    public function boot(): void
    {
        \app('queue')->failing([QueueFailureListener::class, 'handle']);
    }
}
```

<a name="deferred-providers"></a>
### Deferred Providers

If your provider is **only** registering bindings in the service container or performing component registration (such as calling `loadViewComponentsAs()`), you should defer its registration until one of the provided services is actually needed. Deferring the loading of such a provider will improve the performance of your application because it is not loaded from the filesystem on every request.

To defer a provider you must follow the logic of `MailServiceProvider` from your `\App\Application` file. Implementing the `MacropaySolutions\Kernel\Contracts\Support\DeferrableProvider` interface will not help.

> [!NOTE]  
> Component and view helper registrations like `$this->loadViewComponentsAs()` **must only be invoked from a deferred service provider** (such as `MailServiceProvider`). This guarantees that component registration closures never get pushed into the container during standard HTTP API requests.

<a name="registering-providers"></a>
## Registering Providers

If your provider defines a `boot` method, register it directly on the `$app` instance inside `bootstrap/app.php`:

```php
$app->register(App\Providers\AppServiceProvider::class);
```

If a Service Provider's `register()` logic has been refactored into `App\Application::registerExplicitBindingsMap()`, its `$app->register()` line in `bootstrap/app.php` should be commented out or removed entirely.