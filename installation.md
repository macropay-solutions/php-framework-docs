---
title: Installation
description: Requirements and initialization steps for the PHP Framework.
context: framework-setup
---

# Installation

- [Installation](#installation)
    - [Server Requirements](#server-requirements)
    - [Installing PHP Framework](#installing-php-framework)
    - [Configuration](#configuration)
- [Installing Packages](#installing-packages)
    - [Basic Installation](#basic-installation)
    - [Using Package Assets](#using-package-assets)
    - [Framework Core Packages](#framework-core-packages)

<a name="installation"></a>
## Installation


<a name="server-requirements"></a>
### Server Requirements

- PHP >= 8.2
- OpenSSL PHP Extension
- PDO PHP Extension
- Mbstring PHP Extension
- See more in [PHP Kernel composer.json](https://github.com/macropay-solutions/php-kernel/blob/production/composer.json) and [PHP Framework composer.json](https://github.com/macropay-solutions/php-framework/blob/production/composer.json)


<a name="installing-php-framework"></a>
### Installing PHP Framework

PHP Framework utilizes [Composer](http://getcomposer.org) to manage its dependencies. So, before using PHP Framework, make sure you have Composer installed on your machine.

Install PHP Framework by issuing the Composer `create-project` command in your terminal:

    composer create-project --prefer-dist macropay-solutions/php-framework:^1.0 blog



### Serving Your Application

To serve your project locally, you may use the built-in PHP development server:

    php -S localhost:8000 -t public

<a name="configuration"></a>
### Configuration

All the configuration options for the PHP Framework are stored in the `.env` file. Once PHP Framework is installed, you should also [configure your local environment](/php-framework-docs/configuration#environment-configuration).

#### Application Key

The application key is automatically generated and injected into your `.env` file during the `composer create-project` process.

<a name="installing-packages"></a>
## Installing Packages

In this framework, package installation requires **manual asset copying** instead of `vendor:publish` commands to ensure absolute zero-overhead during boot.

<a name="basic-installation"></a>
### Basic Installation

#### Step 1: Require Package

    composer require my-org/my-package

#### Step 2: Register Provider (if needed)

Edit `bootstrap/app.php` and manually register the service provider:

    $app->register(\MyOrg\MyPackage\MyPackageProvider::class);

#### Step 3: Copy Configuration (if package has config/)

    cp vendor/my-org/my-package/config/my-package.php config/

Register the configuration in `bootstrap/app.php`:

    if (!$app->configurationIsCached()) {
        $app->configure('app');
        $app->configure('my-package');
    }

#### Step 4: Copy Views (if package has resources/views/)

    mkdir -p resources/views/vendor/my-package
    cp -r vendor/my-org/my-package/resources/views/* resources/views/vendor/my-package/

#### Step 5: Copy Language Files (if package has resources/lang/)

    cp -r vendor/my-org/my-package/resources/lang/en/* resources/lang/en/
    cp -r vendor/my-org/my-package/resources/lang/es/* resources/lang/es/

<a name="using-package-assets"></a>
### Using Package Assets

#### Views

Reference copied views using dot-notation:

    return view('vendor.my-package.invoice', ['order' => $order]);

#### Translations

Reference copied language files natively:

    echo __('my-package.welcome');

#### Configuration

Access copied config:

    $apiKey = config('my-package.api-key');
    $ttl = config('my-package.cache.ttl');

<a name="framework-core-packages"></a>
### Framework Core Packages

These are already copied/published in the php-framework template.

#### Notifications

    mkdir -p resources/views/vendor/notifications
    cp -r vendor/macropay-solutions/php-kernel/kernel/Notifications/resources/views/* \
          resources/views/vendor/notifications/

Then reference in your mail classes:

    return view('vendor.notifications.email', ['data' => $data]);

#### Pagination

    mkdir -p resources/views/vendor/pagination
    cp -r vendor/macropay-solutions/php-kernel/kernel/Pagination/resources/views/* \
          resources/views/vendor/pagination/
