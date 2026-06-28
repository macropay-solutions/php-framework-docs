---
title: Cache
description: Guide to configuring and using caching, cache tags, and atomic locks in PHP Framework.
context: cache
---

# Cache

- [Introduction](#introduction)
- [Configuration](#configuration)
  - [Driver Prerequisites](#driver-prerequisites)
- [Cache Usage](#cache-usage)
  - [Obtaining a Cache Instance](#obtaining-a-cache-instance)
  - [Retrieving Items From the Cache](#retrieving-items-from-the-cache)
  - [Storing Items in the Cache](#storing-items-in-the-cache)
  - [Removing Items From the Cache](#removing-items-from-the-cache)
  - [Extending Items TTL in the Cache](#extending-items-ttl-in-the-cache)
  - [The Cache Helper](#the-cache-helper)
- [Cache Tags](#cache-tags)
  - [Storing Tagged Cache Items](#storing-tagged-cache-items)
  - [Accessing Tagged Cache Items](#accessing-tagged-cache-items)
  - [Removing Tagged Cache Items](#removing-tagged-cache-items)
  - [The Cascading Expiration Hierarchy](#the-cascading-expiration-hierarchy)
- [Atomic Locks](#atomic-locks)
  - [Driver Prerequisites](#lock-driver-prerequisites)
  - [Managing Locks](#managing-locks)
  - [Managing Locks Across Processes](#managing-locks-across-processes)
- [Adding Custom Cache Drivers](#adding-custom-cache-drivers)
  - [Writing the Driver](#writing-the-driver)
  - [Registering the Driver](#registering-the-driver)
- [Events](#events)

<a name="introduction"></a>
## Introduction

Some of the data retrieval or processing tasks performed by your application could be CPU intensive or take several seconds to complete. When this is the case, it is common to cache the retrieved data for a time so it can be retrieved quickly on subsequent requests for the same data. The cached data is usually stored in a very fast data store such as [Memcached](https://memcached.org) or [Redis](https://redis.io).

Thankfully, Framework provides an expressive, unified API for various cache backends, allowing you to take advantage of their blazing fast data retrieval and speed up your web application.

<a name="configuration"></a>
## Configuration

Your application's cache configuration file is located at `config/cache.php`. In this file, you may specify which cache driver you would like to be used by default throughout your application. Framework supports popular caching backends like [Memcached](https://memcached.org), [Redis](https://redis.io), [DynamoDB](https://aws.amazon.com/dynamodb), and relational databases out of the box. In addition, a file based cache driver is available, while `array` and "null" cache drivers provide convenient cache backends for your automated tests.

The cache configuration file also contains various other options, which are documented within the file, so make sure to read over these options. By default, Framework is configured to use the `file` cache driver, which stores the serialized, cached objects on the server's filesystem. For larger applications, it is recommended that you use a more robust driver such as Memcached or Redis. You may even configure multiple cache configurations for the same driver.

<a name="driver-prerequisites"></a>
### Driver Prerequisites

<a name="prerequisites-database"></a>
#### Database

When using the `database` cache driver, you will need to set up a table to contain the cache items. You'll find an example schema declaration for the table below, using the database connection's schema builder:

    \app('db')->connection()->getSchemaBuilder()->create('cache', function (Blueprint $table) {
        $table->string('key')->unique();
        $table->text('value');
        $table->integer('expiration');
    });

> [!NOTE]  
> You may also use the `php run cache:table` Run command to generate a migration with the proper schema.

<a name="memcached"></a>
#### Memcached

Using the Memcached driver requires the [Memcached PECL package](https://pecl.php.net/package/memcached) to be installed. You may list all of your Memcached servers in the `config/cache.php` configuration file. This file already contains a `memcached.servers` entry to get you started:

    'memcached' => [
        'servers' => [
            [
                'host' => env('MEMCACHED_HOST', '127.0.0.1'),
                'port' => env('MEMCACHED_PORT', 11211),
                'weight' => 100,
            ],
        ],
    ],

If needed, you may set the `host` option to a UNIX socket path. If you do this, the `port` option should be set to `0`:

    'memcached' => [
        [
            'host' => '/var/run/memcached/memcached.sock',
            'port' => 0,
            'weight' => 100
        ],
    ],

<a name="redis"></a>
#### Redis

Before using a Redis cache with Framework, you will need to either install the PhpRedis PHP extension via PECL or install the `predis/predis` package (~1.0) via Composer.

For more information on configuring Redis, consult its [Framework documentation page](/redis#configuration).

<a name="dynamodb"></a>
#### DynamoDB

Before using the [DynamoDB](https://aws.amazon.com/dynamodb) cache driver, you must create a DynamoDB table to store all the cached data. Typically, this table should be named `cache`. However, you should name the table based on the value of the `stores.dynamodb.table` configuration value within your application's `cache` configuration file.

This table should also have a string partition key with a name that corresponds to the value of the `stores.dynamodb.attributes.key` configuration item within your application's `cache` configuration file. By default, the partition key should be named `key`.

<a name="cache-usage"></a>
## Cache Usage

<a name="obtaining-a-cache-instance"></a>
### Obtaining a Cache Instance

To obtain a cache store instance, you may resolve it directly via dependency injection or via the application container. The `MacropaySolutions\Kernel\Contracts\Cache\Repository` contract provides convenient access to the underlying implementations:

    <?php

    namespace App\Controllers;

    use MacropaySolutions\Kernel\Contracts\Cache\Repository as CacheRepository;

    class UserController
    {
        /**
         * Show a list of all users of the application.
         */
        public function index(CacheRepository $cache): array
        {
            $value = $cache->get('key');

            return [
                // ...
            ];
        }
    }

<a name="accessing-multiple-cache-stores"></a>
#### Accessing Multiple Cache Stores

Using the container, you may access various cache stores via the `store` method. The key passed to the `store` method should correspond to one of the stores listed in the `stores` configuration array in your `cache` configuration file:

    $value = \app('cache')->store('file')->get('foo');

    \app('cache')->store('redis')->put('bar', 'baz', 600); // 10 Minutes

<a name="retrieving-items-from-the-cache"></a>
### Retrieving Items From the Cache

The `get` method is used to retrieve items from the cache. If the item does not exist in the cache, `null` will be returned. If you wish, you may pass a second argument to the `get` method specifying the default value you wish to be returned if the item doesn't exist:

    $value = \app('cache')->get('key');

    $value = \app('cache')->get('key', 'default');

You may even pass a closure as the default value. The result of the closure will be returned if the specified item does not exist in the cache. Passing a closure allows you to defer the retrieval of default values from a database or other external service:

    $value = \app('cache')->get('key', function () {
        return \app('db')->table(/* ... */)->get();
    });

<a name="determining-item-existence"></a>
#### Determining Item Existence

The `has` method may be used to determine if an item exists in the cache. This method will also return `false` if the item exists but its value is `null`:

    if (\app('cache')->has('key')) {
        // ...
    }

<a name="incrementing-decrementing-values"></a>
#### Incrementing / Decrementing Values

The `increment` and `decrement` methods may be used to adjust the value of integer items in the cache. Both of these methods accept an optional second argument indicating the amount by which to increment or decrement the item's value:

    // Initialize the value if it does not exist...
    \app('cache')->add('key', 0, now()->addHours(4));

    // Increment or decrement the value...
    \app('cache')->increment('key');
    \app('cache')->increment('key', $amount);
    \app('cache')->decrement('key');
    \app('cache')->decrement('key', $amount);

<a name="retrieve-store"></a>
#### Retrieve and Store

Sometimes you may wish to retrieve an item from the cache, but also store a default value if the requested item doesn't exist. For example, you may wish to retrieve all users from the cache or, if they don't exist, retrieve them from the database and add them to the cache. You may do this using the `remember` method:

    $value = \app('cache')->remember('users', $seconds, function () {
        return \app('db')->table('users')->get();
    });

If the item does not exist in the cache, the closure passed to the `remember` method will be executed and its result will be placed in the cache.

You may use the `rememberForever` method to retrieve an item from the cache or store it forever if it does not exist:

    $value = \app('cache')->rememberForever('users', function () {
        return \app('db')->table('users')->get();
    });

<a name="retrieve-delete"></a>
#### Retrieve and Delete

If you need to retrieve an item from the cache and then delete the item, you may use the `pull` method. Like the `get` method, `null` will be returned if the item does not exist in the cache:

    $value = \app('cache')->pull('key');

<a name="storing-items-in-the-cache"></a>
### Storing Items in the Cache

You may use the `put` method to store items in the cache:

    \app('cache')->put('key', 'value', $seconds = 10);

If the storage time is not passed to the `put` method, the item will be stored indefinitely:

    \app('cache')->put('key', 'value');

Instead of passing the number of seconds as an integer, you may also pass a `DateTime` instance representing the desired expiration time of the cached item:

    \app('cache')->put('key', 'value', now()->addMinutes(10));

<a name="store-if-not-present"></a>
#### Store if Not Present

The `add` method will only add the item to the cache if it does not already exist in the cache store. The method will return `true` if the item is actually added to the cache. Otherwise, the method will return `false`. The `add` method is an atomic operation:

    \app('cache')->add('key', 'value', $seconds);

<a name="storing-items-forever"></a>
#### Storing Items Forever

The `forever` method may be used to store an item in the cache permanently. Since these items will not expire, they must be manually removed from the cache using the `forget` method:

    \app('cache')->forever('key', 'value');

> [!NOTE]  
> If you are using the Memcached driver, items that are stored "forever" may be removed when the cache reaches its size limit.

<a name="removing-items-from-the-cache"></a>
### Removing Items From the Cache

You may remove items from the cache using the `forget` method:

    \app('cache')->forget('key');

You may also remove items by providing a zero or negative number of expiration seconds:

    \app('cache')->put('key', 'value', 0);

    \app('cache')->put('key', 'value', -5);

    \app('cache')->touch('key', 0);

You may clear the entire cache using the `flush` method:

    \app('cache')->flush();

> [!WARNING]  
> Flushing the cache does not respect your configured cache "prefix" and will remove all entries from the cache. Consider this carefully when clearing a cache which is shared by other applications.

<a name="extending-items-ttl-in-the-cache"></a>
### Extending Items TTL in the Cache

You may adjust the time to live of a cache key without retrieving its underlying value when using the `memcached`, `dynamodb`, `redis`, and `database` drivers. In other stores, the value is retrieved and then overwritten with the new expiration time.

    \app('cache')->touch(string $key, \DateTimeInterface|\DateInterval|int|null $ttl = null)

    \app('cache')->touch('key', null); // Retain forever

    \app('cache')->touch('key', 50); // Expires in 50 seconds from now

> [!WARNING]  
> Passing a `0` TTL will explicitly remove the cache key from the store!

<a name="the-cache-helper"></a>
### The Cache Helper

In addition to resolving from the container, you may also use the global `cache` function to retrieve and store data via the cache. When the `cache` function is called with a single, string argument, it will return the value of the given key:

    $value = cache('key');

If you provide an array of key / value pairs and an expiration time to the function, it will store values in the cache for the specified duration:

    cache(['key' => 'value'], $seconds);

    cache(['key' => 'value'], now()->addMinutes(10));

When the `cache` function is called without any arguments, it returns an instance of the `MacropaySolutions\Kernel\Contracts\Cache\Factory` implementation, allowing you to call other caching methods:

    cache()->remember('users', $seconds, function () {
        return \app('db')->table('users')->get();
    });

<a name="cache-tags"></a>
## Cache Tags

> **Warning** > Cache tags are not supported when using the `file`, `dynamodb`, or `database` cache drivers. Furthermore, this architecture strictly requires an atomic cache store (such as `redis`, `memcached` or `apc`) that natively supports atomic `add` and `increment` primitives.

<a name="storing-tagged-cache-items"></a>
### Storing Tagged Cache Items

Cache tags allow you to tag related items in the cache and then flush all cached values that have been assigned a given tag. You may access a tagged cache by passing in an ordered array of tag names. For example, let's access a tagged cache and `put` a value into the cache:

    \app('cache')->tags(['people', 'artists'])->put('John', $john, $seconds);

    \app('cache')->tags(['people', 'authors'])->put('Anne', $anne, $seconds);

#### The Storage Security Ceiling

To prevent permanent memory leaks across multi-tier atomic networks, all tagged entries are bound to a strict storage ceiling defined by the framework container (`Container::TAGGED_CACHE_TTL_CAP_SECONDS`).

Consequently, calling the `forever` method on a tagged cache will not store the item indefinitely. Instead, it will cap the record's maximum lifespan to the internal boundary (defaults to 7200 seconds, or 2 hours), ensuring silent garbage collection if a tag generation falls inactive:

    // This item will honor the maximum security ceiling (e.g., 2 hours)
    \app('cache')->tags(['people', 'artists'])->forever('John', $john);

##### ⚠️ Critical Architectural Warning: Decouple JWT Blacklists from Tagged Cache
*(This applies to JWT Blacklists and **any other logic that uses a tagged cache with a TTL longer than 2 hours!**)*

Forcing a flat token blacklist into a relational tagged cache ecosystem introduces a critical security vulnerability and an expensive performance drain.

**The Security Loophole (Premature Blacklist Eviction)**
* **The Conflict:** To optimize memory, highly efficient tagging engines enforce short TTL caps (e.g., 2 hours) so tracking indices can expire and reset. However, a secure JWT blacklist requires a long, unclipped lifespan (e.g., 14 days) to keep logged-out tokens invalidated.
* **The Threat:** Forcing JWT through a tagging driver clips its 14-day lifetime down to the short business cap. Once that cap hits, the cache evicts the blacklist entry. Because the token's cryptographic signature is still valid, **logged-out or stolen tokens are resurrected, exposing the app to Token Replay Attacks.**

> [!WARNING]  
> Raising the global cache cap to 14 days to fix this ruins your business caching, causing massive tracking pointer bloat and stopping sequence recycling.

**The Performance Drain (Double Initialization Storm)**
By default, standard package storage drivers often probe for tagging support using dynamic `method_exists()` checks, forcing redundant internal key sorting and metadata allocation on every single API gateway hit.

**The Resolution: Pure Flat Keyspace Decoupling**
Do not use tags for the authentication layer. Implement the package's storage contract directly, utilizing the framework's raw cache repository.

This forces blacklisted token IDs to write directly to the raw cache pool as flat, un-tagged key-value pairs. The tokens securely retain their unclipped 14-day lifespan, the redundant allocation overhead is completely eliminated, and your business tags keep their tight lifecycles.

<a name="accessing-tagged-cache-items"></a>
### Accessing Tagged Cache Items

Items stored via tags may not be accessed without also providing the exact tags that were used to store the value. To retrieve a tagged cache item, pass the same ordered list of tags to the `tags` method and then call the `get` method with the key you wish to retrieve:

    $john = \app('cache')->tags(['people', 'artists'])->get('John');

    $anne = \app('cache')->tags(['people', 'authors'])->get('Anne');

Behind the scenes, the framework calculates a real-time, cryptographic composite hash prefix using the active isolated version string of every tag in your set. This allows the system to resolve the precise data address on reads instantly, keeping the hot-path completely clear of structural database queries or lookup interceptors.

<a name="removing-tagged-cache-items"></a>
### Removing Tagged Cache Items

You may flush all items that are assigned a tag or list of tags.

The framework employs an **Atomic Lazy Eviction** model. Executing a `flush` does not loop through keys or trigger blocking resource deletions on the cluster. Instead, it fires a single, microsecond-level atomic increment on the tag's generation identifier. The old keyspace hash is instantly rendered obsolete and orphaned:

    \app('cache')->tags(['people', 'authors'])->flush();

This statement instantly alters the computed namespace signature for both `people` and `authors`. The next time an application task attempts to read `Anne` or `John`, the math yields a different hash lookup address, returning a clean cache miss natively with zero performance penalty.

In contrast, this statement would advance only the version tracker assigned to `authors`, meaning `Anne` would hit an immediate cache miss, but `John` would remain safely readable:

    \app('cache')->tags('authors')->flush();

<a name="the-cascading-expiration-hierarchy"></a>
### The Cascading Expiration Hierarchy

Because flushes are managed lazily via structural version shifts, old payloads and tracking references are left to expire naturally. Memory cleanup is fully decentralized and governed by an automated, staggered "Russian Doll" time-to-live sequence mapping:

* **Tier 4 (Data Payloads):** Dropped first natively by the storage cluster at the user's defined lifespan or the maximum `ttlCap`.
* **Tiers 2 & 3 (Sequence Counters & Pointers):** Outlive the data layer by exactly 5 seconds (`ttlCap + 5s`), acting as temporary buffers to escort high-concurrency write operations safely to their graves.
* **Tier 1 (Master Generation Version):** Outlives all underlying components by a factor of two (`ttlCap * 2`). This structural anchor prevents premature evictions from accidentally recycling version sequences and resurrecting stale historical generations.

If an entire tag goes completely dark and receives zero read or write traffic throughout this silent decay window, both tracking pillars drop entirely out of RAM. The next cold write operation hitting the cluster triggers atomic fallback gates, gracefully rolling both sequence and version indices back to their baseline state (`1`) with absolute thread isolation.

<a name="atomic-locks"></a>
## Atomic Locks

> [!WARNING]  
> To utilize this feature, your application must be using the `memcached`, `redis`, `dynamodb`, `database`, `file`, or `array` cache driver as your application's default cache driver. In addition, all servers must be communicating with the same central cache server.

<a name="lock-driver-prerequisites"></a>
### Driver Prerequisites

<a name="atomic-locks-prerequisites-database"></a>
#### Database

When using the `database` cache driver, you will need to setup a table to contain your application's cache locks. You'll find an example schema declaration for the table below:

    \app('db')->connection()->getSchemaBuilder()->create('cache_locks', function (Blueprint $table) {
        $table->string('key')->primary();
        $table->string('owner');
        $table->integer('expiration');
    });

> [!NOTE]  
> If you used the `cache:table` Run command to create the database driver's cache table, the migration created by that command already includes a definition for the `cache_locks` table.

<a name="managing-locks"></a>
### Managing Locks

Atomic locks allow for for the manipulation of distributed locks without worrying about race conditions. You may create and manage locks using the `lock` method:

    $lock = \app('cache')->lock('foo', 10);

    if ($lock->get()) {
        // Lock acquired for 10 seconds...

        $lock->release();
    }

The `get` method also accepts a closure. After the closure is executed, Framework will automatically release the lock:

    \app('cache')->lock('foo', 10)->get(function () {
        // Lock acquired for 10 seconds and automatically released...
    });

If the lock is not available at the moment you request it, you may instruct Framework to wait for a specified number of seconds. If the lock can not be acquired within the specified time limit, an `MacropaySolutions\Kernel\Contracts\Cache\LockTimeoutException` will be thrown:

    use MacropaySolutions\Kernel\Contracts\Cache\LockTimeoutException;

    $lock = \app('cache')->lock('foo', 10);

    try {
        $lock->block(5);

        // Lock acquired after waiting a maximum of 5 seconds...
    } catch (LockTimeoutException $e) {
        // Unable to acquire lock...
    } finally {
        $lock?->release();
    }

The example above may be simplified by passing a closure to the `block` method. When a closure is passed to this method, Framework will attempt to acquire the lock for the specified number of seconds and will automatically release the lock once the closure has been executed:

    \app('cache')->lock('foo', 10)->block(5, function () {
        // Lock acquired after waiting a maximum of 5 seconds...
    });

Lock can be refreshed or prolonged:

    $lock = \app('cache')->lock('foo', 100);

    if ($lock->get()) {
        // Lock acquired for 100 seconds...
        $start = \microtime(true);

        // loop start
          // custom logic

          if (\round((\microtime(true) - $start) * 1000, 2) > 90) {
             $lock->refresh(100);
          }
        // end loop

        $lock->release();
    }

<a name="managing-locks-across-processes"></a>
### Managing Locks Across Processes

Sometimes, you may wish to acquire a lock in one process and release it in another process. For example, you may acquire a lock during a web request and wish to release the lock at the end of a queued job that is triggered by that request. In this scenario, you should pass the lock's scoped "owner token" to the queued job so that the job can re-instantiate the lock using the given token.

In the example below, we will dispatch a queued job if a lock is successfully acquired. In addition, we will pass the lock's owner token to the queued job via the lock's `owner` method:

    $podcast = Podcast::getQuery()->find($id);

    $lock = \app('cache')->lock('processing', 120);

    if ($lock->get()) {
        ProcessPodcast::dispatch($podcast, $lock->owner());
    }

Within our application's `ProcessPodcast` job, we can restore and release the lock using the owner token:

    \app('cache')->restoreLock('processing', $this->owner)->release();

If you would like to release a lock without respecting its current owner, you may use the `forceRelease` method:

    \app('cache')->lock('processing')->forceRelease();

<a name="adding-custom-cache-drivers"></a>
## Adding Custom Cache Drivers

<a name="writing-the-driver"></a>
### Writing the Driver

To create our custom cache driver, we first need to implement the `MacropaySolutions\Kernel\Contracts\Cache\Store` [contract](/contracts). So, a MongoDB cache implementation might look something like this:

    <?php

    namespace App\Extensions;

    use MacropaySolutions\Kernel\Contracts\Cache\Store;

    class MongoStore implements Store
    {
        public function get($key) {}
        public function many(array $keys) {}
        public function put($key, $value, $seconds) {}
        public function putMany(array $values, $seconds) {}
        public function increment($key, $value = 1) {}
        public function decrement($key, $value = 1) {}
        public function forever($key, $value) {}
        public function forget($key) {}
        public function flush() {}
        public function getPrefix() {}
    }

We just need to implement each of these methods using a MongoDB connection. For an example of how to implement each of these methods, take a look at the `MacropaySolutions\Kernel\Cache\MemcachedStore` in the [PHP-Kernel source code](https://github.com/macropay-solutions/php-kernel). Once our implementation is complete, we can finish our custom driver registration by extending the cache manager:

    \app('cache')->extend('mongo', function ($app) {
        return \app('cache')->repository(new MongoStore);
    });

> [!NOTE]  
> If you're wondering where to put your custom cache driver code, you could create an `Extensions` namespace within your `app` directory. However, keep in mind that Framework does not have a rigid application structure and you are free to organize your application according to your preferences.

<a name="registering-the-driver"></a>
### Registering the Driver

To register the custom cache driver with Framework, we will use the `extend` method on the cache manager. Since other service providers may attempt to read cached values within their `boot` method, we will register our custom driver within a `booting` callback. By using the `booting` callback, we can ensure that the custom driver is registered just before the `boot` method is called on our application's service providers but after the `register` method is called on all the service providers. We will register our `booting` callback within the `register` method of our application's `App\Providers\AppServiceProvider` class:

    <?php

    namespace App\Providers;

    use App\Extensions\MongoStore;
    use MacropaySolutions\Kernel\Support\ServiceProvider;

    class AppServiceProvider extends ServiceProvider
    {
        /**
         * Register any application services.
         */
        public function register(): void
        {
            $this->app->booting(function () {
                 $this->app->make('cache')->extend('mongo', function ($app) {
                     return $this->app->make('cache')->repository(new MongoStore);
                 });
             });
        }

        /**
         * Bootstrap any application services.
         */
        public function boot(): void
        {
            // ...
        }
    }

The first argument passed to the `extend` method is the name of the driver. This will correspond to your `driver` option in the `config/cache.php` configuration file. The second argument is a closure that should return an `MacropaySolutions\Kernel\Cache\Repository` instance. The closure will be passed an `$app` instance, which is an instance of the [service container](/container).

Once your extension is registered, update your `config/cache.php` configuration file's `driver` option to the name of your extension.

<a name="events"></a>
## Events

To execute code on every cache operation, you may listen for the [events](/events) fired by the cache. Typically, you should place these event listeners within your application's `App\Providers\EventServiceProvider` class:

    use App\Listeners\LogCacheHit;
    use App\Listeners\LogCacheMissed;
    use App\Listeners\LogKeyForgotten;
    use App\Listeners\LogKeyWritten;
    use MacropaySolutions\Kernel\Cache\Events\CacheHit;
    use MacropaySolutions\Kernel\Cache\Events\CacheMissed;
    use MacropaySolutions\Kernel\Cache\Events\KeyForgotten;
    use MacropaySolutions\Kernel\Cache\Events\KeyWritten;
    
    /**
     * The event listener mappings for the application.
     *
     * @var array
     */
    protected $listen = [
        CacheHit::class => [
            LogCacheHit::class,
        ],

        CacheMissed::class => [
            LogCacheMissed::class,
        ],

        KeyForgotten::class => [
            LogKeyForgotten::class,
        ],

        KeyWritten::class => [
            LogKeyWritten::class,
        ],
    ];
