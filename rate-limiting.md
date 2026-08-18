---
title: Rate Limiting
description: Limiting the frequency of actions and requests in PHP Framework.
context: rate-limiting
---
# Rate Limiting

- [Introduction](#introduction)
    - [Cache Configuration](#cache-configuration)
- [Basic Usage](#basic-usage)
    - [Manually Incrementing Attempts](#manually-incrementing-attempts)
    - [Clearing Attempts](#clearing-attempts)

<a name="introduction"></a>
## Introduction

Framework includes a simple to use rate limiting abstraction which, in conjunction with your application's [cache](/cache), provides an easy way to limit any action during a specified window of time.

> [!Important]
> This rate limiter is designed for application-level rate limiting of specific actions within your application (e.g., limiting how many messages a user can send per minute). It does not replace server-level or infrastructure-level rate limiting. You should continue to implement rate limiting at the server level (via nginx, Apache, load balancers, or cloud providers) to protect your infrastructure from request floods and DDoS attacks. Use this framework rate limiter in conjunction with infrastructure-level protections.

> [!NOTE]  
> If you are interested in rate limiting incoming HTTP requests, please consult the [rate limiter middleware documentation](routing#rate-limiting).

<a name="cache-configuration"></a>
### Cache Configuration

Typically, the rate limiter utilizes your default application cache as defined by the `default` key within your application's `cache` configuration file. However, you may specify which cache driver the rate limiter should use by defining a `limiter` key within your application's `cache` configuration file:

    'default' => 'memcached',

    'limiter' => 'redis',

<a name="basic-usage"></a>
## Basic Usage

The `RateLimiter` service may be used to interact with the rate limiter. The simplest method offered by the rate limiter is the `attempt` method, which rate limits a given callback for a given number of seconds.

The `attempt` method returns `false` when the callback has no remaining attempts available; otherwise, the `attempt` method will return the callback's result or `true`. The first argument accepted by the `attempt` method is a rate limiter "key", which may be any string of your choosing that represents the action being rate limited:

    $executed = app(\MacropaySolutions\Kernel\Cache\RateLimiter::class)->attempt(
        'send-message:'.$user->id,
        $perMinute = 5,
        function() {
            // Send message...
        }
    );

    if (!$executed) {
      return 'Too many messages sent!';
    }

If necessary, you may provide a fourth argument to the `attempt` method, which is the "decay rate", or the number of seconds until the available attempts are reset. For example, we can modify the example above to allow five attempts every two minutes:

    $executed = app(\MacropaySolutions\Kernel\Cache\RateLimiter::class)->attempt(
        'send-message:'.$user->id,
        $perTwoMinutes = 5,
        function() {
            // Send message...
        },
        $decayRate = 120,
    );

<a name="manually-incrementing-attempts"></a>
### Manually Incrementing Attempts

If you would like to manually interact with the rate limiter, a variety of other methods are available. For example, you may invoke the `tooManyAttempts` method to determine if a given rate limiter key has exceeded its maximum number of allowed attempts per minute:

    if (app(\MacropaySolutions\Kernel\Cache\RateLimiter::class)->tooManyAttempts('send-message:'.$user->id, $perMinute = 5)) {
        return 'Too many attempts!';
    }

    app(\MacropaySolutions\Kernel\Cache\RateLimiter::class)->increment('send-message:'.$user->id);

    // Send message...

Alternatively, you may use the `remaining` method to retrieve the number of attempts remaining for a given key. If a given key has retries remaining, you may invoke the `increment` method to increment the number of total attempts:

    if (app(\MacropaySolutions\Kernel\Cache\RateLimiter::class)->remaining('send-message:'.$user->id, $perMinute = 5)) {
        app(\MacropaySolutions\Kernel\Cache\RateLimiter::class)->increment('send-message:'.$user->id);

        // Send message...
    }

If you would like to increment the value for a given rate limiter key by more than one, you may provide the desired amount to the `increment` method:

    app(\MacropaySolutions\Kernel\Cache\RateLimiter::class)->increment('send-message:'.$user->id, amount: 5);

<a name="determining-limiter-availability"></a>
#### Determining Limiter Availability

When a key has no more attempts left, the `availableIn` method returns the number of seconds remaining until more attempts will be available:

    if (app(\MacropaySolutions\Kernel\Cache\RateLimiter::class)->tooManyAttempts('send-message:'.$user->id, $perMinute = 5)) {
        $seconds = app(\MacropaySolutions\Kernel\Cache\RateLimiter::class)->availableIn('send-message:'.$user->id);

        return 'You may try again in '.$seconds.' seconds.';
    }

    app(\MacropaySolutions\Kernel\Cache\RateLimiter::class)->increment('send-message:'.$user->id);

    // Send message...

<a name="clearing-attempts"></a>
### Clearing Attempts

You may reset the number of attempts for a given rate limiter key using the `clear` method. For example, you may reset the number of attempts when a given message is read by the receiver:

    use App\Models\Message;

    /**
     * Mark the message as read.
     */
    public function read(Message $message): Message
    {
        $message->markAsRead();

        app(\MacropaySolutions\Kernel\Cache\RateLimiter::class)->clear('send-message:'.$message->user_id);

        return $message;
    }
