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

The `MacropaySolutions\Kernel\Http\Request` instance extends `MacropaySolutions\Kernel\Http\Base\Request` to analyze incoming execution context.

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

Extract bearer tokens directly from the `Authorization` header. If the header is missing, it returns an empty string:

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
    composer require zendframework/zend-diactoros

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

Access user payload components across all HTTP verbs using `input`, `get` or `getFiltered`:

    $name = $request->input('name');
    $name = $request->get('name');
    $name = $request->get('name', 'Sally'); // With default fallback

Safely sanitize parameters with `getFiltered`, which implements native `filter_var` mechanics. Missing fields return an empty string rather than `null`:

    $sanitized = $request->getFiltered('queryParam');

Restrict data retrieval explicitly to query string values using the `query` method:

    $search = $request->query('search');
    $search = $request->query('search', 'default_term');
    $allQuery = $request->query(); // Returns entire query array

This will return a valid string or false (it uses `filter_var` in the background). See `\App\Request::getFiltered` for more details.
Also when the field is missing it will return empty string not null!

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

    class TrustProxies
    {
        protected array $proxies = [
            '192.168.1.1',
            '192.168.1.2',
        ];

        protected int $headers = Request::HEADER_X_FORWARDED_FOR | 
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

    class TrustHosts
    {
        public function hosts(): array
        {
            return [
                'framework.test',
                'api.framework.test',
            ];
        }
    }
