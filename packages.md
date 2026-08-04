---
title: Package Development
description: Guide to building and integrating custom packages for PHP-Framework.
context: packages
---

# Package Development

- [Introduction](#introduction)
- [Service Providers](#service-providers)
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
    │   ├── MyPackageProvider.php
    │   └── Services/
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

Service providers are the connection point between your package and PHP-Framework. A service provider is responsible for binding things into the framework's service container (which directly manages its own explicit `$availableBindings` map) and informing PHP-Framework where to load package resources.

A service provider extends the `MacropaySolutions\Kernel\Support\ServiceProvider` class and contains two methods: `register` and `boot`.

### What Works vs. What Does NOT Work

Because the framework is heavily optimized for zero-overhead boot times, dynamic asset registration methods have been removed.

**What Works (Do This):**
Only use the `register()` method with container bindings:

```php
public function register(): void
{
    $this->app->singleton(InvoiceService::class, function () {
        return new InvoiceService(config('my-package'));
    });
}
```

**What Does NOT Work (MISSING Methods):**
- ❌ `loadViewsFrom()` — Use manual file copying instead.
- ❌ `loadTranslationsFrom()` — Use manual file copying instead.
- ❌ `loadJsonTranslationsFrom()` — Use manual file copying instead.
- ❌ `loadMigrationsFrom()` — Use manual file copying instead.
- ❌ `publishes()` — Use manual file copying instead.

### Registering Your Package Provider

For consumers to use your package, they must explicitly register your service provider in their application. This is done in `bootstrap/app.php`:

```php
$app->register(\Vendor\Package\PackageServiceProvider::class);
```

This ensures full control over package initialization and avoids automatic discovery that could introduce performance overhead or unwanted side effects.

If your service provider's boot method is empty, it can be registered as a deferred provider analog to `\MacropaySolutions\Kernel\Mail\MailServiceProvider`. See `\App\Application::registerMailBindings`. Alternatively, the bindings can be manually registered into `\App\Application::registerExplicitBindingsMap`.

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

When users deploy their application and run `php run autowiring:cache`, the framework will read these paths and cache the reflection data for all specified constructors and methods. See `app.autowirings` config for more details.

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

If your package contains routes, you may load them using the `loadRoutesFrom` method. This method will automatically determine if the application's routes are cached and will not load your routes file if the routes have already been cached:

```php
public function boot(): void
{
    $this->loadRoutesFrom(__DIR__ . '/../routes/api.php');
}
```

**Strict Architectural Constraint:** Route Closures are completely forbidden by the internal engine backed by `fast-route` and will throw a `RuntimeException`. All routes defined in your package MUST point to a Controller class method. Using an absolute namespace is recommended to prevent namespace grouping collisions:

```php
// routes/api.php
$router->post('/package/action', '\Vendor\Package\Http\Controllers\PackageController@action');
```

### Migrations

If your package contains database migrations, instruct your users to copy them in their migrations folder.


Once registered, they will automatically run when consumers execute `php run migrate`.

> **Warning:** You should instruct your users to manually copy the migration files into their application's `database/migrations` folder as the `loadMigrationsFrom` method does NOT exist in the ServiceProvider.

### Views & Language Files

**Note:** PHP-Framework is optimized for stateless JSON APIs. The `loadViewsFrom` and `loadTranslationsFrom` helpers (as well as double-colon namespaces like `namespace::view` and `namespace::file.key`) **do NOT exist** to eliminate runtime container closure bindings and filesystem scans on every request.

If your package includes HTML views or translation files, instruct users to copy them directly into the application space (`resources/views/vendor/{package}` and `resources/lang/en/{file}.php`). Resources are then referenced using standard dot-notation:

```php
// Resolves resources/views/vendor/my-package/mail.template.php
return view('vendor.my-package.mail', $data);
```

## Commands

To register your package's console commands, you may use the `commands` method in your service provider's `boot` method when the application is running in console mode:

```php
use Vendor\Package\Console\Commands\InstallCommand;
use Vendor\Package\Console\Commands\NetworkCommand;

public function boot(): void
{
    if ($this->app->runningInConsole()) {
        $this->commands([
            InstallCommand::class,
            NetworkCommand::class,
        ]);
    }
}
```

> **NOTE:** Please register these commands in your `composer.json` for the autowiring cache to optimize their execution.

## Manual Asset Installation

Because PHP-Framework is tailored for maximum API performance, if your package requires configuration files, public assets, database migrations, or HTML view templates (such as pagination or email layouts) to be moved into the consuming application, you must document the manual copy commands directly in your `README.md` so consumers can add them to their installation steps or CI/CD build scripts.
