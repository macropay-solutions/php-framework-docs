---
title: HTTP Routing
description: Core HTTP routing definitions, parameters, groups, and domain restrictions for PHP Framework.
context: routing
---

# HTTP Routing

- [Basic Routing](#basic-routing)
- [Cache Routing](#cache-routing)
- [Domain Routing](#domain-routing)
- [Route Parameters](#route-parameters)
  - [Required Parameters](#required-parameters)
  - [Optional Parameters](#optional-parameters)
  - [Regular Expression Constraints](#parameters-regular-expression-constraints)
- [Named Routes](#named-routes)
- [Route Groups](#route-groups)
  - [Middleware](#route-group-middleware)
  - [Namespaces](#route-group-namespaces)
  - [Route Prefixes](#route-group-prefixes)

<a name="basic-routing"></a>
## Basic Routing

You will define All the routes for your application in the `routes/web.php` file. The most basic PHP-Framework routes simply accept a URI and a controller mapping string:

    $router->get('foo', 'HomeController@index');
    $router->post('foo', 'HomeController@store');

    <?php

    namespace App\Http\Controllers;

    use MacropaySolutions\Kernel\Http\Request;
    use MacropaySolutions\Kernel\Http\Response;

    class HomeController extends \MacropaySolutions\Framework\Routing\Controller
    {
        public function store(Request $request): Response
        {
            //
        }
    }

## Cache Routing

These commands are available in PHP-Framework template. Routes are auto-cached on `--no-dev` composer environments via an Ahead-of-Time (AOT) compiler.

    php run route:cache

    php run route:clear

## Domain Routing

PHP-Framework enables restricting routes to specific `domain` and generating **absolute URLs** for named routes via `route` function.

To enable domain enforcement and absolute URL generation, the domain parameter must include valid URLs without trailing slash or port.
- Correct:

  'domain' => 'https://api.example.com'
  'domain' => ['https://api.example.com', 'https://api-custom.example.com']

- Incorrect:

  'domain' => 'api.example.com'
  'domain' => 'https://api.example.com/'
  'domain' => 'https://api.example.com:80'

If a `domain` is provided without a scheme (e.g., `api.example.com`) or app.url has no scheme:
- named routes will fall back to relative paths (e.g., `/users`) because the system cannot safely assume the intended protocol.
- the domain check will be skipped, and the route will resolve on any host hitting the application.

When a valid URL is provided in the route group OR app.url config, all named routes (`route('name')`) will automatically return the full absolute URL.  Example: `https://api.example.com/v1/profile`

The matching logic is port-agnostic. A configuration for `http://localhost` will correctly match a request to `http://localhost:8000`, making it safe for local microservice development.

> **Note:**
>
> All aliases and URIs must be unique despite having multiple domains! Use prefixes for route uri to accomplish that.
>
> When domains is a list of urls, `route('alias')` will use/return the last url from the list.
>
> When same uri is registered multiple times `route('alias')` will use/return the last url from the last registration.
>
> In debug mode same alias used multiple times will be logged.
>
> The `domain` key can be used in route groups.
>
> Create and use/uncomment the GLOBAL (mandatory GLOBAL) middleware \App\Http\Middleware\TrustProxies::class, in \App\Application::registerExplicitBindingsMap or in bootstrap/app.php:

    $app->middleware([
        \App\Http\Middleware\TrustProxies::class,
    ]);

This is needed for proper domain check from current request. See [requests](/requests#configuring-trusted-proxies) for more details. Failing to do this will silently result in 404 responses!

#### Available Router Methods

The router allows you to register routes that respond to any HTTP verb using target controller mapping actions:

    $router->query($uri, $action);
    $router->get($uri, $action);
    $router->post($uri, $action);
    $router->put($uri, $action);
    $router->patch($uri, $action);
    $router->delete($uri, $action);
    $router->options($uri, $action);
    $router->match(['GET', 'POST'], $uri, $action);

### The Action Parameter

The `$action` parameter can be a simple controller mapping string, or an array containing the target controller and additional route attributes (like route names, middleware, or specific domains):

    // String Action
    $router->get('users', 'UserController@index');

    // Array Action
    $router->get('users', [
        'uses' => 'UserController@index',
        'as' => 'users.index',
        'middleware' => 'auth',
        'domain' => 'https://api.example.com'
    ]);

<a name="route-parameters"></a>
## Route Parameters

<a name="required-parameters"></a>
### Required Parameters

Of course, sometimes you will need to capture segments of the URI within your route. For example, you may need to capture a user's ID from the URL. You may do so by defining route parameters:

    $router->get('user/{id}', 'UserController@show');

You may define as many route parameters as required by your route:

    $router->get('posts/{postId}/comments/{commentId}', 'CommentController@show');

Route parameters are always encased within "curly" braces. The parameters will be passed into your controller's method when the route is executed.

> **Note:** Route parameters cannot contain the `-` character. Use an underscore (`_`) instead.

<a name="optional-parameters"></a>
### Optional Parameters

You may define optional route parameters by enclosing part of the route URI definition in `[...]`. So, for example, `/foo[bar]` will match both `/foo` and `/foobar`. Optional parameters are only supported in a trailing position of the URI. In other words, you may not place an optional parameter in the middle of a route definition:

    $router->get('user[/{name}]', 'UserController@index');

<a name="parameters-regular-expression-constraints"></a>
### Regular Expression Constraints

You may constrain the format of your route parameters by defining a regular expression in your route definition. Standard dynamic regex filters do not impact the high-speed routing tier:

    $router->get('user/{name:[A-Za-z]+}', 'UserController@showByName');

<a name="named-routes"></a>
## Named Routes

Named routes allow the convenient generation of URLs for specific routes. You may specify a name for a route using the `as` array key when defining the route:

    $router->get('profile', ['as' => 'profile', 'uses' => 'UserController@showProfile']);

You may also specify route names for controller actions:

    $router->get('profile', [
        'as' => 'profile', 'uses' => 'UserController@showProfile'
    ]);

#### Generating URLs To Named Routes

Once you have assigned a name to a given route, you may use the route's name when generating URLs via the global `route` function:

    $url = route('profile');
    $url = route('profile', ['id' => 3]);

If the named route defines parameters, you may pass the parameters as the second argument to the `route` function. The given parameters will automatically be inserted into the URL in their correct positions:

    $router->get('user/{id}/profile', ['as' => 'profile', 'uses' => 'UserController@showProfile']);

    $url = route('profile', ['id' => 1]);

<a name="route-groups"></a>
## Route Groups

Route groups allow you to share route attributes, such as middleware or namespaces, across a large number of routes without needing to define those attributes on each individual route. Shared attributes are specified in an array format as the first parameter to the `$router->group` method.

To learn more about route groups, we'll walk through several common use-cases for the feature.

<a name="route-group-middleware"></a>
### Middleware

To assign middleware to all routes within a group, you may use the `middleware` key in the group attribute array. Middleware will be executed in the order you define this array:

    $router->group(['middleware' => 'auth'], function () use ($router) {
        $router->get('/', 'DashboardController@index');
        $router->get('user/profile', 'UserController@profile');
    });

<a name="route-group-namespaces"></a>
### Namespaces

Another common use-case for route groups is assigning the same PHP namespace to a group of controllers. You may use the `namespace` parameter in your group attribute array to specify the namespace for all controllers within the group:

    $router->group(['namespace' => 'Admin'], function() use ($router)
    {
        // Using The "App\Http\Controllers\Admin" Namespace...

        $router->group(['namespace' => 'User'], function() use ($router) {
            // Using The "App\Http\Controllers\Admin\User" Namespace...
        });
    });

<a name="route-group-prefixes"></a>
### Route Prefixes

The `prefix` group attribute may be used to prefix each route in the group with a given URI. For example, you may want to prefix all route URIs within the group with `admin`:

    $router->group(['prefix' => 'admin'], function () use ($router) {
        $router->get('users', 'AdminUserController@index');
    });

You may also use the `prefix` parameter to specify common parameters for your grouped routes:

    $router->group(['prefix' => 'accounts/{accountId}'], function () use ($router) {
        $router->get('detail', 'AccountController@detail');
    });

> **Note: The PHP-Kernel Hybrid Trie Engine**
>
> PHP-Kernel features a zero-regex tiered routing design architecture. Requests are resolved via an instant O(1) Hash Shield for static routes, and an O(K) Native Prefix Tree matching paths token-by-token.
>
> **1. Trailing Slash Normalization**
> Because the Trie walking mechanism evaluates string delimiters natively, endpoints like `/users/1` and `/users/1/` map to the identical execution leaf automatically without triggering extra middleware overhead.
>
> **2. The 404 Firewall & 405 Behavior**
> To maximize network scanning defense and eliminate lookup leaks, basic and dynamic routes handled by the native Trie engine will return a `404 Not Found` instead of `405 Method Not Allowed`. However, **Complex Routes handled by the `nikic/fast-route` fallback will still return a `405`** to retain strict compliance.
>
> **3. Understanding Complex Route Fallbacks**
> During the `route:cache` AOT compilation phase, the builder separates standard hot-paths from "Complex Routes." Pure dynamic paths (even those utilizing inline validation parameters like `/user/{id:\d+}`) are **not** complex and run on the raw Trie.
>
> True Complex Routes are isolated into an independent queue and delegated to a highly scoped instance of `FastRoute` only if the initial Trie walk misses:
> * **Complex Path Rules:** Greedy wildcards (`/{any:.*}`), multi-variable inline formats (`/export-{date}.csv`), or adjacent variables without explicit separation (`/{code}{number}`).
>
> For maximum throughput and immunity to resource-draining malicious routing requests, keep your routing file entirely clean of the Complex Queue.

> **CRITICAL ARCHITECTURAL WARNING**
>
> Because PHP-Framework's high-performance router utilizes native `strtok()` loops to parse URI nodes with zero memory allocations, **this framework is strictly designed for a stateless, isolated PHP-FPM execution architecture.**
>
> Do **NOT** run PHP-Framework with long-running, multithreaded, or coroutine-based application servers (such as **Swoole, OpenSwoole, or RoadRunner**).
>
> Because `strtok()` relies on a single global internal pointer within the PHP thread state, concurrent asynchronous requests sharing the same process worker will overwrite each other's routing tokens mid-flight. This will result in critical security vulnerabilities, including routing desynchronization, cross-user data leaks, and authentication middleware bypasses.
