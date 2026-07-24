---
title: Events
description: Guide to configuring, dispatching, and managing events and listeners in PHP Framework.
context: events
---

# Events

- [Introduction](#introduction)
- [Registering Events and Listeners](#registering-events-and-listeners)
    - [Generating Events and Listeners](#generating-events-and-listeners)
    - [Manually Registering Events](#manually-registering-events)
    - [Event Discovery](#event-discovery)
    - [Model Observers](#model-observers)
- [Defining Events](#defining-events)
- [Defining Listeners](#defining-listeners)
- [Queued Event Listeners](#queued-event-listeners)
    - [Queueable Array Callables (Recommended)](#queueable-array-callables-recommended)
    - [Manually Interacting With the Queue](#manually-interacting-with-the-queue)
    - [Queued Event Listeners and Database Transactions](#queued-event-listeners-and-database-transactions)
    - [Handling Failed Jobs](#handling-failed-jobs)
- [Dispatching Events](#dispatching-events)
    - [Payload Array Shapes: Class vs. String Events](#payload-array-shapes-class-vs-string-events)
    - [Dispatching Events After Database Transactions](#dispatching-events-after-database-transactions)
- [Event Subscribers](#event-subscribers)
    - [Writing Event Subscribers](#writing-event-subscribers)
    - [Registering Event Subscribers](#registering-event-subscribers)
- [Testing](#testing)
    - [Faking a Subset of Events](#faking-a-subset-of-events)
    - [Scoped Events Fakes](#scoped-event-fakes)

<a name="introduction"></a>
## Introduction

Framework's events provide a simple observer pattern implementation, allowing you to subscribe and listen for various events that occur within your application. Event classes are typically stored in the `app/Events` directory, while their listeners are stored in `app/Listeners`. Don't worry if you don't see these directories in your application as they will be created for you as you generate events and listeners using Run console commands.

Events serve as a great way to decouple various aspects of your application, since a single event can have multiple listeners that do not depend on each other. For example, you may wish to send a Slack notification to your user each time an order has shipped. Instead of coupling your order processing code to your Slack notification code, you can raise an `App\Events\OrderShipped` event which a listener can receive and use to dispatch a Slack notification.

#### Generators

The Framework includes generator commands to automatically scaffold events and listeners for you, see [Run](/run). These generated classes provide the basic structure of every event and listener.

<a name="registering-events-and-listeners"></a>
## Registering Events and Listeners

The `App\Providers\EventServiceProvider` included with your Framework application provides a convenient place to register all of your application's event listeners. The `listen` property contains an array of all events (keys) and their listeners (values). You may add as many events to this array as your application requires. For example, let's add an `OrderShipped` event:

    use App\Events\OrderShipped;
    use App\Listeners\SendShipmentNotification;

    /**
     * The event listener mappings for the application.
     *
     * @var array<class-string, array<int, class-string>>
     */
    protected $listen = [
        OrderShipped::class => [
            SendShipmentNotification::class,
        ],
    ];

> [!NOTE]  
> The `event:list` command may be used to display a list of all events and listeners registered by your application.

<a name="generating-events-and-listeners"></a>
### Generating Events and Listeners

Of course, manually creating the files for each event and listener is cumbersome. Instead, add listeners and events to your `EventServiceProvider` and use the `event:generate` Run command. This command will generate any events or listeners that are listed in your `EventServiceProvider` that do not already exist:

    php run event:generate

Alternatively, you may use the `make:event` and `make:listener` Run commands to generate individual events and listeners:

    php run make:event PodcastProcessed

    php run make:listener SendPodcastNotification --event=PodcastProcessed

<a name="manually-registering-events"></a>
### Manually Registering Events

Typically, events should be registered via the `EventServiceProvider` `$listen` array; however, you may also register class-based event listeners manually in the `boot` method of your `EventServiceProvider`:

    use App\Events\PodcastProcessed;
    use App\Listeners\SendPodcastNotification;

    /**
     * Register any other events for your application.
     */
    public function boot(): void
    {
        \app('events')->listen(
            PodcastProcessed::class,
            SendPodcastNotification::class,
        );
    }

<a name="queueable-array-callables-recommended"></a>
#### Queueable Array Callables (Recommended)

You must wrap an array callable within the `MacropaySolutions\Kernel\Events\queueableArray` function. This leverages the **Storable Array Callables** engine, completely bypassing PHP object serialization to prevent PHP Object Injection vulnerabilities.

    use App\Events\OrderCreated;
    use App\Models\Order;
    use App\Services\InventoryManager;
    use function MacropaySolutions\Kernel\Events\queueableArray;

    \app('events')->listen(
        OrderCreated::class,
        queueableArray([InventoryManager::class, 'reserveStock'])
    );

> [!NOTE]  
> **Automatic Model Serialization:** Any `Model` or `Collection` instances passed as arguments to a queued array callable are automatically serialized into lightweight database identifier arrays via `SerializesModelsHelper` and re-fetched fresh from the database on worker execution.
> [!WARNING]  
> **The Queued Event Trap (Silent Data Loss):** When firing events that trigger Queued Listeners, you **must not** pass instantiated objects (other than `Model` or `Collection`) inside an Event object payload. The queue's transport layer relies entirely on `json_encode()`. Any standard objects passed into the array callable are ignored by the model serializer and silently flattened into associative arrays, permanently destroying the instances. You must refactor your events to pass primitive data only.

<a name="wildcard-event-listeners"></a>
#### Wildcard Event Listeners

You may even register listeners using the `*` as a wildcard parameter, allowing you to catch multiple events on the same listener. Wildcard listeners receive the event name as their first argument and the entire event data array as their second argument:

    \app('events')->listen('event.*', \App\Listeners\WildcardListener::class);

<a name="event-discovery"></a>
### Event Discovery

Instead of registering events and listeners manually in the `$listen` array of the `EventServiceProvider`, you can enable automatic event discovery. When event discovery is enabled, Framework will automatically find and register your events and listeners by scanning your application's `Listeners` directory. In addition, any explicitly defined events listed in the `EventServiceProvider` will still be registered.

Framework finds event listeners by scanning the listener classes using PHP's reflection services. When Framework finds any listener class method that begins with `handle` or `__invoke`, Framework will register those methods as event listeners for the event that is type-hinted in the method's signature:

    use App\Events\PodcastProcessed;

    class SendPodcastNotification
    {
        /**
         * Handle the given event.
         */
        public function handle(PodcastProcessed $event): void
        {
            // ...
        }
    }

Event discovery is disabled by default, but you can enable it by overriding the `shouldDiscoverEvents` method of your application's `EventServiceProvider`:

    /**
     * Determine if events and listeners should be automatically discovered.
     */
    public function shouldDiscoverEvents(): bool
    {
        return true;
    }

By default, all listeners within your application's `app/Listeners` directory will be scanned. If you would like to define additional directories to scan, you may override the `discoverEventsWithin` method in your `EventServiceProvider`:

    /**
     * Get the listener directories that should be used to discover events.
     *
     * @return array<int, string>
     */
    protected function discoverEventsWithin(): array
    {
        return [
            $this->app->path('Listeners'),
        ];
    }

<a name="model-observers"></a>
#### Model Observers

    /**
     * Determine if events as observers and listeners should be automatically discovered.
     */
    public function shouldDiscoverEventsAsObservers(): bool
    {
        return true;
    }

You may place your Observer in the Observers folder and set auto-discovery to true in your `EventServiceProvider`. By default, it will scan the following directory:

    /**
     * Get the observers directories that should be used to discover events as observers.
     */
    protected function discoverEventsAsObserversWithin(): array
    {
        return [
            $this->app->path() . DIRECTORY_SEPARATOR . 'Observers',
        ];
    }

Once enabled, the Run `event:cache` command will auto-register it; there is no need to manually call the `observe` method. You can also generate observers via the CLI.

> [!NOTE]
> If `shouldDiscoverEventsAsObservers` is enabled, then the `observe` method SHOULD NOT BE CALLED manually!

<a name="event-discovery-in-production"></a>
#### Event Discovery In Production

In production, it is not efficient for the framework to scan all of your listeners on every request. Therefore, during your deployment process, you should run the `event:cache` Run command to cache a manifest of all of your application's events and listeners. This manifest will be used by the framework to speed up the event registration process. The `event:clear` command may be used to destroy the cache.

<a name="defining-events"></a>
## Defining Events

An event class is essentially a data container which holds the information related to the event. For example, let's assume an `App\Events\OrderShipped` event receives an [Obvious ORM](/obvious) object:

    <?php

    namespace App\Events;

    use App\Models\Order;
    use MacropaySolutions\Kernel\Broadcasting\InteractsWithSockets;

    class OrderShipped
    {
        use InteractsWithSockets;

        /**
         * Create a new event instance.
         */
        public function __construct(
            public Order $order,
        ) {}
    }

As you can see, this event class contains no complex logic. It is a container for the `App\Models\Order` instance.

> [!WARNING]  
> **Constructor Mapping & Virtual Property Hooks (PHP 8.4+):**
> Because queued events are reconstructed on the worker via `app($eventClass, $publicProperties)`, every public property extracted via `get_object_vars()` must match an event constructor parameter name.
> - **Virtual Properties Forbidden:** Do not define public virtual properties (e.g., `public string $summary { get => ... }`) on Event objects. Because virtual properties cannot be constructor arguments, worker rehydration will fail with an `Unknown named parameter` exception. Use helper methods instead (e.g., `public function summary(): string`).
> - **Backed Hooks & Asymmetric Visibility Allowed:** Property hooks on constructor-promoted parameters (`public string $x { set => ... }`) and asymmetric visibility (`public private(set) string $y`) are fully supported.

> [!NOTE]
> **Automatic Model & Collection Serialization:** You may freely pass `Model` and `Collection` instances inside event object properties or parameters. When an event handled by a queued listener is dispatched, `SerializesModelsHelper` automatically converts model properties into lightweight database identifier arrays before pushing the job to the queue. On the worker thread, fresh model instances are automatically re-queried from the database before being passed to your listener's `handle` method.

> [!WARNING]  
> **Raw Arrays of Models Forbidden:** Do not pass raw PHP arrays of model instances (e.g., `public array $orders = [$order1, $order2]`). Raw PHP arrays bypass `SerializesModelsHelper`, preventing model identifier conversion and failing worker-side database rehydration. Always use a `Collection` instance for multiple models (e.g., `public Collection $orders`), or pass an array of primitive IDs (e.g., `public array $orderIds = [1, 2]`).

<a name="defining-listeners"></a>
## Defining Listeners

Next, let's take a look at the listener for our example event. Event listeners receive the reconstructed event instance in their `handle` method. Both synchronous (direct) and queued listeners receive the exact same typed event class object.

Within the `handle` method, you may perform any actions necessary to respond to the event:

    <?php

    namespace App\Listeners;

    use App\Events\OrderShipped;

    class SendShipmentNotification
    {
        /**
         * Create the event listener.
         */
        public function __construct()
        {
            // ...
        }

        /**
         * Handle the event.
         */
        public function handle(OrderShipped $event): void
        {
            // Access the model directly...
            $order = $event->order;

            // Send notification...
        }
    }

> [!NOTE]  
> Your event listeners may also type-hint any dependencies they need on their constructors. All event listeners are resolved via the Framework [service container](/container), so dependencies will be injected automatically.

<a name="stopping-the-propagation-of-an-event"></a>
#### Stopping The Propagation Of An Event

Sometimes, you may wish to stop the propagation of an event to other listeners. You may do so by returning `false` from your listener's `handle` method.

<a name="queued-event-listeners"></a>
## Queued Event Listeners

> [!NOTE]  
> **Unified Behavior Across Sync & Queued Execution**
> Whether executed synchronously or processed asynchronously by a queue worker, your listener's `handle` method receives the exact same reconstructed event class instance (`OrderShipped $event`). Any `Model` or `Collection` properties are automatically re-fetched fresh from the database on the worker thread.

Queueing listeners can be beneficial if your listener is going to perform a slow task such as sending an email or making an HTTP request. Before using queued listeners, make sure to [configure your queue](/queues) and start a queue worker on your server or local development environment.

To specify that a listener should be queued, add the `ShouldQueue` interface to the listener class. Listeners generated by the `event:generate` and `make:listener` Run commands already have this interface imported into the current namespace so you can use it immediately:

    <?php

    namespace App\Listeners;

    use App\Events\OrderShipped;
    use MacropaySolutions\Kernel\Contracts\Queue\ShouldQueue;

    class SendShipmentNotification implements ShouldQueue
    {
        /**
         * Handle the queued event on the worker thread.
         */
        public function handle(OrderShipped $event): void
        {
            $order = $event->order;

            // ...
        }
    }

That's it! Now, when an event handled by this listener is dispatched, the listener will automatically be queued by the event dispatcher using Framework's [queue system](/queues). If no exceptions are thrown when the listener is executed by the queue, the queued job will automatically be deleted after it has finished processing.

<a name="customizing-the-queue-connection-queue-name"></a>
#### Customizing The Queue Connection, Name, & Delay

If you would like to customize the queue connection, queue name, or queue delay time of an event listener, you may define the `$connection`, `$queue`, or `$delay` properties on your listener class:

    <?php

    namespace App\Listeners;

    use App\Events\OrderShipped;
    use MacropaySolutions\Kernel\Contracts\Queue\ShouldQueue;

    class SendShipmentNotification implements ShouldQueue
    {
        /**
         * The name of the connection the job should be sent to.
         *
         * @var string|null
         */
        public $connection = 'sqs';

        /**
         * The name of the queue the job should be sent to.
         *
         * @var string|null
         */
        public $queue = 'listeners';

        /**
         * The time (seconds) before the job should be processed.
         *
         * @var int
         */
        public $delay = 60;
    }

If you would like to define the listener's queue connection, queue name, or delay at runtime, you may define `viaConnection`, `viaQueue`, or `withDelay` methods on the listener:

    /**
     * Get the name of the listener's queue connection.
     */
    public function viaConnection(): string
    {
        return 'sqs';
    }

    /**
     * Get the name of the listener's queue.
     */
    public function viaQueue(): string
    {
        return 'listeners';
    }

    /**
     * Get the number of seconds before the job should be processed.
     */
    public function withDelay(OrderShipped $event): int
    {
        return ($event->highPriority ?? false) ? 0 : 60;
    }

<a name="conditionally-queueing-listeners"></a>
#### Conditionally Queueing Listeners

Sometimes, you may need to determine whether a listener should be queued based on some data that are only available at runtime. To accomplish this, a `shouldQueue` method may be added to a listener to determine whether the listener should be queued. If the `shouldQueue` method returns `false`, the listener will not be executed:

    <?php

    namespace App\Listeners;

    use App\Events\OrderCreated;
    use MacropaySolutions\Kernel\Contracts\Queue\ShouldQueue;

    class RewardGiftCard implements ShouldQueue
    {
        /**
         * Reward a gift card to the customer.
         */
        public function handle(OrderCreated $event): void
        {
            // ...
        }

        /**
         * Determine whether the listener should be queued.
         */
        public function shouldQueue(OrderCreated $event): bool
        {
            return $event->order->subtotal >= 5000;
        }
    }

<a name="manually-interacting-with-the-queue"></a>
### Manually Interacting With the Queue

If you need to manually access the listener's underlying queue job's `delete` and `release` methods, you may do so using the `MacropaySolutions\Kernel\Queue\InteractsWithQueue` trait. This trait is imported by default on generated listeners and provides access to these methods:

    <?php

    namespace App\Listeners;

    use App\Events\OrderShipped;
    use MacropaySolutions\Kernel\Contracts\Queue\ShouldQueue;
    use MacropaySolutions\Kernel\Queue\InteractsWithQueue;

    class SendShipmentNotification implements ShouldQueue
    {
        use InteractsWithQueue;

        /**
         * Handle the event.
         */
        public function handle(OrderShipped $event): void
        {
            if (true) {
                $this->release(30);
            }
        }
    }

<a name="queued-event-listeners-and-database-transactions"></a>
### Queued Event Listeners and Database Transactions

When queued listeners are dispatched within database transactions, they may be processed by the queue before the database transaction has committed. When this happens, any updates you have made to models or database records during the database transaction may not yet be reflected in the database. In addition, any models or database records created within the transaction may not exist in the database. If your listener depends on these models, unexpected errors can occur when the job that dispatches the queued listener is processed.

If your queue connection's `after_commit` configuration option is set to `false`, you may still indicate that a particular queued listener should be dispatched after all open database transactions have been committed by implementing the `ShouldHandleEventsAfterCommit` interface on the listener class:

    <?php

    namespace App\Listeners;

    use MacropaySolutions\Kernel\Contracts\Events\ShouldHandleEventsAfterCommit;
    use MacropaySolutions\Kernel\Contracts\Queue\ShouldQueue;
    use MacropaySolutions\Kernel\Queue\InteractsWithQueue;

    class SendShipmentNotification implements ShouldQueue, ShouldHandleEventsAfterCommit
    {
        use InteractsWithQueue;
    }

> [!NOTE]  
> To learn more about working around these issues, please review the documentation regarding [queued jobs and database transactions](/queues#jobs-and-database-transactions).

<a name="handling-failed-jobs"></a>
### Handling Failed Jobs

Sometimes your queued event listeners may fail. If the queued listener exceeds the maximum number of attempts as defined by your queue worker, the `failed` method will be called on your listener. The `failed` method receives the reconstructed event instance and the `Throwable` that caused the failure:

    <?php

    namespace App\Listeners;

    use App\Events\OrderShipped;
    use MacropaySolutions\Kernel\Contracts\Queue\ShouldQueue;
    use MacropaySolutions\Kernel\Queue\InteractsWithQueue;
    use Throwable;

    class SendShipmentNotification implements ShouldQueue
    {
        use InteractsWithQueue;

        /**
         * Handle the event.
         */
        public function handle(OrderShipped $event): void
        {
            // ...
        }

        /**
         * Handle a job failure.
         */
        public function failed(OrderShipped $event, Throwable $exception): void
        {
            // ...
        }
    }

<a name="specifying-queued-listener-maximum-attempts"></a>
#### Specifying Queued Listener Maximum Attempts

If one of your queued listeners is encountering an error, you likely do not want it to keep retrying indefinitely. Therefore, Framework provides various ways to specify how many times or for how long a listener may be attempted.

You may define a `$tries` property on your listener class to specify how many times the listener may be attempted before it is considered to have failed:

    <?php

    namespace App\Listeners;

    use App\Events\OrderShipped;
    use MacropaySolutions\Kernel\Contracts\Queue\ShouldQueue;
    use MacropaySolutions\Kernel\Queue\InteractsWithQueue;

    class SendShipmentNotification implements ShouldQueue
    {
        use InteractsWithQueue;

        /**
         * The number of times the queued listener may be attempted.
         *
         * @var int
         */
        public $tries = 5;
    }

As an alternative to defining how many times a listener may be attempted before it fails, you may define a time at which the listener should no longer be attempted. This allows a listener to be attempted any number of times within a given time frame. To define the time at which a listener should no longer be attempted, add a `retryUntil` method to your listener class. This method should return a `DateTime` instance:

    use DateTime;

    /**
     * Determine the time at which the listener should timeout.
     */
    public function retryUntil(): DateTime
    {
        return now()->addMinutes(5);
    }

<a name="dispatching-events"></a>
## Dispatching Events

To dispatch an event, you may use the global `event` helper function or the application container to fire events throughout your application. You can dispatch events by instantiating the event object or by passing the event class string alongside its parameters:

    <?php

    namespace App\Http\Controllers;

    use App\Events\OrderShipped;
    use App\Http\Controllers\Controller;
    use App\Models\Order;
    use MacropaySolutions\Kernel\Http\RedirectResponse;
    use MacropaySolutions\Kernel\Http\Request;

    class OrderShipmentController extends Controller
    {
        /**
         * Ship the given order.
         */
        public function store(Request $request): RedirectResponse
        {
            $order = Order::query()->findOrFail($request->order_id);

            // Style 1: Passing instantiated Event Object (Recommended)
            \event(new OrderShipped($order));

            // Style 2: Passing Event Class String + Parameters
            // \event(OrderShipped::class, ['order' => $order]);

            return redirect('/orders');
        }
    }

> [!NOTE]  
> Regardless of which dispatch style you choose, the event dispatcher automatically hydrates the `OrderShipped` object so that both synchronous and queued listeners receive the exact same typed `$event` object in `handle(OrderShipped $event)`.

<a name="payload-array-shapes"></a>
### Payload Array Shapes: Class vs. String Events

When dispatching events using arrays, the shape of the array you pass depends entirely on the type of event you are dispatching:

**1. Class-Based Events (Use Associative Arrays)**
When dispatching an event by its class string, the array is passed to the IoC container to build the event object. The array **must be associative**, where the keys exactly match the variable names in the Event's `__construct()` method:

    // ✅ Correct: Container maps 'order' to __construct(Order $order)
    event(OrderShipped::class, ['order' => $order]);

    // ❌ WRONG: Container doesn't know which parameter this belongs to
    event(OrderShipped::class, [$order]);

**2. String-Based Events (Use Indexed Lists)**
When dispatching a purely string-named event, no object is constructed. The array elements are unpacked sequentially into the Listener's `handle()` method. The array **must be a positional list**:

    // ✅ Correct: Maps to handle(User $user, string $ip) in exact order
    event('user.login', [$user, '192.168.1.1']);

    // ⚠️ Works, but keys are ignored!
    // The dispatcher applies `array_values()` before unpacking. The keys are stripped,
    // meaning the array is still passed strictly by its internal order.
    \event('user.login', ['user' => $user, 'ip' => '192.168.1.1']);

<a name="dispatching-events-after-database-transactions"></a>
### Dispatching Events After Database Transactions

Sometimes, you may want to instruct Framework to only dispatch an event after the active database transaction has committed. To do so, you may implement the `ShouldDispatchAfterCommit` interface on the event class.

This interface instructs Framework to not dispatch the event until the current database transaction is committed. If the transaction fails, the event will be discarded. If no database transaction is in progress when the event is dispatched, the event will be dispatched immediately:

    <?php

    namespace App\Events;

    use App\Models\Order;
    use MacropaySolutions\Kernel\Broadcasting\InteractsWithSockets;
    use MacropaySolutions\Kernel\Contracts\Events\ShouldDispatchAfterCommit;

    class OrderShipped implements ShouldDispatchAfterCommit
    {
        use InteractsWithSockets;

        /**
         * Create a new event instance.
         */
        public function __construct(
            public Order $order,
        ) {}
    }

<a name="event-subscribers"></a>
## Event Subscribers

> [!WARNING]
> **Performance Penalty:** Avoid using Event Subscribers in this architecture because they natively bypass the framework's caching mechanisms and actively increase application boot time. They are initialized and registered dynamically on every single request. It is highly recommended to register standard class-based listeners instead, which are safely compiled into memory by the `event:cache` command.

<a name="writing-event-subscribers"></a>
### Writing Event Subscribers

Event subscribers are classes that may subscribe to multiple events from within the subscriber class itself, allowing you to define several event handlers within a single class. Subscribers should define a `subscribe` method, which will be passed an event dispatcher instance. You may call the `listen` method on the given dispatcher to register event listeners:

    <?php

    namespace App\Listeners;

    use MacropaySolutions\Kernel\Auth\Events\Login;
    use MacropaySolutions\Kernel\Auth\Events\Logout;
    use MacropaySolutions\Kernel\Events\Dispatcher;

    class UserEventSubscriber
    {
        /**
         * Handle user login events.
         */
        public function handleUserLogin(Login $event): void {}

        /**
         * Handle user logout events.
         */
        public function handleUserLogout(Logout $event): void {}

        /**
         * Register the listeners for the subscriber.
         */
        public function subscribe(Dispatcher $events): void
        {
            $events->listen(
                Login::class,
                [UserEventSubscriber::class, 'handleUserLogin']
            );

            $events->listen(
                Logout::class,
                [UserEventSubscriber::class, 'handleUserLogout']
            );
        }
    }

If your event listener methods are defined within the subscriber itself, you may find it more convenient to return an array of events and method names from the subscriber's `subscribe` method. Framework will automatically determine the subscriber's class name when registering the event listeners:

    <?php

    namespace App\Listeners;

    use MacropaySolutions\Kernel\Auth\Events\Login;
    use MacropaySolutions\Kernel\Auth\Events\Logout;
    use MacropaySolutions\Kernel\Events\Dispatcher;

    class UserEventSubscriber
    {
        /**
         * Handle user login events.
         */
        public function handleUserLogin(Login $event): void {}

        /**
         * Handle user logout events.
         */
        public function handleUserLogout(Logout $event): void {}

        /**
         * Register the listeners for the subscriber.
         *
         * @return array<string, string>
         */
        public function subscribe(Dispatcher $events): array
        {
            return [
                Login::class => 'handleUserLogin',
                Logout::class => 'handleUserLogout',
            ];
        }
    }

<a name="registering-event-subscribers"></a>
### Registering Event Subscribers

After writing the subscriber, you are ready to register it with the event dispatcher. You may register subscribers using the `$subscribe` property on the `EventServiceProvider`. For example, let's add the `UserEventSubscriber` to the list:

    <?php

    namespace App\Providers;

    use App\Listeners\UserEventSubscriber;
    use MacropaySolutions\Framework\Providers\EventServiceProvider as ServiceProvider;

    class EventServiceProvider extends ServiceProvider
    {
        /**
         * The event listener mappings for the application.
         *
         * @var array
         */
        protected $listen = [
            // ...
        ];

        /**
         * The subscriber classes to register.
         *
         * @var array
         */
        protected $subscribe = [
            UserEventSubscriber::class,
        ];
    }

<a name="testing"></a>
## Testing

When testing code that dispatches events, you may wish to instruct Framework to not actually execute the event's listeners, since the listener's code can be tested directly and separately of the code that dispatches the corresponding event. Of course, to test the listener itself, you may instantiate a listener instance and invoke the `handle` method directly in your test.

<a name="using-eventfake"></a>
### Using EventFake

You may instantiate `MacropaySolutions\KernelDev\Support\Testing\Fakes\EventFake` and swap the `events` instance in the application container:

    <?php

    namespace Tests\Feature;

    use App\Events\OrderFailedToShip;
    use App\Events\OrderShipped;
    use MacropaySolutions\Kernel\Database\Obvious\Model;
    use MacropaySolutions\KernelDev\Support\Testing\Fakes\EventFake;
    use Tests\TestCase;

    class ExampleTest extends TestCase
    {
        /**
         * Test order shipping.
         */
        public function test_orders_can_be_shipped(): void
        {
            $fake = new EventFake(\app('events'));

            \app()->instance('events', $fake);
            Model::setEventDispatcher($fake);
            \app('cache')->refreshEventDispatcher();

            // Perform order shipping...

            // Assert that an event was dispatched...
            $fake->assertDispatched(OrderShipped::class);

            // Assert an event was dispatched twice...
            $fake->assertDispatched(OrderShipped::class, 2);

            // Assert an event was not dispatched...
            $fake->assertNotDispatched(OrderFailedToShip::class);

            // Assert that no events were dispatched...
            $fake->assertNothingDispatched();
        }
    }

You may pass a truth-test callback to the `assertDispatched` or `assertNotDispatched` methods to assert that an event matching the given criteria was dispatched. Since the dispatcher normalizes events into object instances, your callback will receive the event object:

    $fake->assertDispatched(function (OrderShipped $event) use ($order) {
        return $event->order->id === $order->id;
    });

If you would simply like to assert that an event listener is registered for a given event, you may use the `assertListening` method:

    $fake->assertListening(
        OrderShipped::class,
        SendShipmentNotification::class
    );

> [!WARNING]  
> After swapping the container instance with `EventFake`, no event listeners will be executed. So, if your tests use model factories that rely on events, such as creating a UUID during a model's `creating` event, you should swap the fake **after** using your factories.

<a name="faking-a-subset-of-events"></a>
### Faking a Subset of Events

If you only want to fake event listeners for a specific set of events, you may pass an array of event classes to the `EventFake` constructor:

    use MacropaySolutions\Kernel\Database\Obvious\Model;

    /**
     * Test order process.
     */
    public function test_orders_can_be_processed(): void
    {
        $fake = new EventFake(\app('events'), [
            OrderCreated::class,
        ]);

        \app()->instance('events', $fake);
        Model::setEventDispatcher($fake);
        \app('cache')->refreshEventDispatcher();

        $order = Order::factory()->create();

        $fake->assertDispatched(OrderCreated::class);
    }

You may explicitly allow certain events to be dispatched while faking the rest by calling the `except` method:

    $fake->except([
        OrderCreated::class,
    ]);

<a name="using-testcase-testing-helpers"></a>
### Using TestCase Testing Helpers

Alternatively, if your test class extends `MacropaySolutions\KernelDev\Framework\Testing\TestCase`, you can utilize built-in helper methods such as `expectsEvents` and `withoutEvents`:

    <?php

    namespace Tests\Feature;

    use App\Events\OrderShipped;
    use Tests\TestCase;

    class ShipmentTest extends TestCase
    {
        /**
         * Test expected events.
         */
        public function test_expects_events(): void
        {
            $this->expectsEvents([
                OrderShipped::class,
            ]);

            // Perform action that fires OrderShipped...
        }

        /**
         * Test silencing all events.
         */
        public function test_silence_events(): void
        {
            $this->withoutEvents();

            // Perform action with all events silenced...
        }
    }