---
title: Configuration
description: Comprehensive guide to configuring the PHP-Framework ecosystem, including environment isolation, runtime value access, explicit bootstrapping, configuration caching, and maintenance mode mechanics.
context: configuration
---

# Configuration

- [Introduction](#introduction)
- [Environment Configuration](#environment-configuration)
  - [Environment Variable Types](#environment-variable-types)
  - [Retrieving Environment Configuration](#retrieving-environment-configuration)
  - [Determining the Current Environment](#determining-the-current-environment)
- [Accessing Configuration Values](#accessing-configuration-values)
- [Configuration Caching](#configuration-caching)
- [Debug Mode](#debug-mode)

<a name="introduction"></a>
## Introduction

All the configuration files for the Framework framework are stored in the `config` directory. Each option is documented, so feel free to look through the files and get familiar with the options available to you.

These configuration files allow you to configure things like your database connection information, your mail server information, as well as various other core configuration values such as your application timezone and encryption key.

<a name="application-overview"></a>
#### Application Overview

In a hurry? You can get a quick overview of your application's configuration, drivers, and environment via the `about` Run command:

```shell
php run about
```

If you're only interested in a particular section of the application overview output, you may filter for that section using the `--only` option:

```shell
php run about --only=environment
```

Or, to explore a specific configuration file's values in detail, you may use the `config:show` Run command:

```shell
php run config:show database
```

<a name="environment-configuration"></a>
## Environment Configuration

It is often helpful to have different configuration values based on the environment where the application is running. For example, you may wish to use a different cache driver locally than you do on your production server.

To make this a cinch, Framework utilizes the [DotEnv](https://github.com/vlucas/phpdotenv) PHP library. In a fresh Framework installation, the root directory of your application will contain a `.env.example` file that defines many common environment variables. During the Framework installation process, this file will automatically be copied to `.env`.

Framework's default `.env` file contains some common configuration values that may differ based on whether your application is running locally or on a production web server. These values are then retrieved from various Framework configuration files within the `config` directory using Framework's `env` function.

If you are developing with a team, you may wish to continue including a `.env.example` file with your application. By putting placeholder values in the example configuration file, other developers on your team can clearly see which environment variables are needed to run your application.

> [!NOTE]  
> Any variable in your `.env` file can be overridden by external environment variables such as server-level or system-level environment variables.

<a name="environment-file-security"></a>
#### Environment File Security

Your `.env` file should not be committed to your application's source control, since each developer / server using your application could require a different environment configuration. Furthermore, this would be a security risk in the event an intruder gains access to your source control repository, since any sensitive credentials would get exposed.

<a name="additional-environment-files"></a>
#### Additional Environment Files

Before loading your application's environment variables, Framework determines if an `APP_ENV` environment variable has been externally provided or if the `--env` CLI argument has been specified. If so, Framework will attempt to load an `.env.[APP_ENV]` file if it exists. If it does not exist, the default `.env` file will be loaded.

<a name="environment-variable-types"></a>
### Environment Variable Types

All variables in your `.env` files are typically parsed as strings, so some reserved values have been created to allow you to return a wider range of types from the `env()` function:

| `.env` Value | `env()` Value |
|--------------|---------------|
| true         | (bool) true   |
| (true)       | (bool) true   |
| false        | (bool) false  |
| (false)      | (bool) false  |
| empty        | (string) ''   |
| (empty)      | (string) ''   |
| null         | (null) null   |
| (null)       | (null) null   |

If you need to define an environment variable with a value that contains spaces, you may do so by enclosing the value in double quotes:

```ini
APP_NAME="My Application"
```

<a name="retrieving-environment-configuration"></a>
### Retrieving Environment Configuration

All the variables listed in the `.env` file will be loaded into the `$_ENV` PHP super-global when your application receives a request. However, you may use the `env` function to retrieve values from these variables in your configuration files. In fact, if you review the Framework configuration files, you will notice many of the options are already using this function:

    'debug' => env('APP_DEBUG', false),

The second value passed to the `env` function is the "default value". This value will be returned if no environment variable exists for the given key.

<a name="determining-the-current-environment"></a>
### Determining the Current Environment

The current application environment is determined via the `APP_ENV` variable from your `.env` file. You may access this value via the `environment` method on the application container instance:

    $environment = \app()->environment();

You may also pass arguments to the `environment` method to determine if the environment matches a given value. The method will return `true` if the environment matches any of the given values:

    if (\app()->environment('local')) {
        // The environment is local
    }

    if (\app()->environment('local', 'staging')) {
        // The environment is either local OR staging...
    }

> [!NOTE]  
> The current application environment detection can be overridden by defining a server-level `APP_ENV` environment variable.

<a name="accessing-configuration-values"></a>
## Accessing Configuration Values

You may easily access your configuration values using the global `config` function from anywhere in your application. The configuration values may be accessed using "dot" syntax, which includes the name of the file and option you wish to access. A default value may also be specified and will be returned if the configuration option does not exist:

    $value = config('app.timezone');

    // Retrieve a default value if the configuration value does not exist...
    $value = config('app.timezone', 'Asia/Seoul');

To set configuration values at runtime, you may pass an array to the `config` function:

    config(['app.timezone' => 'America/Chicago']);

Before a configuration file can be used, you should load it into the application using the `configure` method. This must be explicitly declared within your `bootstrap/app.php` file:

    $app->configure('app');
    $app->configure('database');
    $app->configure('cache');

---

<a name="configuration-caching"></a>
## Configuration Caching

To give your application a speed boost, you should cache all of your configuration files into a single file using the `config:cache` Run command. This will combine all the configuration options for your application into a single file which can be quickly loaded by the framework.

You should typically run the `php run config:cache` command as part of your production deployment process. The command should not be run during local development as configuration options will frequently need to be changed during the course of your application's development.

> [NOTE]  
> In addition to manual compilation via the console tool, configurations are **automatically cached by the framework within `--no-dev` Composer deployment environments**.

Once the configuration has been cached, your application's `.env` file will not be loaded by the framework during requests or Run commands; therefore, the `env` function will only return external, system level environment variables.

For this reason, you should ensure you are only calling the `env` function from within your application's configuration (`config`) files. You can see many examples of this by examining Framework's default configuration files. Configuration values may be accessed from anywhere in your application using the `config` function [described above](#accessing-configuration-values).

The `config:clear` command may be used to purge the cached configuration:

```shell
php run config:clear
```

> [!WARNING]  
> If you execute the `config:cache` command during your deployment process (or when building an optimized `--no-dev` environment), you should be sure that you are only calling the `env()` function from within your configuration files. Once the configuration has been cached, the `.env` file will not be loaded; therefore, the `env` function will only return external, system level environment variables.

<a name="debug-mode"></a>
## Debug Mode

The `debug` option in your `config/app.php` configuration file determines how much information about an error is actually displayed to the user. By default, this option is set to respect the value of the `APP_DEBUG` environment variable, which is stored in your `.env` file.

> [!WARNING]  
> For local development, you should set the `APP_DEBUG` environment variable to `true`. **In your production or public test environment, this value should always be `false`. If the variable is set to `true` in production, you risk exposing sensitive configuration values to your application's end users.**