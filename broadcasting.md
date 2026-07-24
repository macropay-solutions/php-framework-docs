---
title: Broadcasting
description: Guide to configuring and using event broadcasting, WebSockets, presence channels, and real-time client events in PHP Framework.
context: broadcasting
---

# Broadcasting

- [Introduction](#introduction)
- [Server Side Installation](#server-side-installation)
  - [Configuration](#configuration)
  - [Pusher Channels](#pusher-channels)
  - [Open Source Alternatives](#open-source-alternatives)
- [Concept Overview](#concept-overview)
  - [Using an Example Application](#using-example-application)
- [Defining Broadcast Events](#defining-broadcast-events)
  - [Broadcast Name](#broadcast-name)
  - [Broadcast Data](#broadcast-data)
  - [Broadcast Queue](#broadcast-queue)
  - [Broadcast Conditions](#broadcast-conditions)
  - [Broadcasting and Database Transactions](#broadcasting-and-database-transactions)
- [Authorizing Channels](#authorizing-channels)
  - [Defining Authorization Routes](#defining-authorization-routes)
  - [Defining Authorization Callbacks](#defining-authorization-callbacks)
  - [Defining Channel Classes](#defining-channel-classes)
- [Broadcasting Events](#broadcasting-events)
  - [Only to Others](#only-to-others)
  - [Customizing the Connection](#customizing-the-connection)
- [Presence Channels](#presence-channels)
  - [Authorizing Presence Channels](#authorizing-presence-channels)
  - [Broadcasting to Presence Channels](#broadcasting-to-presence-channels)
- [Model Broadcasting](#model-broadcasting)
  - [Model Broadcasting Conventions](#model-broadcasting-conventions)
- [Client Events](#client-events)
- [Notifications](#notifications)

<a name="introduction"></a>
## Introduction

In many modern web applications, WebSockets are used to implement realtime, live-updating user interfaces. When some data is updated on the server, a message is typically sent over a WebSocket connection to be handled by the client. WebSockets provide a more efficient alternative to continually polling your application's server for data changes that should be reflected in your UI.

For example, imagine your application is able to export a user's data to a CSV file and email it to them. However, creating this CSV file takes several minutes so you choose to create and mail the CSV within a [queued job](/queues). When the CSV has been created and mailed to the user, we can use event broadcasting to dispatch an `App\Events\UserDataExported` event that is received by our application's JavaScript. Once the event is received, we can display a message to the user that their CSV has been emailed to them without them ever needing to refresh the page.

To assist you in building these types of features, Framework makes it easy to "broadcast" your server-side Framework [events](/events) over a WebSocket connection. Broadcasting your Framework events allows you to share the same event names and data between your server-side Framework application and your client-side JavaScript application.

The core concepts behind broadcasting are simple: clients connect to named channels on the frontend, while your Framework application broadcasts events to these channels on the backend. These events can contain any additional data you wish to make available to the frontend.

<a name="supported-drivers"></a>
#### Supported Drivers

By default, Framework includes a server-side broadcasting driver for you: [Pusher Channels](https://pusher.com/channels).

> [!NOTE]  
> Before diving into event broadcasting, make sure you have read Framework's documentation on [events and listeners](/events).

<a name="server-side-installation"></a>
## Server Side Installation

To get started using Framework's event broadcasting, we need to do some configuration within the Framework application as well as install a few packages.

Event broadcasting is accomplished by a server-side broadcasting driver that broadcasts your Framework events so that a JavaScript library can receive them within the browser client. Don't worry - we'll walk through each part of the installation process step-by-step.

<a name="configuration"></a>
### Configuration

All of your application's event broadcasting configuration is stored in the `config/broadcasting.php` configuration file. Framework supports several broadcast drivers out of the box: [Pusher Channels](https://pusher.com/channels), [Redis](/redis), and a `log` driver for local development and debugging. Additionally, a `null` driver is included which allows you to totally disable broadcasting during testing. A configuration example is included for each of these drivers in the `config/broadcasting.php` configuration file.

<a name="broadcast-service-provider"></a>
#### Broadcast Service Provider

Before broadcasting any events, you will first need to register the `App\Providers\BroadcastServiceProvider`. In new Framework applications, you only need to uncomment this provider in the `providers` array of your `config/app.php` configuration file. This `BroadcastServiceProvider` contains the code necessary to register the broadcast authorization routes and callbacks.

<a name="queue-configuration"></a>
#### Queue Configuration

You will also need to configure and run a [queue worker](/queues). All event broadcasting is done via queued jobs so that the response time of your application is not seriously affected by events being broadcast.

<a name="pusher-channels"></a>
### Pusher Channels

If you plan to broadcast your events using [Pusher Channels](https://pusher.com/channels), you should install the Pusher Channels PHP SDK using the Composer package manager:

```shell
composer require pusher/pusher-php-server
```

Next, you should configure your Pusher Channels credentials in the `config/broadcasting.php` configuration file. An example Pusher Channels configuration is already included in this file, allowing you to quickly specify your key, secret, and application ID. Typically, these values should be set via the `PUSHER_APP_KEY`, `PUSHER_APP_SECRET`, and `PUSHER_APP_ID` [environment variables](/configuration#environment-configuration):

```ini
PUSHER_APP_ID=your-pusher-app-id
PUSHER_APP_KEY=your-pusher-key
PUSHER_APP_SECRET=your-pusher-secret
PUSHER_APP_CLUSTER=mt1
```

The `config/broadcasting.php` file's `pusher` configuration also allows you to specify additional `options` that are supported by Channels, such as the cluster.

Next, you will need to change your broadcast driver to `pusher` in your `.env` file:

```ini
BROADCAST_DRIVER=pusher
```

Finally, you are ready to configure your frontend client to receive the broadcast events.

<a name="pusher-compatible-open-source-alternatives"></a>
#### Open Source Pusher Alternatives

[soketi](https://docs.soketi.app/) provides a Pusher compatible WebSocket server for Framework, allowing you to leverage the full power of Framework broadcasting without a commercial WebSocket provider. For more information on installing and using open source packages for broadcasting, please consult our documentation on [open source alternatives](#open-source-alternatives).

<a name="open-source-alternatives"></a>
### Open Source Alternatives

<a name="open-source-alternatives-node"></a>
#### Node

[Soketi](https://github.com/soketi/soketi) is a Node based, Pusher compatible WebSocket server for Framework. Under the hood, Soketi utilizes µWebSockets.js for extreme scalability and speed. This package allows you to leverage the full power of Framework broadcasting without a commercial WebSocket provider. For more information on installing and using this package, please consult its [official documentation](https://docs.soketi.app/).

<a name="concept-overview"></a>
## Concept Overview

Framework's event broadcasting allows you to broadcast your server-side Framework events to your client-side JavaScript application using a driver-based approach to WebSockets. Currently, Framework ships with the [Pusher Channels](https://pusher.com/channels) driver. The events may be easily consumed on the client-side using your preferred WebSocket client.

Events are broadcast over "channels", which may be specified as public or private. Any visitor to your application may subscribe to a public channel without any authentication or authorization; however, in order to subscribe to a private channel, a user must be authenticated and authorized to listen on that channel.

> [!NOTE]  
> If you would like to explore open source alternatives to Pusher, check out the [open source alternatives](#open-source-alternatives).

<a name="using-example-application"></a>
### Using an Example Application

Before diving into each component of event broadcasting, let's take a high level overview using an e-commerce store as an example.

In our application, let's assume we have a page that allows users to view the shipping status for their orders. Let's also assume that an `OrderShipmentStatusUpdated` event is fired when a shipping status update is processed by the application:

    use App\Events\OrderShipmentStatusUpdated;

    OrderShipmentStatusUpdated::dispatch($order);

<a name="the-shouldbroadcast-interface"></a>
#### The `ShouldBroadcast` Interface

When a user is viewing one of their orders, we don't want them to have to refresh the page to view status updates. Instead, we want to broadcast the updates to the application as they are created. So, we need to mark the `OrderShipmentStatusUpdated` event with the `ShouldBroadcast` interface. This will instruct Framework to broadcast the event when it is fired:

    <?php

    namespace App\Events;

    use App\Models\Order;
    use MacropaySolutions\Kernel\Broadcasting\Channel;
    use MacropaySolutions\Kernel\Broadcasting\InteractsWithSockets;
    use MacropaySolutions\Kernel\Broadcasting\PresenceChannel;
    use MacropaySolutions\Kernel\Contracts\Broadcasting\ShouldBroadcast;
    use MacropaySolutions\Kernel\Queue\SerializesModels;

    class OrderShipmentStatusUpdated implements ShouldBroadcast
    {
        /**
         * The order instance.
         *
         * @var \App\Models\Order
         */
        public $order;
    }

The `ShouldBroadcast` interface requires our event to define a `broadcastOn` method. This method is responsible for returning the channels that the event should broadcast on. An empty stub of this method is already defined on generated event classes, so we only need to fill in its details. We only want the creator of the order to be able to view status updates, so we will broadcast the event on a private channel that is tied to the order:

    use MacropaySolutions\Kernel\Broadcasting\Channel;
    use MacropaySolutions\Kernel\Broadcasting\PrivateChannel;

    /**
     * Get the channel the event should broadcast on.
     */
    public function broadcastOn(): Channel
    {
        return new PrivateChannel('orders.'.$this->order->id);
    }

If you wish the event to broadcast on multiple channels, you may return an `array` instead:

    use MacropaySolutions\Kernel\Broadcasting\PrivateChannel;

    /**
     * Get the channels the event should broadcast on.
     *
     * @return array<int, \MacropaySolutions\Kernel\Broadcasting\Channel>
     */
    public function broadcastOn(): array
    {
        return [
            new PrivateChannel('orders.'.$this->order->id),
            // ...
        ];
    }

<a name="example-application-authorizing-channels"></a>
#### Authorizing Channels

Remember, users must be authorized to listen on private channels. We may define our channel authorization rules in our application's `routes/channels.php` file. In this example, we need to verify that any user attempting to listen on the private `orders.1` channel is actually the creator of the order:

    use App\Models\Order;
    use App\Models\User;

    Broadcast::channel('orders.{orderId}', function (User $user, int $orderId) {
        return $user->id === Order::findOrNew($orderId)->user_id;
    });

The `channel` method accepts two arguments: the name of the channel and a callback which returns `true` or `false` indicating whether the user is authorized to listen on the channel.

All authorization callbacks receive the currently authenticated user as their first argument and any additional wildcard parameters as their subsequent arguments. In this example, we are using the `{orderId}` placeholder to indicate that the "ID" portion of the channel name is a wildcard.

<a name="listening-for-event-broadcasts"></a>
#### Listening for Event Broadcasts

Next, all that remains is to listen for the event in your JavaScript application using your preferred WebSocket client. By default, all the event's public properties will be included on the broadcast event.

<a name="defining-broadcast-events"></a>
## Defining Broadcast Events

To inform Framework that a given event should be broadcast, you must implement the `MacropaySolutions\Kernel\Contracts\Broadcasting\ShouldBroadcast` interface on the event class. This interface is already imported into all event classes generated by the framework so you may easily add it to any of your events.

The `ShouldBroadcast` interface requires you to implement a single method: `broadcastOn`. The `broadcastOn` method should return a channel or array of channels that the event should broadcast on. The channels should be instances of `Channel`, `PrivateChannel`, or `PresenceChannel`. Instances of `Channel` represent public channels that any user may subscribe to, while `PrivateChannels` and `PresenceChannels` represent private channels that require [channel authorization](#authorizing-channels):

    <?php

    namespace App\Events;

    use App\Models\User;
    use MacropaySolutions\Kernel\Broadcasting\Channel;
    use MacropaySolutions\Kernel\Broadcasting\InteractsWithSockets;
    use MacropaySolutions\Kernel\Broadcasting\PresenceChannel;
    use MacropaySolutions\Kernel\Broadcasting\PrivateChannel;
    use MacropaySolutions\Kernel\Contracts\Broadcasting\ShouldBroadcast;
    use MacropaySolutions\Kernel\Queue\SerializesModels;

    class ServerCreated implements ShouldBroadcast
    {
        use SerializesModels;

        /**
         * Create a new event instance.
         */
        public function __construct(
            public User $user,
        ) {}

        /**
         * Get the channels the event should broadcast on.
         *
         * @return array<int, \MacropaySolutions\Kernel\Broadcasting\Channel>
         */
        public function broadcastOn(): array
        {
            return [
                new PrivateChannel('user.' . $this->user->id),
            ];
        }
    }

After implementing the `ShouldBroadcast` interface, you only need to [fire the event](/events) as you normally would. Once the event has been fired, a [queued job](/queues) will automatically broadcast the event using your specified broadcast driver.

<a name="broadcast-name"></a>
### Broadcast Name

By default, Framework will broadcast the event using the event's class name. However, you may customize the broadcast name by defining a `broadcastAs` method on the event:

    /**
     * The event's broadcast name.
     */
    public function broadcastAs(): string
    {
        return 'server.created';
    }

If you customize the broadcast name using the `broadcastAs` method, you should make sure your frontend client listens for the exact event name without any assumed namespace prefixes.

<a name="broadcast-data"></a>
### Broadcast Data

When an event is broadcast, all of its `public` properties are automatically serialized and broadcast as the event's payload, allowing you to access any of its public data from your JavaScript application. So, for example, if your event has a single public `$user` property that contains an Obvious model, the event's broadcast payload would be:

```json
{
  "user": {
    "id": 1,
    "name": "Name Name"
    ...
  }
}
```

However, if you wish to have more fine-grained control over your broadcast payload, you may add a `broadcastWith` method to your event. This method should return the array of data that you wish to broadcast as the event payload:

    /**
     * Get the data to broadcast.
     *
     * @return array<string, mixed>
     */
    public function broadcastWith(): array
    {
        return ['id' => $this->user->id];
    }

<a name="broadcast-queue"></a>
### Broadcast Queue

By default, each broadcast event is placed on the default queue for the default queue connection specified in your `queue.php` configuration file. You may customize the queue connection and name used by the broadcaster by defining `connection` and `queue` properties on your event class:

    /**
     * The name of the queue connection to use when broadcasting the event.
     *
     * @var string
     */
    public $connection = 'redis';

    /**
     * The name of the queue on which to place the broadcasting job.
     *
     * @var string
     */
    public $queue = 'default';

Alternatively, you may customize the queue name by defining a `broadcastQueue` method on your event:

    /**
     * The name of the queue on which to place the broadcasting job.
     */
    public function broadcastQueue(): string
    {
        return 'default';
    }

If you would like to broadcast your event using the `sync` queue instead of the default queue driver, you can implement the `ShouldBroadcastNow` interface instead of `ShouldBroadcast`:

    <?php

    use MacropaySolutions\Kernel\Contracts\Broadcasting\ShouldBroadcastNow;

    class OrderShipmentStatusUpdated implements ShouldBroadcastNow
    {
        // ...
    }

<a name="broadcast-conditions"></a>
### Broadcast Conditions

Sometimes you want to broadcast your event only if a given condition is true. You may define these conditions by adding a `broadcastWhen` method to your event class:

    /**
     * Determine if this event should broadcast.
     */
    public function broadcastWhen(): bool
    {
        return $this->order->value > 100;
    }

<a name="broadcasting-and-database-transactions"></a>
#### Broadcasting and Database Transactions

When broadcast events are dispatched within database transactions, they may be processed by the queue before the database transaction has committed. When this happens, any updates you have made to models or database records during the database transaction may not yet be reflected in the database. In addition, any models or database records created within the transaction may not exist in the database. If your event depends on these models, unexpected errors can occur when the job that broadcasts the event is processed.

If your queue connection's `after_commit` configuration option is set to `false`, you may still indicate that a particular broadcast event should be dispatched after all open database transactions have been committed by implementing the `ShouldDispatchAfterCommit` interface on the event class:

    <?php

    namespace App\Events;

    use MacropaySolutions\Kernel\Contracts\Broadcasting\ShouldBroadcast;
    use MacropaySolutions\Kernel\Contracts\Events\ShouldDispatchAfterCommit;
    use MacropaySolutions\Kernel\Queue\SerializesModels;

    class ServerCreated implements ShouldBroadcast, ShouldDispatchAfterCommit
    {
        use SerializesModels;
    }

> [!NOTE]  
> To learn more about working around these issues, please review the documentation regarding [queued jobs and database transactions](/queues#jobs-and-database-transactions).

<a name="authorizing-channels"></a>
## Authorizing Channels

Private channels require you to authorize that the currently authenticated user can actually listen on the channel. This is accomplished by making an HTTP request to your Framework application with the channel name and allowing your application to determine if the user can listen on that channel. When connecting to private channels, your frontend client must make an HTTP request to authorize the subscription. You will need to define the proper routes to respond to these requests.

<a name="defining-authorization-routes"></a>
### Defining Authorization Routes

Thankfully, Framework makes it easy to define the routes to respond to channel authorization requests. In the `App\Providers\BroadcastServiceProvider` included with your Framework application, you will see a call to the `Broadcast::routes` method. This method will register the `/broadcasting/auth` route to handle authorization requests:

    Broadcast::routes();

The `Broadcast::routes` method will automatically place its routes within the `web` middleware group; however, you may pass an array of route attributes to the method if you would like to customize the assigned attributes:

    Broadcast::routes($attributes);

<a name="defining-authorization-callbacks"></a>
### Defining Authorization Callbacks

Next, we need to define the logic that will actually determine if the currently authenticated user can listen to a given channel. This is done in the `routes/channels.php` file that is included with your application. In this file, you may use the `Broadcast::channel` method to register channel authorization callbacks:

    use App\Models\User;

    Broadcast::channel('orders.{orderId}', function (User $user, int $orderId) {
        return $user->id === Order::findOrNew($orderId)->user_id;
    });

The `channel` method accepts two arguments: the name of the channel and a callback which returns `true` or `false` indicating whether the user is authorized to listen on the channel.

All authorization callbacks receive the currently authenticated user as their first argument and any additional wildcard parameters as their subsequent arguments. In this example, we are using the `{orderId}` placeholder to indicate that the "ID" portion of the channel name is a wildcard.

You may view a list of your application's broadcast authorization callbacks using the `channel:list` Run command:

```shell
php run channel:list
```

<a name="authorization-callback-model-binding"></a>
#### Authorization Callback Model Binding

Just like HTTP routes, channel routes may also take advantage of implicit and explicit [route model binding](/routing#route-model-binding). For example, instead of receiving a string or numeric order ID, you may request an actual `Order` model instance:

    use App\Models\Order;
    use App\Models\User;

    Broadcast::channel('orders.{order}', function (User $user, Order $order) {
        return $user->id === $order->user_id;
    });

> [!WARNING]  
> Unlike HTTP route model binding, channel model binding does not support automatic [implicit model binding scoping](/routing#implicit-model-binding-scoping). However, this is rarely a problem because most channels can be scoped based on a single model's unique, primary key.

<a name="authorization-callback-authentication"></a>
#### Authorization Callback Authentication

Private and presence broadcast channels authenticate the current user via your application's default authentication guard. If the user is not authenticated, channel authorization is automatically denied and the authorization callback is never executed. However, you may assign multiple, custom guards that should authenticate the incoming request if necessary:

    Broadcast::channel('channel', function () {
        // ...
    }, ['guards' => ['web', 'admin']]);

<a name="defining-channel-classes"></a>
### Defining Channel Classes

If your application is consuming many different channels, your `routes/channels.php` file could become bulky. So, instead of using closures to authorize channels, you may use channel classes. To generate a channel class, use the `make:channel` Run command. This command will place a new channel class in the `App/Broadcasting` directory.

```shell
php run make:channel OrderChannel
```

Next, register your channel in your `routes/channels.php` file:

    use App\Broadcasting\OrderChannel;

    Broadcast::channel('orders.{order}', OrderChannel::class);

Finally, you may place the authorization logic for your channel in the channel class' `join` method. This `join` method will house the same logic you would have typically placed in your channel authorization closure. You may also take advantage of channel model binding:

    <?php

    namespace App\Broadcasting;

    use App\Models\Order;
    use App\Models\User;

    class OrderChannel
    {
        /**
         * Create a new channel instance.
         */
        public function __construct()
        {
            // ...
        }

        /**
         * Authenticate the user's access to the channel.
         */
        public function join(User $user, Order $order): array|bool
        {
            return $user->id === $order->user_id;
        }
    }

> [!NOTE]  
> Like many other classes in Framework, channel classes will automatically be resolved by the [service container](/container). So, you may type-hint any dependencies required by your channel in its constructor.

<a name="broadcasting-events"></a>
## Broadcasting Events

Once you have defined an event and marked it with the `ShouldBroadcast` interface, you only need to fire the event using the event's dispatch method. The event dispatcher will notice that the event is marked with the `ShouldBroadcast` interface and will queue the event for broadcasting:

    use App\Events\OrderShipmentStatusUpdated;

    OrderShipmentStatusUpdated::dispatch($order);

<a name="only-to-others"></a>
### Only to Others

When building an application that utilizes event broadcasting, you may occasionally need to broadcast an event to all subscribers to a given channel except for the current user. You may accomplish this using the `broadcast` helper and the `toOthers` method:

    use App\Events\OrderShipmentStatusUpdated;

    broadcast(new OrderShipmentStatusUpdated($update))->toOthers();

To better understand when you may want to use the `toOthers` method, let's imagine a task list application where a user may create a new task by entering a task name. To create a task, your application might make a request to a `/task` URL which broadcasts the task's creation and returns a JSON representation of the new task. When your JavaScript application receives the response from the end-point, it might directly insert the new task into its task list like so:

```js
axios.post('/task', task)
        .then((response) => {
          this.tasks.push(response.data);
        });
```

However, remember that we also broadcast the task's creation. If your JavaScript application is also listening for this event in order to add tasks to the task list, you will have duplicate tasks in your list: one from the end-point and one from the broadcast. You may solve this by using the `toOthers` method to instruct the broadcaster to not broadcast the event to the current user.

> [!WARNING]  
> Your event must use the `MacropaySolutions\Kernel\Broadcasting\InteractsWithSockets` trait in order to call the `toOthers` method.

<a name="only-to-others-configuration"></a>
#### Configuration

If you are using a global HTTP client like [Axios](https://github.com/mzabriskie/axios) in your JavaScript application, you will need to manually configure it to send the `X-Socket-ID` header with all outgoing requests. When you call the `toOthers` method, Framework will extract the socket ID from the header and instruct the broadcaster to not broadcast to any connections with that socket ID.

<a name="customizing-the-connection"></a>
### Customizing the Connection

If your application interacts with multiple broadcast connections and you want to broadcast an event using a broadcaster other than your default, you may specify which connection to push an event to using the `via` method:

    use App\Events\OrderShipmentStatusUpdated;

    broadcast(new OrderShipmentStatusUpdated($update))->via('pusher');

Alternatively, you may specify the event's broadcast connection by calling the `broadcastVia` method within the event's constructor. However, before doing so, you should ensure that the event class uses the `InteractsWithBroadcasting` trait:

    <?php

    namespace App\Events;

    use MacropaySolutions\Kernel\Broadcasting\Channel;
    use MacropaySolutions\Kernel\Broadcasting\InteractsWithBroadcasting;
    use MacropaySolutions\Kernel\Broadcasting\InteractsWithSockets;
    use MacropaySolutions\Kernel\Broadcasting\PresenceChannel;
    use MacropaySolutions\Kernel\Broadcasting\PrivateChannel;
    use MacropaySolutions\Kernel\Contracts\Broadcasting\ShouldBroadcast;
    use MacropaySolutions\Kernel\Queue\SerializesModels;

    class OrderShipmentStatusUpdated implements ShouldBroadcast
    {
        use InteractsWithBroadcasting;

        /**
         * Create a new event instance.
         */
        public function __construct()
        {
            $this->broadcastVia('pusher');
        }
    }

<a name="presence-channels"></a>
## Presence Channels

Presence channels build on the security of private channels while exposing the additional feature of awareness of who is subscribed to the channel. This makes it easy to build powerful, collaborative application features such as notifying users when another user is viewing the same page or listing the inhabitants of a chat room.

<a name="authorizing-presence-channels"></a>
### Authorizing Presence Channels

All presence channels are also private channels; therefore, users must be [authorized to access them](#authorizing-channels). However, when defining authorization callbacks for presence channels, you will not return `true` if the user is authorized to join the channel. Instead, you should return an array of data about the user.

The data returned by the authorization callback will be made available to the presence channel event listeners in your JavaScript application. If the user is not authorized to join the presence channel, you should return `false` or `null`:

    use App\Models\User;

    Broadcast::channel('chat.{roomId}', function (User $user, int $roomId) {
        if ($user->canJoinRoom($roomId)) {
            return ['id' => $user->id, 'name' => $user->name];
        }
    });

<a name="broadcasting-to-presence-channels"></a>
### Broadcasting to Presence Channels

Presence channels may receive events just like public or private channels. Using the example of a chatroom, we may want to broadcast `NewMessage` events to the room's presence channel. To do so, we'll return an instance of `PresenceChannel` from the event's `broadcastOn` method:

    /**
     * Get the channels the event should broadcast on.
     *
     * @return array<int, \MacropaySolutions\Kernel\Broadcasting\Channel>
     */
    public function broadcastOn(): array
    {
        return [
            new PresenceChannel('chat.'.$this->message->room_id),
        ];
    }

As with other events, you may use the `broadcast` helper and the `toOthers` method to exclude the current user from receiving the broadcast:

    broadcast(new NewMessage($message));

    broadcast(new NewMessage($message))->toOthers();

<a name="model-broadcasting"></a>
## Model Broadcasting

> [!WARNING]  
> Before reading the following documentation about model broadcasting, we recommend you become familiar with the general concepts of Framework's model broadcasting services as well as how to manually create and listen to broadcast events.

It is common to broadcast events when your application's [Obvious models](/obvious) are created, updated, or deleted. Of course, this can easily be accomplished by manually [defining custom events for Obvious model state changes](/obvious#events) and marking those events with the `ShouldBroadcast` interface.

However, if you are not using these events for any other purposes in your application, it can be cumbersome to create event classes for the sole purpose of broadcasting them. To remedy this, Framework allows you to indicate that an Obvious model should automatically broadcast its state changes.

To get started, your Obvious model should use the `MacropaySolutions\Kernel\Database\Obvious\BroadcastsEvents` trait. In addition, the model should define a `broadcastOn` method, which will return an array of channels that the model's events should broadcast on:

```php
<?php

namespace App\Models;

use MacropaySolutions\Kernel\Broadcasting\Channel;
use MacropaySolutions\Kernel\Broadcasting\PrivateChannel;
use MacropaySolutions\Kernel\Database\Obvious\BroadcastsEvents;
use MacropaySolutions\Kernel\Database\Obvious\Factories\HasFactory;
use MacropaySolutions\Kernel\Database\Obvious\Model;
use MacropaySolutions\Kernel\Database\Obvious\Relations\BelongsTo;

class Post extends Model
{
    use BroadcastsEvents, HasFactory;

    /**
     * Get the user that the post belongs to.
     */
    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class);
    }

    /**
     * Get the channels that model events should broadcast on.
     *
     * @return array<int, \MacropaySolutions\Kernel\Broadcasting\Channel|\MacropaySolutions\Kernel\Database\Obvious\Model>
     */
    public function broadcastOn(string $event): array
    {
        return [$this, $this->user];
    }
}
```

Once your model includes this trait and defines its broadcast channels, it will begin automatically broadcasting events when a model instance is created, updated, deleted, trashed, or restored.

In addition, you may have noticed that the `broadcastOn` method receives a string `$event` argument. This argument contains the type of event that has occurred on the model and will have a value of `created`, `updated`, `deleted`, `trashed`, or `restored`. By inspecting the value of this variable, you may determine which channels (if any) the model should broadcast to for a particular event:

```php
/**
 * Get the channels that model events should broadcast on.
 *
 * @return array<string, array<int, \MacropaySolutions\Kernel\Broadcasting\Channel|\MacropaySolutions\Kernel\Database\Obvious\Model>>
 */
public function broadcastOn(string $event): array
{
    return match ($event) {
        'deleted' => [],
        default => [$this, $this->user],
    };
}
```

<a name="customizing-model-broadcasting-event-creation"></a>
#### Customizing Model Broadcasting Event Creation

Occasionally, you may wish to customize how Framework creates the underlying model broadcasting event. You may accomplish this by defining a `newBroadcastableEvent` method on your Obvious model. This method should return an `MacropaySolutions\Kernel\Database\Obvious\BroadcastableModelEventOccurred` instance:

```php
use MacropaySolutions\Kernel\Database\Obvious\BroadcastableModelEventOccurred;

/**
 * Create a new broadcastable model event for the model.
 */
protected function newBroadcastableEvent(string $event): BroadcastableModelEventOccurred
{
    return (new BroadcastableModelEventOccurred(
        $this, $event
    ))->dontBroadcastToCurrentUser();
}
```

<a name="model-broadcasting-conventions"></a>
### Model Broadcasting Conventions

<a name="model-broadcasting-channel-conventions"></a>
#### Channel Conventions

As you may have noticed, the `broadcastOn` method in the model example above did not return `Channel` instances. Instead, Obvious models were returned directly. If an Obvious model instance is returned by your model's `broadcastOn` method (or is contained in an array returned by the method), Framework will automatically instantiate a private channel instance for the model using the model's class name and primary key identifier as the channel name.

So, an `App\Models\User` model with an `id` of `1` would be converted into an `MacropaySolutions\Kernel\Broadcasting\PrivateChannel` instance with a name of `App.Models.User.1`. Of course, in addition to returning Obvious model instances from your model's `broadcastOn` method, you may return complete `Channel` instances in order to have full control over the model's channel names:

```php
use MacropaySolutions\Kernel\Broadcasting\PrivateChannel;

/**
 * Get the channels that model events should broadcast on.
 *
 * @return array<int, \MacropaySolutions\Kernel\Broadcasting\Channel>
 */
public function broadcastOn(string $event): array
{
    return [
        new PrivateChannel('user.'.$this->id)
    ];
}
```

If you plan to explicitly return a channel instance from your model's `broadcastOn` method, you may pass an Obvious model instance to the channel's constructor. When doing so, Framework will use the model channel conventions discussed above to convert the Obvious model into a channel name string:

```php
return [new Channel($this->user)];
```

If you need to determine the channel name of a model, you may call the `broadcastChannel` method on any model instance. For example, this method returns the string `App.Models.User.1` for an `App\Models\User` model with an `id` of `1`:

```php
$user->broadcastChannel()
```

<a name="model-broadcasting-event-conventions"></a>
#### Event Conventions

Since model broadcast events are not associated with an "actual" event within your application's `App\Events` directory, they are assigned a name and a payload based on conventions. Framework's convention is to broadcast the event using the class name of the model (not including the namespace) and the name of the model event that triggered the broadcast.

So, for example, an update to the `App\Models\Post` model would broadcast an event to your client-side application as `PostUpdated` with the following payload:

```json
{
  "model": {
    "id": 1,
    "title": "My first post"
    ...
  },
  ...
  "socket": "someSocketId",
}
```

The deletion of the `App\Models\User` model would broadcast an event named `UserDeleted`.

If you would like, you may define a custom broadcast name and payload by adding a `broadcastAs` and `broadcastWith` method to your model. These methods receive the name of the model event / operation that is occurring, allowing you to customize the event's name and payload for each model operation. If `null` is returned from the `broadcastAs` method, Framework will use the model broadcasting event name conventions discussed above when broadcasting the event:

```php
/**
 * The model event's broadcast name.
 */
public function broadcastAs(string $event): string|null
{
    return match ($event) {
        'created' => 'post.created',
        default => null,
    };
}

/**
 * Get the data to broadcast for the model.
 *
 * @return array<string, mixed>
 */
public function broadcastWith(string $event): array
{
    return match ($event) {
        'created' => ['title' => $this->title],
        default => ['model' => $this],
    };
}
```

<a name="client-events"></a>
## Client Events

> [!NOTE]  
> When using [Pusher Channels](https://pusher.com/channels), you must enable the "Client Events" option in the "App Settings" section of your [application dashboard](https://dashboard.pusher.com/) in order to send client events.

Sometimes you may wish to broadcast an event to other connected clients without hitting your Framework application at all. This can be particularly useful for things like "typing" notifications, where you want to alert users of your application that another user is typing a message on a given screen.

To broadcast and listen for client events, you will need to use your chosen WebSocket client's native SDK (such as the Pusher JS library).

<a name="notifications"></a>
## Notifications

By pairing event broadcasting with [notifications](/notifications), your JavaScript application may receive new notifications as they occur without needing to refresh the page. Before getting started, be sure to read over the documentation on using [the broadcast notification channel](/notifications#broadcast-notifications).

Once you have configured a notification to use the broadcast channel, you may listen for the broadcast events using your frontend client. Remember, the channel name should match the class name of the entity receiving the notifications. A channel authorization callback for the `App.Models.User.{id}` channel is included in the default `BroadcastServiceProvider` that ships with the Framework framework.
