---
title: HTTP Requests
description: Accessing and processing incoming HTTP requests, input data, headers, and file uploads.
context: requests
---

# HTTP Requests

- [Accessing The Request](#accessing-the-request)
  - [Basic Request Information](#basic-request-information)
  - [Request Headers](#request-headers)
  - [Request IP Address](#request-ip-address)
  - [Content Negotiation](#content-negotiation)
  - [PSR-7 Requests](#psr-7-requests)
- [Retrieving Input](#retrieving-input)
  - [Where Each Accessor Reads From](#where-each-accessor-reads-from)
  - [Automatic Stream Wrapper Protection (Security)](#automatic-stream-wrapper-protection-security)
  - [Input Presence and Mutation](#input-presence-and-mutation)
  - [Cookies](#cookies)
  - [Files](#files)
- [Configuring Trusted Proxies](#configuring-trusted-proxies)
- [Configuring Trusted Hosts](#configuring-trusted-hosts)

<a name="accessing-the-request"></a>
## Accessing The Request

Obtain the current HTTP request instance via dependency injection by type-hinting `MacropaySolutions\Kernel\Http\Request` on your controller method. The service container injects the instance automatically:

    <?php

    namespace App\Http\Controllers;

    use MacropaySolutions\Kernel\Http\Request;
    use MacropaySolutions\Kernel\Http\Response;

    class UserController extends \MacropaySolutions\Framework\Routing\Controller
    {
        public function store(Request $request): Response
        {
            $slowName = $request->input('name');
            $fasterName = $request->get('name');
            $filteredName = $request->getFiltered('name');
        }
    }

If the controller method requires input from a route parameter, list the route arguments after the injected dependencies:

    $router->put('user/{id}', 'UserController@update');

    public function update(Request $request, string $id): Response
    {
        // Access route parameter $id directly
    }

<a name="basic-request-information"></a>
### Basic Request Information

The `MacropaySolutions\Kernel\Http\Request` instance extends `Symfony\Component\HttpFoundation\Request` to analyze incoming execution context.

#### Retrieving The Request URI and Host

The `path` method returns the request's URI path info. If the incoming request targets `http://domain.com/foo/bar`, `path` returns `foo/bar`:

    $uri = $request->path();

The `is` method verifies if the incoming request URI matches a pattern string. The `*` character acts as a wildcard:

    if ($request->is('admin/*')) {
        //
    }

Retrieve URLs or the request host with the following methods:

    $url = $request->url(); // Without Query String
    $url = $request->fullUrl(); // With Query String

    // Host utilities
    $host = $request->host();
    $httpHost = $request->httpHost();
    $schemeAndHttpHost = $request->schemeAndHttpHost();

Modify the current query string dynamically with URL generation extensions:

    // Appends parameters
    $url = $request->fullUrlWithQuery(['type' => 'phone']);

    // Strips parameters
    $url = $request->fullUrlWithoutQuery(['type']);

#### Retrieving The Request Method

    $method = $request->method();

    if ($request->isMethod('POST')) {
        //
    }

<a name="request-headers"></a>
### Request Headers

Retrieve a request header using the `header` method. If absent, it returns `null`. Pass a fallback value as the optional second argument:

    $value = $request->header('X-Header-Name');
    $value = $request->header('X-Header-Name', 'default');

    if ($request->hasHeader('X-Header-Name')) {
        // Header exists
    }

Extract bearer tokens directly from the `Authorization` header. If the header is missing, it returns `null`:

    $token = $request->bearerToken();

<a name="request-ip-address"></a>
### Request IP Address

    $ipAddress = $request->ip();

    // Returns all client IP addresses forwarded by proxies
    $ipAddresses = $request->ips();

<a name="content-negotiation"></a>
### Content Negotiation

Analyze the `Accept` header format using built-in helper wrappers:

    $contentTypes = $request->getAcceptableContentTypes();

    if ($request->accepts(['text/html', 'application/json'])) {
        //
    }

    $preferred = $request->prefers(['text/html', 'application/json']);

    if ($request->expectsJson()) {
        // Request expects JSON response
    }

<a name="psr7-requests"></a>
### PSR-7 Requests

To utilize PSR-7 standard interfaces for incoming HTTP messages, install the necessary bridging components:

    composer require symfony/psr-http-message-bridge
    composer require nyholm/psr7

Type-hint the compliant PSR-7 interface directly on your controller definitions:

    <?php

    namespace App\Http\Controllers;

    class UserController extends \MacropaySolutions\Framework\Routing\Controller
    {
        public function store(\Psr\Http\Message\ServerRequestInterface $request): Response
        {
            $queryParams = $request->getQueryParams();
            $bodyParams = $request->getParsedBody();

            $name = $bodyParams['name'] ?? null;
        }
    }

<a name="retrieving-input"></a>
## Retrieving Input

#### Retrieving Input Values

Access user payload components across all HTTP verbs using `input`, `get` or `getFiltered`. They do not read the same sources in the same order: see [Where Each Accessor Reads From](#where-each-accessor-reads-from) before choosing one.

    $name = $request->input('name');
    $name = $request->get('name');
    $name = $request->get('name', 'Sally'); // With default fallback

Safely sanitize parameters with `getFiltered`, which implements native `filter_var` mechanics. Missing fields return an empty string rather than `null`:

    $sanitized = $request->getFiltered('queryParam');

It uses `filter_var` in the background (see `Request::getFiltered` for details). With the default filter:

| Input | Result |
|---|---|
| a string | the string |
| a scalar (`5`, `true` from JSON) | the value as a string (`"5"`, `"1"`) |
| an array or object | `false`, or `null` with `options: \FILTER_NULL_ON_FAILURE` |
| missing | `''` (empty string, not `null`) |

So `(string)$request->getFiltered('name')` is always a string, and an array sent instead of a string becomes `''`.

Restrict data retrieval explicitly to query string values using the `query` method:

    $search = $request->query('search');
    $search = $request->query('search', 'default_term');
    $allQuery = $request->query(); // Returns entire query array

<a name="where-each-accessor-reads-from"></a>
### Where Each Accessor Reads From

A request carries values from four sources: the client input (body and query string), uploaded files, the route parameters (path) and the request attributes (data set by your middleware, never by the client). The accessors do not combine them the same way:

| Accessor | Attributes | Body / query                                                                  | Files | Route parameters | Dot notation |
|---|---|-------------------------------------------------------------------------------|---|---|---|
| `get()`, `getFiltered()` | **first** | when: POST/PUT/PATCH/QUERY body before query, other methods query before body | no | **never** | no |
| `input()` | no | body merged over query (GET/HEAD: query only)                                 | no | no | yes |
| `query()` | no | query only                                                                    | no | no | no |
| `all()`, `only()`, `except()`, `has()`, `filled()` | no | as `input()`                                                                  | **yes** (a file wins over a field with the same name) | no | yes |
| `$request->name`, `$request['name']` | no | as `all()`                                                                    | yes | **last** (fallback) | yes |
| `isset($request['name'])` | no | as `all()`                                                                    | yes | yes | yes |
| `isset($request->name)` | no | as `$request->name` | yes | yes | yes |
| `route('name')` | no | no                                                                            | no | only | no |
| `file('name')`, `hasFile('name')` | no | no                                                                            | only | no | yes |
| `attributes->get('name')` | only | no                                                                            | no | no | no |

Route parameters are never stored in the attributes: the router keeps the current route as `[status, action, parameters]` and `route()` reads it. Without a route (console, queue workers) `route()` returns `null` and `route('name', $default)` returns `$default`.

Consequences to keep in mind:

- For `/users/{id}?id=5`, `$request->id` returns the query's `5`, `getFiltered('id')` returns `5` and `route('id')` returns the path value.
- An attribute set by middleware overrides client input with the same name in `get()` and `getFiltered()`, but not in `input()`, `all()` or `$request->name`. Prefix your attribute keys so they never match an input name.
- `getFiltered('id')` returns `''` when `id` is only in the path; it never falls back to the route.
- `$request->photo` returns the `UploadedFile` when a file was sent, while `getFiltered('photo')` returns `''`; use `hasFile()` to check for files.
- The two `isset()` forms disagree: `isset($request->name)` is `false` when the value is `null`, like normal PHP, while `isset($request['name'])` is `true` whenever the key exists, even with a `null` value. To check whether a value was sent, use `has()` (the key exists) or `filled()` (it exists and is not empty) instead of either.

**Read every value from its own source:**

    $id = $request->route('id');                         // path
    $name = (string)$request->getFiltered('name');       // client input
    $photo = $request->file('photo');                    // uploaded file
    $clientId = $request->attributes->get('_client_id'); // set by your middleware

Avoid dynamic properties (`$request->name`) and array access (`$request['name']`): they merge input, files and route parameters, return untyped values, and rebuild the merged input on every call.

#### Native JSON Ingestion (Zero-Overhead)

PHP-Kernel intercepts `application/json` or `+json` Content-Types at the absolute lowest level (`Request::capture()`). 

It directly parses `php://input` natively and hydrates the base request object before the framework even fully boots. This bypasses Symfony's internal logic entirely, meaning JSON payload values are instantly available with zero processing overhead using the standard data retrieval methods:

    $name = $request->get('user_name');
    $name = $request->getFiltered('user_name');

You do not need to use the dedicated `$request->json()` method; JSON payloads are treated natively as standard input.

#### Form Array and JSON Traversal fallback

When working with explicitly verified array data structure inputs or JSON payloads where dot-notation traversal is mandatory, utilize `input` carefully:

    $name = $request->input('products.0.name');
    $names = $request->input('products.*.name');

#### Type Casting Input Wrappers

    // Boolean resolution (Returns true for 1, "1", true, "true", "on", "yes")
    $archived = $request->boolean('archived');

    // Date resolution (Returns Carbon instance)
    $birthday = $request->date('birthday');
    $elapsed = $request->date('elapsed', '!H:i', 'Europe/Madrid');

    // Backed Enum resolution
    $status = $request->enum('status', App\Enums\Status::class);

#### Bulk Payload Subsets

    $allInput = $request->all();

    $subset = $request->only(['username', 'password']);
    $subset = $request->only('username', 'password');

    $excluded = $request->except(['credit_card']);
    $excluded = $request->except('credit_card');

### Automatic Stream Wrapper Protection (Security)

By default, the framework provides a defense-in-depth layer to help mitigate Local File Inclusion (LFI) and Remote Code Execution (RCE) vulnerabilities leveraging malicious stream wrappers (`php://` and `phar://`).

This automatic guard recursively inspects both array keys and values across all primary request data fetchers, including `get()`, `post()`, `cookie()`, `input()`, `query()`, `route()`, `all()`, `header()`, `server()`, `file()`, `json()`, dynamic properties (`__get`), and `getFiltered()`.

If a dangerous stream wrapper is detected, the framework will silently strip the payload and return your specified `$default` value (or `null`) instead:

    // If 'theme' contains "php://filter/read=...", this safely returns 'default.php'
    $theme = $request->get('theme', 'default.php');
    
    // If an attacker smuggles a wrapper in an array key, the key is safely unset.
    $payload = $request->input('payload');

While this regex permits normal `http://` and `https://` URLs, it targets strings starting with `php://` or `phar://` (including chained wrappers like compress.zlib://php://). Legitimate inputs starting with these schemes (e.g., in documentation or markdown editors) will be filtered.

> [!WARNING]
> This is a defense-in-depth measure, not a complete RCE boundary. Values can still bypass this guard if developers read from raw property bags (e.g., `$request->query->get('key')`) or manually decode/transform values after retrieval. For file operations, you must still implement strict allowlists mapping user-facing identifiers to server-controlled paths.

> [!NOTE] To fetch the raw value use: `$request->query->get('key')`, `$request->request->get('key')` and so on. Beware that this removes the protection unless you wrap the result into `$request->filterDangerousWrapper(....);`
> 
> If a dangerous stream wrapper is detected in route parameters, the route will not be matched, resulting in 404 response.

<a name="input-presence-and-mutation"></a>
### Input Presence and Mutation

    if ($request->has('name')) {}
    if ($request->has(['name', 'email'])) {}
    if ($request->hasAny(['name', 'email'])) {}

    // Executed only if value is present
    $request->whenHas('name', function (string $input) {
        // 
    });

    // Check if value is present and is not an empty string
    if ($request->filled('name')) {}
    if ($request->anyFilled(['name', 'email'])) {}

    if ($request->missing('name')) {}

#### Mutating Input Mid-Flight

Manually inject or overwrite parameter data keys inside the active request instance payload using the following mutation utilities:

    $request->merge(['votes' => 0]);

    $request->mergeIfMissing(['votes' => 0]);

    $request->forceReplace(['votes' => 0]);

    $request->forceOffsetUnset('votes');

<a name="cookies"></a>
### Cookies

Retrieve processed incoming cookie attributes directly from the request instance. Cookie values are verified against encryption tamper signs automatically:

    $value = $request->cookie('cookie_name');

<a name="files"></a>
### Files

#### Retrieving and Validating Uploaded Files

Access file inputs from the request wrapper via the `file` method. This extracts an `MacropaySolutions\Kernel\Http\UploadedFile` instance:

    $file = $request->file('photo');

    if ($request->hasFile('photo')) {
        // File exists
    }

    if ($request->file('photo')->isValid()) {
        // File uploaded without serialization errors
    }

Extract target properties from the instance directly:

    $path = $request->file('photo')->path();
    $extension = $request->file('photo')->extension();

#### Storing Uploaded Files

If the system utilizes the native `MacropaySolutions\Kernel\Filesystem` service configuration provider layers, file objects can be dispatched straight to target storage adapters (local, public, or cloud systems like S3) using the `store` engine:

    $path = $request->file('photo')->store('images');
    $path = $request->file('photo')->store('images', 's3');

    // Store with explicit naming structure
    $path = $request->file('photo')->storeAs('images', 'filename.jpg');
    $path = $request->file('photo')->storeAs('images', 'filename.jpg', 's3');

#### Manual File Movement

To move files strictly bypassing storage manager virtualization abstraction logic, invoke `move` to relocate the resource directly within physical disk sectors:

    $request->file('photo')->move($destinationPath);
    $request->file('photo')->move($destinationPath, $fileName);

<a name="configuring-trusted-proxies"></a>
## Configuring Trusted Proxies

When running applications behind network load balancers terminating TLS certificates, customize trusted upstream network proxy configurations explicitly within your global proxy middleware components to guarantee accurate request schema evaluation:

    <?php

    namespace App\Http\Middleware;

    use MacropaySolutions\Kernel\Http\Request;

    class TrustProxies extends \MacropaySolutions\Kernel\Http\Middleware\TrustProxies
    {
        protected $proxies = [
            '192.168.1.1',
            '192.168.1.2',
        ];

        protected $headers = Request::HEADER_X_FORWARDED_FOR | 
            Request::HEADER_X_FORWARDED_HOST | 
            Request::HEADER_X_FORWARDED_PORT | 
            Request::HEADER_X_FORWARDED_PROTO;
    }

To trust all forwarding cloud network components unconditionally where absolute backend subnets are unknown, utilize wildcard markers:

    protected $proxies = '*';

<a name="configuring-trusted-hosts"></a>
## Configuring Trusted Hosts

To defend applications from HTTP Host header injection vulnerabilities, configure custom middleware filters to inspect and strictly limit accepted domain configurations allowed to penetrate the runtime layer:

    <?php

    namespace App\Http\Middleware;

    class TrustHosts extends \MacropaySolutions\Kernel\Http\Middleware\TrustHosts
    {
        public function hosts(): array
        {
            return [
                'framework.test',
                'api.framework.test',
            ];
        }
    }
