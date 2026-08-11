---
title: Deployment
description: Guide to server requirements, Nginx configuration, and production optimizations for deploying PHP-Framework applications.
context: deployment
---

# Deployment


> **CRITICAL ARCHITECTURAL WARNING**
>
> Because PHP-Framework's high-performance router utilizes native `strtok()` loops to parse URI nodes with zero memory allocations, **this framework is strictly designed for a stateless, isolated PHP-FPM execution architecture.**
>
> Do **NOT** run PHP-Framework with long-running, multithreaded, or coroutine-based application servers (such as **Swoole, OpenSwoole, or RoadRunner**).
>
> Because `strtok()` relies on a single global internal pointer within the PHP thread state, concurrent asynchronous requests sharing the same process worker will overwrite each other's routing tokens mid-flight. This will result in critical security vulnerabilities, including routing desynchronization, cross-user data leaks, and authentication middleware bypasses.

- [Introduction](#introduction)
- [Server Requirements](#server-requirements)
- [Server Configuration](#server-configuration)
    - [Nginx](#nginx)
- [Optimization](#optimization)
    - [Autoloader Optimization](#autoloader-optimization)
    - [Caching Configuration](#caching-configuration)
    - [Caching Events](#caching-events)
    - [Caching Routes](#caching-routes)
    - [Caching Views](#caching-views)
    - [Caching Autowiring](#caching-autowiring)
    - [Caching Commands](#caching-commands)
    - [Merging Cached Files](#merging-cached-files)
- [Debug Mode](#debug-mode)

<a name="introduction"></a>
## Introduction

When you're ready to deploy your PHP-Framework application to production, there are some important things you can do to make sure your application is running as efficiently as possible. In this document, we'll cover some great starting points for making sure your PHP-Framework application is deployed properly.

<a name="server-requirements"></a>
## Server Requirements

PHP-Framework has a few system requirements. You should ensure that your web server has the following minimum PHP version and extensions:

<div class="content-list" markdown="1">

- PHP >= 8.2
- Ctype PHP Extension
- cURL PHP Extension
- DOM PHP Extension
- Fileinfo PHP Extension
- Filter PHP Extension
- Hash PHP Extension
- Mbstring PHP Extension
- OpenSSL PHP Extension
- PCRE PHP Extension
- PDO PHP Extension
- Session PHP Extension
- Tokenizer PHP Extension
- XML PHP Extension
- See more in [PHP Kernel composer.json](https://github.com/macropay-solutions/php-kernel/blob/production/composer.json) and [PHP-Framework composer.json](https://github.com/macropay-solutions/php-framework/blob/production/composer.json)

</div>

<a name="server-configuration"></a>
## Server Configuration

<a name="nginx"></a>
### Nginx

If you are deploying your application to a server that is running Nginx, you may use the following configuration file as a starting point for configuring your web server. Most likely, this file will need to be customized depending on your server's configuration.

Please ensure, like the configuration below, your web server directs all requests to your application's `public/index.php` file. You should never attempt to move the `index.php` file to your project's root, as serving the application from the project root will expose many sensitive configuration files to the public Internet:

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name example.com;
    root /srv/example.com/public;

    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-Content-Type-Options "nosniff";

    index index.php;

    charset utf-8;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location = /favicon.ico { access_log off; log_not_found off; }
    location = /robots.txt  { access_log off; log_not_found off; }

    error_page 404 /index.php;

    location ~ \.php$ {
        fastcgi_pass unix:/var/run/php/php8.2-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        include fastcgi_params;
    }

    location ~ /\.(?!well-known).* {
        deny all;
    }
}
```

<a name="optimization"></a>
## Optimization

<a name="autoloader-optimization"></a>
### Autoloader Optimization

When deploying to production, make sure that you are optimizing Composer's class autoloader map so Composer can quickly find the proper file to load for a given class:

```shell
composer install --optimize-autoloader --no-dev
```

> [!NOTE]  
> In addition to optimizing the autoloader, you should always be sure to include a `composer.lock` file in your project's source control repository. Your project's dependencies can be installed much faster when a `composer.lock` file is present.

By using the `--classmap-authoritative` flag, Composer:

- disables filesystem scanning for classes not found in the classmap (dynamically generated class aliases not explicitly in the map will still work provided they are handled by a custom autoloader prepend, such as: `spl_autoload_register([$this, 'load'], true, true);`).
- improves performance because `class_exists()` calls return immediately — no fallback to PSR-4/PSR-0 filesystem checks.
- assumes the classmap is complete: if a class isn’t in the map, it doesn’t exist.

> [!NOTE]
> Class aliases created with class_alias() are not included in the classmap automatically, so those classes may fail to autoload.

When using `--no-scripts` flag, be sure to call:

```shell
composer dump-autoload
```
or explicitly call all the cache commands from the composer.json->scripts->post-autoload-dump of the template you are using (PHP-Framework). This will improve the boot time of the application. See more below.

<a name="caching-configuration"></a>
### Caching Configuration

When deploying your application to production, you should make sure that you run the `config:cache` Run command during your deployment process:

```shell
php run config:cache
```

This command will combine all of PHP-Framework's configuration files into a single, cached file, which greatly reduces the number of trips the framework must make to the filesystem when loading your configuration values.

> [!WARNING]  
> If you execute the `config:cache` command during your deployment process, you should be sure that you are only calling the `env` function from within your configuration files. Once the configuration has been cached, the `.env` file will not be loaded and all calls to the `env` function for `.env` variables will return `null`.

<a name="caching-events"></a>
### Caching Events

If your application is utilizing [event discovery](/events#event-discovery), you should cache your application's event to listener mappings during your deployment process. This can be accomplished by invoking the `event:cache` Run command during deployment:

```shell
php run event:cache
```

> [!NOTE]
>It can also include the observers.

<a name="caching-routes"></a>
### Caching Routes

If you are building a large application with many routes, you should make sure that you are running the `route:cache` Run command during your deployment process:

```shell
php run route:cache
```

This command reduces all of your route registrations into a single method call within a cached file, improving the performance of route registration when registering hundreds of routes.

<a name="caching-views"></a>
### Caching Views

When deploying your application to production, you should make sure that you run the `view:cache` Run command during your deployment process:

```shell
php run view:cache
```

This command precompiles all your Template views so they are not compiled on demand, improving the performance of each request that returns a view.

<a name="caching-autowiring"></a>
### Caching Autowiring

When deploying your application to production, you should make sure that you run the `autowiring:cache` Run command during your deployment process:

```shell
php run autowiring:cache
```

This avoids runtime reflection on method and construct autowire, improving the performance.

Packages can auto add to this config via their composer.json:

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

To configure what classes should be cached, add in` app.autowiring` config the list:
These are the defaults:
```php
    /**
     * run autowiring:cache source paths for public methods (except __construct which is implicitly handled)
     * The CallQueuedHandler, controllers, middlewares, built-in commands, service providers, macroable classes
     * + other classes resolved from Container during the autowiring:cache command execution are handled automatically
     * 'path' can be a single class FQN or a directory path
     * This allows you to add autowiring to any constructor/method from a class if you want
     * when you resolve that class from the container or use BoundMethod::call to call that method.
     * Use '*' for all public methods.
     * Packages can auto add to this config via their composer.json:
     * {
     *  "extra": {
     *   "php-framework": {
     *    "autowiring": [
     *     {
     *      "path": "src/ExampleFolder",
     *      "methods": []
     *     },
     *     {
     *      "path": "\\Vendor\\ExampleClass",
     *      "methods": []
     *     }
     *    ]
     *   }
     *  }
     * }
     */
    'autowiring' => [
        [
            'path' => \app()->path() . DIRECTORY_SEPARATOR . 'Console' . DIRECTORY_SEPARATOR . 'Commands',
            'methods' => ['handle', '__invoke'],
        ],
        [
            'path' => \app()->path() . DIRECTORY_SEPARATOR . 'Jobs',
            'methods' => ['handle', '__invoke'],
        ],
        [
            'path' => \app()->path() . DIRECTORY_SEPARATOR . 'Http' . DIRECTORY_SEPARATOR . 'Requests',
            'methods' => ['validator', 'authorize', 'after', 'rules'],
        ],
        [
            'path' => \app()->path() . DIRECTORY_SEPARATOR . 'Listeners',
            'methods' => [],
        ],
    ],

```
Check `bootstrap/cache/autowiring.php`.
> [!NOTE]
> **Autowiring Default Parameter Precedence**
> If you want to prioritize default parameter values (such as `= null`) over attempting to autowire and instantiate unbound classes, you can override the `DEFAULT_PARAMETER_TAKES_PRECEDENCE_WHEN_AUTOWIRING` constant to `true` in your `\App\Application` class. This provides an additional performance boost by safely bypassing dependency resolution attempts (and potential native PHP `\Error`s) for abstract classes or unbound interfaces that have default values. This applies if the parameter is not provided.

> [!NOTE]
> If the parameters are sent as array list, concrete will be instantiated directly with them. On failure, it will default to the old reflection but at the cost of building an Exception.
>
>If the first parameters are sent as list and the last one(s) need to be auto-resolved, the above exception scenario will happen, which is slow. Always send all parameters as list, in the right order.
>
> The method autowiring (so non __construct) does not support parameters as list!
>
> The listeners are resolved from container to check if they should be queued.
>
> This allows you to add autowiring to any constructor/method from a class if you want when you resolve that class from the container or use BoundMethod::call to call that method.

<a name="caching-commands"></a>
### Caching Commands

When deploying your application to production, you should make sure that you run the `commands:cache` Run command during your deployment process:

```shell
php run commands:cache
```

This avoids runtime reflection and instantiation on all commands, improving the performance.

<a name="merging-cached-files"></a>
### Merging Cached Files

Call this AFTER all cache commands have already run:

```shell
php run merge-cached-files:cache
```

This improves boot speed.


<a name="debug-mode"></a>
## Debug Mode

The debug option in your config/app.php configuration file determines how much information about an error is actually displayed to the user. By default, this option is set to respect the value of the `APP_DEBUG` environment variable, which is stored in your application's `.env` file.

> [!WARNING]  
> **In your production environment, this value should always be `false`. If the `APP_DEBUG` variable is set to `true` in production, you risk exposing sensitive configuration values to your application's end users.**
