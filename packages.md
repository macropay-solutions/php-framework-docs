---
title: Package Development
description: Guide to building and integrating custom packages for PHP-Framework.
context: packages
---

# Package Development

- [Introduction](#introduction)
- [Service Providers](#service-providers)
  - [Bindings Without a Provider](#bindings-without-a-provider)
  - [When a Provider Is Needed](#when-a-provider-is-needed)
  - [Registering Your Package Provider](#registering-your-package-provider)
- [Autowiring Discovery](#autowiring-discovery)
- [Resources](#resources)
  - [Configuration](#configuration)
  - [Routes](#routes)
  - [Migrations](#migrations)
  - [Views & Language Files](#views--language-files)
- [Commands](#commands)
- [Manual Asset Installation](#manual-asset-installation)

## Introduction

Packages are the primary way of adding functionality to PHP-Framework. Some packages are stand-alone and work with any PHP framework via your `composer.json` file, while others are specifically intended for use with PHP-Framework.

*Note: You must always use global helper functions (e.g., `app()`, `config()`, `response()`) or autowiring when building packages.*

### Recommended Package Structure

Packages should include copyable asset directories for consumers to easily publish manually:

    my-org/my-package/
    ├── src/
    │   ├── Factories/
    │   └── Services/
    ├── routes/
    │   └── api.php
    ├── config/
    │   └── my-package.php
    ├── resources/
    │   ├── views/
    │   │   ├── invoice.template.php
    │   │   └── mail/
    │   │       └── welcome.template.php
    │   └── lang/
    │       ├── en/
    │       │   └── messages.php
    │       └── es/
    │           └── messages.php
    ├── composer.json
    └── README.md

## Service Providers

Service providers are optional and not encouraged. Instantiating a provider and calling its methods on every request adds boot overhead, and a package does not need one for its bindings, routes or commands (see the sections below).

### Bindings Without a Provider

Ship your factories as array callables and document the entries consumers must add to their `App\Application::$bindings` property:

```php
// App\Application
protected array $bindings = [
    // ...
    \Vendor\Package\InvoiceService::class => [
        'concrete' => [\Vendor\Package\Factories\InvoiceFactory::class, 'create'],
        'shared' => true,
    ],
];
```

```php
// Vendor\Package\Factories\InvoiceFactory
public static function create(): InvoiceService
{
    return new InvoiceService(\config('my-package'));
}
```

These bindings are resolved only when the service is requested, so they cost nothing during boot. See [Service Providers & Container Bindings](providers#zero-overhead-container-bindings).

### When a Provider Is Needed

Write a provider only when your package must configure a service as it is created, such as adding a failure listener to the queue or a guard to auth. Make it deferred and put that logic in `register()` behind `afterResolving()`, like the template's `App\Providers\AuthServiceProvider`. It then runs once, the first time the service is resolved, and never on requests that do not use the service:

```php
<?php

namespace Vendor\Package;

use MacropaySolutions\Kernel\Container\Container;
use MacropaySolutions\Kernel\Contracts\Support\DeferrableProvider;
use MacropaySolutions\Kernel\Queue\QueueManager;
use MacropaySolutions\Kernel\Support\ServiceProvider;

class PackageServiceProvider extends ServiceProvider implements DeferrableProvider
{
    public function register(): void
    {
        // instances/singletons run their resolving events only once
        $this->app->afterResolving('queue', [$this, 'configureQueue']);
    }

    public function configureQueue(QueueManager $queue, Container $app): void
    {
        $queue->failing([\Vendor\Package\Listeners\QueueFailureListener::class, 'handle']);
    }
}
```

Avoid `boot()`: a booted provider runs on every request, whether the request needs the package or not.

Because the framework is heavily optimized for zero-overhead boot times, dynamic asset registration methods have been removed:

- ❌ `loadViewsFrom()` — Use manual file copying instead.
- ❌ `loadTranslationsFrom()` — Use manual file copying instead.
- ❌ `loadJsonTranslationsFrom()` — Use manual file copying instead.
- ❌ `loadMigrationsFrom()` — Use manual file copying instead.
- ❌ `publishes()` — Use manual file copying instead.

### Registering Your Package Provider

Providers are never discovered automatically. Consumers register a deferred provider from the `App\Application` method that registers the service it configures, the way the template registers `App\Providers\AuthServiceProvider` in `registerAuthBindings()`. The provider is then loaded only when that service is first needed:

```php
// App\Application
protected function registerQueueBindings()
{
    parent::registerQueueBindings();
    $this->register(\Vendor\Package\PackageServiceProvider::class);
}
```

Do not register it in `bootstrap/app.php` with `$app->register()`, which would load it on every request.

## Autowiring Discovery

While service providers must be manually registered, PHP-Framework does support autodiscovery for **autowiring configuration**. To avoid runtime reflection and improve performance, packages can automatically append their classes to the `autowiring:cache` command by declaring them in their `composer.json` file:

```json
{
    "extra": {
        "php-framework": {
            "autowiring": [
                {
                    "path": "src/ExampleFolder",
                    "methods": []
                },
                {
                    "path": "\\Vendor\\ExampleClass",
                    "methods": []
                }
            ]
        }
    }
}
```

When users deploy their application and run `php run autowiring:cache`, the framework will read these paths and cache the reflection data for all specified constructors and methods. See `app.autowiring` config for more details.

The same command writes the [OPcache preload](deployment#opcache-preloading) script. It lists the `MacropaySolutions\` and `App\` classes plus every autowired class, so declaring your classes here also preloads them in PHP-FPM, whatever their namespace.

## Resources

### Configuration

Package configuration files should be stored in a `config` directory within your package. To make these available to consumers, they must call `$app->configure('package_config')` in `bootstrap/app.php`:

```php
if (!$app->configurationIsCached()) {
    $app->configure('app');
    $app->configure('crufd_wizard');
    $app->configure('package_config');
}
```

The consumer can then manually copy the package's configuration file to their application's `config` directory.

> **Warning:** Configuration files must not contain closures, as they will break the `config:cache` command. All configuration values must be static arrays, strings, integers, or other non-callable values.

### Routes

Ship your routes in a file and have consumers require it from their `App\Router::registerRoutes()` method, next to their own routes file. This needs no provider, and `route:cache` includes the package routes:

```php
// App\Router
public function registerRoutes(): void
{
    $this->group([
        'namespace' => 'App\Http\Controllers',
    ], static function (\App\Router $router): void {
        require $router->app->basePath('routes/web.php');
        require $router->app->basePath('vendor/vendor/package/routes/api.php');
    });
}
```

**Strict Architectural Constraint:** Route Closures are completely forbidden by the internal engine backed by `fast-route` and will throw a `RuntimeException`. All routes defined in your package MUST point to a Controller class method. Use array callables: they hold the fully qualified class name, so the consumer's group namespace is never prepended to them:

```php
// routes/api.php
/** @var \App\Router $router */
$router->post('/package/action', [\Vendor\Package\Http\Controllers\PackageController::class, 'action']);
```

A `Controller@method` string also works, but only with an absolute namespace (a leading `\`); otherwise the group namespace is prepended to it.

`ServiceProvider::loadRoutesFrom()` is meant only for local development and tests, where routes are not cached, such as your package's own test suite booting the application with your provider. `route:cache` builds a new router and runs only `App\Router::registerRoutes()`, so routes loaded from a provider would be missing from the cache: `loadRoutesFrom()` throws a `RuntimeException` once routes are cached, and the route cache files must then be deleted from `bootstrap/cache`, because `php run route:clear` boots the provider too. `loadRoutesFrom()` requires the file inside the provider, where `$this` is the provider and there is no `$router` variable, so the file must use `$this->app->router`. To keep one file that works both ways, start it with:

```php
// routes/api.php
/** @var \App\Router $router */
$router ??= $this->app->router;
```

From `registerRoutes()` the `$router` of the group is used; from `loadRoutesFrom()` it falls back to the application router.

### Migrations

Because `loadMigrationsFrom()` does NOT exist in `ServiceProvider`, package migrations must reside in the application's `database/migrations` directory.

#### Automated Installation (Recommended)

Add the migration folder mapping to your package's `composer.json`:

    "extra": {
        "publish-assets": {
            "your-path/migrations": "database/migrations"
        }
    }

When consumers install or update your package, Composer automatically places the migrations into their `database/migrations` directory without overwriting existing files.

#### Manual Installation Alternative

If consumers do not use the asset publisher plugin, instruct them in your `README.md` to copy the migration files manually:

    cp -r vendor/my-vendor/my-package/your-path/migrations/* database/migrations/

### Views & Language Files

Because `loadViewsFrom()` and `loadTranslationsFrom()` do **NOT** exist in PHP-Framework, view templates and language files must reside in the application space.

Instruct consumers to publish these via `publish-assets`:

    "extra": {
        "publish-assets": {
            "resources/views": "resources/views/vendor/my-package",
            "resources/lang": "resources/lang"
        }
    }

View files in the application space are then rendered using standard dot-notation:

    return view('vendor.my-package.invoice', $data);

### Commands

Consumers list your package's console commands in the `$commands` property of their `App\Console\Kernel`, so no provider is needed:

```php
// App\Console\Kernel
protected $commands = [
    \Vendor\Package\Console\Commands\InstallCommand::class,
    \Vendor\Package\Console\Commands\NetworkCommand::class,
];
```

> **NOTE:** Please register these commands in your `composer.json` for the autowiring cache to optimize their execution.

## Manual Asset Installation

Because PHP-Framework is tailored for maximum API performance, if your package requires configuration files, public assets, database migrations, or HTML view templates (such as pagination or email layouts) to be moved into the consuming application, you must document the manual copy commands directly in your `README.md` so consumers can add them to their installation steps or CI/CD build scripts.

### Automated Asset Installation (Composer Alternative)

While documenting manual copy commands works, the recommended approach is to utilize the framework's built-in Composer asset publisher plugin. The `macropay-solutions/php-kernel-dev` dependency ships with a Composer plugin that completely bypasses the HTTP boot sequence while automating asset deployment.

In your package's `composer.json`, instruct the plugin which files to publish by defining the `publish-assets` configuration within the `extra` block:

    {
        "extra": {
            "publish-assets": {
                "config/my-package.php": "config/my-package.php",
                "database/migrations": "database/migrations",
                "resources/views": "resources/views/vendor/my-package",
                "resources/lang": "resources/lang",
                "public": "public/vendor/my-package"
            }
        }
    }

The keys are the original package local path and the values are the destination path.

> **NOTE**
> When consumers **install** or **update** your package, Composer will automatically synchronize these files into their application space **without overwriting** any existing files. If you need them republished, simply delete the local copies and run `composer reinstall <vendor>/<package>`.
> Consumers will still need to manually register the service provider.