---
title: Authentication
description: Guide to stateless authentication and API token management in PHP Framework.
context: authentication
---

# Authentication

- [Introduction](#introduction)
  - [Session Authentication (Opt-In)](#session-authentication-opt-in)
  - [Database Considerations](#database-considerations)
- [Getting Started](#getting-started)
  - [Authentication Service Provider](#authentication-service-provider)
  - [Accessing The Authenticated User](#accessing-the-authenticated-user)
- [Protecting Routes](#protecting-routes)
- [Adding Custom Guards](#adding-custom-guards)
- [Adding Custom User Providers](#adding-custom-user-providers)
  - [The User Provider Contract](#the-user-provider-contract)
  - [The Authenticatable Contract](#the-authenticatable-contract)
- [Events](#events)

<a name="introduction"></a>
## Introduction

Authentication in Framework is configured specifically for stateless execution by default. Since Framework does not maintain session state out of the box ([but it can be explicitly enabled](/session)), incoming requests are authenticated via a stateless mechanism such as API tokens or Bearer tokens.

At its core, Framework's authentication facilities are composed of **guards** and **providers**:
* **Guards** define how users are authenticated for each request (e.g., inspecting a request header or token).
* **Providers** define how users are retrieved from persistent storage (such as Obvious ORM or the database query builder).

Your application's authentication configuration is managed via `config/auth.php`.

<a name="session-authentication-opt-in"></a>
### Session Authentication (Opt-In)

While Framework is optimized for stateless JSON APIs by default, stateful session-based authentication is fully supported. Because session storage and cookie handling are disabled by default for performance, utilizing session guards requires an explicit opt-in:

1. Enable the session container bindings and aliases inside `App\Application.php`.
2. Declare the `MacropaySolutions\Kernel\Session\Middleware\StartSession` middleware on your targeted route group.
3. Configure the `session` guard in `config/auth.php`.

<a name="database-considerations"></a>
### Database Considerations

By default, Framework includes an `App\Models\User` Obvious model in your `app/Models` directory. This model may be used with the default Obvious authentication driver. If your application is not using Obvious, you may use the `database` authentication provider which uses the database query builder.

When building the database schema for the `App\Models\User` model, ensure the password column is at least 60 characters in length.

<a name="getting-started"></a>
## Getting Started

<a name="authentication-service-provider"></a>
#### Authentication Service Provider

> [!NOTE]  
> Before using Framework's authentication features, ensure the `AuthServiceProvider` registration call is active in your `bootstrap/app.php` file.

The `AuthServiceProvider` located in your `app/Providers` directory contains a call to `app('auth')->viaRequest`. The `viaRequest` method accepts a Closure that is executed when an incoming request requires authentication. Within this Closure, you may resolve your `App\Models\User` instance using any strategy required:

```php
use App\Models\User;
use MacropaySolutions\Kernel\Http\Request;

$this->app['auth']->viaRequest('api', function (Request $request): ?User {
    $token = (string) $request->getFiltered('api_token');

    if ($token === '') {
        return null;
    }

    return User::query()->where('api_token', $token)->first();
});
```

You may retrieve the authenticated user using an API token in the request headers or query string, a Bearer token, or any custom header logic.

If your project does not use Obvious ORM, you may return an instance of `MacropaySolutions\Kernel\Auth\GenericUser`. This class accepts an array of attributes as its constructor argument:

```php
use MacropaySolutions\Kernel\Auth\GenericUser;

return new GenericUser(['id' => 1, 'name' => 'Name']);
```

<a name="accessing-the-authenticated-user"></a>
#### Accessing The Authenticated User

You may use the `auth()->user()` global helper to retrieve the currently authenticated user. Alternatively, you may access the user via the `MacropaySolutions\Kernel\Http\Request` instance in your controller method:

```php
namespace App\Http\Controllers;

use MacropaySolutions\Kernel\Http\Request;
use MacropaySolutions\Kernel\Http\Response;

class UserController extends Controller
{
    /**
     * Display the authenticated user profile.
     */
    public function profile(Request $request): Response
    {
        $user = $request->user();

        return \response()->json([
            'status' => 'success',
            'data' => $user->toArray(),
        ]);
    }
}
```

To check whether the incoming request is authenticated, use `auth()->check()`:

```php
if (\auth()->check()) {
    return \response()->json(['status' => 'authenticated']);
}
```

<a name="protecting-routes"></a>
## Protecting Routes

Route middleware can be used to only allow authenticated users to access specific endpoints. Framework includes the `auth` middleware, which references `MacropaySolutions\Kernel\Auth\Middleware\Authenticate`. Attach the middleware to your route definitions inside your route files:

```php
$router->get('user/profile', [
    'middleware' => 'auth',
    'uses' => 'UserController@profile',
]);
```

<a name="adding-custom-guards"></a>
## Adding Custom Guards

You may define custom authentication guards using the `extend` method on `app('auth')`. Place your call to the `extend` method within the `boot` method of your `AuthServiceProvider`:

```php
namespace App\Providers;

use App\Services\Auth\JwtGuard;
use MacropaySolutions\Kernel\Contracts\Auth\Guard;
use MacropaySolutions\Kernel\Support\ServiceProvider;

class AuthServiceProvider extends ServiceProvider
{
    /**
     * Register authentication services.
     */
    public function boot(): void
    {
        \app('auth')->extend('jwt', function ($app, string $name, array $config): Guard {
            $provider = \app('auth')->createUserProvider($config['provider']);

            return new JwtGuard($provider);
        });
    }
}
```

Once your custom guard is defined, reference it within the `guards` configuration array in `config/auth.php`.

<a name="adding-custom-user-providers"></a>
## Adding Custom User Providers

If you are not using a traditional database table to store users, you may extend Framework with a custom user provider using the `provider` method on `app('auth')`:

```php
namespace App\Providers;

use App\Extensions\MongoUserProvider;
use MacropaySolutions\Kernel\Contracts\Auth\UserProvider;
use MacropaySolutions\Kernel\Support\ServiceProvider;

class AuthServiceProvider extends ServiceProvider
{
    /**
     * Register authentication services.
     */
    public function boot(): void
    {
        \app('auth')->provider('mongo', function ($app, array $config): UserProvider {
            return new MongoUserProvider($app->make('mongo.connection'));
        });
    }
}
```

<a name="the-user-provider-contract"></a>
### The User Provider Contract

Implementations of `MacropaySolutions\Kernel\Contracts\Auth\UserProvider` handle fetching an `MacropaySolutions\Kernel\Contracts\Auth\Authenticatable` instance out of persistent storage:

```php
namespace MacropaySolutions\Kernel\Contracts\Auth;

interface UserProvider
{
    public function retrieveById($identifier);
    public function retrieveByToken($identifier, $token);
    public function updateRememberToken(Authenticatable $user, $token);
    public function retrieveByCredentials(array $credentials);
    public function validateCredentials(Authenticatable $user, array $credentials);
}
```

<a name="the-authenticatable-contract"></a>
### The Authenticatable Contract

Classes representing authenticated users must implement `MacropaySolutions\Kernel\Contracts\Auth\Authenticatable`:

```php
namespace MacropaySolutions\Kernel\Contracts\Auth;

interface Authenticatable
{
    public function getAuthIdentifierName();
    public function getAuthIdentifier();
    public function getAuthPassword();
    public function getRememberToken();
    public function setRememberToken($value);
    public function getRememberTokenName();
}
```

<a name="events"></a>
## Events

Framework dispatches events during the authentication lifecycle:
* `MacropaySolutions\Kernel\Auth\Events\Attempting`
* `MacropaySolutions\Kernel\Auth\Events\Authenticated`
* `MacropaySolutions\Kernel\Auth\Events\Failed`
* `MacropaySolutions\Kernel\Auth\Events\Logout`
