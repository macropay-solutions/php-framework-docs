---
title: HTTP Client
description: Making outgoing HTTP requests using PHP Framework.
context: http-client
---
# HTTP Client

- [Introduction](#introduction)
- [Making Requests](#making-requests)
    - [Request Data](#request-data)
    - [Headers](#headers)
    - [Authentication](#authentication)
    - [Timeout](#timeout)
    - [Retries](#retries)
    - [Error Handling](#error-handling)
    - [Guzzle Middleware](#guzzle-middleware)
    - [Guzzle Options](#guzzle-options)
- [Concurrent Requests](#concurrent-requests)
- [Macros](#macros)
- [Testing](#testing)
    - [Faking Responses](#faking-responses)
    - [Inspecting Requests](#inspecting-requests)
    - [Preventing Stray Requests](#preventing-stray-requests)
- [Events](#events)

<a name="introduction"></a>
## Introduction

Framework provides an expressive, minimal API around the [Guzzle HTTP client](http://docs.guzzlephp.org/en/stable/), allowing you to quickly make outgoing HTTP requests to communicate with other web applications. Framework's wrapper around Guzzle is focused on its most common use cases and a wonderful developer experience.

Before getting started, you should ensure that you have installed the Guzzle package as a dependency of your application. By default, Framework automatically includes this dependency. However, if you have previously removed the package, you may install it again via Composer:

```shell
composer require guzzlehttp/guzzle
```

<a name="making-requests"></a>
## Making Requests

To make requests, you may use the `head`, `get`, `post`, `put`, `patch`, and `delete` methods provided by the `app(\MacropaySolutions\Kernel\Http\Client\Factory::class')`. First, let's examine how to make a basic `GET` request to another URL:



    $response = \app(\MacropaySolutions\Kernel\Http\Client\Factory::class)->get('http://example.com');

The `get` method returns an instance of `MacropaySolutions\Kernel\Http\Client\Response`, which provides a variety of methods that may be used to inspect the response:

    $response->body() : string;
    $response->json($key = null, $default = null) : array|mixed;
    $response->object() : object;
    $response->collect($key = null) : MacropaySolutions\Kernel\Support\Collection;
    $response->status() : int;
    $response->successful() : bool;
    $response->redirect(): bool;
    $response->failed() : bool;
    $response->clientError() : bool;
    $response->header($header) : string;
    $response->headers() : array;

The `MacropaySolutions\Kernel\Http\Client\Response` object also implements the PHP `ArrayAccess` interface, allowing you to access JSON response data directly on the response:

    return \app(\MacropaySolutions\Kernel\Http\Client\Factory::class)->get('http://example.com/users/1')['name'];

In addition to the response methods listed above, the following methods may be used to determine if the response has a given status code:

    $response->ok() : bool;                  // 200 OK
    $response->created() : bool;             // 201 Created
    $response->accepted() : bool;            // 202 Accepted
    $response->noContent() : bool;           // 204 No Content
    $response->movedPermanently() : bool;    // 301 Moved Permanently
    $response->found() : bool;               // 302 Found
    $response->badRequest() : bool;          // 400 Bad Request
    $response->unauthorized() : bool;        // 401 Unauthorized
    $response->paymentRequired() : bool;     // 402 Payment Required
    $response->forbidden() : bool;           // 403 Forbidden
    $response->notFound() : bool;            // 404 Not Found
    $response->requestTimeout() : bool;      // 408 Request Timeout
    $response->conflict() : bool;            // 409 Conflict
    $response->unprocessableEntity() : bool; // 422 Unprocessable Entity
    $response->tooManyRequests() : bool;     // 429 Too Many Requests
    $response->serverError() : bool;         // 500 Internal Server Error

<a name="uri-templates"></a>
#### URI Templates

The HTTP client also allows you to construct request URLs using the [URI template specification](https://www.rfc-editor.org/rfc/rfc6570). To define the URL parameters that can be expanded by your URI template, you may use the `withUrlParameters` method:

```php
\app(\MacropaySolutions\Kernel\Http\Client\Factory::class)->withUrlParameters([
    'endpoint' => 'https://macropay-solutions.com',
    'page' => 'docs',
    'version' => '9.x',
    'topic' => 'validation',
])->get('{+endpoint}/{page}/{version}/{topic}');
```

<a name="dumping-requests"></a>
#### Dumping Requests

If you would like to dump the outgoing request instance before it is sent and terminate the script's execution, you may add the `dd` method to the beginning of your request definition:

    return \app(\MacropaySolutions\Kernel\Http\Client\Factory::class)->dd()->get('http://example.com');

<a name="request-data"></a>
### Request Data

Of course, it is common when making `POST`, `PUT`, and `PATCH` requests to send additional data with your request, so these methods accept an array of data as their second argument. By default, data will be sent using the `application/json` content type:



    $response = \app(\MacropaySolutions\Kernel\Http\Client\Factory::class)->post('http://example.com/users', [
        'name' => 'Steve',
        'role' => 'Network Administrator',
    ]);

<a name="get-request-query-parameters"></a>
#### GET Request Query Parameters

When making `GET` requests, you may either append a query string to the URL directly or pass an array of key / value pairs as the second argument to the `get` method:

    $response = \app(\MacropaySolutions\Kernel\Http\Client\Factory::class)->get('http://example.com/users', [
        'name' => 'Surname',
        'page' => 1,
    ]);

Alternatively, the `withQueryParameters` method may be used:

    \app(\MacropaySolutions\Kernel\Http\Client\Factory::class)->retry(3, 100)->withQueryParameters([
        'name' => 'Surname',
        'page' => 1,
    ])->get('http://example.com/users')

<a name="sending-form-url-encoded-requests"></a>
#### Sending Form URL Encoded Requests

If you would like to send data using the `application/x-www-form-urlencoded` content type, you should call the `asForm` method before making your request:

    $response = \app(\MacropaySolutions\Kernel\Http\Client\Factory::class)->asForm()->post('http://example.com/users', [
        'name' => 'Sara',
        'role' => 'Privacy Consultant',
    ]);

<a name="sending-a-raw-request-body"></a>
#### Sending a Raw Request Body

You may use the `withBody` method if you would like to provide a raw request body when making a request. The content type may be provided via the method's second argument:

    $response = \app(\MacropaySolutions\Kernel\Http\Client\Factory::class)->withBody(
        base64_encode($photo), 'image/jpeg'
    )->post('http://example.com/photo');

<a name="multi-part-requests"></a>
#### Multi-Part Requests

If you would like to send files as multi-part requests, you should call the `attach` method before making your request. This method accepts the name of the file and its contents. If needed, you may provide a third argument which will be considered the file's filename, while a fourth argument may be used to provide headers associated with the file:

    $response = \app(\MacropaySolutions\Kernel\Http\Client\Factory::class)->attach(
        'attachment', file_get_contents('photo.jpg'), 'photo.jpg', ['Content-Type' => 'image/jpeg']
    )->post('http://example.com/attachments');

Instead of passing the raw contents of a file, you may pass a stream resource:

    $photo = fopen('photo.jpg', 'r');

    $response = \app(\MacropaySolutions\Kernel\Http\Client\Factory::class)->attach(
        'attachment', $photo, 'photo.jpg'
    )->post('http://example.com/attachments');

<a name="headers"></a>
### Headers

Headers may be added to requests using the `withHeaders` method. This `withHeaders` method accepts an array of key / value pairs:

    $response = \app(\MacropaySolutions\Kernel\Http\Client\Factory::class)->withHeaders([
        'X-First' => 'foo',
        'X-Second' => 'bar'
    ])->post('http://example.com/users', [
        'name' => 'Surname',
    ]);

You may use the `accept` method to specify the content type that your application is expecting in response to your request:

    $response = \app(\MacropaySolutions\Kernel\Http\Client\Factory::class)->accept('application/json')->get('http://example.com/users');

For convenience, you may use the `acceptJson` method to quickly specify that your application expects the `application/json` content type in response to your request:

    $response = \app(\MacropaySolutions\Kernel\Http\Client\Factory::class)->acceptJson()->get('http://example.com/users');

The `withHeaders` method merges new headers into the request's existing headers. If needed, you may replace all the headers entirely using the `replaceHeaders` method:

```php
$response = \app(\MacropaySolutions\Kernel\Http\Client\Factory::class)->withHeaders([
    'X-Original' => 'foo',
])->replaceHeaders([
    'X-Replacement' => 'bar',
])->post('http://example.com/users', [
    'name' => 'Surname',
]);
```

<a name="authentication"></a>
### Authentication

You may specify basic and digest authentication credentials using the `withBasicAuth` and `withDigestAuth` methods, respectively:

    // Basic authentication...
    $response = \app(\MacropaySolutions\Kernel\Http\Client\Factory::class)->withBasicAuth('name@domain.com', 'secret')->post(/* ... */);

    // Digest authentication...
    $response = \app(\MacropaySolutions\Kernel\Http\Client\Factory::class)->withDigestAuth('name@domain.com', 'secret')->post(/* ... */);

<a name="bearer-tokens"></a>
#### Bearer Tokens

If you would like to quickly add a bearer token to the request's `Authorization` header, you may use the `withToken` method:

    $response = \app(\MacropaySolutions\Kernel\Http\Client\Factory::class)->withToken('token')->post(/* ... */);

<a name="timeout"></a>
### Timeout

The `timeout` method may be used to specify the maximum number of seconds to wait for a response. By default, the HTTP client will timeout after 30 seconds:

    $response = \app(\MacropaySolutions\Kernel\Http\Client\Factory::class)->timeout(3)->get(/* ... */);

If the given timeout is exceeded, an instance of `MacropaySolutions\Kernel\Http\Client\ConnectionException` will  be thrown.

You may specify the maximum number of seconds to wait while trying to connect to a server using the `connectTimeout` method:

    $response = \app(\MacropaySolutions\Kernel\Http\Client\Factory::class)->connectTimeout(3)->get(/* ... */);

<a name="retries"></a>
### Retries

If you would like the HTTP client to automatically retry the request if a client or server error occurs, you may use the `retry` method. The `retry` method accepts the maximum number of times the request should be attempted and the number of milliseconds that Framework should wait in between attempts:

    $response = \app(\MacropaySolutions\Kernel\Http\Client\Factory::class)->retry(3, 100)->post(/* ... */);

If you would like to manually calculate the number of milliseconds to sleep between attempts, you may pass a closure as the second argument to the `retry` method:

    use Exception;

    $response = \app(\MacropaySolutions\Kernel\Http\Client\Factory::class)->retry(3, function (int $attempt, Exception $exception) {
        return $attempt * 100;
    })->post(/* ... */);

For convenience, you may also provide an array as the first argument to the `retry` method. This array will be used to determine how many milliseconds to sleep between subsequent attempts:

    $response = \app(\MacropaySolutions\Kernel\Http\Client\Factory::class)->retry([100, 200])->post(/* ... */);

If needed, you may pass a third argument to the `retry` method. The third argument should be a callable that determines if the retries should actually be attempted. For example, you may wish to only retry the request if the initial request encounters an `ConnectionException`:

    use Exception;
    use MacropaySolutions\Kernel\Http\Client\PendingRequest;

    $response = \app(\MacropaySolutions\Kernel\Http\Client\Factory::class)->retry(3, 100, function (Exception $exception, PendingRequest $request) {
        return $exception instanceof ConnectionException;
    })->post(/* ... */);

If a request attempt fails, you may wish to make a change to the request before a new attempt is made. You can achieve this by modifying the request argument provided to the callable you provided to the `retry` method. For example, you might want to retry the request with a new authorization token if the first attempt returned an authentication error:

    use Exception;
    use MacropaySolutions\Kernel\Http\Client\PendingRequest;
    use MacropaySolutions\Kernel\Http\Client\RequestException;

    $response = \app(\MacropaySolutions\Kernel\Http\Client\Factory::class)->withToken($this->getToken())->retry(2, 0, function (Exception $exception, PendingRequest $request) {
        if (!$exception instanceof RequestException || $exception->response->status() !== 401) {
            return false;
        }

        $request->withToken($this->getNewToken());

        return true;
    })->post(/* ... */);

If all the requests fail, an instance of `MacropaySolutions\Kernel\Http\Client\RequestException` will be thrown. If you would like to disable this behavior, you may provide a `throw` argument with a value of `false`. When disabled, the last response received by the client will be returned after all retries have been attempted:

    $response = \app(\MacropaySolutions\Kernel\Http\Client\Factory::class)->retry(3, 100, throw: false)->post(/* ... */);

> [!WARNING]  
> If all the requests fail because of a connection issue, a `MacropaySolutions\Kernel\Http\Client\ConnectionException` will still be thrown even when the `throw` argument is set to `false`.

<a name="error-handling"></a>
### Error Handling

Unlike Guzzle's default behavior, Framework's HTTP client wrapper does not throw exceptions on client or server errors (`400` and `500` level responses from servers). You may determine if one of these errors was returned using the `successful`, `clientError`, or `serverError` methods:

    // Determine if the status code is >= 200 and < 300...
    $response->successful();

    // Determine if the status code is >= 400...
    $response->failed();

    // Determine if the response has a 400 level status code...
    $response->clientError();

    // Determine if the response has a 500 level status code...
    $response->serverError();

    // Immediately execute the given callback if there was a client or server error...
    $response->onError(callable $callback);

<a name="throwing-exceptions"></a>
#### Throwing Exceptions

If you have a response instance and would like to throw an instance of `MacropaySolutions\Kernel\Http\Client\RequestException` if the response status code indicates a client or server error, you may use the `throw` or `throwIf` methods:

    use MacropaySolutions\Kernel\Http\Client\Response;

    $response = \app(\MacropaySolutions\Kernel\Http\Client\Factory::class)->post(/* ... */);

    // Throw an exception if a client or server error occurred...
    $response->throw();

    // Throw an exception if an error occurred and the given condition is true...
    $response->throwIf($condition);

    // Throw an exception if an error occurred and the given closure resolves to true...
    $response->throwIf(fn (Response $response) => true);

    // Throw an exception if an error occurred and the given condition is false...
    $response->throwUnless($condition);

    // Throw an exception if an error occurred and the given closure resolves to false...
    $response->throwUnless(fn (Response $response) => false);

    // Throw an exception if the response has a specific status code...
    $response->throwIfStatus(403);

    // Throw an exception unless the response has a specific status code...
    $response->throwUnlessStatus(200);

    return $response['user']['id'];

The `MacropaySolutions\Kernel\Http\Client\RequestException` instance has a public `$response` property which will allow you to inspect the returned response.

The `throw` method returns the response instance if no error occurred, allowing you to chain other operations onto the `throw` method:

    return \app(\MacropaySolutions\Kernel\Http\Client\Factory::class)->post(/* ... */)->throw()->json();

If you would like to perform some additional logic before the exception is thrown, you may pass a closure to the `throw` method. The exception will be thrown automatically after the closure is invoked, so you do not need to re-throw the exception from within the closure:

    use MacropaySolutions\Kernel\Http\Client\Response;
    use MacropaySolutions\Kernel\Http\Client\RequestException;

    return \app(\MacropaySolutions\Kernel\Http\Client\Factory::class)->post(/* ... */)->throw(function (Response $response, RequestException $e) {
        // ...
    })->json();

<a name="guzzle-middleware"></a>
### Guzzle Middleware

Since Framework's HTTP client is powered by Guzzle, you may take advantage of [Guzzle Middleware](https://docs.guzzlephp.org/en/stable/handlers-and-middleware.html) to manipulate the outgoing request or inspect the incoming response. To manipulate the outgoing request, register a Guzzle middleware via the `withRequestMiddleware` method:


    use Psr\Http\Message\RequestInterface;

    $response = \app(\MacropaySolutions\Kernel\Http\Client\Factory::class)->withRequestMiddleware(
        function (RequestInterface $request) {
            return $request->withHeader('X-Example', 'Value');
        }
    )->get('http://example.com');

Likewise, you can inspect the incoming HTTP response by registering a middleware via the `withResponseMiddleware` method:


    use Psr\Http\Message\ResponseInterface;

    $response = \app(\MacropaySolutions\Kernel\Http\Client\Factory::class)->withResponseMiddleware(
        function (ResponseInterface $response) {
            $header = $response->getHeader('X-Example');

            // ...

            return $response;
        }
    )->get('http://example.com');

<a name="global-middleware"></a>
#### Global Middleware

Sometimes, you may want to register a middleware that applies to every outgoing request and incoming response. To accomplish this, you may use the `globalRequestMiddleware` and `globalResponseMiddleware` methods. Typically, these methods should be invoked in the `boot` method of your application's `AppServiceProvider`:

```php

\app(\MacropaySolutions\Kernel\Http\Client\Factory::class)->globalRequestMiddleware(fn ($request) => $request->withHeader(
    'User-Agent', 'Example Application/1.0'
));

\app(\MacropaySolutions\Kernel\Http\Client\Factory::class)->globalResponseMiddleware(fn ($response) => $response->withHeader(
    'X-Finished-At', now()->toDateTimeString()
));
```

<a name="guzzle-options"></a>
### Guzzle Options

You may specify additional [Guzzle request options](http://docs.guzzlephp.org/en/stable/request-options.html) using the `withOptions` method. The `withOptions` method accepts an array of key / value pairs:

    $response = \app(\MacropaySolutions\Kernel\Http\Client\Factory::class)->withOptions([
        'debug' => true,
    ])->get('http://example.com/users');

<a name="concurrent-requests"></a>
## Concurrent Requests

Sometimes, you may wish to make multiple HTTP requests concurrently. In other words, you want several requests to be dispatched at the same time instead of issuing the requests sequentially. This can lead to substantial performance improvements when interacting with slow HTTP APIs.

Thankfully, you may accomplish this using the `pool` method. The `pool` method accepts a closure which receives an `MacropaySolutions\Kernel\Http\Client\Pool` instance, allowing you to easily add requests to the request pool for dispatching:

    use MacropaySolutions\Kernel\Http\Client\Pool;


    $responses = \app(\MacropaySolutions\Kernel\Http\Client\Factory::class)->pool(fn (Pool $pool) => [
        $pool->get('http://localhost/first'),
        $pool->get('http://localhost/second'),
        $pool->get('http://localhost/third'),
    ]);

    return $responses[0]->ok() &&
           $responses[1]->ok() &&
           $responses[2]->ok();

As you can see, each response instance can be accessed based on the order it was added to the pool. If you wish, you can name the requests using the `as` method, which allows you to access the corresponding responses by name:

    use MacropaySolutions\Kernel\Http\Client\Pool;


    $responses = \app(\MacropaySolutions\Kernel\Http\Client\Factory::class)->pool(fn (Pool $pool) => [
        $pool->as('first')->get('http://localhost/first'),
        $pool->as('second')->get('http://localhost/second'),
        $pool->as('third')->get('http://localhost/third'),
    ]);

    return $responses['first']->ok();

<a name="customizing-concurrent-requests"></a>
#### Customizing Concurrent Requests

The `pool` method cannot be chained with other HTTP client methods such as the `withHeaders` or `middleware` methods. If you want to apply custom headers or middleware to pooled requests, you should configure those options on each request in the pool:

```php
use MacropaySolutions\Kernel\Http\Client\Pool;

$headers = [
    'X-Example' => 'example',
];

$responses = \app(\MacropaySolutions\Kernel\Http\Client\Factory::class)->pool(fn (Pool $pool) => [
    $pool->withHeaders($headers)->get('http://framework.test/test'),
    $pool->withHeaders($headers)->get('http://framework.test/test'),
    $pool->withHeaders($headers)->get('http://framework.test/test'),
]);
```

<a name="macros"></a>
## Macros

The Framework HTTP client allows you to define "macros", which can serve as a fluent, expressive mechanism to configure common request paths and headers when interacting with services throughout your application. To get started, you may define the macro within the `boot` method of your application's `App\Providers\AppServiceProvider` class:

```php

/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    \app(\MacropaySolutions\Kernel\Http\Client\Factory::class)->macro('github', function () {
        return \app(\MacropaySolutions\Kernel\Http\Client\Factory::class)->withHeaders([
            'X-Example' => 'example',
        ])->baseUrl('https://github.com');
    });
}
```

Once your macro has been configured, you may invoke it from anywhere in your application to create a pending request with the specified configuration:

```php
$response = \app(\MacropaySolutions\Kernel\Http\Client\Factory::class)->github()->get('/');
```

<a name="testing"></a>
## Testing

Many Framework services provide functionality to help you easily and expressively write tests, and Framework's HTTP client is no exception. The `app(\MacropaySolutions\Kernel\Http\Client\Factory::class')`'s `fake` method allows you to instruct the HTTP client to return stubbed / dummy responses when requests are made.

<a name="faking-responses"></a>
### Faking Responses

For example, to instruct the HTTP client to return empty, `200` status code responses for every request, you may call the `fake` method with no arguments:



    $factory = app(\MacropaySolutions\Kernel\Http\Client\Factory::class);
    app()->instance(\MacropaySolutions\Kernel\Http\Client\Factory::class, $factory->fake());$response = \app(\MacropaySolutions\Kernel\Http\Client\Factory::class)->post(/* ... */);

    $response = \app(\MacropaySolutions\Kernel\Http\Client\Factory::class)->post(/* ... */);

<a name="faking-specific-urls"></a>
#### Faking Specific URLs

Alternatively, you may pass an array to the `fake` method. The array's keys should represent URL patterns that you wish to fake and their associated responses. The `*` character may be used as a wildcard character. Any requests made to URLs that have not been faked will actually be executed. You may use the `app(\MacropaySolutions\Kernel\Http\Client\Factory::class')`'s `response` method to construct stub / fake responses for these endpoints:

    $factory = app(\MacropaySolutions\Kernel\Http\Client\Factory::class);
    app()->instance(\MacropaySolutions\Kernel\Http\Client\Factory::class, $factory->fake([
        // Stub a JSON response for GitHub endpoints...
        'github.com/*' => $factory->response(['foo' => 'bar'], 200, $headers),

        // Stub a string response for Google endpoints...
        'google.com/*' => $factory->response('Hello World', 200, $headers),
    ]));

If you would like to specify a fallback URL pattern that will stub all unmatched URLs, you may use a single `*` character:

    $factory = app(\MacropaySolutions\Kernel\Http\Client\Factory::class);
    app()->instance(\MacropaySolutions\Kernel\Http\Client\Factory::class, $factory->fake([
        // Stub a JSON response for GitHub endpoints...
        'github.com/*' => $factory->response(['foo' => 'bar'], 200, ['Headers']),

        // Stub a string response for all other endpoints...
        '*' => $factory->response('Hello World', 200, ['Headers']),
    ]));

<a name="faking-response-sequences"></a>
#### Faking Response Sequences

Sometimes you may need to specify that a single URL should return a series of fake responses in a specific order. You may accomplish this using the `\app(\MacropaySolutions\Kernel\Http\Client\Factory::class)->sequence` method to build the responses:

    $factory = app(\MacropaySolutions\Kernel\Http\Client\Factory::class);
    app()->instance(\MacropaySolutions\Kernel\Http\Client\Factory::class, $factory->fake([
        // Stub a series of responses for GitHub endpoints...
        'github.com/*' => $factory->sequence()
                                ->push('Hello World', 200)
                                ->push(['foo' => 'bar'], 200)
                                ->pushStatus(404),
    ]));

When all the responses in a response sequence have been consumed, any further requests will cause the response sequence to throw an exception. If you would like to specify a default response that should be returned when a sequence is empty, you may use the `whenEmpty` method:

    $factory = app(\MacropaySolutions\Kernel\Http\Client\Factory::class);
    app()->instance(\MacropaySolutions\Kernel\Http\Client\Factory::class, $factory->fake([
        // Stub a series of responses for GitHub endpoints...
        'github.com/*' => $factory->sequence()
                                ->push('Hello World', 200)
                                ->push(['foo' => 'bar'], 200)
                                ->whenEmpty($factory->response()),
    ]));

If you would like to fake a sequence of responses but do not need to specify a specific URL pattern that should be faked, you may use the `\app(\MacropaySolutions\Kernel\Http\Client\Factory::class)->fakeSequence` method:

    $factory = app(\MacropaySolutions\Kernel\Http\Client\Factory::class);
    app()->instance(\MacropaySolutions\Kernel\Http\Client\Factory::class, $factory->fakeSequence()
            ->push('Hello World', 200)
            ->whenEmpty($factory->response()));

<a name="fake-callback"></a>
#### Fake Callback

If you require more complicated logic to determine what responses to return for certain endpoints, you may pass a closure to the `fake` method. This closure will receive an instance of `MacropaySolutions\Kernel\Http\Client\Request` as well as an array of options. The closure should return a response instance. Within your closure, you may perform whatever logic is necessary to determine what type of response to return:

    use MacropaySolutions\Kernel\Http\Client\Request;

    app()->instance(\MacropaySolutions\Kernel\Http\Client\Factory::class, $factory->fake(function (Request $request, array $options) use ($factory) {
        return $factory->response('Hello World', 200);
    }));

<a name="preventing-stray-requests"></a>
### Preventing Stray Requests

If you would like to ensure that all requests sent via the HTTP client have been faked throughout your individual test or complete test suite, you can call the `preventStrayRequests` method. After calling this method, any requests that do not have a corresponding fake response will throw an exception rather than making the actual HTTP request:



    $factory = app(\MacropaySolutions\Kernel\Http\Client\Factory::class);
    app()->instance(\MacropaySolutions\Kernel\Http\Client\Factory::class, $factory->preventStrayRequests());

    app()->instance(\MacropaySolutions\Kernel\Http\Client\Factory::class, $factory->fake([
        'https://github.com/*' => $factory->response('ok'),
    ]));

    // An "ok" response is returned...
    \app(\MacropaySolutions\Kernel\Http\Client\Factory::class)->get('https://github.com/macropay-solutions/php-kernel');

    // An exception is thrown...
    \app(\MacropaySolutions\Kernel\Http\Client\Factory::class)->get('https://domain.com');

<a name="inspecting-requests"></a>
### Inspecting Requests

When faking responses, you may occasionally wish to inspect the requests the client receives in order to make sure your application is sending the correct data or headers. You may accomplish this by calling the `\app(\MacropaySolutions\Kernel\Http\Client\Factory::class)->assertSent` method after calling `\app(\MacropaySolutions\Kernel\Http\Client\Factory::class)->fake`.

The `assertSent` method accepts a closure which will receive an `MacropaySolutions\Kernel\Http\Client\Request` instance and should return a boolean value indicating if the request matches your expectations. In order for the test to pass, at least one request must have been issued matching the given expectations:

    use MacropaySolutions\Kernel\Http\Client\Request;


    $factory = app(\MacropaySolutions\Kernel\Http\Client\Factory::class);
    app()->instance(\MacropaySolutions\Kernel\Http\Client\Factory::class, $factory->fake());

    \app(\MacropaySolutions\Kernel\Http\Client\Factory::class)->withHeaders([
        'X-First' => 'foo',
    ])->post('http://example.com/users', [
        'name' => 'Surname',
        'role' => 'Developer',
    ]);

    \app(\MacropaySolutions\Kernel\Http\Client\Factory::class)->assertSent(function (Request $request) {
        return $request->hasHeader('X-First', 'foo') &&
               $request->url() == 'http://example.com/users' &&
               $request['name'] == 'Surname' &&
               $request['role'] == 'Developer';
    });

If needed, you may assert that a specific request was not sent using the `assertNotSent` method:

    use MacropaySolutions\Kernel\Http\Client\Request;


    $factory = app(\MacropaySolutions\Kernel\Http\Client\Factory::class);
    app()->instance(\MacropaySolutions\Kernel\Http\Client\Factory::class, $factory->fake());

    \app(\MacropaySolutions\Kernel\Http\Client\Factory::class)->post('http://example.com/users', [
        'name' => 'Surname',
        'role' => 'Developer',
    ]);

    \app(\MacropaySolutions\Kernel\Http\Client\Factory::class)->assertNotSent(function (Request $request) {
        return $request->url() === 'http://example.com/posts';
    });

You may use the `assertSentCount` method to assert how many requests were "sent" during the test:

    $factory = app(\MacropaySolutions\Kernel\Http\Client\Factory::class);
    app()->instance(\MacropaySolutions\Kernel\Http\Client\Factory::class, $factory->fake());

    \app(\MacropaySolutions\Kernel\Http\Client\Factory::class)->assertSentCount(5);

Or, you may use the `assertNothingSent` method to assert that no requests were sent during the test:

    $factory = app(\MacropaySolutions\Kernel\Http\Client\Factory::class);
    app()->instance(\MacropaySolutions\Kernel\Http\Client\Factory::class, $factory->fake());

    \app(\MacropaySolutions\Kernel\Http\Client\Factory::class)->assertNothingSent();

<a name="recording-requests-and-responses"></a>
#### Recording Requests / Responses

You may use the `recorded` method to gather all requests and their corresponding responses. The `recorded` method returns a collection of arrays that contains instances of `MacropaySolutions\Kernel\Http\Client\Request` and `MacropaySolutions\Kernel\Http\Client\Response`:

```php
$factory = app(\MacropaySolutions\Kernel\Http\Client\Factory::class);
app()->instance(\MacropaySolutions\Kernel\Http\Client\Factory::class, $factory->fake([
    'https://domain.com' => $factory->response(status: 500),
    'https://subdomain.domain.com/' =>$factory->response(),
]));

\app(\MacropaySolutions\Kernel\Http\Client\Factory::class)->get('https://domain.com');
\app(\MacropaySolutions\Kernel\Http\Client\Factory::class)->get('https://subdomain.domain.com/');

$recorded = \app(\MacropaySolutions\Kernel\Http\Client\Factory::class)->recorded();

[$request, $response] = $recorded[0];
```

Additionally, the `recorded` method accepts a closure which will receive an instance of `MacropaySolutions\Kernel\Http\Client\Request` and `MacropaySolutions\Kernel\Http\Client\Response` and may be used to filter request / response pairs based on your expectations:

```php
use MacropaySolutions\Kernel\Http\Client\Request;
use MacropaySolutions\Kernel\Http\Client\Response;

$factory = app(\MacropaySolutions\Kernel\Http\Client\Factory::class);
app()->instance(\MacropaySolutions\Kernel\Http\Client\Factory::class, $factory->fake([
    'https://domain.com' => $factory->response(status: 500),
    'https://a.domain.com/' =>$factory->response(),
]));

\app(\MacropaySolutions\Kernel\Http\Client\Factory::class)->get('https://domain.com');
\app(\MacropaySolutions\Kernel\Http\Client\Factory::class)->get('https://a.domain.com/');

$recorded = \app(\MacropaySolutions\Kernel\Http\Client\Factory::class)->recorded(function (Request $request, Response $response) {
    return $request->url() !== 'https://domain.com' &&
           $response->successful();
});
```

<a name="events"></a>
## Events

Framework fires three events during the process of sending HTTP requests. The `RequestSending` event is fired prior to a request being sent, while the `ResponseReceived` event is fired after a response is received for a given request. The `ConnectionFailed` event is fired if no response is received for a given request.

The `RequestSending` and `ConnectionFailed` events both contain a public `$request` property that you may use to inspect the `MacropaySolutions\Kernel\Http\Client\Request` instance. Likewise, the `ResponseReceived` event contains a `$request` property as well as a `$response` property which may be used to inspect the `MacropaySolutions\Kernel\Http\Client\Response` instance. You may register event listeners for this event in your `App\Providers\EventServiceProvider` service provider:

    /**
     * The event listener mappings for the application.
     *
     * @var array
     */
    protected $listen = [
        'MacropaySolutions\Kernel\Http\Client\Events\RequestSending' => [
            'App\Listeners\LogRequestSending',
        ],
        'MacropaySolutions\Kernel\Http\Client\Events\ResponseReceived' => [
            'App\Listeners\LogResponseReceived',
        ],
        'MacropaySolutions\Kernel\Http\Client\Events\ConnectionFailed' => [
            'App\Listeners\LogConnectionFailed',
        ],
    ];
