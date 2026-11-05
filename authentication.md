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
  - [The Authenticatable Trait and Token Rotation](#the-authenticatable-trait-and-token-rotation)
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

When building the database schema for the `App\Models\User` model, ensure the password column is at least 60 characters in length. If utilizing stateful session authentication with "Remember Me" support, your table must also include a nullable string column named `remember_token` (e.g., `VARCHAR(100)`).

<a name="getting-started"></a>
## Getting Started

<a name="authentication-service-provider"></a>
#### Authentication Service Provider

The `AuthServiceProvider` located in your `app/Providers` directory contains a call to `app('auth')->viaRequest`. The `viaRequest` method accepts a Closure that is executed when an incoming request requires authentication. Within this Closure, you may resolve your `App\Models\User` instance using any strategy required.

For maximum security against timing attacks and database leaks, it is recommended to query the user by an identifier first, and then verify their token or secret using a constant-time cryptographic function like `password_verify`:

```php
use App\Models\User;
use MacropaySolutions\Kernel\Http\Request;

$this->app['auth']->viaRequest('api', function (Request$request): ?User {
       $apiToken = $request->input('api_token');

       return (\is_string($apiToken) && $apiToken !== '') ?
           User::query()->where('api_token', \hash('sha256', $apiToken))->first() :
           null;
      //or
      $id = $request->header('X-API-ID');
      $pass = $request->header('X-API-PASS');

      if (!\is_string($id) || $id === '' || !\is_string($pass) || $pass === '') {
          return null;
      }

      $user = User::query()->find($id) ?? new User();

      if (
          \password_verify($pass, $user->getAttributeValue('pass') ??
              '$2y$10$TKh8H1.PfQx37YgCzwiKb.KjNyWgaHb9cbcoQgdIVFlYg7B77UdFm') // hash for `secret`
          && $user->exists
      ) {
          return $user;
      }

      return null;
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
use MacropaySolutions\Kernel\Http\JsonResponse;

class UserController extends Controller
{
    /**
     * Display the authenticated user profile.
     */
    public function profile(Request $request): JsonResponse
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

Route middleware can be used to only allow authenticated users to access specific endpoints. Framework includes the `auth` middleware, which references `App\Http\Middleware\Authenticate` (uncomment the `auth` alias in the `$routeMiddleware` property of `App\Application`). Attach the middleware to your route definitions inside your route files:

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

<a name="the-authenticatable-trait-and-token-rotation"></a>
### The Authenticatable Trait and Token Rotation

Framework provides the `MacropaySolutions\Kernel\Auth\Authenticatable` trait to satisfy the `Authenticatable` contract on your Obvious models>.

#### Automatic Remember Token Rotation

Whenever an existing user's password attribute is modified on a model instance (`$user->password = ...` or `$user->update(['password' => ...])`), the trait intercepts the write in `Authenticatable::setAttribute()` and automatically rotates the `remember_token` to a new random 60-character string.

This mechanism invalidates all active "Remember Me" sessions across other devices on the server side without exposing raw password hashes or derivatives in the client recaller cookie (`ID|Token`).

> [!WARNING]
> **Direct Query Builder Updates & Mutator Bypass**
> Direct query builder updates (such as `User::query()->where(...)->update(['password' => ...])` or `app('db')->table('users')->update(...)`) execute raw SQL queries and bypass Obvious model mutators, lifecycle hooks, and `setAttribute()`.
>
> When performing bulk updates directly through the query builder, the `remember_token` will not be rotated automatically. You must explicitly pass a newly generated `remember_token` in the update payload to prevent old remember-me cookies from remaining valid against the updated credentials:

    use MacropaySolutions\Kernel\Support\Str;

    // Explicitly rotate the remember_token during direct query builder updates
    User::query()->where('id', $userId)->update([
        'password' => $newHashedPassword,
        'remember_token' => Str::random(60),
    ]);

Failing to rotate the `remember_token` when updating a password will result in:
- **Old sessions remain fully valid:** Any previously issued "Remember Me" cookies (`ID|Token`) on other browsers, devices, or in the hands of an attacker will continue to authenticate successfully.
- **Global logout bypass:** Because the password hash derivative was removed from the cookie and `AuthenticateSession` no longer compares the cookie against the database password, the `remember_token` in the database is now the sole gatekeeper for persistent sessions. If it does not change, changing the password does nothing to revoke existing sessions.
- **No automatic session termination on credential compromise:** If an account is reset due to being compromised, the attacker retains access indefinitely until their local cookie reaches its natural expiration (e.g., 400 days) or the user explicitly clicks "Log out" to invoke `cycleRememberToken()`.

<a name="events"></a>
## Events

Framework dispatches events during the authentication lifecycle:
* `MacropaySolutions\Kernel\Auth\Events\Attempting`
* `MacropaySolutions\Kernel\Auth\Events\Authenticated`
* `MacropaySolutions\Kernel\Auth\Events\Failed`
* `MacropaySolutions\Kernel\Auth\Events\Logout`
