---
title: HTTP Responses
description: Guide to creating and returning HTTP responses, JSON responses, file downloads, and redirects in PHP Framework.
context: responses
---

# HTTP Responses

- [Basic Responses](#basic-responses)
  - [Attaching Headers To Responses](#attaching-headers-to-responses)
- [Other Response Types](#other-response-types)
  - [JSON Responses](#json-responses)
  - [File Downloads](#file-downloads)
- [Redirects](#redirects)
  - [Redirecting To Named Routes](#redirecting-named-routes)

<a name="basic-responses"></a>
## Basic Responses

All routes and controllers should return an HTTP response to be sent back to the user's browser. Framework provides several different ways to return responses. Route closures are strictly forbidden by the framework and will throw a `RuntimeException`, so all routes must point directly to a Controller class method.

The most basic response is returning a string from a controller action:

```php
namespace App\Http\Controllers;

class HomeController extends Controller
{
    public function index(): string
    {
        return 'Hello World';
    }
}
```

You can register the route pointing to this controller method:

```php
$router->get('/', [HomeController::class, 'index']);
```

The returned string will automatically be converted into an HTTP response by the framework.

#### Response Objects

However, for most controller actions, you will be returning a full `MacropaySolutions\Kernel\Http\Response` instance. Returning a full `Response` instance allows you to customize the response's HTTP status code and headers. A `Response` instance inherits from the `Symfony\Component\HttpFoundation\Response` class, providing a variety of methods for building HTTP responses:

```php
namespace App\Http\Controllers;

use MacropaySolutions\Kernel\Http\Response;

class HomeController extends Controller
{
    public function home(): Response
    {
        return (new Response($content, $status))
            ->header('Content-Type', $value);
    }
}
```

For convenience, you may also use the global `response` helper function:

```php
namespace App\Http\Controllers;

use MacropaySolutions\Kernel\Http\Response;

class HomeController extends Controller
{
    public function home(): Response
    {
        return \response($content, $status)
            ->header('Content-Type', $value);
    }
}
```

> [!NOTE]  
> For a full list of available `Response` methods, check out the [Symfony API documentation](https://symfony.com/doc/current/components/http_foundation.html#response).

<a name="attaching-headers-to-responses"></a>
#### Attaching Headers To Responses

Keep in mind that most response methods are chainable, allowing for the fluent building of responses. For example, you may use the `header` method to add a series of headers to the response before sending it back to the user:

```php
return response($content)
    ->header('Content-Type', $type)
    ->header('X-Header-One', 'Header Value')
    ->header('X-Header-Two', 'Header Value');
```

Or, you may use the `withHeaders` method to specify an array of headers to be added to the response:

```php
return response($content)
    ->withHeaders([
        'Content-Type' => $type,
        'X-Header-One' => 'Header Value',
        'X-Header-Two' => 'Header Value',
    ]);
```

<a name="other-response-types"></a>
## Other Response Types

The `response` helper function may be used to conveniently generate other types of response instances. When the `response` helper is called without arguments, an implementation of the `MacropaySolutions\Kernel\Contracts\Routing\ResponseFactory` class is returned. This class provides several helpful methods for generating responses.

<a name="json-responses"></a>
#### JSON Responses

The `json` method will automatically set the `Content-Type` header to `application/json`, as well as convert the given array into JSON using the `json_encode` PHP function:

```php
return response()->json(['name' => 'Abigail', 'state' => 'CA']);
```

You can optionally provide a status code and an array of additional headers:

```php
return response()->json(['error' => 'Unauthorized'], 401, ['X-Header-One' => 'Header Value']);
```

If you would like to create a JSONP response, you may use the `json` method in addition to `setCallback`:

```php
return response()
    ->json(['name' => 'Abigail', 'state' => 'CA'])
    ->setCallback($request->input('callback'));
```

<a name="file-downloads"></a>
#### File Downloads

The `download` method may be used to generate a response that forces the user's browser to download the file at the given path. The `download` method accepts a file name as the second argument, which determines the file name seen by the user downloading the file. Finally, you may pass an array of HTTP headers as the third argument:

```php
return response()->download($pathToFile);
```

Or with custom filename and headers:

```php
return response()->download($pathToFile, $name, $headers);
```

> [!NOTE]  
> Symfony HttpFoundation, which manages file downloads, requires the file being downloaded to have an ASCII file name.

<a name="redirects"></a>
## Redirects

Redirect responses are instances of the `MacropaySolutions\Kernel\Http\RedirectResponse` class, and contain the proper headers needed to redirect the user to another URL. The simplest method to generate a `RedirectResponse` is to use the global `redirect` helper method within a controller action:

```php
$router->get('dashboard', [DashboardController::class, 'index']);
```

```php
namespace App\Http\Controllers;

use MacropaySolutions\Kernel\Http\RedirectResponse;

class DashboardController extends Controller
{
    public function index(): RedirectResponse
    {
        return redirect('home/dashboard');
    }
}
```

<a name="redirecting-named-routes"></a>
#### Redirecting To Named Routes

When you call the `redirect` helper function with no parameters, an instance of `MacropaySolutions\Kernel\Routing\Redirector` is returned, allowing you to call any method on the `Redirector` instance. For example, to generate a `RedirectResponse` to a named route, you may use the `route` method:

```php
return redirect()->route('login');
```

If your route has parameters, you may pass them as the second argument to the `route` method:

```php
// For a route with the following URI: profile/{id}

return redirect()->route('profile', ['id' => 1]);
```

If you are redirecting to a route with an "ID" parameter that is being populated from an Obvious model, you may simply pass the model itself. The ID will be extracted automatically:

```php
return redirect()->route('profile', [$user]);
```
