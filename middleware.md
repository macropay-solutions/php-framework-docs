---
title: HTTP Middleware
description: Guide to defining, registering, and managing stateless HTTP middleware within PHP-Framework.
context: middleware
---

# HTTP Middleware

- [Introduction](#introduction)
- [Defining Middleware](#defining-middleware)
  - [*Before* / *After* Middleware](#before-after-middleware)
- [Registering Middleware](#registering-middleware)
  - [Global Middleware](#global-middleware)
  - [Assigning Middleware To Routes](#assigning-middleware-to-routes)
- [Middleware Parameters](#middleware-parameters)
- [Terminable Middleware](#terminable-middleware)
- [Password Confirmation Middleware](#password-confirmation-middleware)

<a name="introduction"></a>
## Introduction

HTTP middleware provide a convenient mechanism for inspecting, filtering, and intercepting HTTP requests entering your stateless application. For example, Framework includes a middleware that verifies if the client making the request is authenticated. If the user is not authenticated, the middleware will return a structured JSON error response or trigger an explicit redirect helper. However, if the user is authenticated, the middleware will allow the request to proceed further into the application.

Of course, additional middleware can be written to perform a variety of tasks tailored specifically for performance-optimized, stateless JSON APIs. A CORS middleware might be responsible for adding the proper headers to all responses leaving your application, while a metrics middleware might log request duration and memory footprint for application telemetry.

> [!NOTE]  
> Automated global view or session variables (such as legacy cookie or session layers) are completely disabled by default to optimize memory allocation. To utilize state-dependent middleware, you must actively complete the explicit component opt-in flow inside your `composer.json` and `App\Application.php` binding layers.

All custom middleware must be stored within the `app/Http/Middleware` directory.

<a name="defining-middleware"></a>
## Defining Middleware

To create a new middleware, copy the base template or duplicate an existing class like `ExampleMiddleware` included within the project baseline. In this example middleware, we will only allow access to the route if the supplied security token matches our secret key via the `getFiltered` parameter scanner. Otherwise, we will redirect the users back to the "home" URI.

    <?php

    namespace App\Http\Middleware;

    use Closure;
    use MacropaySolutions\Kernel\Http\Request;
    use Symfony\Component\HttpFoundation\Response;

    class EnsureTokenIsValid
    {
        /**
         * Handle an incoming request
         */
        public function handle(Request $request, Closure $next): Response
        {
            if ($request->getFiltered('token') !== 'my-secret-token') {
                return redirect('home');
            }

            return $next($request);
        }
    }

As you can see, if the given `token` does not match our secret token, the middleware will return an HTTP redirect response; otherwise, the request will be passed further into the application execution loop. To pass the request deeper into the application (allowing the middleware to "pass"), you call the `$next` callback with the `$request`.

It is best to envision middleware as a series of "layers" HTTP requests must pass through before they trigger a controller action. Each layer can examine the request and even reject it entirely before the application executes core logic.

> [!NOTE]  
> All middleware are resolved directly via the dependency injection [service container](/container), meaning you may type-hint any necessary dependencies you require within your middleware's constructor method.

<a name="before-after-middleware"></a>
### *Before* / *After* Middleware

Whether a middleware runs before or after a request is handled depends on the structural placement of the code relative to the execution of the `$next` callback. For example, the following middleware performs its task **before** the request is handled by the application:

    <?php

    namespace App\Http\Middleware;

    use Closure;
    use MacropaySolutions\Kernel\Http\Request;
    use Symfony\Component\HttpFoundation\Response;

    class BeforeMiddleware
    {
        public function handle(Request $request, Closure $next): Response
        {
            // Perform action before application execution

            return $next($request);
        }
    }

Conversely, this middleware performs its task **after** the request is handled by the application layer:

    <?php

    namespace App\Http\Middleware;

    use Closure;
    use MacropaySolutions\Kernel\Http\Request;
    use Symfony\Component\HttpFoundation\Response;

    class AfterMiddleware
    {
        public function handle(Request $request, Closure $next): Response
        {
            $response = $next($request);

            // Perform action after application execution

            return $response;
        }
    }

<a name="registering-middleware"></a>
## Registering Middleware

<a name="global-middleware"></a>
### Global Middleware

If you want a middleware to run during every single HTTP request sent to your application, you must list it within your core application setup container inside the `app/Application.php` class.

To run the request filter globally **before** the high-performance routing engine attempts to find a matching path, append the class to the `$middleware` array property:

    protected $middleware = [
        \App\Http\Middleware\EnsureTokenIsValid::class,
    ];

To run the request filter globally across all routes **after** a valid route has been evaluated and successfully matched, list the class within the `$foundRouteMiddleware` array property:

    protected array $foundRouteMiddleware = [
        \App\Http\Middleware\AfterRoutingMetricsLogger::class,
    ];

<a name="assigning-middleware-to-routes"></a>
### Assigning Middleware To Routes

> [!WARNING]  
> **Route Closures are Forbidden:** The routing architecture strictly forbids closures and will throw a `RuntimeException`. All routes must point explicitly to a controller class method via string syntax or a structured array mailable layout.

If you would like to assign a specific middleware to individual routes, you may first register a short-hand alias key by passing an array to the `$app->routeMiddleware()` method within your `bootstrap/app.php` file:

    $app->routeMiddleware([
        'auth' => \App\Http\Middleware\Authenticate::class,
        'role' => \App\Http\Middleware\EnsureUserHasRole::class,
    ]);

Once the middleware shorthand has been declared, you can assign it using the `middleware` key option inside your route definition array:

    $router->get('admin/profile', [
        'middleware' => 'auth',
        'uses' => 'App\Http\Controllers\AdminProfileController@show',
    ]);

You may pass an array of aliases to assign multiple middleware steps to a single route:

    $router->get('dashboard', [
        'middleware' => ['first', 'second'],
        'uses' => 'App\Http\Controllers\DashboardController::class@index',
    ]);

> [!NOTE]  
> Utilizing Fully Qualified Names (FQN) inside your middleware options matrix will result in a faster total application boot time by avoiding redundant loop checks.

<a name="middleware-parameters"></a>
## Middleware Parameters

Middleware can also receive custom parameters passed straight from the routing layer. For instance, if your API needs to verify that an authenticated user has an explicit "role" value before allowing a database mutation, you can pass that role type directly.

Custom parameters will be passed as trailing arguments to the middleware's `handle` method immediately following the `$next` closure argument:

    <?php

    namespace App\Http\Middleware;

    use Closure;
    use MacropaySolutions\Kernel\Http\Request;
    use Symfony\Component\HttpFoundation\Response;

    class EnsureUserHasRole
    {
        /**
         * Run the incoming request filter against custom parameters.
         */
        public function handle(Request $request, Closure $next, string $role): Response
        {
            if (!$request->user()->hasRole($role)) {
                return response()->json(['error' => 'Unauthorized Role'], 403); // or throw
            }

            return $next($request);
        }
    }

Middleware parameters are assigned when mapping your route definitions by separating the short-hand middleware alias and the argument string with a `:` character. Multiple parameter values should be separated using commas:

    $router->put('post/{id}', [
        'middleware' => 'role:editor,publisher',
        'uses' => 'App\Http\Controllers\PostController@update',
    ]);

<a name="terminable-middleware"></a>
## Terminable Middleware

Sometimes a middleware may need to complete resource cleanup or background work after the raw HTTP response payload has already been dispatched to the client's browser. For instance, the framework's internal session manager logs stateless payload data to storage *after* the request transmission finishes.

To achieve this behavior, define your middleware as "terminable" by adding a public `terminate` method directly to the class structure:

    <?php

    namespace App\Http\Middleware;

    use Closure;
    use MacropaySolutions\Kernel\Http\Request;
    use Symfony\Component\HttpFoundation\Response;

    class StartSession
    {
        public function handle(Request $request, Closure $next): Response
        {
            return $next($request);
        }

        /**
         * Handle structural tasks after the response has been sent to the client.
         */
        public function terminate(Request $request, Response $response): void
        {
            // Commit session state or flush memory allocations...
        }
    }

The `terminate` method must accept both a `MacropaySolutions\Kernel\Http\Request` and a `Symfony\Component\HttpFoundation\Response` instance. Once implemented, register the terminable middleware as a global or route-specific entry point inside your configuration loops.

When calling the `terminate` method on your middleware, Framework will resolve a fresh instance of the middleware from the [service container](/container). If you would like to use the same middleware instance when the `handle` and `terminate` methods are called, register the middleware with the container using the container's `singleton` method.

    protected function registerExplicitBindingsMap(): void
    {
        $this->bindings = [
            \App\Http\Middleware\StartSession::class => [
                'concrete' => fn($app) => new \App\Http\Middleware\StartSession(),
                'shared' => true // Enforces singleton status
            ],
        ];
    }

<a name="password-confirmation-middleware"></a>
## Password Confirmation Middleware

Framework includes a built-in `RequirePassword` middleware (`MacropaySolutions\Kernel\Auth\Middleware\RequirePassword`) to protect sensitive endpoints—such as updating billing information or changing account settings—by ensuring the user has confirmed their password within a given timeframe.

By default, the password confirmation window remains valid for **3 hours (10,800 seconds)** before requiring re-confirmation via the session (`auth.password_confirmed_at`).

> [!NOTE]  
> **Session Requirement:** Because `RequirePassword` reads from `$request->session()`, sessions must be explicitly enabled on the route or group (e.g., via the `StartSession` middleware). Stateless API routes without an active session layer will be unable to persist the password confirmation timestamp.

#### Registering the Middleware

Register the middleware alias in your `app/Application.php` file:

```php
    protected $routeMiddleware = [
        'password.confirm' => \MacropaySolutions\Kernel\Auth\Middleware\RequirePassword::class,
    ];
// or use it with its full FQN to gain execution speed.
```

#### Protecting Routes

Attach the middleware to sensitive routes:

```php
$router->post('user/security/keys', [
    'middleware' => 'password.confirm',
    'uses' => 'SecurityController@store',
]);
```

#### Customizing Redirect Routes and Timeouts

If an unconfirmed web request hits the endpoint, it redirects to the `password.confirm` named route by default. If the request expects JSON (`Accept: application/json`), the middleware returns a `423 Locked` HTTP response instead.

You can customize the target redirect route or timeout window (in seconds) using the static `using()` helper method:

```php
use MacropaySolutions\Kernel\Auth\Middleware\RequirePassword;

// Custom redirect route and a 15-minute timeout window
$router->post('user/billing/card', [
    'middleware' => RequirePassword::using('auth.custom_confirm', 900),
    'uses' => 'BillingController@update',
]);
```
