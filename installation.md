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