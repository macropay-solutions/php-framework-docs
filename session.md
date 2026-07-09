---
title: HTTP Session
description: Explicit session opt-in flow, configuration, drivers, and custom handlers for PHP-Framework.
context: session
---

# HTTP Session

- [Introduction](#introduction)
  - [Explicit Session Opt-In Flow](#explicit-session-opt-in-flow)
  - [Configuration](#configuration)
  - [Driver Prerequisites](#driver-prerequisites)
- [Interacting With the Session](#interacting-with-the-session)
  - [Retrieving Data](#retrieving-data)
  - [Storing Data](#storing-data)
  - [Flash Data](#flash-data)
  - [Deleting Data](#deleting-data)
  - [Regenerating the Session ID](#regenerating-the-session-id)
- [Session Blocking](#session-blocking)
- [Adding Custom Session Drivers](#adding-custom-session-drivers)
  - [Implementing the Driver](#implementing-the-driver)
  - [Registering the Driver](#registering-the-driver)

<a name="introduction"></a>
## Introduction

Since HTTP driven applications are stateless, sessions provide a way to store information about the user across multiple requests. That user information is typically placed in a persistent store / backend that can be accessed from subsequent requests.

PHP-Framework is built from the ground up as a stateless, performance-optimized micro-framework tailored for JSON APIs. Because of this lean, share-nothing architecture, HTTP sessions are completely disabled by default.

When explicitly enabled, sessions provide a structured way to store user data across requests. PHP-Framework ships with support for a variety of expressive backends, including [Memcached](https://memcached.org), [Redis](https://redis.io), and relational databases.

<a name="explicit-session-opt-in-flow"></a>
### Explicit Session Opt-In Flow

To utilize cookies and sessions, you must actively toggle them on within the application lifecycle:

1. **Composer Realignment**: Remove cookie and session exclusions from the `exclude-from-classmap` collection inside your `composer.json` file.
2. **Container Activation**: Open `App\Application.php` and uncomment the native session and cookie service container bindings.
3. **Middleware Isolation**: Manually declare the necessary session middlewares *only* on the specific controller route buckets that require state preservation.

> [!WARNING]  
> Automatic session blocks are not available. Though manual `.withErrors()` view chaining remains fully supported for manual redirect implementations.

#### Mandatory Session Middleware Stack

When targeting standard HTML views or routes that handle CSRF verification, register the following sequential middlewares on your router group:

    \MacropaySolutions\Kernel\Cookie\Middleware\EncryptCookies::class,
    \MacropaySolutions\Kernel\Cookie\Middleware\AddQueuedCookiesToResponse::class,
    \MacropaySolutions\Kernel\Session\Middleware\StartSession::class,
    \MacropaySolutions\Session\Middleware\VerifyCsrfToken::class,

<a name="configuration"></a>
### Configuration

Your application's session configuration file is stored at `config/session.php`. Be sure to review the options available to you in this file. By default, Framework is configured to use the `file` session driver, which will work well for many applications. If your application will be load balanced across multiple web servers, you should choose a centralized store that all servers can access, such as Redis or a database.

The session `driver` configuration option defines where session data will be stored for each request. Framework ships with several great drivers out of the box:

<div class="content-list" markdown="1">

- `file` - sessions are stored in `storage/framework/sessions`.
- `cookie` - sessions are stored in secure, encrypted cookies.
- `database` - sessions are stored in a relational database.
- `memcached` / `redis` - sessions are stored in one of these fast, cache based stores.
- `dynamodb` - sessions are stored in AWS DynamoDB.
- `array` - sessions are stored in a PHP array and will not be persisted.

</div>

> [!NOTE]  
> The array driver is primarily used during [testing](/testing) and prevents the data stored in the session from being persisted.

<a name="driver-prerequisites"></a>
### Driver Prerequisites

#### Database Storage

When utilizing the `database` session driver, you must establish a dedicated storage table. To learn more about database migrations, you may consult the complete [migration documentation](/migrations). You can execute the `session:table` Run command to generate this migration blueprint seamlessly:

```shell
php run session:table

php run migrate
```

The compiled migration executes the following structured table assignment directly through the database connection engine:

    use MacropaySolutions\Kernel\Database\Schema\Blueprint;

    \app('db')->connection()->getSchemaBuilder()->create('sessions', function (Blueprint $table) {
        $table->string('id')->primary();
        $table->foreignId('user_id')->nullable()->index();
        $table->string('ip_address', 45)->nullable();
        $table->text('user_agent')->nullable();
        $table->text('payload');
        $table->integer('last_activity')->index();
    });

<a name="redis"></a>
#### Redis

Before using Redis sessions with Framework, you will need to either install the PhpRedis PHP extension via PECL or install the `predis/predis` package (~1.0) via Composer. For more information on configuring Redis, consult Framework's [Redis documentation](/redis#configuration).

> [!NOTE]  
> In the `session` configuration file, the `connection` option may be used to specify which Redis connection is used by the session.

<a name="interacting-with-the-session"></a>
## Interacting With the Session

<a name="retrieving-data"></a>
### Retrieving Data

There are two primary ways of working with session data in Framework: the global `session` helper and via a `Request` instance. First, let's look at accessing the session via a `Request` instance, which can be type-hinted on a controller method. Remember, controller method dependencies are automatically injected via the Framework [service container](/container):

#### Via Request Injection

Type-hint the core `MacropaySolutions\Kernel\Http\Request` instance on your targeted controller method to access the underlying session payload:

    <?php

    namespace App\Http\Controllers;

    use MacropaySolutions\Kernel\Http\Request;
    use MacropaySolutions\Kernel\View\View;

    class ProfileController extends \MacropaySolutions\Framework\Routing\Controller
    {
        public function show(Request $request): View
        {
            $value = $request->session()->get('user_key', 'default_fallback');

            return view('user.profile', ['data' => $value]);
        }
    }

When you retrieve an item from the session, you may also pass a default value as the second argument to the `get` method. This default value will be returned if the specified key does not exist in the session. If you pass a closure as the default value to the `get` method and the requested key does not exist, the closure will be executed and its result returned:

    $value = $request->session()->get('key', 'default');

    $value = $request->session()->get('key', function () {
        return 'default';
    });

<a name="the-global-session-helper"></a>
#### The Global Session Helper

The `session()` helper responds dynamically based on input signatures. Passing a string pulls session data, while passing a key-value array writes it:

    namespace App\Http\Controllers;

    class HomeController extends \MacropaySolutions\Framework\Routing\Controller
    {
        public function index()
        {
            // Retrieve data
            $analytics = session('tracker_id');

            // Store data
            session(['session_active' => true]);
        }
    }

> [!NOTE]  
> There is little practical difference between using the session via an HTTP request instance versus using the global `session` helper. Both methods are [testable](/testing) via the `assertSessionHas` method which is available in all of your test cases.

<a name="retrieving-all-session-data"></a>
#### Retrieving All Session Data

If you would like to retrieve all the data in the session, you may use the `all` method:

    $data = $request->session()->all();

<a name="retrieving-a-portion-of-the-session-data"></a>
#### Retrieving a Portion of the Session Data

The `only` and `except` methods may be used to retrieve a subset of the session data:

    $data = $request->session()->only(['username', 'email']);

    $data = $request->session()->except(['username', 'email']);

<a name="determining-if-an-item-exists-in-the-session"></a>
#### Determining if an Item Exists in the Session

To determine if an item is present in the session, you may use the `has` method. The `has` method returns `true` if the item is present and is not `null`:

    if ($request->session()->has('users')) {
        // ...
    }

To determine if an item is present in the session, even if its value is `null`, you may use the `exists` method:

    if ($request->session()->exists('users')) {
        // ...
    }

To determine if an item is not present in the session, you may use the `missing` method. The `missing` method returns `true` if the item is not present:

    if ($request->session()->missing('users')) {
        // ...
    }

<a name="storing-data"></a>
### Storing Data

To store data in the session, you will typically use the request instance's `put` method or the global `session` helper:

    // Via a request instance...
    $request->session()->put('key', 'value');

    // Via the global "session" helper...
    session(['key' => 'value']);

<a name="pushing-to-array-session-values"></a>
#### Pushing to Array Session Values

The `push` method may be used to push a new value onto a session value that is an array. For example, if the `user.teams` key contains an array of team names, you may push a new value onto the array like so:

    $request->session()->push('user.teams', 'developers');

<a name="retrieving-deleting-an-item"></a>
#### Retrieving and Deleting an Item

The `pull` method will retrieve and delete an item from the session in a single statement:

    $value = $request->session()->pull('key', 'default');

<a name="incrementing-and-decrementing-session-values"></a>
#### Incrementing and Decrementing Session Values

If your session data contains an integer you wish to increment or decrement, you may use the `increment` and `decrement` methods:

    $request->session()->increment('count');

    $request->session()->increment('count', $incrementBy = 2);

    $request->session()->decrement('count');

    $request->session()->decrement('count', $decrementBy = 2);

<a name="flash-data"></a>
### Flash Data

Sometimes you may wish to store items in the session for the next request. You may do so using the `flash` method. Data stored in the session using this method will be available immediately and during the subsequent HTTP request. After the subsequent HTTP request, the flashed data will be deleted. Flash data is primarily useful for short-lived status messages:

    $request->session()->flash('status', 'Task was successful!');

If you need to persist your flash data for several requests, you may use the `reflash` method, which will keep all the flash data for an additional request. If you only need to keep specific flash data, you may use the `keep` method:

    $request->session()->reflash();

    $request->session()->keep(['username', 'email']);

To persist your flash data only for the current request, you may use the `now` method:

    $request->session()->now('status', 'Task was successful!');

<a name="deleting-data"></a>
### Deleting Data

The `forget` method will remove a piece of data from the session. If you would like to remove all data from the session, you may use the `flush` method:

    // Forget a single key...
    $request->session()->forget('name');

    // Forget multiple keys...
    $request->session()->forget(['name', 'status']);

    $request->session()->flush();

<a name="regenerating-the-session-id"></a>
### Regenerating the Session ID

Regenerating the session ID is often done in order to prevent malicious users from exploiting a [session fixation](https://owasp.org/www-community/attacks/Session_fixation) attack on your application.

Framework automatically regenerates the session ID during authentication; however, if you need to manually regenerate the session ID, you may use the `regenerate` method:

    $request->session()->regenerate();

If you need to regenerate the session ID and remove all data from the session in a single statement, you may use the `invalidate` method:

    $request->session()->invalidate();

<a name="session-blocking"></a>
## Session Blocking

> [!WARNING]  
> To utilize session blocking, your application must be using a cache driver that supports [atomic locks](/cache#atomic-locks). Currently, those cache drivers include the `memcached`, `dynamodb`, `redis`, `database`, `file`, and `array` drivers. In addition, you may not use the `cookie` session driver.

By default, PHP-Framework allows requests using the same session to execute concurrently. For example, if an API client executes two parallel HTTP requests, they will both process at the exact same time. For many stateless applications, this is not a problem; however, session data corruption can occur if concurrent requests hit different application endpoints that simultaneously read and write conflicting data to the session backend.

To mitigate this risk, PHP-Framework provides atomic serialization functionality that handles concurrent requests sharing a session ID. Blocking is not applied dynamically via fluid route chaining or custom route middleware. Instead, it is managed **globally via the configuration layer**.

### Enabling Global Session Serialization

When active, the core session manager intercepts the initialization phase inside the standard `StartSession` middleware and wraps the request lifetime within an atomic cache lock.

To configure session blocking, adjust the dedicated configuration keys inside your `config/session.php` file:

    return [

        // ... core session parameters

        /*
        |--------------------------------------------------------------------------
        | Global Session Blocking
        |--------------------------------------------------------------------------
        */

        'block' => true,

        'block_store' => 'redis',

        'block_lock_seconds' => 10,

        'block_wait_seconds' => 10,
    ];

#### Configuration Options

* **`block`**: A boolean flag (`true`/`false`) determining whether the framework should globally force concurrent user session requests into a serialized execution queue.
* **`block_store`**: Specifies the explicit cache driver connection responsible for handling the underlying atomic lock records.
* **`block_lock_seconds`**: Defines the maximum duration (in seconds) that an application request can hold a session lock before it is automatically released.
* **`block_wait_seconds`**: Defines the maximum number of seconds a secondary concurrent request will wait while attempting to obtain the session lock. An explicit `MacropaySolutions\Kernel\Contracts\Cache\LockTimeoutException` will be thrown if the request is unable to obtain the atomic lock within this threshold.

#### Lifecycle Execution

The entire blocking sequence is handled natively inside the mandatory session middleware stack you apply to your routes:

    \MacropaySolutions\Kernel\Session\Middleware\StartSession::class

When `block` is enabled in your configuration, this middleware automatically determines if it should secure an atomic lock before executing your target controller methods, cleanly decoupling serialization safety from your routing tree.

<a name="adding-custom-session-drivers"></a>
## Adding Custom Session Drivers

<a name="implementing-the-driver"></a>
### Implementing the Driver

If none of the existing session drivers fit your application's needs, Framework makes it possible to write your own session handler. Your custom session driver should implement PHP's built-in `SessionHandlerInterface`. This interface contains just a few simple methods. A stubbed MongoDB implementation looks like the following:

    <?php

    namespace App\Extensions;

    class MongoSessionHandler implements \SessionHandlerInterface
    {
        public function open($savePath, $sessionName): bool { return true; }
        public function close(): bool { return true; }
        public function read($sessionId): string { return ''; }
        public function write($sessionId, $data): bool { return true; }
        public function destroy($sessionId): bool { return true; }
        public function gc($lifetime): int|bool { return true; }
    }

> [!NOTE]  
> Framework does not ship with a directory to contain your extensions. You are free to place them anywhere you like. In this example, we have created an `Extensions` directory to house the `MongoSessionHandler`.

Since the purpose of these methods is not readily understandable, let's quickly cover what each of the methods do:

<div class="content-list" markdown="1">

- The `open` method would typically be used in file based session store systems. Since Framework ships with a `file` session driver, you will rarely need to put anything in this method. You can simply leave this method empty.
- The `close` method, like the `open` method, can also usually be disregarded. For most drivers, it is not needed.
- The `read` method should return the string version of the session data associated with the given `$sessionId`. There is no need to do any serialization or other encoding when retrieving or storing session data in your driver, as Framework will perform the serialization for you.
- The `write` method should write the given `$data` string associated with the `$sessionId` to some persistent storage system, such as MongoDB or another storage system of your choice.  Again, you should not perform any serialization - Framework will have already handled that for you.
- The `destroy` method should remove the data associated with the `$sessionId` from persistent storage.
- The `gc` method should destroy all session data that is older than the given `$lifetime`, which is a UNIX timestamp. For self-expiring systems like Memcached and Redis, this method may be left empty.

</div>

<a name="registering-the-driver"></a>
### Registering the Driver

You must interact directly with the underlying session manager resolved out of the service container instance. Extend your backend engine within the `boot` lifecycle phase of a custom service provider:

    <?php

    namespace App\Providers;

    use App\Extensions\MongoSessionHandler;
    use MacropaySolutions\Kernel\Support\ServiceProvider;

    class SessionServiceProvider extends ServiceProvider
    {
        public function register(): void
        {
            //
        }

        public function boot(): void
        {
            // Accessing the manager instance directly via the app container helper
            \app('session')->extend('mongo', function ($app) {
                return new MongoSessionHandler();
            });
        }
    }

Once registered, assign `mongo` to your application's active driver variable inside `config/session.php`.