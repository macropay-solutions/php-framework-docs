# Notifications

- [Introduction](#introduction)
- [Generating Notifications](#generating-notifications)
- [Sending Notifications](#sending-notifications)
  - [Using the Notifiable Trait](#using-the-notifiable-trait)
  - [Using the ChannelManager](#using-the-channelmanager)
  - [Specifying Delivery Channels](#specifying-delivery-channels)
  - [Queueing Notifications](#queueing-notifications)
  - [On-Demand Notifications](#on-demand-notifications)
- [Mail Notifications](#mail-notifications)
  - [Formatting Mail Messages](#formatting-mail-messages)
  - [Customizing the Sender](#customizing-the-sender)
  - [Customizing the Recipient](#customizing-the-recipient)
  - [Customizing the Subject](#customizing-the-subject)
  - [Customizing the Mailer](#customizing-the-mailer)
  - [Customizing the Templates](#customizing-the-templates)
  - [Attachments](#mail-attachments)
  - [Adding Tags and Metadata](#adding-tags-metadata)
  - [Customizing the Symfony Message](#customizing-the-symfony-message)
  - [Using Mailables](#using-mailables)
  - [Previewing Mail Notifications](#previewing-mail-notifications)
- [Markdown Mail Notifications](#markdown-mail-notifications)
  - [Generating the Message](#generating-the-message)
  - [Writing the Message](#writing-the-message)
  - [Customizing the Components](#customizing-the-components)
- [Database Notifications](#database-notifications)
  - [Prerequisites](#database-prerequisites)
  - [Formatting Database Notifications](#formatting-database-notifications)
  - [Accessing the Notifications](#accessing-the-notifications)
  - [Marking Notifications as Read](#marking-notifications-as-read)
- [Broadcast Notifications](#broadcast-notifications)
  - [Prerequisites](#broadcast-prerequisites)
  - [Formatting Broadcast Notifications](#formatting-broadcast-notifications)
  - [Listening for Notifications](#listening-for-notifications)
- [Localizing Notifications](#localizing-notifications)
- [Testing](#testing)
- [Notification Events](#notification-events)
- [Custom Channels](#custom-channels)

<a name="introduction"></a>
## Introduction

Typically, notifications should be short, informational messages that notify users of something that occurred in your application. For example, if you are writing a billing application, you might send an "Invoice Paid" notification to your users via the email and SMS channels.

<a name="generating-notifications"></a>
## Generating Notifications

In Framework, each notification is represented by a single class that is typically stored in the `app/Notifications` directory. Don't worry if you don't see this directory in your application - it will be created for you when you run the `make:notification` Run command:

```shell
php run make:notification InvoicePaid
```

This command will place a fresh notification class in your `app/Notifications` directory. Each notification class contains a `via` method and a variable number of message building methods, such as `toMail` or `toDatabase`, that convert the notification to a message tailored for that particular channel.

<a name="sending-notifications"></a>
## Sending Notifications

<a name="using-the-notifiable-trait"></a>
### Using the Notifiable Trait

Notifications may be sent in two ways: using the `notify` method of the `Notifiable` trait. The `Notifiable` trait is included on your application's `App\Models\User` model by default:

    <?php

    namespace App\Models;

    use MacropaySolutions\Kernel\Database\Obvious\Model;
    use MacropaySolutions\Kernel\Notifications\Notifiable;

    class User extends Model
    {
        use Notifiable;
    }

The `notify` method that is provided by this trait expects to receive a notification instance:

    use App\Notifications\InvoicePaid;

    $user->notify(new InvoicePaid($invoice));

> [!NOTE]  
> Remember, you may use the `Notifiable` trait on any of your models. You are not limited to only including it on your `User` model.

<a name="using-the-channelmanager"></a>
### Using the ChannelManager

Alternatively, you may send notifications directly via the container's `ChannelManager`. This approach is useful when you need to send a notification to multiple notifiable entities such as a collection of users. To send notifications using the manager, pass all the notifiable entities and the notification instance to the `send` method:

    use MacropaySolutions\Kernel\Notifications\ChannelManager;

    \app(ChannelManager::class)->send($users, new InvoicePaid($invoice));

You can also send notifications immediately using the `sendNow` method. This method will send the notification immediately even if the notification implements the `ShouldQueue` interface:

    \app(ChannelManager::class)->sendNow($developers, new DeploymentCompleted($deployment));

<a name="specifying-delivery-channels"></a>
### Specifying Delivery Channels

Every notification class has a `via` method that determines on which channels the notification will be delivered. Notifications may be sent on the `mail`, `database`, `broadcast`, `vonage`, and `slack` channels.


The `via` method receives a `$notifiable` instance, which will be an instance of the class to which the notification is being sent. You may use `$notifiable` to determine which channels the notification should be delivered on:

    /**
     * Get the notification's delivery channels.
     *
     * @return array<int, string>
     */
    public function via(object $notifiable): array
    {
        return $notifiable->prefers_sms ? ['vonage'] : ['mail', 'database'];
    }

<a name="queueing-notifications"></a>
### Queueing Notifications

> [!WARNING]  
> Before queueing notifications you should configure your queue and [start a worker](/queues#running-the-queue-worker).

Sending notifications can take time, especially if the channel needs to make an external API call to deliver the notification. To speed up your application's response time, let your notification be queued by adding the `ShouldQueue` interface and `Queueable` trait to your class. The interface and trait are already imported for all notifications generated using the `make:notification` command, so you may immediately add them to your notification class:

    <?php

    namespace App\Notifications;

    use MacropaySolutions\Kernel\Bus\Queueable;
    use MacropaySolutions\Kernel\Contracts\Queue\ShouldQueue;
    use MacropaySolutions\Kernel\Notifications\Notification;

    class InvoicePaid extends Notification implements ShouldQueue
    {
        use Queueable;

        // ...
    }

Once the `ShouldQueue` interface has been added to your notification, you may send the notification like normal. Framework will detect the `ShouldQueue` interface on the class and automatically queue the delivery of the notification:

    $user->notify(new InvoicePaid($invoice));

When queueing notifications, a queued job will be created for each recipient and channel combination. For example, six jobs will be dispatched to the queue if your notification has three recipients and two channels.

>[!NOTE]
> Avoid piling up email addresses when sending emails in a loop via queue with the same mailable:

    $maillable = (new ChildMailable())->onQueue('emails');

    foreach ([...] as $email) {
        Mail::to($email)->queue($maillable);
        $message->to = []; // without this the emails will pile up in the $maillable object
    }

or

    foreach ([...] as $email) {
        Mail::to($email)->queue((new ChildMailable())->onQueue('emails'));
    }

<a name="model-serialization-and-rehydration"></a>
#### Automatic Model Serialization & Rehydration

When passing Obvious ORM Models (`QueueableEntity`) or Collections (`QueueableCollection`) as public properties on your Notification, the framework **does not** serialize the entire object or its loaded relationships. It safely converts it into a lightweight `ModelIdentifier` and automatically rehydrates a completely fresh instance of the model on the worker. This guarantees your background workers always operate on the most up-to-date database state.

> [!CAUTION]  
> **The Protected / Private Property Loss Trap:** The queue transport layer relies entirely on `get_object_vars()` and `json_encode()` to flatten queued Notifications into 100% object-free JSON payloads. As a result, **all `protected` and `private` instance properties are permanently stripped during serialization**.
>
> **The Recommended Fix:** To pass data securely, assign it strictly to `public` properties, or bypass object dispatch entirely by pushing a Storable Array Callable directly to the queue.

> [!WARNING]  
> **The Primitive Property Rule (Silent Data Loss):** Because the transport layer uses `json_encode()`, passing an Object (that is not a `QueueableCollection` or `QueueableEntity`) as a public property to your Notification will **not** throw an exception on dispatch. Instead, it will be silently flattened into JSON. When the worker receives it, it will be decoded as a plain PHP associative array. If your Notification methods expect an actual instance, the worker will crash. You **must** pass primitive data (like an `$invoiceId`) and fetch the records inside your message building methods (`toMail`, `toDatabase`, etc.). Composite primary columns are not supported out of the box, For them, you must manually pass primitives and query the model in the job.
>
> **The Constructor Rule:** Because the worker rebuilds your Notification via the Dependency Injection container (`\app($class)`) before hydrating its public properties, **the constructor must be resolvable by the container**. Any stateful arguments (like `$invoiceId` or `$user`) must be optional (`= null`), or you will trigger an `ArgumentCountError` on the worker. To avoid reflection you can add these classes in your `app.autowiring` config.
>
> **The "Dumb" Constructor Caveat:** Because the worker passes `null` to build the initial shell, your constructor must *only* be used for property assignment. Do not execute database queries or business logic on the passed arguments inside the constructor, as it will crash the worker.
>
> **❌ BAD (Crashes on Worker):** `$this->amount = $invoice->calculateTotal();` // Crashes because $invoice is null on the worker!
> **✅ GOOD (Safe):** Fetch the amounts and process logic inside the `toMail()` or `toDatabase()` methods. The framework will have fully hydrated the real `$invoice` property by the time those methods execute.

<a name="delaying-notifications"></a>
#### Delaying Notifications

If you would like to delay the delivery of the notification, you may chain the `delay` method onto your notification instantiation:

    $delay = now()->addMinutes(10);

    $user->notify((new InvoicePaid($invoice))->delay($delay));

<a name="delaying-notifications-per-channel"></a>
#### Delaying Notifications per Channel

You may pass an array to the `delay` method to specify the delay amount for specific channels:

    $user->notify((new InvoicePaid($invoice))->delay([
        'mail' => now()->addMinutes(5),
        'sms' => now()->addMinutes(10),
    ]));

Alternatively, you may define a `withDelay` method on the notification class itself. The `withDelay` method should return an array of channel names and delay values:

    /**
     * Determine the notification's delivery delay.
     *
     * @return array<string, \MacropaySolutions\Kernel\Support\Carbon>
     */
    public function withDelay(object $notifiable): array
    {
        return [
            'mail' => now()->addMinutes(5),
            'sms' => now()->addMinutes(10),
        ];
    }

<a name="customizing-the-notification-queue-connection"></a>
#### Customizing the Notification Queue Connection

By default, queued notifications will be queued using your application's default queue connection. If you would like to specify a different connection that should be used for a particular notification, you may call the `onConnection` method from your notification's constructor:

    <?php

    namespace App\Notifications;

    use MacropaySolutions\Kernel\Bus\Queueable;
    use MacropaySolutions\Kernel\Contracts\Queue\ShouldQueue;
    use MacropaySolutions\Kernel\Notifications\Notification;

    class InvoicePaid extends Notification implements ShouldQueue
    {
        use Queueable;

        /**
         * Create a new notification instance.
         */
        public function __construct()
        {
            $this->onConnection('redis');
        }
    }

Or, if you would like to specify a specific queue connection that should be used for each notification channel supported by the notification, you may define a `viaConnections` method on your notification. This method should return an array of channel name / queue connection name pairs:

    /**
     * Determine which connections should be used for each notification channel.
     *
     * @return array<string, string>
     */
    public function viaConnections(): array
    {
        return [
            'mail' => 'redis',
            'database' => 'sync',
        ];
    }

<a name="customizing-notification-channel-queues"></a>
#### Customizing Notification Channel Queues

If you would like to specify a specific queue that should be used for each notification channel supported by the notification, you may define a `viaQueues` method on your notification. This method should return an array of channel name / queue name pairs:

    /**
     * Determine which queues should be used for each notification channel.
     *
     * @return array<string, string>
     */
    public function viaQueues(): array
    {
        return [
            'mail' => 'mail-queue',
            'slack' => 'slack-queue',
        ];
    }

<a name="queued-notifications-and-database-transactions"></a>
#### Queued Notifications and Database Transactions

When queued notifications are dispatched within database transactions, they may be processed by the queue before the database transaction has committed. When this happens, any updates you have made to models or database records during the database transaction may not yet be reflected in the database. In addition, any models or database records created within the transaction may not exist in the database. If your notification depends on these models, unexpected errors can occur when the job that sends the queued notification is processed.

If your queue connection's `after_commit` configuration option is set to `false`, you may still indicate that a particular queued notification should be dispatched after all open database transactions have been committed by calling the `afterCommit` method when sending the notification:

    use App\Notifications\InvoicePaid;

    $user->notify((new InvoicePaid($invoice))->afterCommit());

Alternatively, you may call the `afterCommit` method from your notification's constructor:

    <?php

    namespace App\Notifications;

    use MacropaySolutions\Kernel\Bus\Queueable;
    use MacropaySolutions\Kernel\Contracts\Queue\ShouldQueue;
    use MacropaySolutions\Kernel\Notifications\Notification;

    class InvoicePaid extends Notification implements ShouldQueue
    {
        use Queueable;

        /**
         * Create a new notification instance.
         */
        public function __construct()
        {
            $this->afterCommit();
        }
    }

> [!NOTE]  
> To learn more about working around these issues, please review the documentation regarding [queued jobs and database transactions](/queues#jobs-and-database-transactions).

<a name="determining-if-the-queued-notification-should-be-sent"></a>
#### Determining if a Queued Notification Should Be Sent

After a queued notification has been dispatched for the queue for background processing, it will typically be accepted by a queue worker and sent to its intended recipient.

However, if you would like to make the final determination on whether the queued notification should be sent after it is being processed by a queue worker, you may define a `shouldSend` method on the notification class. If this method returns `false`, the notification will not be sent:

    /**
     * Determine if the notification should be sent.
     */
    public function shouldSend(object $notifiable, string $channel): bool
    {
        return $this->invoice->isPaid();
    }

<a name="on-demand-notifications"></a>
### On-Demand Notifications

Sometimes you may need to send a notification to someone who is not stored as a "user" of your application. To do this, instantiate an `AnonymousNotifiable` and use its `route` method to specify ad-hoc notification routing information before sending the notification:

    use MacropaySolutions\Kernel\Broadcasting\Channel;
    use MacropaySolutions\Kernel\Notifications\AnonymousNotifiable;

    (new AnonymousNotifiable())
        ->route('mail', 'surname@example.com')
        ->route('vonage', '5555555555')
        ->route('slack', '#slack-channel')
        ->route('broadcast', [new Channel('channel-name')])
        ->notify(new InvoicePaid($invoice));

If you would like to provide the recipient's name when sending an on-demand notification to the `mail` route, you may provide an array that contains the email address as the key and the name as the value of the first element in the array:

    (new AnonymousNotifiable())->route('mail', [
        'barrett@example.com' => 'Barrett Blair',
    ])->notify(new InvoicePaid($invoice));

Using the `routes` method, you may provide ad-hoc routing information for multiple notification channels at once:

    (new AnonymousNotifiable())->routes([
        'mail' => ['barrett@example.com' => 'Barrett Blair'],
        'vonage' => '5555555555',
    ])->notify(new InvoicePaid($invoice));

<a name="mail-notifications"></a>
## Mail Notifications

<a name="formatting-mail-messages"></a>
### Formatting Mail Messages

If a notification supports being sent as an email, you should define a `toMail` method on the notification class. This method will receive a `$notifiable` entity and should return an `MacropaySolutions\Kernel\Notifications\Messages\MailMessage` instance.

The `MailMessage` class contains a few simple methods to help you build transactional email messages. Mail messages may contain lines of text as well as a "call to action". Let's take a look at an example `toMail` method:

    /**
     * Get the mail representation of the notification.
     */
    public function toMail(object $notifiable): MailMessage
    {
        $url = url('/invoice/'.$this->invoice->id);

        return (new MailMessage)
                    ->greeting('Hello!')
                    ->line('One of your invoices has been paid!')
                    ->lineIf($this->amount > 0, "Amount paid: {$this->amount}")
                    ->action('View Invoice', $url)
                    ->line('Thank you for using our application!');
    }

> [!NOTE]  
> Note we are using `$this->invoice->id` in our `toMail` method. You may pass any data your notification needs to generate its message into the notification's constructor.

In this example, we register a greeting, a line of text, a call to action, and then another line of text. These methods provided by the `MailMessage` object make it simple and fast to format small transactional emails. The mail channel will then translate the message components into a beautiful, responsive HTML email template with a plain-text counterpart.

> [!NOTE]  
> When sending mail notifications, be sure to set the `name` configuration option in your `config/app.php` configuration file. This value will be used in the header and footer of your mail notification messages.

<a name="error-messages"></a>
#### Error Messages

Some notifications inform users of errors, such as a failed invoice payment. You may indicate that a mail message is regarding an error by calling the `error` method when building your message. When using the `error` method on a mail message, the call to action button will be red instead of black:

    /**
     * Get the mail representation of the notification.
     */
    public function toMail(object $notifiable): MailMessage
    {
        return (new MailMessage)
                    ->error()
                    ->subject('Invoice Payment Failed')
                    ->line('...');
    }

<a name="other-mail-notification-formatting-options"></a>
#### Other Mail Notification Formatting Options

Instead of defining the "lines" of text in the notification class, you may use the `view` method to specify a custom template that should be used to render the notification email:

    /**
     * Get the mail representation of the notification.
     */
    public function toMail(object $notifiable): MailMessage
    {
        return (new MailMessage)->view(
            'mail.invoice.paid', ['invoice' => $this->invoice]
        );
    }

You may specify a plain-text view for the mail message by passing the view name as the second element of an array that is given to the `view` method:

    /**
     * Get the mail representation of the notification.
     */
    public function toMail(object $notifiable): MailMessage
    {
        return (new MailMessage)->view(
            ['mail.invoice.paid', 'mail.invoice.paid-text'],
            ['invoice' => $this->invoice]
        );
    }

Or, if your message only has a plain-text view, you may utilize the `text` method:

    /**
     * Get the mail representation of the notification.
     */
    public function toMail(object $notifiable): MailMessage
    {
        return (new MailMessage)->text(
            'mail.invoice.paid-text', ['invoice' => $this->invoice]
        );
    }

<a name="customizing-the-sender"></a>
### Customizing the Sender

By default, the email's sender / from address is defined in the `config/mail.php` configuration file. However, you may specify the from address for a specific notification using the `from` method:

    /**
     * Get the mail representation of the notification.
     */
    public function toMail(object $notifiable): MailMessage
    {
        return (new MailMessage)
                    ->from('barrett@example.com', 'Barrett Blair')
                    ->line('...');
    }

<a name="customizing-the-recipient"></a>
### Customizing the Recipient

When sending notifications via the `mail` channel, the notification system will automatically look for an `email` property on your notifiable entity. You may customize which email address is used to deliver the notification by defining a `routeNotificationForMail` method on the notifiable entity:

    <?php

    namespace App\Models;

    use MacropaySolutions\Kernel\Database\Obvious\Model;
    use MacropaySolutions\Kernel\Notifications\Notifiable;
    use MacropaySolutions\Kernel\Notifications\Notification;

    class User extends Model
    {
        use Notifiable;

        /**
         * Route notifications for the mail channel.
         *
         * @return  array<string, string>|string
         */
        public function routeNotificationForMail(Notification $notification): array|string
        {
            // Return email address only...
            return $this->email_address;

            // Return email address and name...
            return [$this->email_address => $this->name];
        }
    }

<a name="customizing-the-subject"></a>
### Customizing the Subject

By default, the email's subject is the class name of the notification formatted to "Title Case". So, if your notification class is named `InvoicePaid`, the email's subject will be `Invoice Paid`. If you would like to specify a different subject for the message, you may call the `subject` method when building your message:

    /**
     * Get the mail representation of the notification.
     */
    public function toMail(object $notifiable): MailMessage
    {
        return (new MailMessage)
                    ->subject('Notification Subject')
                    ->line('...');
    }

<a name="customizing-the-mailer"></a>
### Customizing the Mailer

By default, the email notification will be sent using the default mailer defined in the `config/mail.php` configuration file. However, you may specify a different mailer at runtime by calling the `mailer` method when building your message:

    /**
     * Get the mail representation of the notification.
     */
    public function toMail(object $notifiable): MailMessage
    {
        return (new MailMessage)
                    ->mailer('postmark')
                    ->line('...');
    }

<a name="customizing-the-templates"></a>
### Customizing the Templates

You can modify the HTML and plain-text template used by mail notifications by copying the notification view files from the kernel package (`vendor/macropay-solutions/php-kernel/kernel/Notifications/resources/views`) directly into your application's `resources/views/vendor/notifications` directory.

Once placed in `resources/views/vendor/notifications/email.template.php`, the framework will load the custom template using standard dot-notation (`vendor.notifications.email`).

<a name="mail-attachments"></a>
### Attachments

To add attachments to an email notification, use the `attach` method while building your message. The `attach` method accepts the absolute path to the file as its first argument:

    /**
     * Get the mail representation of the notification.
     */
    public function toMail(object $notifiable): MailMessage
    {
        return (new MailMessage)
                    ->greeting('Hello!')
                    ->attach('/path/to/file');
    }

> [!NOTE]  
> The `attach` method offered by notification mail messages also accepts [attachable objects](/mail#attachable-objects). Please consult the comprehensive [attachable object documentation](/mail#attachable-objects) to learn more.

When attaching files to a message, you may also specify the display name and / or MIME type by passing an `array` as the second argument to the `attach` method:

    /**
     * Get the mail representation of the notification.
     */
    public function toMail(object $notifiable): MailMessage
    {
        return (new MailMessage)
                    ->greeting('Hello!')
                    ->attach('/path/to/file', [
                        'as' => 'name.pdf',
                        'mime' => 'application/pdf',
                    ]);
    }

Unlike attaching files in mailable objects, you may not attach a file directly from a storage disk using `attachFromStorage`. You should rather use the `attach` method with an absolute path to the file on the storage disk. Alternatively, you could return a [mailable](/mail#generating-mailables) from the `toMail` method:

    use App\Mail\InvoicePaid as InvoicePaidMailable;

    /**
     * Get the mail representation of the notification.
     */
    public function toMail(object $notifiable): Mailable
    {
        return (new InvoicePaidMailable($this->invoice))
                    ->to($notifiable->email)
                    ->attachFromStorage('/path/to/file');
    }

When necessary, multiple files may be attached to a message using the `attachMany` method:

    /**
     * Get the mail representation of the notification.
     */
    public function toMail(object $notifiable): MailMessage
    {
        return (new MailMessage)
                    ->greeting('Hello!')
                    ->attachMany([
                        '/path/to/forge.svg',
                        '/path/to/vapor.svg' => [
                            'as' => 'Logo.svg',
                            'mime' => 'image/svg+xml',
                        ],
                    ]);
    }

<a name="raw-data-attachments"></a>
#### Raw Data Attachments

The `attachData` method may be used to attach a raw string of bytes as an attachment. When calling the `attachData` method, you should provide the filename that should be assigned to the attachment:

    /**
     * Get the mail representation of the notification.
     */
    public function toMail(object $notifiable): MailMessage
    {
        return (new MailMessage)
                    ->greeting('Hello!')
                    ->attachData($this->pdf, 'name.pdf', [
                        'mime' => 'application/pdf',
                    ]);
    }

<a name="adding-tags-metadata"></a>
### Adding Tags and Metadata

Some third-party email providers such as Mailgun and Postmark support message "tags" and "metadata", which may be used to group and track emails sent by your application. You may add tags and metadata to an email message via the `tag` and `metadata` methods:

    /**
     * Get the mail representation of the notification.
     */
    public function toMail(object $notifiable): MailMessage
    {
        return (new MailMessage)
                    ->greeting('Comment Upvoted!')
                    ->tag('upvote')
                    ->metadata('comment_id', $this->comment->id);
    }

If your application is using the Mailgun driver, you may consult Mailgun's documentation for more information on [tags](https://documentation.mailgun.com/en/latest/user_manual.html#tagging-1) and [metadata](https://documentation.mailgun.com/en/latest/user_manual.html#attaching-data-to-messages). Likewise, the Postmark documentation may also be consulted for more information on their support for [tags](https://postmarkapp.com/blog/tags-support-for-smtp) and [metadata](https://postmarkapp.com/support/article/1125-custom-metadata-faq).

If your application is using Amazon SES to send emails, you should use the `metadata` method to attach [SES "tags"](https://docs.aws.amazon.com/ses/latest/APIReference/API_MessageTag.html) to the message.

<a name="customizing-the-symfony-message"></a>
### Customizing the Symfony Message

The `withSymfonyMessage` method of the `MailMessage` class allows you to register a closure which will be invoked with the Symfony Message instance before sending the message. This gives you an opportunity to deeply customize the message before it is delivered:

    use Symfony\Component\Mime\Email;

    /**
     * Get the mail representation of the notification.
     */
    public function toMail(object $notifiable): MailMessage
    {
        return (new MailMessage)
                    ->withSymfonyMessage(function (Email $message) {
                        $message->getHeaders()->addTextHeader(
                            'Custom-Header', 'Header Value'
                        );
                    });
    }

<a name="using-mailables"></a>
### Using Mailables

If needed, you may return a full [mailable object](/mail) from your notification's `toMail` method. When returning a `Mailable` instead of a `MailMessage`, you will need to specify the message recipient using the mailable object's `to` method:

    use App\Mail\InvoicePaid as InvoicePaidMailable;
    use MacropaySolutions\Kernel\Mail\Mailable;

    /**
     * Get the mail representation of the notification.
     */
    public function toMail(object $notifiable): Mailable
    {
        return (new InvoicePaidMailable($this->invoice))
                    ->to($notifiable->email);
    }

<a name="mailables-and-on-demand-notifications"></a>
#### Mailables and On-Demand Notifications

If you are sending an [on-demand notification](#on-demand-notifications), the `$notifiable` instance given to the `toMail` method will be an instance of `MacropaySolutions\Kernel\Notifications\AnonymousNotifiable`, which offers a `routeNotificationFor` method that may be used to retrieve the email address the on-demand notification should be sent to:

    use App\Mail\InvoicePaid as InvoicePaidMailable;
    use MacropaySolutions\Kernel\Notifications\AnonymousNotifiable;
    use MacropaySolutions\Kernel\Mail\Mailable;

    /**
     * Get the mail representation of the notification.
     */
    public function toMail(object $notifiable): Mailable
    {
        $address = $notifiable instanceof AnonymousNotifiable
            ? $notifiable->routeNotificationFor('mail')
            : $notifiable->email;

        return (new InvoicePaidMailable($this->invoice))
                    ->to($address);
    }

<a name="previewing-mail-notifications"></a>
### Previewing Mail Notifications

When designing a mail notification template, it is convenient to quickly preview the rendered mail message in your browser like a typical Template template. For this reason, Framework allows you to return any mail message generated by a mail notification directly from a controller. When a `MailMessage` is returned, it will be rendered and displayed in the browser, allowing you to quickly preview its design without needing to send it to an actual email address:

    use App\Http\Controllers\Controller;
    use App\Models\Invoice;
    use App\Notifications\InvoicePaid;
    use MacropaySolutions\Kernel\Notifications\Messages\MailMessage;

    class NotificationPreviewController extends Controller
    {
        public function show(): MailMessage
        {
            $invoice = Invoice::query()->find(1);

            return (new InvoicePaid($invoice))->toMail($invoice->user);
        }
    }

<a name="markdown-mail-notifications"></a>
## Markdown Mail Notifications

Markdown mail notifications allow you to take advantage of the pre-built templates of mail notifications, while giving you more freedom to write longer, customized messages. Since the messages are written in Markdown, Framework is able to render beautiful, responsive HTML templates for the messages while also automatically generating a plain-text counterpart.

<a name="generating-the-message"></a>
### Generating the Message

To generate a notification with a corresponding Markdown template, you may use the `--markdown` option of the `make:notification` Run command:

```shell
php run make:notification InvoicePaid --markdown=mail.invoice.paid
```

Like all other mail notifications, notifications that use Markdown templates should define a `toMail` method on their notification class. However, instead of using the `line` and `action` methods to construct the notification, use the `markdown` method to specify the name of the Markdown template that should be used. An array of data you wish to make available to the template may be passed as the method's second argument:

    /**
     * Get the mail representation of the notification.
     */
    public function toMail(object $notifiable): MailMessage
    {
        $url = url('/invoice/'.$this->invoice->id);

        return (new MailMessage)
                    ->subject('Invoice Paid')
                    ->markdown('mail.invoice.paid', ['url' => $url]);
    }

<a name="writing-the-message"></a>
### Writing the Message

Markdown mail notifications use a combination of Template components and Markdown syntax which allow you to easily construct notifications while leveraging Framework's pre-crafted notification components:

```template
<x-mail::message>
# Invoice Paid

Your invoice has been paid!

<x-mail::button :url="$url">
View Invoice
</x-mail::button>

Thanks,<br>
{{ config('app.name') }}
</x-mail::message>
```

<a name="button-component"></a>
#### Button Component

The button component renders a centered button link. The component accepts two arguments, a `url` and an optional `color`. Supported colors are `primary`, `green`, and `red`. You may add as many button components to a notification as you wish:

```template
<x-mail::button :url="$url" color="green">
View Invoice
</x-mail::button>
```

<a name="panel-component"></a>
#### Panel Component

The panel component renders the given block of text in a panel that has a slightly different background color than the rest of the notification. This allows you to draw attention to a given block of text:

```template
<x-mail::panel>
This is the panel content.
</x-mail::panel>
```

<a name="table-component"></a>
#### Table Component

The table component allows you to transform a Markdown table into an HTML table. The component accepts the Markdown table as its content. Table column alignment is supported using the default Markdown table alignment syntax:

```template
<x-mail::table>
| Framework       | Table         | Example  |
| ------------- |:-------------:| --------:|
| Col 2 is      | Centered      | $10      |
| Col 3 is      | Right-Aligned | $20      |
</x-mail::table>
```

<a name="customizing-the-components"></a>
### Customizing the Components

You may copy all the Markdown notification component templates to your own application for customization. Copy the view templates from the kernel's mail resources into your application's `resources/views/vendor/mail` directory.


The `resources/views/vendor/mail` directory should contain an `html` and a `text` directory, each containing their respective representations of every available component. You are free to customize these components however you like.

<a name="customizing-the-css"></a>
#### Customizing the CSS

After exporting the components, the `resources/views/vendor/mail/html/themes` directory will contain a `default.css` file. You may customize the CSS in this file and your styles will automatically be in-lined within the HTML representations of your Markdown notifications.

If you would like to build an entirely new theme for Framework's Markdown components, you may place a CSS file within the `html/themes` directory. After naming and saving your CSS file, update the `theme` option of the `mail` configuration file to match the name of your new theme.

To customize the theme for an individual notification, you may call the `theme` method while building the notification's mail message. The `theme` method accepts the name of the theme that should be used when sending the notification:

    /**
     * Get the mail representation of the notification.
     */
    public function toMail(object $notifiable): MailMessage
    {
        return (new MailMessage)
                    ->theme('invoice')
                    ->subject('Invoice Paid')
                    ->markdown('mail.invoice.paid', ['url' => $url]);
    }

<a name="database-notifications"></a>
## Database Notifications

<a name="database-prerequisites"></a>
### Prerequisites

The `database` notification channel stores the notification information in a database table. This table will contain information such as the notification type as well as a JSON data structure that describes the notification.

You can query the table to display the notifications in your application's user interface. But, before you can do that, you will need to create a database table to hold your notifications. You may use the `notifications:table` command to generate a [migration](/migrations) with the proper table schema:

```shell
php run notifications:table

php run migrate
```

> [!NOTE]  
> If your notifiable models are using [UUID or ULID primary keys](/obvious#uuid-and-ulid-keys), you should replace the `morphs` method with [`uuidMorphs`](/migrations#column-method-uuidMorphs) or [`ulidMorphs`](/migrations#column-method-ulidMorphs) in the notification table migration.

<a name="formatting-database-notifications"></a>
### Formatting Database Notifications

If a notification supports being stored in a database table, you should define a `toDatabase` or `toArray` method on the notification class. This method will receive a `$notifiable` entity and should return a plain PHP array. The returned array will be encoded as JSON and stored in the `data` column of your `notifications` table. Let's take a look at an example `toArray` method:

    /**
     * Get the array representation of the notification.
     *
     * @return array<string, mixed>
     */
    public function toArray(object $notifiable): array
    {
        return [
            'invoice_id' => $this->invoice->id,
            'amount' => $this->invoice->amount,
        ];
    }

When the notification is stored in your application's database, the `type` column will be populated with the notification's class name. However, you may customize this behavior by defining a `databaseType` method on your notification class:

    /**
     * Get the notification's database type.
     *
     * @return string
     */
    public function databaseType(object $notifiable): string
    {
        return 'invoice-paid';
    }

<a name="todatabase-vs-toarray"></a>
#### `toDatabase` vs. `toArray`

The `toArray` method is also used by the `broadcast` channel to determine which data to broadcast to your JavaScript powered frontend. If you would like to have two different array representations for the `database` and `broadcast` channels, you should define a `toDatabase` method instead of a `toArray` method.

<a name="accessing-the-notifications"></a>
### Accessing the Notifications

Once notifications are stored in the database, you need a convenient way to access them from your notifiable entities. The `MacropaySolutions\Kernel\Notifications\Notifiable` trait, which is included on Framework's default `App\Models\User` model, includes a `notifications` [Obvious relationship](/obvious-relationships) that returns the notifications for the entity. To fetch notifications, you may access this method like any other Obvious relationship. By default, notifications will be sorted by the `created_at` timestamp with the most recent notifications at the beginning of the collection:

    $user = App\Models\User::query()->find(1);

    foreach ($user->notifications as $notification) {
        echo $notification->type;
    }

If you want to retrieve only the "unread" notifications, you may use the `unreadNotifications` relationship. Again, these notifications will be sorted by the `created_at` timestamp with the most recent notifications at the beginning of the collection:

    $user = App\Models\User::query()->find(1);

    foreach ($user->unreadNotifications as $notification) {
        echo $notification->type;
    }

> [!NOTE]  
> To access your notifications from your JavaScript client, you should define a notification controller for your application which returns the notifications for a notifiable entity, such as the current user. You may then make an HTTP request to that controller's URL from your JavaScript client.

<a name="marking-notifications-as-read"></a>
### Marking Notifications as Read

Typically, you will want to mark a notification as "read" when a user views it. The `MacropaySolutions\Kernel\Notifications\Notifiable` trait provides a `markAsRead` method, which updates the `read_at` column on the notification's database record:

    $user = App\Models\User::query()->find(1);

    foreach ($user->unreadNotifications as $notification) {
        $notification->markAsRead();
    }

However, instead of looping through each notification, you may use the `markAsRead` method directly on a collection of notifications:

    $user->unreadNotifications->markAsRead();

You may also use a mass-update query to mark all the notifications as read without retrieving them from the database:

    $user = App\Models\User::query()->find(1);

    $user->unreadNotifications()->update(['read_at' => now()]);

You may `delete` the notifications to remove them from the table entirely:

    $user->notifications()->delete();

<a name="broadcast-notifications"></a>
## Broadcast Notifications

<a name="broadcast-prerequisites"></a>
### Prerequisites

Before broadcasting notifications, you should configure and be familiar with Framework's [event broadcasting](/broadcasting) services. Event broadcasting provides a way to react to server-side Framework events from your JavaScript powered frontend.

<a name="formatting-broadcast-notifications"></a>
### Formatting Broadcast Notifications

The `broadcast` channel broadcasts notifications using Framework's [event broadcasting](/broadcasting) services, allowing your JavaScript powered frontend to catch notifications in realtime. If a notification supports broadcasting, you can define a `toBroadcast` method on the notification class. This method will receive a `$notifiable` entity and should return a `BroadcastMessage` instance. If the `toBroadcast` method does not exist, the `toArray` method will be used to gather the data that should be broadcast. The returned data will be encoded as JSON and broadcast to your JavaScript powered frontend. Let's take a look at an example `toBroadcast` method:

    use MacropaySolutions\Kernel\Notifications\Messages\BroadcastMessage;

    /**
     * Get the broadcastable representation of the notification.
     */
    public function toBroadcast(object $notifiable): BroadcastMessage
    {
        return new BroadcastMessage([
            'invoice_id' => $this->invoice->id,
            'amount' => $this->invoice->amount,
        ]);
    }

<a name="broadcast-queue-configuration"></a>
#### Broadcast Queue Configuration

All broadcast notifications are queued for broadcasting. If you would like to configure the queue connection or queue name that is used to queue the broadcast operation, you may use the `onConnection` and `onQueue` methods of the `BroadcastMessage`:

    return (new BroadcastMessage($data))
                    ->onConnection('sqs')
                    ->onQueue('broadcasts');

<a name="customizing-the-notification-type"></a>
#### Customizing the Notification Type

In addition to the data you specify, all broadcast notifications also have a `type` field containing the full class name of the notification. If you would like to customize the notification `type`, you may define a `broadcastType` method on the notification class:

    /**
     * Get the type of the notification being broadcast.
     */
    public function broadcastType(): string
    {
        return 'broadcast.message';
    }

<a name="listening-for-notifications"></a>
### Listening for Notifications

Notifications will broadcast on a private channel formatted using a `{notifiable}.{id}` convention. So, if you are sending a notification to an `App\Models\User` instance with an ID of `1`, the notification will be broadcast on the `App.Models.User.1` private channel.

<a name="customizing-the-notification-channel"></a>
#### Customizing the Notification Channel

If you would like to customize which channel that an entity's broadcast notifications are broadcast on, you may define a `receivesBroadcastNotificationsOn` method on the notifiable entity:

    <?php

    namespace App\Models;

    use MacropaySolutions\Kernel\Broadcasting\PrivateChannel;
    use MacropaySolutions\Kernel\Database\Obvious\Model;
    use MacropaySolutions\Kernel\Notifications\Notifiable;

    class User extends Model
    {
        use Notifiable;

        /**
         * The channels the user receives notification broadcasts on.
         */
        public function receivesBroadcastNotificationsOn(): string
        {
            return 'users.'.$this->id;
        }
    }

<a name="sms-notifications"></a>

## Localizing Notifications

Framework allows you to send notifications in a locale other than the HTTP request's current locale, and will even remember this locale if the notification is queued.

To accomplish this, the `MacropaySolutions\Kernel\Notifications\Notification` class offers a `locale` method to set the desired language. The application will change into this locale when the notification is being evaluated and then revert back to the previous locale when evaluation is complete:

    $user->notify((new InvoicePaid($invoice))->locale('es'));

Localization of multiple notifiable entries may also be achieved via the `ChannelManager`:

    \app(ChannelManager::class)->locale('es')->send(
        $users, new InvoicePaid($invoice)
    );

<a name="user-preferred-locales"></a>
### User Preferred Locales

Sometimes, applications store each user's preferred locale. By implementing the `HasLocalePreference` contract on your notifiable model, you may instruct Framework to use this stored locale when sending a notification:

    use MacropaySolutions\Kernel\Contracts\Translation\HasLocalePreference;

    class User extends Model implements HasLocalePreference
    {
        /**
         * Get the user's preferred locale.
         */
        public function preferredLocale(): string
        {
            return $this->locale;
        }
    }

Once you have implemented the interface, Framework will automatically use the preferred locale when sending notifications and mailables to the model. Therefore, there is no need to call the `locale` method when using this interface:

    $user->notify(new InvoicePaid($invoice));

<a name="testing"></a>
## Testing

To prevent notifications from being sent during tests, you must manually swap the `ChannelManager` in the application container with an instance of `NotificationFake`. Typically, sending notifications is unrelated to the code you are actually testing. Most likely, it is sufficient to simply assert that Framework was instructed to send a given notification.

After swapping the container binding, you may then assert that notifications were instructed to be sent to users and even inspect the data the notifications received:

    <?php

    namespace Tests\Feature;

    use App\Notifications\OrderShipped;
    use MacropaySolutions\Kernel\Notifications\ChannelManager;
    use MacropaySolutions\KernelDev\Support\Testing\Fakes\NotificationFake;
    use Tests\TestCase;

    class ExampleTest extends TestCase
    {
        public function test_orders_can_be_shipped(): void
        {
            $fake = new NotificationFake();
            \app()->instance(ChannelManager::class, $fake);

            // Perform order shipping...

            // Assert that no notifications were sent...
            $fake->assertNothingSent();

            // Assert a notification was sent to the given users...
            $fake->assertSentTo(
                [$user], OrderShipped::class
            );

            // Assert a notification was not sent...
            $fake->assertNotSentTo(
                [$user], AnotherNotification::class
            );

            // Assert that a given number of notifications were sent...
            $fake->assertCount(3);
        }
    }

You may pass a closure to the `assertSentTo` or `assertNotSentTo` methods in order to assert that a notification was sent that passes a given "truth test". If at least one notification was sent that passes the given truth test then the assertion will be successful:

    $fake->assertSentTo(
        $user,
        function (OrderShipped $notification, array $channels) use ($order) {
            return $notification->order->id === $order->id;
        }
    );

<a name="on-demand-notifications"></a>
#### On-Demand Notifications

If the code you are testing sends [on-demand notifications](#on-demand-notifications), you can test that the on-demand notification was sent via the `assertSentOnDemand` method:

    $fake->assertSentOnDemand(OrderShipped::class);

By passing a closure as the second argument to the `assertSentOnDemand` method, you may determine if an on-demand notification was sent to the correct "route" address:

    $fake->assertSentOnDemand(
        OrderShipped::class,
        function (OrderShipped $notification, array $channels, object $notifiable) use ($user) {
            return $notifiable->routes['mail'] === $user->email;
        }
    );

<a name="notification-events"></a>
## Notification Events

<a name="notification-sending-event"></a>
#### Notification Sending Event

When a notification is sending, the `MacropaySolutions\Kernel\Notifications\Events\NotificationSending` [event](/events) is dispatched by the notification system. This contains the "notifiable" entity and the notification instance itself. You may register listeners for this event in your application's `EventServiceProvider`:

    use App\Listeners\CheckNotificationStatus;
    use MacropaySolutions\Kernel\Notifications\Events\NotificationSending;
    
    /**
     * The event listener mappings for the application.
     *
     * @var array
     */
    protected $listen = [
        NotificationSending::class => [
            CheckNotificationStatus::class,
        ],
    ];

The notification will not be sent if an event listener for the `NotificationSending` event returns `false` from its `handle` method:

    use MacropaySolutions\Kernel\Notifications\Events\NotificationSending;

    /**
     * Handle the event.
     */
    public function handle(NotificationSending $event): bool
    {
        return false;
    }

Within an event listener, you may access the `notifiable`, `notification`, and `channel` properties on the event to learn more about the notification recipient or the notification itself:

    /**
     * Handle the event.
     */
    public function handle(NotificationSending $event): void
    {
        // $event->channel
        // $event->notifiable
        // $event->notification
    }

<a name="notification-sent-event"></a>
#### Notification Sent Event

When a notification is sent, the `MacropaySolutions\Kernel\Notifications\Events\NotificationSent` [event](/events) is dispatched by the notification system. This contains the "notifiable" entity and the notification instance itself. You may register listeners for this event in your `EventServiceProvider`:

    use App\Listeners\LogNotification;
    use MacropaySolutions\Kernel\Notifications\Events\NotificationSent;
    
    /**
     * The event listener mappings for the application.
     *
     * @var array
     */
    protected $listen = [
        NotificationSent::class => [
            LogNotification::class,
        ],
    ];

> [!NOTE]  
> After registering listeners in your `EventServiceProvider`, use the `event:generate` Run command to quickly generate listener classes.

Within an event listener, you may access the `notifiable`, `notification`, `channel`, and `response` properties on the event to learn more about the notification recipient or the notification itself:

    /**
     * Handle the event.
     */
    public function handle(NotificationSent $event): void
    {
        // $event->channel
        // $event->notifiable
        // $event->notification
        // $event->response
    }

<a name="custom-channels"></a>
## Custom Channels

Framework ships with a handful of notification channels, but you may want to write your own drivers to deliver notifications via other channels. Framework makes it simple. To get started, define a class that contains a `send` method. The method should receive two arguments: a `$notifiable` and a `$notification`.

Within the `send` method, you may call methods on the notification to retrieve a message object understood by your channel and then send the notification to the `$notifiable` instance however you wish:

    <?php

    namespace App\Notifications;

    use MacropaySolutions\Kernel\Notifications\Notification;

    class VoiceChannel
    {
        /**
         * Send the given notification.
         */
        public function send(object $notifiable, Notification $notification): void
        {
            $message = $notification->toVoice($notifiable);

            // Send notification to the $notifiable instance...
        }
    }

Once your notification channel class has been defined, you may return the class name from the `via` method of any of your notifications. In this example, the `toVoice` method of your notification can return whatever object you choose to represent voice messages. For example, you might define your own `VoiceMessage` class to represent these messages:

    <?php

    namespace App\Notifications;

    use App\Notifications\Messages\VoiceMessage;
    use App\Notifications\VoiceChannel;
    use MacropaySolutions\Kernel\Bus\Queueable;
    use MacropaySolutions\Kernel\Contracts\Queue\ShouldQueue;
    use MacropaySolutions\Kernel\Notifications\Notification;

    class InvoicePaid extends Notification
    {
        use Queueable;

        /**
         * Get the notification channels.
         */
        public function via(object $notifiable): string
        {
            return VoiceChannel::class;
        }

        /**
         * Get the voice representation of the notification.
         */
        public function toVoice(object $notifiable): VoiceMessage
        {
            // ...
        }
    }
