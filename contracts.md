---
title: Contracts
description: Core interfaces defining services and architectural boundaries in PHP Framework.
context: contracts
---
# Contracts

- [Introduction](#introduction)
    - [Contracts vs. Container Helpers](#contracts-vs-helpers)
- [When to Use Contracts](#when-to-use-contracts)
- [How to Use Contracts](#how-to-use-contracts)
- [Contract Reference](#contract-reference)

<a name="introduction"></a>
## Introduction

Framework's "contracts" are a set of interfaces that define the core services provided by the framework. For example, an `MacropaySolutions\Kernel\Contracts\Queue\Queue` contract defines the methods needed for queueing jobs, while the `MacropaySolutions\Kernel\Contracts\Mail\Mailer` contract defines the methods needed for sending e-mail.

Each contract has a corresponding implementation provided by the framework. For example, Framework provides a queue implementation with a variety of drivers, and a mailer implementation that is powered by [Symfony Mailer](https://symfony.com/doc/6.0/mailer.html).

All the Framework contracts live in [their own GitHub repository](https://github.com/kernel/contracts). This provides a quick reference point for all available contracts, as well as a single, decoupled package that may be utilized when building packages that interact with Framework services.

<a name="contracts-vs-helpers"></a>
### Contracts vs. Container Helpers

Framework's helper functions provide a simple way of utilizing Framework's services without needing to type-hint and resolve contracts out of the service container.

Unlike helpers, which do not require you to inject them in your class' constructor, contracts allow you to define explicit dependencies for your classes. Some developers prefer to explicitly define their dependencies in this way and therefore prefer to use contracts, while other developers enjoy the convenience of global helpers.

<a name="when-to-use-contracts"></a>
## When to Use Contracts

The decision to use contracts or global helpers will come down to personal taste and the tastes of your development team. Both contracts and global functions can be used to create robust, well-tested Framework applications.

If you are building a package that integrates with multiple PHP frameworks you may wish to use the `kernel/contracts` package to define your integration with Framework's services.

<a name="how-to-use-contracts"></a>
## How to Use Contracts

So, how do you get an implementation of a contract? It's actually quite simple.

Many types of classes in Framework are resolved through the [service container](/container), including controllers, event listeners, middleware, queued jobs. So, to get an implementation of a contract, you can just "type-hint" the interface in the constructor of the class being resolved.

For example, take a look at this event listener:

    <?php

    namespace App\Listeners;

    use App\Events\OrderWasPlaced;
    use App\Models\User;
    use MacropaySolutions\Kernel\Contracts\Redis\Factory;

    class CacheOrderInformation
    {
        /**
         * Create a new event handler instance.
         */
        public function __construct(
            protected Factory $redis,
        ) {}

        /**
         * Handle the event.
         */
        public function handle(OrderWasPlaced $event): void
        {
            // ...
        }
    }

When the event listener is resolved, the service container will read the type-hints on the constructor of the class, and inject the appropriate value. To learn more about registering things in the service container, check out [its documentation](/container).

<a name="contract-reference"></a>
## Contract Reference

This table provides a quick reference to all the Framework contracts and their equivalent container bindings:

| Contract                                                                                                                                                                                      | Container Binding         |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------|
| [MacropaySolutions\Kernel\Contracts\Auth\Access\Authorizable](https://github.com/macropay-solutions/php-kernel/tree/production/kernel/Contracts/Auth/Access/Authorizable.php)                 |  &nbsp;                   |
| [MacropaySolutions\Kernel\Contracts\Auth\Access\Gate](https://github.com/macropay-solutions/php-kernel/tree/production/kernel/Contracts/Auth/Access/Gate.php)                                 | `Gate`                    |
| [MacropaySolutions\Kernel\Contracts\Auth\Authenticatable](https://github.com/macropay-solutions/php-kernel/tree/production/kernel/Contracts/Auth/Authenticatable.php)                         |  &nbsp;                   |
| [MacropaySolutions\Kernel\Contracts\Auth\CanResetPassword](https://github.com/macropay-solutions/php-kernel/tree/production/kernel/Contracts/Auth/CanResetPassword.php)                       | &nbsp;                    |
| [MacropaySolutions\Kernel\Contracts\Auth\Factory](https://github.com/macropay-solutions/php-kernel/tree/production/kernel/Contracts/Auth/Factory.php)                                         | `Auth`                    |
| [MacropaySolutions\Kernel\Contracts\Auth\Guard](https://github.com/macropay-solutions/php-kernel/tree/production/kernel/Contracts/Auth/Guard.php)                                             | `Auth::guard()`           |
| [MacropaySolutions\Kernel\Contracts\Auth\PasswordBroker](https://github.com/macropay-solutions/php-kernel/tree/production/kernel/Contracts/Auth/PasswordBroker.php)                           | `Password::broker()`      |
| [MacropaySolutions\Kernel\Contracts\Auth\PasswordBrokerFactory](https://github.com/macropay-solutions/php-kernel/tree/production/kernel/Contracts/Auth/PasswordBrokerFactory.php)             | `Password`                |
| [MacropaySolutions\Kernel\Contracts\Auth\StatefulGuard](https://github.com/macropay-solutions/php-kernel/tree/production/kernel/Contracts/Auth/StatefulGuard.php)                             | &nbsp;                    |
| [MacropaySolutions\Kernel\Contracts\Auth\SupportsBasicAuth](https://github.com/macropay-solutions/php-kernel/tree/production/kernel/Contracts/Auth/SupportsBasicAuth.php)                     | &nbsp;                    |
| [MacropaySolutions\Kernel\Contracts\Auth\UserProvider](https://github.com/macropay-solutions/php-kernel/tree/production/kernel/Contracts/Auth/UserProvider.php)                               | &nbsp;                    |
| [MacropaySolutions\Kernel\Contracts\Bus\Dispatcher](https://github.com/macropay-solutions/php-kernel/tree/production/kernel/Contracts/Bus/Dispatcher.php)                                     | `Bus`                     |
| [MacropaySolutions\Kernel\Contracts\Bus\QueueingDispatcher](https://github.com/macropay-solutions/php-kernel/tree/production/kernel/Contracts/Bus/QueueingDispatcher.php)                     | `Bus::dispatchToQueue()`  |
| [MacropaySolutions\Kernel\Contracts\Broadcasting\Factory](https://github.com/macropay-solutions/php-kernel/tree/production/kernel/Contracts/Broadcasting/Factory.php)                         | `Broadcast`               |
| [MacropaySolutions\Kernel\Contracts\Broadcasting\Broadcaster](https://github.com/macropay-solutions/php-kernel/tree/production/kernel/Contracts/Broadcasting/Broadcaster.php)                 | `Broadcast::connection()` |
| [MacropaySolutions\Kernel\Contracts\Broadcasting\ShouldBroadcast](https://github.com/macropay-solutions/php-kernel/tree/production/kernel/Contracts/Broadcasting/ShouldBroadcast.php)         | &nbsp;                    |
| [MacropaySolutions\Kernel\Contracts\Broadcasting\ShouldBroadcastNow](https://github.com/macropay-solutions/php-kernel/tree/production/kernel/Contracts/Broadcasting/ShouldBroadcastNow.php)   | &nbsp;                    |
| [MacropaySolutions\Kernel\Contracts\Cache\Factory](https://github.com/macropay-solutions/php-kernel/tree/production/kernel/Contracts/Cache/Factory.php)                                       | `cache`                   |
| [MacropaySolutions\Kernel\Contracts\Cache\Lock](https://github.com/macropay-solutions/php-kernel/tree/production/kernel/Contracts/Cache/Lock.php)                                             | &nbsp;                    |
| [MacropaySolutions\Kernel\Contracts\Cache\LockProvider](https://github.com/macropay-solutions/php-kernel/tree/production/kernel/Contracts/Cache/LockProvider.php)                             | &nbsp;                    |
| [MacropaySolutions\Kernel\Contracts\Cache\Repository](https://github.com/macropay-solutions/php-kernel/tree/production/kernel/Contracts/Cache/Repository.php)                                 | `app('cache')->driver()`         |
| [MacropaySolutions\Kernel\Contracts\Cache\Store](https://github.com/macropay-solutions/php-kernel/tree/production/kernel/Contracts/Cache/Store.php)                                           | &nbsp;                    |
| [MacropaySolutions\Kernel\Contracts\Config\Repository](https://github.com/macropay-solutions/php-kernel/tree/production/kernel/Contracts/Config/Repository.php)                               | `Config`                  |
| [MacropaySolutions\Kernel\Contracts\Console\Application](https://github.com/macropay-solutions/php-kernel/tree/production/kernel/Contracts/Console/Application.php)                           | &nbsp;                    |
| [MacropaySolutions\Kernel\Contracts\Console\Kernel](https://github.com/macropay-solutions/php-kernel/tree/production/kernel/Contracts/Console/Kernel.php)                                     | `MacropaySolutions\Kernel\Contracts\Console\Kernel`       |
| [MacropaySolutions\Kernel\Contracts\Container\Container](https://github.com/macropay-solutions/php-kernel/tree/production/kernel/Contracts/Container/Container.php)                           | `app`                     |
| [MacropaySolutions\Kernel\Contracts\Cookie\Factory](https://github.com/macropay-solutions/php-kernel/tree/production/kernel/Contracts/Cookie/Factory.php)                                     | `Cookie`                  |
| [MacropaySolutions\Kernel\Contracts\Cookie\QueueingFactory](https://github.com/macropay-solutions/php-kernel/tree/production/kernel/Contracts/Cookie/QueueingFactory.php)                     | `Cookie::queue()`         |
| [MacropaySolutions\Kernel\Contracts\Database\ModelIdentifier](https://github.com/macropay-solutions/php-kernel/tree/production/kernel/Contracts/Database/ModelIdentifier.php)                 | &nbsp;                    |
| [MacropaySolutions\Kernel\Contracts\Debug\ExceptionHandler](https://github.com/macropay-solutions/php-kernel/tree/production/kernel/Contracts/Debug/ExceptionHandler.php)                     | &nbsp;                    |
| [MacropaySolutions\Kernel\Contracts\Encryption\Encrypter](https://github.com/macropay-solutions/php-kernel/tree/production/kernel/Contracts/Encryption/Encrypter.php)                         | `encrypter`               |
| [MacropaySolutions\Kernel\Contracts\Events\Dispatcher](https://github.com/macropay-solutions/php-kernel/tree/production/kernel/Contracts/Events/Dispatcher.php)                               | `Event`                   |
| [MacropaySolutions\Kernel\Contracts\Filesystem\Cloud](https://github.com/macropay-solutions/php-kernel/tree/production/kernel/Contracts/Filesystem/Cloud.php)                                 | `app('filesystem')->cloud()`        |
| [MacropaySolutions\Kernel\Contracts\Filesystem\Factory](https://github.com/macropay-solutions/php-kernel/tree/production/kernel/Contracts/Filesystem/Factory.php)                             | `Storage`                 |
| [MacropaySolutions\Kernel\Contracts\Filesystem\Filesystem](https://github.com/macropay-solutions/php-kernel/tree/production/kernel/Contracts/Filesystem/Filesystem.php)                       | `app('filesystem')->disk()`         |
| [MacropaySolutions\Kernel\Contracts\Foundation\Application](https://github.com/macropay-solutions/php-kernel/tree/production/kernel/Contracts/Foundation/Application.php)                     | `App`                     |
| [MacropaySolutions\Kernel\Contracts\Hashing\Hasher](https://github.com/macropay-solutions/php-kernel/tree/production/kernel/Contracts/Hashing/Hasher.php)                                     | `hash`                    |
| [MacropaySolutions\Kernel\Contracts\Http\Kernel](https://github.com/macropay-solutions/php-kernel/tree/production/kernel/Contracts/Http/Kernel.php)                                           | &nbsp;                    |
| [MacropaySolutions\Kernel\Contracts\Mail\MailQueue](https://github.com/macropay-solutions/php-kernel/tree/production/kernel/Contracts/Mail/MailQueue.php)                                     | `Mail::queue()`           |
| [MacropaySolutions\Kernel\Contracts\Mail\Mailable](https://github.com/macropay-solutions/php-kernel/tree/production/kernel/Contracts/Mail/Mailable.php)                                       | &nbsp;                    |
| [MacropaySolutions\Kernel\Contracts\Mail\Mailer](https://github.com/macropay-solutions/php-kernel/tree/production/kernel/Contracts/Mail/Mailer.php)                                           | `Mail`                    |
| [MacropaySolutions\Kernel\Contracts\Notifications\Dispatcher](https://github.com/macropay-solutions/php-kernel/tree/production/kernel/Contracts/Notifications/Dispatcher.php)                 | `Notification`            |
| [MacropaySolutions\Kernel\Contracts\Notifications\Factory](https://github.com/macropay-solutions/php-kernel/tree/production/kernel/Contracts/Notifications/Factory.php)                       | `Notification`            |
| [MacropaySolutions\Kernel\Contracts\Pagination\LengthAwarePaginator](https://github.com/macropay-solutions/php-kernel/tree/production/kernel/Contracts/Pagination/LengthAwarePaginator.php)   | &nbsp;                    |
| [MacropaySolutions\Kernel\Contracts\Pagination\Paginator](https://github.com/macropay-solutions/php-kernel/tree/production/kernel/Contracts/Pagination/Paginator.php)                         | &nbsp;                    |
| [MacropaySolutions\Kernel\Contracts\Pipeline\Hub](https://github.com/macropay-solutions/php-kernel/tree/production/kernel/Contracts/Pipeline/Hub.php)                                         | &nbsp;                    |
| [MacropaySolutions\Kernel\Contracts\Pipeline\Pipeline](https://github.com/macropay-solutions/php-kernel/tree/production/kernel/Contracts/Pipeline/Pipeline.php)                               | `pipeline`                |
| [MacropaySolutions\Kernel\Contracts\Queue\EntityResolver](https://github.com/macropay-solutions/php-kernel/tree/production/kernel/Contracts/Queue/EntityResolver.php)                         | &nbsp;                    |
| [MacropaySolutions\Kernel\Contracts\Queue\Factory](https://github.com/macropay-solutions/php-kernel/tree/production/kernel/Contracts/Queue/Factory.php)                                       | `Queue`                   |
| [MacropaySolutions\Kernel\Contracts\Queue\Job](https://github.com/macropay-solutions/php-kernel/tree/production/kernel/Contracts/Queue/Job.php)                                               | &nbsp;                    |
| [MacropaySolutions\Kernel\Contracts\Queue\Monitor](https://github.com/macropay-solutions/php-kernel/tree/production/kernel/Contracts/Queue/Monitor.php)                                       | `Queue`                   |
| [MacropaySolutions\Kernel\Contracts\Queue\Queue](https://github.com/macropay-solutions/php-kernel/tree/production/kernel/Contracts/Queue/Queue.php)                                           | `Queue::connection()`     |
| [MacropaySolutions\Kernel\Contracts\Queue\QueueableCollection](https://github.com/macropay-solutions/php-kernel/tree/production/kernel/Contracts/Queue/QueueableCollection.php)               | &nbsp;                    |
| [MacropaySolutions\Kernel\Contracts\Queue\QueueableEntity](https://github.com/macropay-solutions/php-kernel/tree/production/kernel/Contracts/Queue/QueueableEntity.php)                       | &nbsp;                    |
| [MacropaySolutions\Kernel\Contracts\Queue\ShouldQueue](https://github.com/macropay-solutions/php-kernel/tree/production/kernel/Contracts/Queue/ShouldQueue.php)                               | &nbsp;                    |
| [MacropaySolutions\Kernel\Contracts\Redis\Factory](https://github.com/macropay-solutions/php-kernel/tree/production/kernel/Contracts/Redis/Factory.php)                                       | `Redis`                   |
| [MacropaySolutions\Framework\Http\ResponseFactory](https://github.com/macropay-solutions/php-kernel/tree/production/src/Http/ResponseFactory.php)                                              | `Response`                |
| [MacropaySolutions\Framework\Routing\UrlGenerator](https://github.com/macropay-solutions/php-kernel/tree/production/src/Routing/UrlGenerator.php)                                             | `URL`                     |
| [MacropaySolutions\Kernel\Contracts\Routing\UrlRoutable](https://github.com/macropay-solutions/php-kernel/tree/production/kernel/Contracts/Routing/UrlRoutable.php)                           | &nbsp;                    |
| [MacropaySolutions\Kernel\Contracts\Session\Session](https://github.com/macropay-solutions/php-kernel/tree/production/kernel/Contracts/Session/Session.php)                                   | `Session::driver()`       |
| [MacropaySolutions\Kernel\Contracts\Support\Arrayable](https://github.com/macropay-solutions/php-kernel/tree/production/kernel/Contracts/Support/Arrayable.php)                               | &nbsp;                    |
| [MacropaySolutions\Kernel\Contracts\Support\Htmlable](https://github.com/macropay-solutions/php-kernel/tree/production/kernel/Contracts/Support/Htmlable.php)                                 | &nbsp;                    |
| [MacropaySolutions\Kernel\Contracts\Support\Jsonable](https://github.com/macropay-solutions/php-kernel/tree/production/kernel/Contracts/Support/Jsonable.php)                                 | &nbsp;                    |
| [MacropaySolutions\Kernel\Contracts\Support\MessageBag](https://github.com/macropay-solutions/php-kernel/tree/production/kernel/Contracts/Support/MessageBag.php)                             | &nbsp;                    |
| [MacropaySolutions\Kernel\Contracts\Support\MessageProvider](https://github.com/macropay-solutions/php-kernel/tree/production/kernel/Contracts/Support/MessageProvider.php)                   | &nbsp;                    |
| [MacropaySolutions\Kernel\Contracts\Support\Renderable](https://github.com/macropay-solutions/php-kernel/tree/production/kernel/Contracts/Support/Renderable.php)                             | &nbsp;                    |
| [MacropaySolutions\Kernel\Contracts\Support\Responsable](https://github.com/macropay-solutions/php-kernel/tree/production/kernel/Contracts/Support/Responsable.php)                           | &nbsp;                    |
| [MacropaySolutions\Kernel\Contracts\Translation\Loader](https://github.com/macropay-solutions/php-kernel/tree/production/kernel/Contracts/Translation/Loader.php)                             | &nbsp;                    |
| [MacropaySolutions\Kernel\Contracts\Translation\Translator](https://github.com/macropay-solutions/php-kernel/tree/production/kernel/Contracts/Translation/Translator.php)                     | `Lang`                    |
| [MacropaySolutions\Kernel\Contracts\Validation\Factory](https://github.com/macropay-solutions/php-kernel/tree/production/kernel/Contracts/Validation/Factory.php)                             | `Validator`               |
| [MacropaySolutions\Kernel\Contracts\Validation\ImplicitRule](https://github.com/macropay-solutions/php-kernel/tree/production/kernel/Contracts/Validation/ImplicitRule.php)                   | &nbsp;                    |
| [MacropaySolutions\Kernel\Contracts\Validation\Rule](https://github.com/macropay-solutions/php-kernel/tree/production/kernel/Contracts/Validation/Rule.php)                                   | &nbsp;                    |
| [MacropaySolutions\Kernel\Contracts\Validation\ValidatesWhenResolved](https://github.com/macropay-solutions/php-kernel/tree/production/kernel/Contracts/Validation/ValidatesWhenResolved.php) | &nbsp;                    |
| [MacropaySolutions\Kernel\Contracts\Validation\Validator](https://github.com/macropay-solutions/php-kernel/tree/production/kernel/Contracts/Validation/Validator.php)                         | `Validator::make()`       |
| [MacropaySolutions\Kernel\Contracts\View\Engine](https://github.com/macropay-solutions/php-kernel/tree/production/kernel/Contracts/View/Engine.php)                                           | &nbsp;                    |
| [MacropaySolutions\Kernel\Contracts\View\Factory](https://github.com/macropay-solutions/php-kernel/tree/production/kernel/Contracts/View/Factory.php)                                         | `View`                    |
| [MacropaySolutions\Kernel\Contracts\View\View](https://github.com/macropay-solutions/php-kernel/tree/production/kernel/Contracts/View/View.php)                                               | `View::make()`            |
