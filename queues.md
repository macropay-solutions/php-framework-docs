---
title: Queues
description: Guide to configuring, dispatching, and managing queued jobs and storable array callables in PHP Framework.
context: queues
---

# Queues

- [Introduction](#introduction)
  - [Connections vs. Queues](#connections-vs-queues)
  - [Driver Notes and Prerequisites](#driver-prerequisites)
- [Queueing Storable Array Callables (Recommended)](#queueing-storable-array-callables-recommended)
  - [Basic Dispatching](#basic-dispatching)
  - [Chains & Batches](#chains-and-batches-arrays)
  - [Job Chaining Exceptions](#job-chaining-exceptions)
  - [Job Middleware and Rate Limiting](#job-middleware-and-rate-limiting)
  - [Strict Security Mode](#strict-security-mode)
  - [Encrypted Array Callables](#encrypted-array-callables)
- [Creating Jobs](#creating-jobs)
  - [Generating Job Classes](#generating-job-classes)
  - [Class Structure](#class-structure)
  - [Unique Jobs](#unique-jobs)
  - [Encrypted Jobs](#encrypted-jobs)
- [Job Middleware](#job-middleware)
  - [Rate Limiting](#rate-limiting)
  - [Preventing Job Overlaps](#preventing-job-overlaps)
  - [Throttling Exceptions](#throttling-exceptions)
- [Dispatching Jobs](#dispatching-jobs)
  - [Delayed Dispatching](#delayed-dispatching)
  - [Synchronous Dispatching](#synchronous-dispatching)
  - [Jobs & Database Transactions](#jobs-and-database-transactions)
  - [Job Chaining](#job-chaining)
  - [Customizing The Queue and Connection](#customizing-the-queue-and-connection)
  - [Specifying Max Job Attempts / Timeout Values](#max-job-attempts-and-timeout)
  - [Error Handling](#error-handling)
- [Job Batching](#job-batching)
  - [Defining Batchable Jobs](#defining-batchable-jobs)
  - [Dispatching Batches](#dispatching-batches)
  - [Chains and Batches](#chains-and-batches)
  - [Adding Jobs to Batches](#adding-jobs-to-batches)
  - [Inspecting Batches](#inspecting-batches)
  - [Cancelling Batches](#cancelling-batches)
  - [Batch Failures](#batch-failures)
  - [Pruning Batches](#pruning-batches)
  - [Storing Batches in DynamoDB](#storing-batches-in-dynamodb)
- [Queueing Closures](#queueing-closures)
- [Running the Queue Worker](#running-the-queue-worker)
  - [The `queue:work` Command](#the-queue-work-command)
  - [Queue Priorities](#queue-priorities)
  - [Queue Workers and Deployment](#queue-workers-and-deployment)
  - [Job Expirations and Timeouts](#job-expirations-and-timeouts)
- [Supervisor Configuration](#supervisor-configuration)
- [Dealing With Failed Jobs](#dealing-with-failed-jobs)
  - [Cleaning Up After Failed Jobs](#cleaning-up-after-failed-jobs)
  - [Retrying Failed Jobs](#retrying-failed-jobs)
  - [Ignoring Missing Models](#ignoring-missing-models)
  - [Pruning Failed Jobs](#pruning-failed-jobs)
  - [Storing Failed Jobs in DynamoDB](#storing-failed-jobs-in-dynamodb)
  - [Disabling Failed Job Storage](#disabling-failed-job-storage)
  - [Failed Job Events](#failed-job-events)
- [Clearing Jobs From Queues](#clearing-jobs-from-queues)
- [Monitoring Your Queues](#monitoring-your-queues)
- [Testing](#testing)
  - [Faking a Subset of Jobs](#faking-a-subset-of-jobs)
  - [Testing Job Chains](#testing-job-chains)
  - [Testing Job Batches](#testing-job-batches)
- [Job Events](#job-events)

<a name="introduction"></a>
## Introduction

While building your web application, you may have some tasks, such as parsing and storing an uploaded CSV file, that take too long to perform during a typical web request. Thankfully, Framework allows you to easily create queued jobs that may be processed in the background. By moving time intensive tasks to a queue, your application can respond to web requests with blazing speed and provide a better user experience to your customers.

Framework queues provide a unified queueing API across a variety of different queue backends, such as [Amazon SQS](https://aws.amazon.com/sqs/), [Redis](https://redis.io), or even a relational database.

Framework's queue configuration options are stored in your application's `config/queue.php` configuration file. In this file, you will find connection configurations for each of the queue drivers that are included with the framework, including the database, [Amazon SQS](https://aws.amazon.com/sqs/), [Redis](https://redis.io), and [Beanstalkd](https://beanstalkd.github.io/) drivers, as well as a synchronous driver that will execute jobs immediately (for use during local development). A `null` queue driver is also included which discards queued jobs.

<a name="connections-vs-queues"></a>
### Connections vs. Queues

Before getting started with Framework queues, it is important to understand the distinction between "connections" and "queues". In your `config/queue.php` configuration file, there is a `connections` configuration array. This option defines the connections to backend queue services such as Amazon SQS, Beanstalk, or Redis. However, any given queue connection may have multiple "queues" which may be thought of as different stacks or piles of queued jobs.

Note that each connection configuration example in the `queue` configuration file contains a `queue` attribute. This is the default queue that jobs will be dispatched to when they are sent to a given connection. In other words, if you dispatch a job without explicitly defining which queue it should be dispatched to, the job will be placed on the queue that is defined in the `queue` attribute of the connection configuration:

    use App\Jobs\ProcessPodcast;

    // This array callable is sent to the default connection's default queue...
    \dispatch([ProcessPodcast::class, 'handle']);

    // This array callable is sent to the default connection's "emails" queue...
    \dispatch([ProcessPodcast::class, 'handle'])->onQueue('emails');

Some applications may not need to ever push jobs onto multiple queues, instead preferring to have one simple queue. However, pushing jobs to multiple queues can be especially useful for applications that wish to prioritize or segment how jobs are processed, since the Framework queue worker allows you to specify which queues it should process by priority. For example, if you push jobs to a `high` queue, you may run a worker that gives them higher processing priority:

    php run queue:work --queue=high,default

<a name="driver-prerequisites"></a>
### Driver Notes and Prerequisites

<a name="database"></a>
#### Database

In order to use the `database` queue driver, you will need a database table to hold the jobs. To generate a migration that creates this table, run the `queue:table` Run command. Once the migration has been created, you may migrate your database using the `migrate` command:

    php run queue:table

    php run migrate

Finally, don't forget to instruct your application to use the `database` driver by updating the `QUEUE_CONNECTION` variable in your application's `.env` file:

    QUEUE_CONNECTION=database

<a name="redis"></a>
#### Redis

In order to use the `redis` queue driver, you should configure a Redis database connection in your `config/database.php` configuration file.

> [!WARNING]  
> The `serializer` and `compression` Redis options are not supported by the `redis` queue driver.

**Redis Cluster**

If your Redis queue connection uses a Redis Cluster, your queue names must contain a [key hash tag](https://redis.io/docs/reference/cluster-spec/#hash-tags). This is required in order to ensure all the Redis keys for a given queue are placed into the same hash slot:

    'redis' => [
        'driver' => 'redis',
        'connection' => 'default',
        'queue' => '{default}',
        'retry_after' => 90,
    ],

**Blocking**

When using the Redis queue, you may use the `block_for` configuration option to specify how long the driver should wait for a job to become available before iterating through the worker loop and re-polling the Redis database.

Adjusting this value based on your queue load can be more efficient than continually polling the Redis database for new jobs. For instance, you may set the value to `5` to indicate that the driver should block for five seconds while waiting for a job to become available:

    'redis' => [
        'driver' => 'redis',
        'connection' => 'default',
        'queue' => 'default',
        'retry_after' => 90,
        'block_for' => 5,
    ],

> [!WARNING]  
> Setting `block_for` to `0` will cause queue workers to block indefinitely until a job is available. This will also prevent signals such as `SIGTERM` from being handled until the next job has been processed.

<a name="other-driver-prerequisites"></a>
#### Other Driver Prerequisites

The following dependencies are needed for the listed queue drivers. These dependencies may be installed via the Composer package manager:

<div class="content-list" markdown="1">

- Amazon SQS: `aws/aws-sdk-php ~3.0`
- Beanstalkd: `pda/pheanstalk ~4.0`
- Redis: `predis/predis ~1.0` or phpredis PHP extension

</div>

<a name="queueing-storable-array-callables-recommended"></a>
## Queueing Storable Array Callables (Recommended)

Framework supports **Storable Array Callables** as a secure, high-performance alternative to traditional object-based jobs by eliminating serialization overhead. Because of the risk of PHP Object Injection (POI) vulnerabilities, this is the default and only allowed queueing mechanism unless strict security mode is explicitly disabled.

#### Basic Dispatching

No need to create a Job class for simple logic. The container will autowire dependencies into the target method.

    use App\Services\EmailService;

    // Primitive arguments passed explicitly. Dependencies autowired.
    \dispatch([EmailService::class, 'sendWelcomeEmail', ['userId' => $user->id]]);

#### Chains and Batches Arrays

Storable array callables seamlessly integrate with Framework's batching and chaining systems.

    use App\Services\ImageProcessor;

    \app('bus')->batch([
        [ImageProcessor::class, 'optimize', ['path' => 'photo1.jpg']],
        [ImageProcessor::class, 'optimize', ['path' => 'photo2.jpg']],
    ])->dispatch();

> [!WARNING]
> **Important Limitations:** To guarantee 100% protection against PHP Object Injection, `dispatch` strictly forbids objects in payloads or chained arrays. Obvious models are not automatically serialized and re-fetched natively; developers are responsible for passing primitive IDs and querying fresh database state inside their array callables. Alternatively, if capturing the exact current state of the model is important, you can pass its array shape (e.g., `$model->toArray()`).

#### Job Chaining Exceptions

When chaining jobs onto an Array Callable, you cannot chain a standard instantiated Job object. You must chain other Array Callables.

**❌ Incorrect (Throws Exception):**

    $job = \MacropaySolutions\Kernel\Queue\CallQueuedCallable::create([ProcessUserRegistration::class, 'handle', ['userId' => 5]]);

    // Fails because `SendWelcomeEmail` is an instantiated object
    $job->chain([
        new SendWelcomeEmail(5) 
    ]);
    
    \dispatch($job);

**✅ Correct (Using Array Callables):**

    $job = \MacropaySolutions\Kernel\Queue\CallQueuedCallable::create([ProcessUserRegistration::class, 'handle', ['userId' => 5]]);

    // Succeeds because the chain uses primitive arrays and strings
    $job->chain([
        [SendWelcomeEmail::class, 'handle', ['userId' => 5]]
    ]);
    
    \dispatch($job);

Failure Callbacks (catch)
Similarly, when defining failure callbacks on the dispatch, you must use the Array Callable syntax instead of standard Closures or invokable objects.

**❌ Incorrect (Throws Exception):**

    // Fails because a Closure is an object under the hood
    \dispatch([ReportGenerator::class, 'run', ['reportId' => 10]])
        ->catch(function (\Throwable $e) {
            // ...
        });

**✅ Correct (Using Array Callables):**

    // Succeeds because the catch callback is a storable array
    \dispatch([ReportGenerator::class, 'run', ['reportId' => 10]])
        ->catch([ReportGenerator::class, 'failed', ['reportId' => 10]]);

#### Job Middleware and Rate Limiting

Because Array Callables are not traditional Job classes, you cannot define a `middleware()` method on them. If you need to apply rate limiting or prevent job overlaps, you should use Framework's caching and throttling instances directly inside your targeted method.

Because the Storable Array Callable engine seamlessly injects the underlying queue worker, you can type-hint the `Job` interface to manually release or fail the job if it is rate-limited!

**✅ Correct (Rate Limiting via Method Injection):**

    use MacropaySolutions\Kernel\Contracts\Queue\Job;

    class EmailService
    {
        // The native Job instance is injected automatically!
        public function sendWelcomeEmail(int $userId, Job $job): void
        {
            \app('redis')->throttle('welcome-emails')
                ->allow(10)->every(60)
                ->then(function () use ($userId) {
                    // Lock obtained, process the email...
                    User::query()->findOrFail($userId)->sendWelcome();
                }, function () use ($job) {
                    // Rate limit exceeded, release the job back to the queue for 10 seconds
                    $job->release(10);
                });
        }
    }

<a name="strict-security-mode"></a>
#### Strict Security Mode

By default, Framework aggressively forbids serialized objects and closures in the queue to completely eliminate the risk of POI vulnerabilities. In the core container, `FORBID_SERIALIZED_OBJECTS_IN_QUEUE` is set to `true`.

If you are migrating a legacy application and need to dispatch traditional object jobs or closures, you must explicitly opt-out of this security by overriding this constant in your `App\Application` class:

    namespace App;

    class Application extends \MacropaySolutions\Framework\Application
    {
        public const FORBID_SERIALIZED_OBJECTS_IN_QUEUE = false;
    }

<a name="encrypted-array-callables"></a>
#### Encrypted Array Callables

Storable Array Callables fully support Framework's built-in payload encryption. If the target class referenced in your array callable implements the `MacropaySolutions\Kernel\Contracts\Queue\ShouldBeEncrypted` interface, the framework will intelligently detect it and automatically encrypt the queued job's metadata and arguments.

    namespace App\CallablesAsArray;

    use MacropaySolutions\Kernel\Contracts\Queue\ShouldBeEncrypted;

    class ProcessPayroll implements ShouldBeEncrypted
    {
        public function handle(int $employeeId): void
        {
            // ...
        }
    }

    // The payload for this array callable will be automatically encrypted...
    \dispatch([ProcessPayroll::class, 'handle', ['employeeId' => 5]]);

<a name="creating-jobs"></a>
## Creating Jobs

> [!WARNING]  
> Creating and dispatching traditional job objects is blocked by default. You must set `FORBID_SERIALIZED_OBJECTS_IN_QUEUE = false` in your `Application` class to use this feature.

<a name="generating-job-classes"></a>
### Generating Job Classes

By default, all the queueable jobs for your application are stored in the `app/Jobs` directory. If the `app/Jobs` directory doesn't exist, it will be created when you run the `make:job` Run command:

    php run make:job ProcessPodcast

The generated class will implement the `MacropaySolutions\Kernel\Contracts\Queue\ShouldQueue` interface, indicating to Framework that the job should be pushed onto the queue to run asynchronously.

<a name="class-structure"></a>
### Class Structure

Job classes are very simple, normally containing only a `handle` method that is invoked when the job is processed by the queue. To get started, let's take a look at an example job class. In this example, we'll pretend we manage a podcast publishing service and need to process the uploaded podcast files before they are published:

    <?php

    namespace App\Jobs;

    use App\Models\Podcast;
    use App\Services\AudioProcessor;
    use MacropaySolutions\Kernel\Bus\Queueable;
    use MacropaySolutions\Kernel\Contracts\Queue\ShouldQueue;
    use MacropaySolutions\Kernel\Queue\InteractsWithQueue;
    use MacropaySolutions\Kernel\Queue\SerializesModels;

    class ProcessPodcast implements ShouldQueue
    {
        use InteractsWithQueue, Queueable, SerializesModels;

        /**
         * Create a new job instance.
         */
        public function __construct(
            public Podcast $podcast,
        ) {}

        /**
         * Execute the job.
         */
        public function handle(AudioProcessor $processor): void
        {
            // Process uploaded podcast...
        }
    }

In this example, note that we were able to pass an [Obvious model](/obvious) directly into the queued job's constructor. Because of the `SerializesModels` trait that the job is using, Obvious models and their loaded relationships will be gracefully serialized and unserialized when the job is processing.

If your queued job accepts an Obvious model in its constructor, only the identifier for the model will be serialized onto the queue. When the job is actually handled, the queue system will automatically re-retrieve the full model instance and its loaded relationships from the database. This approach to model serialization allows for much smaller job payloads to be sent to your queue driver.

<a name="handle-method-dependency-injection"></a>
#### `handle` Method Dependency Injection

The `handle` method is invoked when the job is processed by the queue. Note that we are able to type-hint dependencies on the `handle` method of the job. The Framework [service container](/container) automatically injects these dependencies.

If you would like to take total control over how the container injects dependencies into the `handle` method, you may use the container's `bindMethod` method. The `bindMethod` method accepts a callback which receives the job and the container. Within the callback, you are free to invoke the `handle` method however you wish. Typically, you should call this method from the `boot` method of your `App\Providers\AppServiceProvider` [service provider](/providers):

    use App\Jobs\ProcessPodcast;
    use App\Services\AudioProcessor;
    use MacropaySolutions\Kernel\Contracts\Foundation\Application;

    $this->app->bindMethod([ProcessPodcast::class, 'handle'], function (ProcessPodcast $job, Application $app) {
        return $job->handle($app->make(AudioProcessor::class));
    });

> [!WARNING]  
> Binary data, such as raw image contents, should be passed through the `base64_encode` function before being passed to a queued job. Otherwise, the job may not properly serialize to JSON when being placed on the queue.

<a name="handling-relationships"></a>
#### Queued Relationships

Because all loaded Obvious model relationships also get serialized when a job is queued, the serialized job string can sometimes become quite large. Furthermore, when a job is deserialized and model relationships are re-retrieved from the database, they will be retrieved in their entirety. Any previous relationship constraints that were applied before the model was serialized during the job queueing process will not be applied when the job is deserialized. Therefore, if you wish to work with a subset of a given relationship, you should re-constrain that relationship within your queued job.

Or, to prevent relations from being serialized, you can call the `withoutRelations` method on the model when setting a property value. This method will return an instance of the model without its loaded relationships:

    /**
     * Create a new job instance.
     */
    public function __construct(Podcast $podcast)
    {
        $this->podcast = $podcast->withoutRelations();
    }

<a name="unique-jobs"></a>
### Unique Jobs

> [!WARNING]  
> Unique jobs require a cache driver that supports [locks](/cache#atomic-locks). Currently, the `memcached`, `redis`, `dynamodb`, `database`, `file`, and `array` cache drivers support atomic locks. In addition, unique job constraints do not apply to jobs within batches.

Sometimes, you may want to ensure that only one instance of a specific job is on the queue at any point in time. You may do so by implementing the `ShouldBeUnique` interface on your job class. This interface does not require you to define any additional methods on your class:

    <?php

    use MacropaySolutions\Kernel\Contracts\Queue\ShouldQueue;
    use MacropaySolutions\Kernel\Contracts\Queue\ShouldBeUnique;

    class UpdateSearchIndex implements ShouldQueue, ShouldBeUnique
    {
        ...
    }

In the example above, the `UpdateSearchIndex` job is unique. So, the job will not be dispatched if another instance of the job is already on the queue and has not finished processing.

In certain cases, you may want to define a specific "key" that makes the job unique or you may want to specify a timeout beyond which the job no longer stays unique. To accomplish this, you may define `uniqueId` and `uniqueFor` properties or methods on your job class:

    <?php

    use App\Models\Product;
    use MacropaySolutions\Kernel\Contracts\Queue\ShouldQueue;
    use MacropaySolutions\Kernel\Contracts\Queue\ShouldBeUnique;

    class UpdateSearchIndex implements ShouldQueue, ShouldBeUnique
    {
        /**
         * The product instance.
         *
         * @var \App\Product
         */
        public $product;

        /**
         * The number of seconds after which the job's unique lock will be released.
         *
         * @var int
         */
        public $uniqueFor = 3600;

        /**
         * Get the unique ID for the job.
         */
        public function uniqueId(): string
        {
            return $this->product->id;
        }
    }

In the example above, the `UpdateSearchIndex` job is unique by a product ID. So, any new dispatches of the job with the same product ID will be ignored until the existing job has completed processing. In addition, if the existing job is not processed within one hour, the unique lock will be released and another job with the same unique key can be dispatched to the queue.

> [!WARNING]  
> If your application dispatches jobs from multiple web servers or containers, you should ensure that all of your servers are communicating with the same central cache server so that Framework can accurately determine if a job is unique.

<a name="keeping-jobs-unique-until-processing-begins"></a>
#### Keeping Jobs Unique Until Processing Begins

By default, unique jobs are "unlocked" after a job completes processing or fails all of its retry attempts. However, there may be situations where you would like your job to unlock immediately before it is processed. To accomplish this, your job should implement the `ShouldBeUniqueUntilProcessing` contract instead of the `ShouldBeUnique` contract:

    <?php

    use App\Models\Product;
    use MacropaySolutions\Kernel\Contracts\Queue\ShouldQueue;
    use MacropaySolutions\Kernel\Contracts\Queue\ShouldBeUniqueUntilProcessing;

    class UpdateSearchIndex implements ShouldQueue, ShouldBeUniqueUntilProcessing
    {
        // ...
    }

<a name="unique-job-locks"></a>
#### Unique Job Locks

Behind the scenes, when a `ShouldBeUnique` job is dispatched, Framework attempts to acquire a [lock](/cache#atomic-locks) with the `uniqueId` key. If the lock is not acquired, the job is not dispatched. This lock is released when the job completes processing or fails all of its retry attempts. By default, Framework will use the default cache driver to obtain this lock. However, if you wish to use another driver for acquiring the lock, you may define a `uniqueVia` method that returns the cache driver that should be used:

    use MacropaySolutions\Kernel\Contracts\Cache\Repository;

    class UpdateSearchIndex implements ShouldQueue, ShouldBeUnique
    {
        ...

        /**
         * Get the cache driver for the unique job lock.
         */
        public function uniqueVia(): Repository
        {
            return \app('cache')->driver('redis');
        }
    }

> [!NOTE]  
> If you only need to limit the concurrent processing of a job, use the [`WithoutOverlapping`](/queues#preventing-job-overlaps) job middleware instead.

<a name="encrypted-jobs"></a>
### Encrypted Jobs

Framework allows you to ensure the privacy and integrity of a job's data via [encryption](/encryption). To get started, simply add the `ShouldBeEncrypted` interface to the job class. Once this interface has been added to the class, Framework will automatically encrypt your job before pushing it onto a queue:

    <?php

    use MacropaySolutions\Kernel\Contracts\Queue\ShouldBeEncrypted;
    use MacropaySolutions\Kernel\Contracts\Queue\ShouldQueue;

    class UpdateSearchIndex implements ShouldQueue, ShouldBeEncrypted
    {
        // ...
    }

<a name="job-middleware"></a>
## Job Middleware

> [!WARNING]  
> Job Middleware is typically only applicable to legacy Object Jobs. Array callables should handle middleware logic via direct dependency injection in the executing method.

Job middleware allow you to wrap custom logic around the execution of queued jobs, reducing boilerplate in the jobs themselves. For example, consider the following `handle` method which leverages Framework's Redis rate limiting features to allow only one job to process every five seconds:

    use MacropaySolutions\Kernel\Support\Facades\Redis;

    /**
     * Execute the job.
     */
    public function handle(): void
    {
        Redis::throttle('key')->block(0)->allow(1)->every(5)->then(function () {
            info('Lock obtained...');

            // Handle job...
        }, function () {
            // Could not obtain lock...

            return $this->release(5);
        });
    }

While this code is valid, the implementation of the `handle` method becomes noisy since it is cluttered with Redis rate limiting logic. In addition, this rate limiting logic must be duplicated for any other jobs that we want to rate limit.

Instead of rate limiting in the handle method, we could define a job middleware that handles rate limiting. Framework does not have a default location for job middleware, so you are welcome to place job middleware anywhere in your application. In this example, we will place the middleware in an `app/Jobs/Middleware` directory:

    <?php

    namespace App\Jobs\Middleware;

    use Closure;

    class RateLimited
    {
        /**
         * Process the queued job.
         *
         * @param  \Closure(object): void  $next
         */
        public function handle(object $job, Closure $next): void
        {
            \app('redis')->throttle('key')
                    ->block(0)->allow(1)->every(5)
                    ->then(function () use ($job, $next) {
                        // Lock obtained...

                        $next($job);
                    }, function () use ($job) {
                        // Could not obtain lock...

                        $job->release(5);
                    });
        }
    }

As you can see, like [route middleware](/middleware), job middleware receive the job being processed and a callback that should be invoked to continue processing the job.

After creating job middleware, they may be attached to a job by returning them from the job's `middleware` method. This method does not exist on jobs scaffolded by the `make:job` Run command, so you will need to manually add it to your job class:

    use App\Jobs\Middleware\RateLimited;

    /**
     * Get the middleware the job should pass through.
     *
     * @return array<int, object>
     */
    public function middleware(): array
    {
        return [new RateLimited];
    }

<a name="rate-limiting"></a>
### Rate Limiting

Although we just demonstrated how to write your own rate limiting job middleware, Framework actually includes a rate limiting middleware that you may utilize to rate limit jobs. Like [route rate limiters](/routing#defining-rate-limiters), job rate limiters are defined using the `RateLimiter` facade's `for` method.

For example, you may wish to allow users to backup their data once per hour while imposing no such limit on premium customers. To accomplish this, you may define a `RateLimiter` in the `boot` method of your `AppServiceProvider`:

    use MacropaySolutions\Kernel\Cache\RateLimiting\Limit;

    /**
     * Bootstrap any application services.
     */
    public function boot(): void
    {
        \app(\MacropaySolutions\Kernel\Cache\RateLimiter::class)->for('backups', function (object $job) {
            return $job->user->vipCustomer()
                        ? Limit::none()
                        : Limit::perHour(1)->by($job->user->id);
        });
    }

Once you have defined your rate limit, you may attach the rate limiter to your job using the `MacropaySolutions\Kernel\Queue\Middleware\RateLimited` middleware:

    use MacropaySolutions\Kernel\Queue\Middleware\RateLimited;

    /**
     * Get the middleware the job should pass through.
     *
     * @return array<int, object>
     */
    public function middleware(): array
    {
        return [new RateLimited('backups')];
    }

Releasing a rate limited job back onto the queue will still increment the job's total number of `attempts`. You may wish to tune your `tries` and `maxExceptions` properties on your job class accordingly. Or, you may wish to use the [`retryUntil` method](#time-based-attempts) to define the amount of time until the job should no longer be attempted.

If you do not want a job to be retried when it is rate limited, you may use the `dontRelease` method:

    /**
     * Get the middleware the job should pass through.
     *
     * @return array<int, object>
     */
    public function middleware(): array
    {
        return [(new RateLimited('backups'))->dontRelease()];
    }

> [!NOTE]  
> If you are using Redis, you may use the `MacropaySolutions\Kernel\Queue\Middleware\RateLimitedWithRedis` middleware, which is fine-tuned for Redis and more efficient than the basic rate limiting middleware.

<a name="preventing-job-overlaps"></a>
### Preventing Job Overlaps

Framework includes an `MacropaySolutions\Kernel\Queue\Middleware\WithoutOverlapping` middleware that allows you to prevent job overlaps based on an arbitrary key. This can be helpful when a queued job is modifying a resource that should only be modified by one job at a time.

For example, let's imagine you have a queued job that updates a user's credit score and you want to prevent credit score update job overlaps for the same user ID. To accomplish this, you can return the `WithoutOverlapping` middleware from your job's `middleware` method:

    use MacropaySolutions\Kernel\Queue\Middleware\WithoutOverlapping;

    /**
     * Get the middleware the job should pass through.
     *
     * @return array<int, object>
     */
    public function middleware(): array
    {
        return [new WithoutOverlapping($this->user->id)];
    }

Any overlapping jobs of the same type will be released back to the queue. You may also specify the number of seconds that must elapse before the released job will be attempted again:

    /**
     * Get the middleware the job should pass through.
     *
     * @return array<int, object>
     */
    public function middleware(): array
    {
        return [(new WithoutOverlapping($this->order->id))->releaseAfter(60)];
    }

If you wish to immediately delete any overlapping jobs so that they will not be retried, you may use the `dontRelease` method:

    /**
     * Get the middleware the job should pass through.
     *
     * @return array<int, object>
     */
    public function middleware(): array
    {
        return [(new WithoutOverlapping($this->order->id))->dontRelease()];
    }

The `WithoutOverlapping` middleware is powered by Framework's atomic lock feature. Sometimes, your job may unexpectedly fail or timeout in such a way that the lock is not released. Therefore, you may explicitly define a lock expiration time using the `expireAfter` method. For example, the example below will instruct Framework to release the `WithoutOverlapping` lock three minutes after the job has started processing:

    /**
     * Get the middleware the job should pass through.
     *
     * @return array<int, object>
     */
    public function middleware(): array
    {
        return [(new WithoutOverlapping($this->order->id))->expireAfter(180)];
    }

> [!WARNING]  
> The `WithoutOverlapping` middleware requires a cache driver that supports [locks](/cache#atomic-locks). Currently, the `memcached`, `redis`, `dynamodb`, `database`, `file`, and `array` cache drivers support atomic locks.

<a name="sharing-lock-keys"></a>
#### Sharing Lock Keys Across Job Classes

By default, the `WithoutOverlapping` middleware will only prevent overlapping jobs of the same class. So, although two different job classes may use the same lock key, they will not be prevented from overlapping. However, you can instruct Framework to apply the key across job classes using the `shared` method:

```php
use MacropaySolutions\Kernel\Queue\Middleware\WithoutOverlapping;

class ProviderIsDown
{
    // ...


    public function middleware(): array
    {
        return [
            (new WithoutOverlapping("status:{$this->provider}"))->shared(),
        ];
    }
}

class ProviderIsUp
{
    // ...


    public function middleware(): array
    {
        return [
            (new WithoutOverlapping("status:{$this->provider}"))->shared(),
        ];
    }
}
```

<a name="throttling-exceptions"></a>
### Throttling Exceptions

Framework includes a `MacropaySolutions\Kernel\Queue\Middleware\ThrottlesExceptions` middleware that allows you to throttle exceptions. Once the job throws a given number of exceptions, all further attempts to execute the job are delayed until a specified time interval lapses. This middleware is particularly useful for jobs that interact with third-party services that are unstable.

For example, let's imagine a queued job that interacts with a third-party API that begins throwing exceptions. To throttle exceptions, you can return the `ThrottlesExceptions` middleware from your job's `middleware` method. Typically, this middleware should be paired with a job that implements [time based attempts](#time-based-attempts):

    use DateTime;
    use MacropaySolutions\Kernel\Queue\Middleware\ThrottlesExceptions;

    /**
     * Get the middleware the job should pass through.
     *
     * @return array<int, object>
     */
    public function middleware(): array
    {
        return [new ThrottlesExceptions(10, 5)];
    }

    /**
     * Determine the time at which the job should timeout.
     */
    public function retryUntil(): DateTime
    {
        return now()->addMinutes(5);
    }

The first constructor argument accepted by the middleware is the number of exceptions the job can throw before being throttled, while the second constructor argument is the number of minutes that should elapse before the job is attempted again once it has been throttled. In the code example above, if the job throws 10 exceptions within 5 minutes, we will wait 5 minutes before attempting the job again.

When a job throws an exception but the exception threshold has not yet been reached, the job will typically be retried immediately. However, you may specify the number of minutes such a job should be delayed by calling the `backoff` method when attaching the middleware to the job:

    use MacropaySolutions\Kernel\Queue\Middleware\ThrottlesExceptions;

    /**
     * Get the middleware the job should pass through.
     *
     * @return array<int, object>
     */
    public function middleware(): array
    {
        return [(new ThrottlesExceptions(10, 5))->backoff(5)];
    }

Internally, this middleware uses Framework's cache system to implement rate limiting, and the job's class name is utilized as the cache "key". You may override this key by calling the `by` method when attaching the middleware to your job. This may be useful if you have multiple jobs interacting with the same third-party service and you would like them to share a common throttling "bucket":

    use MacropaySolutions\Kernel\Queue\Middleware\ThrottlesExceptions;

    /**
     * Get the middleware the job should pass through.
     *
     * @return array<int, object>
     */
    public function middleware(): array
    {
        return [(new ThrottlesExceptions(10, 10))->by('key')];
    }

> [!NOTE]  
> If you are using Redis, you may use the `MacropaySolutions\Kernel\Queue\Middleware\ThrottlesExceptionsWithRedis` middleware, which is fine-tuned for Redis and more efficient than the basic exception throttling middleware.

<a name="dispatching-jobs"></a>
## Dispatching Jobs

You may dispatch tasks using the `\dispatch` global helper method. The arguments passed to the `\dispatch` method will automatically wrap array callables or standard job classes (if strict security is bypassed):

    <?php

    namespace App\Http\Controllers;

    use App\Http\Controllers\Controller;
    use App\Jobs\ProcessPodcast;
    use App\Models\Podcast;
    use MacropaySolutions\Kernel\Http\RedirectResponse;
    use MacropaySolutions\Kernel\Http\Request;

    class PodcastController extends Controller
    {
        /**
         * Store a new podcast.
         */
        public function store(Request $request): RedirectResponse
        {
            $podcast = Podcast::create(/* ... */);

            // ...

            \dispatch([ProcessPodcast::class, 'handle', ['podcastId' => $podcast->id]]);

            return redirect('/podcasts');
        }
    }

<a name="delayed-dispatching"></a>
### Delayed Dispatching

If you would like to specify that a job should not be immediately available for processing by a queue worker, you may use the `delay` method when dispatching the job:

    new \MacropaySolutions\Framework\Bus\PendingCallableDispatch(
        \MacropaySolutions\Kernel\Queue\CallQueuedCallable::create(
            [ProcessPodcast::class, 'handle', ['podcastId' => $podcast->id]]
        )->delay(now()->addMinutes(10))
    );

> [!WARNING]  
> The Amazon SQS queue service has a maximum delay time of 15 minutes.

<a name="dispatching-after-the-response-is-sent-to-browser"></a>
#### Dispatching After the Response is Sent to the Browser

Alternatively, the `afterResponse` method delays dispatching a job until after the HTTP response is sent to the user's browser if your web server is using FastCGI:

    \dispatch([SendNotification::class, 'handle', ['userId' => 1]])->afterResponse();

<a name="synchronous-dispatching"></a>
### Synchronous Dispatching

If you would like to dispatch a job immediately (synchronously), you may use the `dispatch_now` method. When using this method, the job will not be queued and will be executed immediately within the current process:

    <?php

    use App\Jobs\ProcessPodcast;

    \dispatch_now([ProcessPodcast::class, 'handle']);


<a name="jobs-and-database-transactions"></a>
### Jobs & Database Transactions

While it is perfectly fine to dispatch jobs within database transactions, you should take special care to ensure that your job will actually be able to execute successfully. When dispatching a job within a transaction, it is possible that the job will be processed by a worker before the parent transaction has committed. When this happens, any updates you have made to models or database records during the database transaction(s) may not yet be reflected in the database. In addition, any models or database records created within the transaction(s) may not exist in the database.

Thankfully, Framework provides several methods of working around this problem. First, you may set the `after_commit` connection option in your queue connection's configuration array:

    'redis' => [
        'driver' => 'redis',
        // ...
        'after_commit' => true,
    ],

When the `after_commit` option is `true`, you may dispatch jobs within database transactions; however, Framework will wait until the open parent database transactions have been committed before actually dispatching the job. Of course, if no database transactions are currently open, the job will be dispatched immediately.

If a transaction is rolled back due to an exception that occurs during the transaction, the jobs that were dispatched during that transaction will be discarded.

> [!NOTE]  
> Setting the `after_commit` configuration option to `true` will also cause any queued event listeners, mailables, notifications, and broadcast events to be dispatched after all open database transactions have been committed.

<a name="specifying-commit-dispatch-behavior-inline"></a>
#### Specifying Commit Dispatch Behavior Inline

If you do not set the `after_commit` queue connection configuration option to `true`, you may still indicate that a specific job should be dispatched after all open database transactions have been committed. To accomplish this, you may chain the `afterCommit` method:

    use App\Jobs\ProcessPodcast;

    new \MacropaySolutions\Framework\Bus\PendingCallableDispatch(
        \MacropaySolutions\Kernel\Queue\CallQueuedCallable::create(
            [ProcessPodcast::class, 'handle', ['podcastId' => $podcast->id]]
        )->afterCommit()
    );


Likewise, if the `after_commit` configuration option is set to `true`, you may indicate that a specific job should be dispatched immediately without waiting for any open database transactions to commit:

    use App\Jobs\ProcessPodcast;

    new \MacropaySolutions\Framework\Bus\PendingCallableDispatch(
        \MacropaySolutions\Kernel\Queue\CallQueuedCallable::create(
            [ProcessPodcast::class, 'handle', ['podcastId' => $podcast->id]]
        )->beforeCommit()
    );

<a name="job-chaining"></a>
### Job Chaining

Job chaining allows you to specify a list of queued tasks that should be run in sequence after the primary task has executed successfully. If one task in the sequence fails, the rest of the tasks will not be run.

You must use the `chain` method provided by the `Queueable` trait directly on your callable instance *before* dispatching it:

    use App\Services\OptimizePodcast;
    use App\Services\ProcessPodcast;
    use App\Services\ReleasePodcast;
    use MacropaySolutions\Kernel\Queue\CallQueuedCallable;

    $job = CallQueuedCallable::create([ProcessPodcast::class, 'handle']);

    $job->chain([
        [OptimizePodcast::class, 'handle', ['podcastId' => 1]],
        [ReleasePodcast::class, 'handle', ['podcastId' => 1]],
    ]);

    new \MacropaySolutions\Framework\Bus\PendingCallableDispatch($job);

> [!WARNING]  
> If you have opted out of Strict Security Mode to use legacy object-based jobs, you may chain them in the exact same manner: `(new ProcessPodcast())->chain([...])`. However, you **cannot** chain Closures, as closure serialization is strictly forbidden by the framework.

> [!WARNING]  
> Deleting jobs using the `$job->delete()` method (via the injected `Job` interface) will not prevent chained jobs from being processed. The chain will only stop executing if a job in the chain fails by throwing an unhandled exception or calling `$job->fail()`.

<a name="chain-connection-queue"></a>
#### Chain Connection and Queue

If you would like to specify the connection and queue that should be used for the chained jobs, you may use the `onConnection` and `onQueue` methods. These methods specify the queue connection and queue name that should be used unless the queued job is explicitly assigned a different connection / queue:

    $job = \MacropaySolutions\Kernel\Queue\CallQueuedCallable::create([ProcessPodcast::class, 'handle', ['podcastId' => 1]]);

    $job->chain([
        [OptimizePodcast::class, 'handle', ['podcastId' => 1]],
        [ReleasePodcast::class, 'handle', ['podcastId' => 1]],
    ]);

    \dispatch($job)->onConnection('redis')->onQueue('podcasts');

<a name="chain-failures"></a>
#### Chain Failures

When chaining jobs, you may use the `catch` method to specify an Array Callable that should be invoked if a job within the chain fails:

    use App\Services\OptimizePodcast;
    use App\Services\ProcessPodcast;
    use App\Services\ReleasePodcast;
    use MacropaySolutions\Kernel\Queue\CallQueuedCallable;

    $job = CallQueuedCallable::create([ProcessPodcast::class, 'handle']);

    $job->chain([
        [OptimizePodcast::class, 'handle', ['podcastId' => 1]],
        [ReleasePodcast::class, 'handle', ['podcastId' => 1]],
    ])->onFailure([ProcessPodcast::class, 'handleChainFailure']);

    new \MacropaySolutions\Framework\Bus\PendingCallableDispatch($job);

or

    $job->chain([
        [OptimizePodcast::class, 'handle', ['podcastId' => 1]],
        [ReleasePodcast::class, 'handle', ['podcastId' => 1]],
    ]);

    (new \MacropaySolutions\Framework\Bus\PendingCallableDispatch($job))->catch([ProcessPodcast::class, 'handleChainFailure']);

<a name="customizing-the-queue-and-connection"></a>
### Customizing The Queue and Connection

<a name="dispatching-to-a-particular-queue"></a>
#### Dispatching to a Particular Queue

By pushing jobs to different queues, you may "categorize" your queued jobs and even prioritize how many workers you assign to various queues. To specify the queue, use the `onQueue` method when dispatching the job:

    \dispatch([ProcessPodcast::class, 'handle', ['podcastId' => $podcast->id]])->onQueue('processing');

<a name="dispatching-to-a-particular-connection"></a>
#### Dispatching to a Particular Connection

If your application interacts with multiple queue connections, you may specify which connection to push a job to using the `onConnection` method:

    \dispatch([ProcessPodcast::class, 'handle', ['podcastId' => $podcast->id]])->onConnection('sqs');

You may chain the `onConnection` and `onQueue` methods together to specify the connection and the queue for a job:

    \dispatch([ProcessPodcast::class, 'handle', ['podcastId' => $podcast->id]])
                  ->onConnection('sqs')
                  ->onQueue('processing');

**SQS FIFO and Fair Queues**

PHP Kernel supports Amazon SQS FIFO (First-In-First-Out) queues through message deduplication.

    \dispatch([JobExample::class, 'handle', ['payload' => $payload]])
        ->onGroup('customer-' . $payload['customer_id']);

Implement a deduplicationId method with string return type in your job to prevent for 5 minutes duplicate dispatches (if not, a default Str::orderedUuid()->toString() will be used meaning duplicates will be processed):

    /**
     * Get the job's deduplication ID.
     */
    public function deduplicationId(): string
    {
        return 'prefix-' . $this->customId;
    }

<a name="max-job-attempts-and-timeout"></a>
### Specifying Max Job Attempts / Timeout Values

<a name="max-attempts"></a>
#### Max Attempts

If one of your queued jobs is encountering an error, you likely do not want it to keep retrying indefinitely. Therefore, Framework provides various ways to specify how many times or for how long a job may be attempted.

One approach to specifying the maximum number of times a job may be attempted is via the `--tries` switch on the Run command line. This will apply to all jobs processed by the worker unless the job being processed specifies the number of times it may be attempted:

    php run queue:work --tries=3

If a job exceeds its maximum number of attempts, it will be considered a "failed" job. For more information on handling failed jobs, consult the [failed job documentation](#dealing-with-failed-jobs). If `--tries=0` is provided to the `queue:work` command, the job will be retried indefinitely.

You may take a more granular approach by defining the maximum number of times a job may be attempted on the job class itself. If the maximum number of attempts is specified on the job, it will take precedence over the `--tries` value provided on the command line:

    <?php

    namespace App\Jobs;

    class ProcessPodcast implements ShouldQueue
    {
        /**
         * The number of times the job may be attempted.
         *
         * @var int
         */
        public $tries = 5;
    }

If you need dynamic control over a particular job's maximum attempts, you may define a `tries` method on the job:

    /**
     * Determine number of times the job may be attempted.
     */
    public function tries(): int
    {
        return 5;
    }

<a name="time-based-attempts"></a>
#### Time Based Attempts

As an alternative to defining how many times a job may be attempted before it fails, you may define a time at which the job should no longer be attempted. This allows a job to be attempted any number of times within a given time frame. To define the time at which a job should no longer be attempted, add a `retryUntil` method to your job class. This method should return a `DateTime` instance:

    use DateTime;

    /**
     * Determine the time at which the job should timeout.
     */
    public function retryUntil(): DateTime
    {
        return now()->addMinutes(10);
    }

> [!NOTE]  
> You may also define a `tries` property or `retryUntil` method on your [queued event listeners](/events#queued-event-listeners).

<a name="max-exceptions"></a>
#### Max Exceptions

Sometimes you may wish to specify that a job may be attempted many times, but should fail if the retries are triggered by a given number of unhandled exceptions (as opposed to being released by the `release` method directly). To accomplish this, you may define a `maxExceptions` property on your job class:

    <?php

    namespace App\Jobs;

    class ProcessPodcast implements ShouldQueue
    {
        /**
         * The number of times the job may be attempted.
         *
         * @var int
         */
        public $tries = 25;

        /**
         * The maximum number of unhandled exceptions to allow before failing.
         *
         * @var int
         */
        public $maxExceptions = 3;

        /**
         * Execute the job.
         */
        public function handle(): void
        {
            \app('redis')->throttle('key')->allow(10)->every(60)->then(function () {
                // Lock obtained, process the podcast...
            }, function () {
                // Unable to obtain lock...
                return $this->release(10);
            });
        }
    }

In this example, the job is released for ten seconds if the application is unable to obtain a Redis lock and will continue to be retried up to 25 times. However, the job will fail if three unhandled exceptions are thrown by the job.

<a name="timeout"></a>
#### Timeout

Often, you know roughly how long you expect your queued jobs to take. For this reason, Framework allows you to specify a "timeout" value. By default, the timeout value is 60 seconds. If a job is processing for longer than the number of seconds specified by the timeout value, the worker processing the job will exit with an error. Typically, the worker will be restarted automatically by a [process manager configured on your server](#supervisor-configuration).

The maximum number of seconds that jobs can run may be specified using the `--timeout` switch on the Run command line:

    php run queue:work --timeout=30

If the job exceeds its maximum attempts by continually timing out, it will be marked as failed.

You may also define the maximum number of seconds a job should be allowed to run on the job class itself. If the timeout is specified on the job, it will take precedence over any timeout specified on the command line:

    <?php

    namespace App\Jobs;

    class ProcessPodcast implements ShouldQueue
    {
        /**
         * The number of seconds the job can run before timing out.
         *
         * @var int
         */
        public $timeout = 120;
    }

Sometimes, IO blocking processes such as sockets or outgoing HTTP connections may not respect your specified timeout. Therefore, when using these features, you should always attempt to specify a timeout using their APIs as well. For example, when using Guzzle, you should always specify a connection and request timeout value.

> [!WARNING]  
> The `pcntl` PHP extension must be installed in order to specify job timeouts. In addition, a job's "timeout" value should always be less than its ["retry after"](#job-expiration) value. Otherwise, the job may be re-attempted before it has actually finished executing or timed out.

<a name="failing-on-timeout"></a>
#### Failing on Timeout

If you would like to indicate that a job should be marked as [failed](#dealing-with-failed-jobs) on timeout, you may define the `$failOnTimeout` property on the job class:

```php
/**
 * Indicate if the job should be marked as failed on timeout.
 *
 * @var bool
 */
public $failOnTimeout = true;
```

<a name="error-handling"></a>
### Error Handling

If an exception is thrown while the job is being processed, the job will automatically be released back onto the queue so it may be attempted again. The job will continue to be released until it has been attempted the maximum number of times allowed by your application. The maximum number of attempts is defined by the `--tries` switch used on the `queue:work` Run command. Alternatively, the maximum number of attempts may be defined on the job class itself. More information on running the queue worker [can be found below](#running-the-queue-worker).

<a name="manually-releasing-a-job"></a>
#### Manually Releasing a Job

Sometimes you may wish to manually release a job back onto the queue so that it can be attempted again at a later time. You may accomplish this by calling the `release` method:

    /**
     * Execute the job.
     */
    public function handle(): void
    {
        // ...

        $this->release();
    }

By default, the `release` method will release the job back onto the queue for immediate processing. However, you may instruct the queue to not make the job available for processing until a given number of seconds has elapsed by passing an integer or date instance to the `release` method:

    $this->release(10);

    $this->release(now()->addSeconds(10));

<a name="manually-failing-a-job"></a>
#### Manually Failing a Job

Occasionally you may need to manually mark a job as "failed". To do so, you may call the `fail` method:

    /**
     * Execute the job.
     */
    public function handle(): void
    {
        // ...

        $this->fail();
    }

If you would like to mark your job as failed because of an exception that you have caught, you may pass the exception to the `fail` method. Or, for convenience, you may pass a string error message which will be converted to an exception for you:

    $this->fail($exception);

    $this->fail('Something went wrong.');

> [!NOTE]  
> For more information on failed jobs, check out the [documentation on dealing with job failures](#dealing-with-failed-jobs).

<a name="job-batching"></a>
## Job Batching

Framework's job batching feature allows you to easily execute a batch of jobs and then perform some action when the batch of jobs has completed executing. Before getting started, you should create a database migration to build a table which will contain meta information about your job batches, such as their completion percentage. This migration may be generated using the `queue:batches-table` Run command:


    php run queue:batches-table

    php run migrate


<a name="defining-batchable-jobs"></a>
### Defining Batchable Jobs

To define a batchable job, you should [create a queueable job](#creating-jobs) as normal; however, you should add the `MacropaySolutions\Kernel\Bus\Batchable` trait to the job class. This trait provides access to a `batch` method which may be used to retrieve the current batch that the job is executing within:

    <?php

    namespace App\Jobs;

    use MacropaySolutions\Kernel\Bus\Batchable;
    use MacropaySolutions\Kernel\Bus\Queueable;
    use MacropaySolutions\Kernel\Contracts\Queue\ShouldQueue;
    use MacropaySolutions\Kernel\Queue\InteractsWithQueue;
    use MacropaySolutions\Kernel\Queue\SerializesModels;

    class ImportCsv implements ShouldQueue
    {
        use Batchable, InteractsWithQueue, Queueable, SerializesModels;

        /**
         * Execute the job.
         */
        public function handle(): void
        {
            if ($this->batch()->cancelled()) {
                // Determine if the batch has been cancelled...

                return;
            }

            // Import a portion of the CSV file...
        }
    }

<a name="dispatching-batches"></a>
### Dispatching Batches

To dispatch a batch of jobs, you should use the `batch` method of the `Bus` facade. Of course, batching is primarily useful when combined with completion callbacks. So, you may use the `then`, `catch`, and `finally` methods to define completion callbacks for the batch. Each of these callbacks will receive an `MacropaySolutions\Kernel\Bus\Batch` instance when they are invoked. In this example, we will imagine we are queueing a batch of jobs that each process a given number of rows from a CSV file:

    use App\Jobs\ImportCsv;
    use App\Services\BatchCallbacks;

    $batch = \app('bus')->batch([
        [ImportCsv::class, 'handle', ['start' => 1, 'end' => 100]],
        [ImportCsv::class, 'handle', ['start' => 101, 'end' => 200]],
    ])->before([BatchCallbacks::class, 'onBefore'])
      ->progress([BatchCallbacks::class, 'onProgress'])
      ->then([BatchCallbacks::class, 'onSuccess'])
      ->catch([BatchCallbacks::class, 'onFailure'])
      ->finally([BatchCallbacks::class, 'onComplete'])
      ->dispatch();

    return $batch->id;

The batch's ID, which may be accessed via the `$batch->id` property, may be used to [query the Framework command bus](#inspecting-batches) for information about the batch after it has been dispatched.

> [!WARNING]  
> Since batch callbacks are serialized and executed at a later time by the Framework queue, you should not use the `$this` variable within the callbacks.

<a name="naming-batches"></a>
#### Naming Batches

To assign an arbitrary name to a batch, you may call the `name` method while defining the batch:

    $batch = \app('bus')->batch([
        // ...
    ])->name('Import CSV')->dispatch();

<a name="batch-connection-queue"></a>
#### Batch Connection and Queue

If you would like to specify the connection and queue that should be used for the batched jobs, you may use the `onConnection` and `onQueue` methods. All batched jobs must execute within the same connection and queue:

    $batch = \app('bus')->batch([
        // ...
    ])->then([BatchCallbacks::class, 'onSuccess'])
    ->onConnection('redis')->onQueue('imports')->dispatch();

<a name="chains-and-batches"></a>
### Chains and Batches

You may define a set of chained jobs within a batch by placing the chained array callables within an array:

    use App\Jobs\ReleasePodcast;
    use App\Jobs\SendPodcastReleaseNotification;

    \app('bus')->batch([
        [
            [ReleasePodcast::class, 'handle', ['id' => 1]],
            [SendPodcastReleaseNotification::class, 'handle', ['id' => 1]],
        ],
        [
            [ReleasePodcast::class, 'handle', ['id' => 2]],
            [SendPodcastReleaseNotification::class, 'handle', ['id' => 2]],
        ],
    ])->then([BatchCallbacks::class, 'onSuccess'])->dispatch();

<a name="adding-jobs-to-batches"></a>
### Adding Jobs to Batches

Sometimes it may be useful to add additional jobs to a batch from within a batched job. This pattern can be useful when you need to batch thousands of jobs which may take too long to dispatch during a web request. So, instead, you may wish to dispatch an initial batch of "loader" jobs that hydrate the batch with even more jobs:

    $batch = \app('bus')->batch([
        // ...
    ])->then([BatchCallbacks::class, 'onSuccess'])->name('Import Contacts')->dispatch();

In this example, we will use the `LoadImportBatch` job to hydrate the batch with additional jobs. To accomplish this, we may use the `add` method on the batch instance that may be accessed via the job's `batch` method:

    use App\Jobs\ImportContacts;
    use MacropaySolutions\Kernel\Support\Collection;

    /**
     * Execute the job.
     */
    public function handle(): void
    {
        if ($this->batch()->cancelled()) {
            return;
        }

        $this->batch()->add([
            [ImportContacts::class, 'handle', ['chunk' => 1]],
            [ImportContacts::class, 'handle', ['chunk' => 2]],
        ]);
    }

> [!WARNING]  
> You may only add jobs to a batch from within a job that belongs to the same batch.

<a name="inspecting-batches"></a>
### Inspecting Batches

The `MacropaySolutions\Kernel\Bus\Batch` instance that is provided to batch completion callbacks has a variety of properties and methods to assist you in interacting with and inspecting a given batch of jobs:

    // The UUID of the batch...
    $batch->id;

    // The name of the batch (if applicable)...
    $batch->name;

    // The number of jobs assigned to the batch...
    $batch->totalJobs;

    // The number of jobs that have not been processed by the queue...
    $batch->pendingJobs;

    // The number of jobs that have failed...
    $batch->failedJobs;

    // The number of jobs that have been processed thus far...
    $batch->processedJobs();

    // The completion percentage of the batch (0-100)...
    $batch->progress();

    // Indicates if the batch has finished executing...
    $batch->finished();

    // Cancel the execution of the batch...
    $batch->cancel();

    // Indicates if the batch has been cancelled...
    $batch->cancelled();

<a name="cancelling-batches"></a>
### Cancelling Batches

Sometimes you may need to cancel a given batch's execution. This can be accomplished by calling the `cancel` method on the `MacropaySolutions\Kernel\Bus\Batch` instance:

    /**
     * Execute the job.
     */
    public function handle(): void
    {
        if ($this->user->exceedsImportLimit()) {
            return $this->batch()->cancel();
        }

        if ($this->batch()->cancelled()) {
            return;
        }
    }

As you may have noticed in the previous examples, batched jobs should typically determine if their corresponding batch has been cancelled before continuing execution. However, for convenience, you may assign the `SkipIfBatchCancelled` [middleware](#job-middleware) to the job instead. As its name indicates, this middleware will instruct Framework to not process the job if its corresponding batch has been cancelled:

    use MacropaySolutions\Kernel\Queue\Middleware\SkipIfBatchCancelled;

    /**
     * Get the middleware the job should pass through.
     */
    public function middleware(): array
    {
        return [new SkipIfBatchCancelled];
    }

<a name="batch-failures"></a>
### Batch Failures

When a batched job fails, the `catch` callback (if assigned) will be invoked. This callback is only invoked for the first job that fails within the batch.

<a name="allowing-failures"></a>
#### Allowing Failures

When a job within a batch fails, Framework will automatically mark the batch as "cancelled". If you wish, you may disable this behavior so that a job failure does not automatically mark the batch as cancelled. This may be accomplished by calling the `allowFailures` method while dispatching the batch:

    $batch = \app('bus')->batch([
        // ...
    ])->then([BatchCallbacks::class, 'onSuccess'])->allowFailures()->dispatch();

<a name="retrying-failed-batch-jobs"></a>
#### Retrying Failed Batch Jobs

For convenience, Framework provides a `queue:retry-batch` Run command that allows you to easily retry all the failed jobs for a given batch. The `queue:retry-batch` command accepts the UUID of the batch whose failed jobs should be retried:

    php run queue:retry-batch 32dbc76c-4f82-4749-b610-a639fe0099b5

<a name="pruning-batches"></a>
### Pruning Batches

Without pruning, the `job_batches` table can accumulate records very quickly. To mitigate this, you should [schedule](/scheduling) the `queue:prune-batches` Run command to run daily:

    $schedule->command('queue:prune-batches')->daily();

By default, all finished batches that are more than 24 hours old will be pruned. You may use the `hours` option when calling the command to determine how long to retain batch data. For example, the following command will delete all batches that finished over 48 hours ago:

    $schedule->command('queue:prune-batches --hours=48')->daily();

Sometimes, your `jobs_batches` table may accumulate batch records for batches that never completed successfully, such as batches where a job failed and that job was never retried successfully. You may instruct the `queue:prune-batches` command to prune these unfinished batch records using the `unfinished` option:

    $schedule->command('queue:prune-batches --hours=48 --unfinished=72')->daily();

Likewise, your `jobs_batches` table may also accumulate batch records for cancelled batches. You may instruct the `queue:prune-batches` command to prune these cancelled batch records using the `cancelled` option:

    $schedule->command('queue:prune-batches --hours=48 --cancelled=72')->daily();

<a name="storing-batches-in-dynamodb"></a>
### Storing Batches in DynamoDB

Framework also provides support for storing batch meta information in [DynamoDB](https://aws.amazon.com/dynamodb) instead of a relational database. However, you will need to manually create a DynamoDB table to store all the batch records.

Typically, this table should be named `job_batches`, but you should name the table based on the value of the `queue.batching.table` configuration value within your application's `queue` configuration file.

<a name="dynamodb-batch-table-configuration"></a>
#### DynamoDB Batch Table Configuration

The `job_batches` table should have a string primary partition key named `application` and a string primary sort key named `id`. The `application` portion of the key will contain your application's name as defined by the `name` configuration value within your application's `app` configuration file. Since the application name is part of the DynamoDB table's key, you can use the same table to store job batches for multiple Framework applications.

In addition, you may define `ttl` attribute for your table if you would like to take advantage of [automatic batch pruning](#pruning-batches-in-dynamodb).

<a name="dynamodb-configuration"></a>
#### DynamoDB Configuration

Next, install the AWS SDK so that your Framework application can communicate with Amazon DynamoDB:

```shell
composer require aws/aws-sdk-php
```

Then, set the `queue.batching.driver` configuration option's value to `dynamodb`. In addition, you should define `key`, `secret`, and `region` configuration options within the `batching` configuration array. These options will be used to authenticate with AWS. When using the `dynamodb` driver, the `queue.batching.database` configuration option is unnecessary:

```php
'batching' => [
    'driver' => env('QUEUE_FAILED_DRIVER', 'dynamodb'),
    'key' => env('AWS_ACCESS_KEY_ID'),
    'secret' => env('AWS_SECRET_ACCESS_KEY'),
    'region' => env('AWS_DEFAULT_REGION', 'us-east-1'),
    'table' => 'job_batches',
],
```

<a name="pruning-batches-in-dynamodb"></a>
#### Pruning Batches in DynamoDB

When utilizing [DynamoDB](https://aws.amazon.com/dynamodb) to store job batch information, the typical pruning commands used to prune batches stored in a relational database will not work. Instead, you may utilize [DynamoDB's native TTL functionality](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/TTL.html) to automatically remove records for old batches.

If you defined your DynamoDB table with a `ttl` attribute, you may define configuration parameters to instruct Framework how to prune batch records. The `queue.batching.ttl_attribute` configuration value defines the name of the attribute holding the TTL, while the `queue.batching.ttl` configuration value defines the number of seconds after which a batch record can be removed from the DynamoDB table, relative to the last time the record was updated:

```php
'batching' => [
    'driver' => env('QUEUE_FAILED_DRIVER', 'dynamodb'),
    'key' => env('AWS_ACCESS_KEY_ID'),
    'secret' => env('AWS_SECRET_ACCESS_KEY'),
    'region' => env('AWS_DEFAULT_REGION', 'us-east-1'),
    'table' => 'job_batches',
    'ttl_attribute' => 'ttl',
    'ttl' => 60 * 60 * 24 * 7, // 7 days...
],
```

<a name="queueing-storable-array-callables-recommended"></a>
## Queueing Storable Array Callables (Recommended)

Framework introduces **Storable Array Callables** as a high-performance, lightweight, and mathematically secure alternative to `SerializableClosure` and traditional object-based jobs. By allowing developers to pass structured arrays instead of closures to the Queue system, we significantly reduce serialization overhead and SQS/Redis payload sizes while maintaining full Dependency Injection support.

This architecture brings a **significant increase in total worker throughput** by eliminating the high overhead of PHP object serialization and reconstruction.

You can pass an array in the format `[Class::class, 'method', ['named' => 'param']]` to almost any framework entry point that previously required a Job object or a Closure.

#### Basic Dispatching

No need to create a Job class for simple logic. The container will autowire dependencies into the target method.

    use App\Services\EmailService;

    // Primitive arguments passed explicitly. Dependencies autowired.
    dispatch([EmailService::class, 'sendWelcomeEmail', ['userId' => $user->id]]);

    // If a developer chains an Object (BLOCKED):
    dispatch(['Class', 'method'])->chain([
        new SendEmailJob()
    ]);
    
    // If a developer chains an Array (ALLOWED & WORKING):
    
    dispatch(['Class', 'method'])->chain([
        ['EmailService', 'sendWelcomeEmail']
    ]);

#### Chains and Batches Arrays

Storable array callables seamlessly integrate with Framework's batching and chaining systems.
```php
    use App\Services\ImageProcessor;
    use MacropaySolutions\Kernel\Support\Facades\Bus;

    \app('bus')->batch([
        [ImageProcessor::class, 'optimize', ['path' => 'photo1.jpg']],
        [ImageProcessor::class, 'optimize', ['path' => 'photo2.jpg']],
    ])->dispatch();
```
> [!WARNING]
> **Important Limitations:** To guarantee 100% protection against PHP Object Injection, `dispatch` strictly forbids objects in payloads or chained arrays. Obvious models are not automatically serialized and re-fetched natively; developers are responsible for passing primitive IDs and querying fresh database state inside their array callables. Alternatively, if capturing the exact current state of the model is important, you can pass its array shape (e.g., $model->toArray()`).

#### Job Chaining Exceptions

When chaining jobs onto an Array Callable, you cannot chain a standard instantiated Job object. You must chain other Array Callables.

**❌ Incorrect (Throws Exception):**
```php
    $job = \MacropaySolutions\Kernel\Queue\CallQueuedCallable::create([ProcessUserRegistration::class, 'handle', ['userId' => 5]]);
    
    // Fails because `SendWelcomeEmail` is an instantiated object
    $job->chain([
        new SendWelcomeEmail(5) 
    ]);
    
    \dispatch($job);
```
**✅ Correct (Using Array Callables):**
```php
    // Succeeds because the chain uses primitive arrays and strings
    $job = \MacropaySolutions\Kernel\Queue\CallQueuedCallable::create([ProcessUserRegistration::class, 'handle', ['userId' => 5]]);

    // Succeeds because the chain uses primitive arrays and strings
    $job->chain([
        [SendWelcomeEmail::class, 'handle', ['userId' => 5]]
    ]);
    
    \dispatch($job);
```
Failure Callbacks (catch)
Similarly, when defining failure callbacks on the dispatch, you must use the Array Callable syntax instead of standard Closures or invokable objects.

**❌ Incorrect (Throws Exception):**
```php
    // Fails because a Closure is an object under the hood
    \dispatch([ReportGenerator::class, 'run', ['reportId' => 10]])
        ->catch(function (\Throwable $e) {
            // ...
        });
```
**✅ Correct (Using Array Callables):**
```php
    // Succeeds because the catch callback is a storable array
    \dispatch([ReportGenerator::class, 'run', ['reportId' => 10]])
        ->catch([ReportGenerator::class, 'failed', ['reportId' => 10]]);
```

#### Job Middleware and Rate Limiting

Because Array Callables are not traditional Job classes, you cannot define a `middleware()` method on them. If you need to apply rate limiting or prevent job overlaps, you should use Framework's caching and throttling facades directly inside your targeted method.

Because the Storable Array Callable engine seamlessly injects the underlying queue worker, you can type-hint the `Job` interface to manually release or fail the job if it is rate-limited!

**✅ Correct (Rate Limiting via Method Injection):**
```php
use MacropaySolutions\Kernel\Contracts\Queue\Job;
use MacropaySolutions\Kernel\Support\Facades\Redis;

class EmailService
{
    // The native Job instance is injected automatically!
    public function sendWelcomeEmail(int $userId, Job $job): void
    {
        Redis::throttle('welcome-emails')
            ->allow(10)->every(60)
            ->then(function () use ($userId) {
                // Lock obtained, process the email...
                User::query()->findOrFail($userId)->sendWelcome();
            }, function () use ($job) {
                // Rate limit exceeded, release the job back to the queue for 10 seconds
                $job->release(10);
            });
    }
}
```

<a name="strict-security-mode"></a>
#### Strict Security Mode
To completely eliminate the risk of PHP Object Injection (POI) vulnerabilities, you may instruct Framework to aggressively reject any serialized objects pushed to the queue. You can enable this strict security boundary by overriding the FORBID_SERIALIZED_OBJECTS_IN_QUEUE constant in your App\Application class:

```PHP
public const FORBID_SERIALIZED_OBJECTS_IN_QUEUE = true;
```
When enabled, the framework will forcefully scan all outbound job payloads. If a developer attempts to dispatch a traditional instantiated job object, a queued closure, or attempts to chain an object into an array callable, the queue dispatcher will immediately throw an InvalidArgumentException.

In this strict mode, only Storable Array Callables, traditional string-based jobs (`Class@method`), and primitive data types (strings, integers, floats, booleans, arrays) are permitted in the queue payload.

<a name="encrypted-array-callables"></a>
#### Encrypted Array Callables
Storable Array Callables fully support Framework's built-in payload encryption. If the target class referenced in your array callable implements the `MacropaySolutions\Kernel\Contracts\Queue\ShouldBeEncrypted` interface, the framework will intelligently detect it and automatically encrypt the queued job's metadata and arguments.

```PHP
namespace App\CallablesAsArray;

use MacropaySolutions\Kernel\Contracts\Queue\ShouldBeEncrypted;

class ProcessPayroll implements ShouldBeEncrypted
{
    public function handle(int $employeeId): void
    {
        // ...
    }
}

// The payload for this array callable will be automatically encrypted...
dispatch([ProcessPayroll::class, 'handle', ['employeeId' => 5]]);
```
<a name="queueing-closures"></a>
## Queueing Closures

> [!WARNING]  
> Queueing closures will not work. For queueing jobs you must explicitly opt out of strict mode in `Application.php` to use this. Consider using [Storable Array Callables](#queueing-storable-array-callables-recommended) instead.


<a name="running-the-queue-worker"></a>
## Running the Queue Worker

<a name="the-queue-work-command"></a>
### The `queue:work` Command

Framework includes an Run command that will start a queue worker and process new jobs as they are pushed onto the queue. You may run the worker using the `queue:work` Run command:

    php run queue:work

> [!NOTE]  
> To keep the `queue:work` process running permanently in the background, you should use a process monitor such as [Supervisor](#supervisor-configuration) to ensure that the queue worker does not stop running.
>
> But be aware that Supervisor is NOT AWARE of signals! That means that it will restart the worker after a `SIGTERM` on scale down or deploy scenario in the time between the signal is received and the container is killed!

You may include the `-v` flag when invoking the `queue:work` command if you would like the processed job IDs to be included in the command's output:


    php run queue:work -v


Remember, queue workers are long-lived processes and store the booted application state in memory. As a result, they will not notice changes in your code base after they have been started. So, during your deployment process, be sure to [restart your queue workers](#queue-workers-and-deployment). In addition, remember that any static state created or modified by your application will not be automatically reset between jobs.

To avoid issues you can use the `--once` flag because the `commands:cache` command will shorten the run boot time. Also, if you want to fail the jobs that exceed the memory limit and result in a fatal error, use the `--fail-on-fatal` flag.

Alternatively, you may run the `queue:listen` command. When using the `queue:listen` command, you don't have to manually restart the worker when you want to reload your updated code or reset the application state; however, this command is significantly less efficient than the `queue:work` command:


    php run queue:listen


<a name="running-multiple-queue-workers"></a>
#### Running Multiple Queue Workers

To assign multiple workers to a queue and process jobs concurrently, you should simply start multiple `queue:work` processes. This can either be done locally via multiple tabs in your terminal or in production using your process manager's configuration settings.

<a name="specifying-the-connection-queue"></a>
#### Specifying the Connection and Queue

You may also specify which queue connection the worker should utilize. The connection name passed to the `work` command should correspond to one of the connections defined in your `config/queue.php` configuration file:


    php run queue:work redis


By default, the `queue:work` command only processes jobs for the default queue on a given connection. However, you may customize your queue worker even further by only processing particular queues for a given connection. For example, if all of your emails are processed in an `emails` queue on your `redis` queue connection, you may issue the following command to start a worker that only processes that queue:


    php run queue:work redis --queue=emails


<a name="processing-a-specified-number-of-jobs"></a>
#### Processing a Specified Number of Jobs

The `--once` option may be used to instruct the worker to only process a single job from the queue:

    php run queue:work --once

The `--max-jobs` option may be used to instruct the worker to process the given number of jobs and then exit. This option may be useful when combined with [Supervisor](#supervisor-configuration) so that your workers are automatically restarted after processing a given number of jobs, releasing any memory they may have accumulated:

```shell
php run queue:work --max-jobs=1000
```

<a name="processing-all-queued-jobs-then-exiting"></a>
#### Processing All Queued Jobs and Then Exiting

The `--stop-when-empty` option may be used to instruct the worker to process all jobs and then exit gracefully. This option can be useful when processing Framework queues within a Docker container if you wish to shutdown the container after the queue is empty:

php run queue:work --stop-when-empty

<a name="processing-jobs-for-a-given-number-of-seconds"></a>
#### Processing Jobs for a Given Number of Seconds

The `--max-time` option may be used to instruct the worker to process jobs for the given number of seconds and then exit. This option may be useful when combined with [Supervisor](#supervisor-configuration) so that your workers are automatically restarted after processing jobs for a given amount of time, releasing any memory they may have accumulated:

# Process jobs for one hour and then exit...

    php run queue:work --max-time=3600


<a name="worker-sleep-duration"></a>
#### Worker Sleep Duration

When jobs are available on the queue, the worker will keep processing jobs with no delay in between jobs. However, the `sleep` option determines how many seconds the worker will "sleep" if there are no jobs available. Of course, while sleeping, the worker will not process any new jobs:

    php run queue:work --sleep=3

<a name="resource-considerations"></a>
#### Resource Considerations

Daemon queue workers do not "reboot" the framework before processing each job. Therefore, you should release any heavy resources after each job completes. For example, if you are doing image manipulation with the GD library, you should free the memory with `imagedestroy` when you are done processing the image.

<a name="queue-priorities"></a>
### Queue Priorities

Sometimes you may wish to prioritize how your queues are processed. For example, in your `config/queue.php` configuration file, you may set the default `queue` for your `redis` connection to `low`. However, occasionally you may wish to push a job to a `high` priority queue like so:

    dispatch([...])->onQueue('high');

To start a worker that verifies that all the `high` queue jobs are processed before continuing to any jobs on the `low` queue, pass a comma-delimited list of queue names to the `work` command:


    php run queue:work --queue=high,low


<a name="queue-workers-and-deployment"></a>
### Queue Workers and Deployment

Since queue workers are long-lived processes, they will not notice changes to your code without being restarted. So, the simplest way to deploy an application using queue workers is to restart the workers during your deployment process. You may gracefully restart all the workers by issuing the `queue:restart` command:


    php run queue:restart


This command will instruct all queue workers to gracefully exit after they finish processing their current job so that no existing jobs are lost. Since the queue workers will exit when the `queue:restart` command is executed, you should be running a process manager such as [Supervisor](#supervisor-configuration) to automatically restart the queue workers.

> [!NOTE]  
> The queue uses the [cache](/cache) to store restart signals, so you should verify that a cache driver is properly configured for your application before using this feature.

<a name="job-expirations-and-timeouts"></a>
### Job Expirations and Timeouts

<a name="job-expiration"></a>
#### Job Expiration

In your `config/queue.php` configuration file, each queue connection defines a `retry_after` option. This option specifies how many seconds the queue connection should wait before retrying a job that is being processed. For example, if the value of `retry_after` is set to `90`, the job will be released back onto the queue if it has been processing for 90 seconds without being released or deleted. Typically, you should set the `retry_after` value to the maximum number of seconds your jobs should reasonably take to complete processing.

> [!WARNING]  
> The only queue connection which does not contain a `retry_after` value is Amazon SQS. SQS will retry the job based on the [Default Visibility Timeout](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/AboutVT.html) which is managed within the AWS console.

<a name="worker-timeouts"></a>
#### Worker Timeouts

The `queue:work` Run command exposes a `--timeout` option. By default, the `--timeout` value is 60 seconds. If a job is processing for longer than the number of seconds specified by the timeout value, the worker processing the job will exit with an error. Typically, the worker will be restarted automatically by a process manager configured on your server:

    php run queue:work --timeout=60

The `retry_after` configuration option and the `--timeout` CLI option are different, but work together to ensure that jobs are not lost and that jobs are only successfully processed once.

> [!WARNING]  
> The `--timeout` value should always be at least several seconds shorter than your `retry_after` configuration value. This will ensure that a worker processing a frozen job is always terminated before the job is retried. If your `--timeout` option is longer than your `retry_after` configuration value, your jobs may be processed twice.

<a name="supervisor-configuration"></a>
## Supervisor Configuration

In production, you need a way to keep your `queue:work` processes running. A `queue:work` process may stop running for a variety of reasons, such as an exceeded worker timeout or the execution of the `queue:restart` command.

For this reason, you need to configure a process monitor that can detect when your `queue:work` processes exit and automatically restart them. In addition, process monitors can allow you to specify how many `queue:work` processes you would like to run concurrently. Supervisor is a process monitor commonly used in Linux environments and we will discuss how to configure it in the following documentation.

<a name="installing-supervisor"></a>
#### Installing Supervisor

Supervisor is a process monitor for the Linux operating system, and will automatically restart your `queue:work` processes if they fail. To install Supervisor on Ubuntu, you may use the following command:

    sudo apt-get install supervisor

> [!NOTE]  
> Supervisor does not account for signals! That means that it will restart your workers if one of them exits later, leading to incomplete processed jobs. Jobs should be idempotent but still...


For more information on Supervisor, consult the [Supervisor documentation](http://supervisord.org/index.html).

<a name="dealing-with-failed-jobs"></a>
## Dealing With Failed Jobs

Sometimes your queued jobs will fail. Don't worry, things don't always go as planned! Framework includes a convenient way to [specify the maximum number of times a job should be attempted](#max-job-attempts-and-timeout). After an asynchronous job has exceeded this number of attempts, it will be inserted into the `failed_jobs` database table. [Synchronously dispatched jobs](/queues#synchronous-dispatching) that fail are not stored in this table and their exceptions are immediately handled by the application.

A migration to create the `failed_jobs` table is typically already present in new Framework applications. However, if your application does not contain a migration for this table, you may use the `queue:failed-table` command to create the migration:

    php run queue:failed-table

    php run migrate

When running a [queue worker](#running-the-queue-worker) process, you may specify the maximum number of times a job should be attempted using the `--tries` switch on the `queue:work` command. If you do not specify a value for the `--tries` option, jobs will only be attempted once or as many times as specified by the job class' `$tries` property:

    php run queue:work redis --tries=3

Using the `--backoff` option, you may specify how many seconds Framework should wait before retrying a job that has encountered an exception. By default, a job is immediately released back onto the queue so that it may be attempted again:

    php run queue:work redis --tries=3 --backoff=3

If you would like to configure how many seconds Framework should wait before retrying a job that has encountered an exception on a per-job basis, you may do so by defining a `backoff` property on your job class:

    /**
     * The number of seconds to wait before retrying the job.
     *
     * @var int
     */
    public $backoff = 3;

If you require more complex logic for determining the job's backoff time, you may define a `backoff` method on your job class:

    /**
    * Calculate the number of seconds to wait before retrying the job.
    */
    public function backoff(): int
    {
        return 3;
    }

You may easily configure "exponential" backoffs by returning an array of backoff values from the `backoff` method. In this example, the retry delay will be 1 second for the first retry, 5 seconds for the second retry, 10 seconds for the third retry, and 10 seconds for every subsequent retry if there are more attempts remaining:

    /**
    * Calculate the number of seconds to wait before retrying the job.
    *
    * @return array<int, int>
    */
    public function backoff(): array
    {
        return [1, 5, 10];
    }

<a name="cleaning-up-after-failed-jobs"></a>
### Cleaning Up After Failed Jobs

When a particular job fails, you may want to send an alert to your users or revert any actions that were partially completed by the job. To accomplish this, you may define a `failed` method on your job class. The `Throwable` instance that caused the job to fail will be passed to the `failed` method:

    <?php

    namespace App\Jobs;

    use App\Models\Podcast;
    use App\Services\AudioProcessor;
    use MacropaySolutions\Kernel\Bus\Queueable;
    use MacropaySolutions\Kernel\Contracts\Queue\ShouldQueue;
    use MacropaySolutions\Kernel\Queue\InteractsWithQueue;
    use MacropaySolutions\Kernel\Queue\SerializesModels;
    use Throwable;

    class ProcessPodcast implements ShouldQueue
    {
        use InteractsWithQueue, Queueable, SerializesModels;

        /**
         * Create a new job instance.
         */
        public function __construct(
            public Podcast $podcast,
        ) {}

        /**
         * Execute the job.
         */
        public function handle(AudioProcessor $processor): void
        {
            // Process uploaded podcast...
        }

        /**
         * Handle a job failure.
         */
        public function failed(?Throwable $exception): void
        {
            // Send user notification of failure, etc...
        }
    }

> [!WARNING]  
> A new instance of the job is instantiated before invoking the `failed` method; therefore, any class property modifications that may have occurred within the `handle` method will be lost.

<a name="retrying-failed-jobs"></a>
### Retrying Failed Jobs

To view all the failed jobs that have been inserted into your `failed_jobs` database table, you may use the `queue:failed` Run command:

    php run queue:failed

The `queue:failed` command will list the job ID, connection, queue, failure time, and other information about the job. The job ID may be used to retry the failed job. For instance, to retry a failed job that has an ID of `ce7bb17c-cdd8-41f0-a8ec-7b4fef4e5ece`, issue the following command:

    php run queue:retry ce7bb17c-cdd8-41f0-a8ec-7b4fef4e5ece

If necessary, you may pass multiple IDs to the command:

    php run queue:retry ce7bb17c-cdd8-41f0-a8ec-7b4fef4e5ece 91401d2c-0784-4f43-824c-34f94a33c24d

You may also retry all the failed jobs for a particular queue:

    php run queue:retry --queue=name

To retry all of your failed jobs, execute the `queue:retry` command and pass `all` as the ID:

    php run queue:retry all

If you would like to delete a failed job, you may use the `queue:forget` command:

    php run queue:forget 91401d2c-0784-4f43-824c-34f94a33c24d


To delete all of your failed jobs from the `failed_jobs` table, you may use the `queue:flush` command:

    php run queue:flush

<a name="ignoring-missing-models"></a>
### Ignoring Missing Models

When injecting an Obvious model into an object job, the model is automatically serialized before being placed on the queue. However, if the model has been deleted, your job may fail with a `ModelNotFoundException`. Array Callables bypass this requirement entirely since you fetch models explicitly in your own method logic.

For convenience, you may choose to automatically delete jobs with missing models by setting your job's `deleteWhenMissingModels` property to `true`. When this property is set to `true`, Framework will quietly discard the job without raising an exception:

    /**
     * Delete the job if its models no longer exist.
     *
     * @var bool
     */
    public $deleteWhenMissingModels = true;

<a name="pruning-failed-jobs"></a>
### Pruning Failed Jobs

You may prune the records in your application's `failed_jobs` table by invoking the `queue:prune-failed` Run command:

    php run queue:prune-failed

By default, all the failed job records that are more than 24 hours old will be pruned. If you provide the `--hours` option to the command, only the failed job records that were inserted within the last N number of hours will be retained. For example, the following command will delete all the failed job records that were inserted more than 48 hours ago:

    php run queue:prune-failed --hours=48

<a name="storing-failed-jobs-in-dynamodb"></a>
### Storing Failed Jobs in DynamoDB

Framework also provides support for storing your failed job records in [DynamoDB](https://aws.amazon.com/dynamodb) instead of a relational database table. However, you must manually create a DynamoDB table to store all the failed job records. Typically, this table should be named `failed_jobs`, but you should name the table based on the value of the `queue.failed.table` configuration value within your application's `queue` configuration file.

The `failed_jobs` table should have a string primary partition key named `application` and a string primary sort key named `uuid`. The `application` portion of the key will contain your application's name as defined by the `name` configuration value within your application's `app` configuration file. Since the application name is part of the DynamoDB table's key, you can use the same table to store failed jobs for multiple Framework applications.

In addition, ensure that you install the AWS SDK so that your Framework application can communicate with Amazon DynamoDB:

```shell
composer require aws/aws-sdk-php
```

Next, set the `queue.failed.driver` configuration option's value to `dynamodb`. In addition, you should define `key`, `secret`, and `region` configuration options within the failed job configuration array. These options will be used to authenticate with AWS. When using the `dynamodb` driver, the `queue.failed.database` configuration option is unnecessary:

```php
'failed' => [
    'driver' => env('QUEUE_FAILED_DRIVER', 'dynamodb'),
    'key' => env('AWS_ACCESS_KEY_ID'),
    'secret' => env('AWS_SECRET_ACCESS_KEY'),
    'region' => env('AWS_DEFAULT_REGION', 'us-east-1'),
    'table' => 'failed_jobs',
],
```

<a name="disabling-failed-job-storage"></a>
### Disabling Failed Job Storage

You may instruct Framework to discard failed jobs without storing them by setting the `queue.failed.driver` configuration option's value to `null`. Typically, this may be accomplished via the `QUEUE_FAILED_DRIVER` environment variable:

    QUEUE_FAILED_DRIVER=null

<a name="failed-job-events"></a>
### Failed Job Events

If you would like to register an event listener that will be invoked when a job fails, you may use the `failing` method on the queue service. We strictly use Array Callables for the event listener to avoid serialization closures:

    <?php

    namespace App\Providers;

    use MacropaySolutions\Kernel\Support\ServiceProvider;

    class AppServiceProvider extends ServiceProvider
    {
        /**
         * Register any application services.
         */
        public function register(): void
        {
            // ...
        }

        /**
         * Bootstrap any application services.
         */
        public function boot(): void
        {
            \app('queue')->failing([\App\Listeners\QueueFailureListener::class, 'handle']);
        }
    }

    class QueueFailureListener
    {
        public function handle(JobFailed $event): void {
            // $event->connectionName
            // $event->job
            // $event->exception
        }
    }

<a name="clearing-jobs-from-queues"></a>
## Clearing Jobs From Queues

If you would like to delete all jobs from the default queue of the default connection, you may do so using the `queue:clear` Run command:

    php run queue:clear

You may also provide the `connection` argument and `queue` option to delete jobs from a specific connection and queue:

    php run queue:clear redis --queue=emails

> [!WARNING]  
> Clearing jobs from queues is only available for the SQS, Redis, and database queue drivers. In addition, the SQS message deletion process takes up to 60 seconds, so jobs sent to the SQS queue up to 60 seconds after you clear the queue might also be deleted.

<a name="monitoring-your-queues"></a>
## Monitoring Your Queues

If your queue receives a sudden influx of jobs, it could become overwhelmed, leading to a long wait time for jobs to complete. If you wish, Framework can alert you when your queue job count exceeds a specified threshold.

To get started, you should schedule the `queue:monitor` command to [run every minute](/scheduling). The command accepts the names of the queues you wish to monitor as well as your desired job count threshold:

    php run queue:monitor redis:default,redis:deployments --max=100

Scheduling this command alone is not enough to trigger a notification alerting you of the queue's overwhelmed status. When the command encounters a queue that has a job count exceeding your threshold, an `MacropaySolutions\Kernel\Queue\Events\QueueBusy` event will be dispatched. You may listen for this event within your application's `EventServiceProvider` in order to send a notification to you or your development team:

    use MacropaySolutions\Kernel\Queue\Events\QueueBusy;

    /**
     * Register any other events for your application.
     */
    public function boot(): void
    {
        \app('events')->listen(QueueBusy::class, [\App\Listeners\QueueBusyListener::class, 'handle']);
    }

    class QueueBusyListener
    {
        public function handle(QueueBusy $event): void {
            // $event->connection
            // $event->queue
            // $event->size
        }
    }

<a name="testing"></a>
## Testing

When testing code that dispatches jobs, you may wish to instruct Framework to not actually execute the job itself, since the job's code can be tested directly and separately of the code that dispatches it. Of course, to test the job itself, you may instantiate a job instance and invoke the `handle` method directly in your test.

You may use the `Queue` facade's `fake` method to prevent queued jobs from actually being pushed to the queue. After calling the `Queue` facade's `fake` method, you may then assert that the application attempted to push jobs to the queue:

    <?php

    namespace Tests\Feature;

    use App\Jobs\ShipOrder;
    use Tests\TestCase;

    class ExampleTest extends TestCase
    {
        public function test_orders_can_be_shipped(): void
        {
            \app('queue')->fake();

            // Perform order shipping...

            // Assert that no jobs were pushed...
            \app('queue')->assertNothingPushed();

            // Assert a Storable Array Callable was pushed to a given queue...
            \app('queue')->assertPushedOn('queue-name', [ShipOrder::class, 'handle']);

            // Assert a job was pushed twice...
            \app('queue')->assertPushed([ShipOrder::class, 'handle'], 2);

            // Assert the total number of jobs that were pushed...
            \app('queue')->assertCount(4);
        }
    }

<a name="faking-a-subset-of-jobs"></a>
### Faking a Subset of Jobs

If you only need to fake specific jobs while allowing your other jobs to execute normally, you may pass the callables that should be faked to the `fake` method:

    public function test_orders_can_be_shipped(): void
    {
        \app('queue')->fake([
            [ShipOrder::class, 'handle'],
        ]);

        // Perform order shipping...

        // Assert a job was pushed twice...
        \app('queue')->assertPushed([ShipOrder::class, 'handle'], 2);
    }

<a name="testing-job-chains"></a>
### Testing Job Chains

To test job chains, you will need to utilize the `Bus` facade's faking capabilities. The `Bus` facade's `assertChained` method may be used to assert that a [chain of jobs](/queues#job-chaining) was dispatched. The `assertChained` method accepts an array of chained jobs as its first argument:

    use App\Jobs\RecordShipment;
    use App\Jobs\ShipOrder;
    use App\Jobs\UpdateInventory;

    \app('bus')->fake();

    // ...

    \app('bus')->assertChained([
        ShipOrder::class,
        RecordShipment::class,
        UpdateInventory::class
    ]);

As you can see in the example above, the array of chained jobs may be an array of the job's class names. However, you may also provide an array of actual job instances. When doing so, Framework will ensure that the job instances are of the same class and have the same property values of the chained jobs dispatched by your application. If you have enabled Strict Security Mode, you must use Storable Array Callables for your assertions:

    // Asserting traditional job objects...
    \app('bus')->assertChained([
        new ShipOrder,
        new RecordShipment,
        new UpdateInventory,
    ]);

    // Asserting Storable Array Callables (Required for Strict Mode)...
    \app('bus')->assertChained([
        [ShipOrderService::class, 'handle', ['orderId' => 1]],
        [RecordShipmentService::class, 'handle', ['orderId' => 1]],
        [UpdateInventoryService::class, 'handle', ['orderId' => 1]],
    ]);

You may use the `assertDispatchedWithoutChain` method to assert that a job was pushed without a chain of jobs:

    \app('bus')->assertDispatchedWithoutChain(ShipOrder::class);

<a name="testing-chained-batches"></a>
#### Testing Chained Batches

If your job chain [contains a batch of jobs](#chains-and-batches), you may assert that the chained batch matches your expectations by inserting a `\app('bus')->chainedBatch` definition within your chain assertion:

    use App\Jobs\ShipOrder;
    use App\Jobs\UpdateInventory;
    use MacropaySolutions\Kernel\Bus\PendingBatch;

    // Asserting a chained batch with traditional job objects...
    \app('bus')->assertChained([
        new ShipOrder,
        \app('bus')->chainedBatch(function (PendingBatch $batch) {
            return $batch->jobs->count() === 3;
        }),
        new UpdateInventory,
    ]);

    // Asserting a chained batch with Storable Array Callables...
    \app('bus')->assertChained([
        [ShipOrderService::class, 'handle'],
        \app('bus')->chainedBatch(function (PendingBatch $batch) {
            return $batch->jobs->count() === 3;
        }),
        [UpdateInventoryService::class, 'handle'],
    ]);

<a name="testing-job-batches"></a>
### Testing Job Batches

The `assertBatched` method may be used to assert that a batch of jobs was dispatched. The closure given to the `assertBatched` method receives an instance of `MacropaySolutions\Kernel\Bus\PendingBatch`, which may be used to inspect the jobs within the batch:

    use MacropaySolutions\Kernel\Bus\PendingBatch;

    \app('bus')->fake();

    // ...

    \app('bus')->assertBatched(function (PendingBatch $batch) {
        return $batch->name == 'import-csv' &&
               $batch->jobs->count() === 10;
    });

You may use the `assertBatchCount` method to assert that a given number of batches were dispatched:

    \app('bus')->assertBatchCount(3);

You may use `assertNothingBatched` to assert that no batches were dispatched:

    \app('bus')->assertNothingBatched();

<a name="testing-job-batch-interaction"></a>
#### Testing Job / Batch Interaction

In addition, you may occasionally need to test an individual job's interaction with its underlying batch. For example, you may need to test if a job cancelled further processing for its batch. To accomplish this, you need to assign a fake batch to the job via the `withFakeBatch` method. The `withFakeBatch` method returns a tuple containing the job instance and the fake batch:

    [$job, $batch] = (new ShipOrder)->withFakeBatch();

    $job->handle();

    $this->assertTrue($batch->cancelled());
    $this->assertEmpty($batch->added);

<a name="job-events"></a>
## Job Events

Using the `before` and `after` methods on the queue service, you may specify callbacks to be executed before or after a queued job is processed. These callbacks are a great opportunity to perform additional logging. Typically, you should call these methods from the `boot` method of a service provider:

    <?php

    namespace App\Providers;

    use MacropaySolutions\Kernel\Support\ServiceProvider;

    class AppServiceProvider extends ServiceProvider
    {
        /**
         * Bootstrap any application services.
         */
        public function boot(): void
        {
            \app('queue')->before([\App\Listeners\QueueEventListener::class, 'onBefore']);
            \app('queue')->after([\App\Listeners\QueueEventListener::class, 'onAfter']);
        }
    }

Using the `looping` method, you may specify callbacks that execute before the worker attempts to fetch a job from a queue. For example, you might register a closure to rollback any transactions that were left open by a previously failed job:

    \app('queue')->looping(function () {
        while (\app('db')->transactionLevel() > 0) {
            \app('db')->rollBack();
        }
    });