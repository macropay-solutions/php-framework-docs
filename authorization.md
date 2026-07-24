---
title: Authorization
description: Guide to gates, policies, and authorization management in PHP Framework.
context: authorization
---

# Authorization

- [Introduction](#introduction)
- [Gates](#gates)
  - [Defining Gates](#defining-gates)
  - [Authorizing Actions via Gates](#authorizing-actions-via-gates)
  - [Gate Responses](#gate-responses)
  - [Intercepting Gate Checks](#intercepting-gate-checks)
- [Policies](#policies)
  - [Creating Policies](#creating-policies)
  - [Registering Policies](#registering-policies)
  - [Writing Policies](#writing-policies)
  - [Guest Users](#guest-users)
  - [Policy Filters](#policy-filters)
- [Authorizing Actions Using Policies](#authorizing-actions-using-policies)
  - [Via the User Model](#via-the-user-model)
  - [Via Controller Helpers](#via-controller-helpers)

<a name="introduction"></a>
## Introduction

In addition to providing [authentication](/authentication) services, Framework provides a simple way to authorize user actions against resources. Framework offers two primary mechanisms for authorizing user actions: **Gates** and **Policies**.

Gates provide a simple closure-based approach to authorization, while Policies group authorization logic around a specific Obvious model or resource.

<a name="gates"></a>
## Gates

<a name="defining-gates"></a>
### Defining Gates

Gates are closures that determine if a user is authorized to perform a given action. Gates are defined within the `boot` method of `App\Providers\AuthServiceProvider` using the `MacropaySolutions\Kernel\Contracts\Auth\Access\Gate` contract resolved from the container.

```php
namespace App\Providers;

use App\Models\Post;
use App\Models\User;
use MacropaySolutions\Kernel\Contracts\Auth\Access\Gate;
use MacropaySolutions\Kernel\Support\ServiceProvider;

class AuthServiceProvider extends ServiceProvider
{
    /**
     * Register authorization services.
     */
    public function boot(): void
    {
        $gate = \app(Gate::class);

        $gate->define('update-post', function (User $user, Post $post): bool {
            return $user->id === $post->user_id;
        });
    }
}
```

<a name="authorizing-actions-via-gates"></a>
### Authorizing Actions via Gates

To authorize an action using gates, use `allows` or `denies` on the resolved `Gate` instance, or `$request->user()->can()` and `$request->user()->cannot()`:

```php
namespace App\Http\Controllers;

use App\Models\Post;
use MacropaySolutions\Kernel\Contracts\Auth\Access\Gate;
use MacropaySolutions\Kernel\Http\Request;
use MacropaySolutions\Kernel\Http\Response;

class PostController extends Controller
{
    /**
     * Update the specified post.
     */
    public function update(Request $request, int|string $id): Response
    {
        $post = Post::query()->findOrFail($id);
        $gate = \app(Gate::class);

        if ($gate->denies('update-post', $post)) {
            \abort(403, 'Unauthorized action.');
        }

        $post->fill([
            'title' => (string) $request->getFiltered('title'),
        ])->save();

        return \response()->json([
            'status' => 'success',
            'data' => $post->toArray(),
        ]);
    }
}
```

<a name="gate-responses"></a>
### Gate Responses

Instead of boolean values, gates may return an `MacropaySolutions\Kernel\Auth\Access\Response` instance to convey detailed error messages or HTTP status codes:

```php
use App\Models\User;
use MacropaySolutions\Kernel\Auth\Access\Response;
use MacropaySolutions\Kernel\Contracts\Auth\Access\Gate;

$gate = \app(Gate::class);

$gate->define('edit-settings', function (User $user): Response {
    return $user->isAdmin
        ? Response::allow()
        : Response::denyWithStatus(404);
});
```

<a name="intercepting-gate-checks"></a>
### Intercepting Gate Checks

To grant all abilities to a specific user type (such as administrators), define a `before` callback:

```php
use App\Models\User;
use MacropaySolutions\Kernel\Contracts\Auth\Access\Gate;

$gate = \app(Gate::class);

$gate->before(function (User $user, string $ability): bool|null {
    if ($user->isAdministrator()) {
        return true;
    }

    return null;
});
```

<a name="policies"></a>
## Policies

<a name="creating-policies"></a>
### Creating Policies

Policies are classes that organize authorization logic around a specific Obvious model. Generate a policy using the `run` console command:

```bash
php run make:policy PostPolicy --model=Post
```

<a name="registering-policies"></a>
### Registering Policies

Register policies in your `AuthServiceProvider` using the `policy` method on the `Gate` contract:

```php
namespace App\Providers;

use App\Models\Post;
use App\Policies\PostPolicy;
use MacropaySolutions\Kernel\Contracts\Auth\Access\Gate;
use MacropaySolutions\Kernel\Support\ServiceProvider;

class AuthServiceProvider extends ServiceProvider
{
    /**
     * Register authorization services.
     */
    public function boot(): void
    {
        $gate = \app(Gate::class);

        $gate->policy(Post::class, PostPolicy::class);
    }
}
```

<a name="writing-policies"></a>
### Writing Policies

Policy methods receive an authenticated `User` instance as their first parameter and the target model as their second parameter:

```php
namespace App\Policies;

use App\Models\Post;
use App\Models\User;

class PostPolicy
{
    /**
     * Determine if the post can be updated by the user.
     */
    public function update(User $user, Post $post): bool
    {
        return $user->id === $post->user_id;
    }
}
```

<a name="guest-users"></a>
### Guest Users

By default, gates and policies return `false` if the incoming request is unauthenticated. To allow unauthenticated checks, make the user argument nullable:

```php
public function update(?User $user, Post $post): bool
{
    return $user?->id === $post->user_id;
}
```

<a name="policy-filters"></a>
### Policy Filters

Define a `before` method on a policy class to run pre-authorization checks:

```php
public function before(User $user, string $ability): bool|null
{
    if ($user->isAdministrator()) {
        return true;
    }

    return null;
}
```

<a name="authorizing-actions-using-policies"></a>
## Authorizing Actions Using Policies

<a name="via-the-user-model"></a>
### Via the User Model

The `App\Models\User` model provides `can` and `cannot` methods:

```php
if ($request->user()->cannot('update', $post)) {
    \abort(403);
}
```

<a name="via-controller-helpers"></a>
### Via Controller Helpers

Base controllers providing the `ProvidesConvenienceMethods` trait include an `authorize` helper. If unauthorized, an `AuthorizationException` is thrown and captured by the exception handler to format a `403` JSON response:

```php
namespace App\Http\Controllers;

use App\Models\Post;
use MacropaySolutions\Kernel\Http\Request;
use MacropaySolutions\Kernel\Http\Response;

class PostController extends Controller
{
    /**
     * Update the specified post.
     */
    public function update(Request $request, int|string $id): Response
    {
        $post = Post::query()->findOrFail($id);

        $this->authorize('update', $post);

        $post->fill([
            'title' => (string) $request->getFiltered('title'),
        ])->save();

        return \response()->json([
            'status' => 'success',
            'data' => $post->toArray(),
        ]);
    }
}
```
