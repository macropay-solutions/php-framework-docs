---
title: Error Handling
description: Guide to exception handling, logging, and JSON error responses in PHP Framework.
context: errors
---

# Error Handling

- [Introduction](#introduction)
- [Configuration](#configuration)
- [The Exception Handler](#the-exception-handler)
    - [Reporting Exceptions](#reporting-exceptions)
    - [The `report` Helper](#the-report-helper)
    - [Exception Log Context](#exception-log-context)
    - [Ignoring Exceptions by Type](#ignoring-exceptions-by-type)
    - [Rendering Exceptions](#rendering-exceptions)
    - [Reportable and Renderable Exceptions](#reportable-and-renderable-exceptions)
- [HTTP Exceptions](#http-exceptions)
    - [The `abort` Helper](#the-abort-helper)
    - [HTTP Response Formatting](#http-response-formatting)

<a name="introduction"></a>
## Introduction

Error and exception handling is configured for every Framework application via the `MacropaySolutions\Framework\Exceptions\Handler` class. All exceptions thrown by your application are logged according to your logging configuration and rendered into HTTP responses.

<a name="configuration"></a>
## Configuration

The `debug` option in your `config/app.php` configuration file controls how much information about an error is exposed in HTTP responses. By default, this option respects the `APP_DEBUG` environment variable stored in your `.env` file.

During local development, set `APP_DEBUG=true`. **In production environments, this value must always be `false`. Setting `APP_DEBUG=true` in production risks exposing sensitive configuration values and stack traces in your API responses.**

<a name="the-exception-handler"></a>
## The Exception Handler

<a name="reporting-exceptions"></a>
### Reporting Exceptions

All unhandled exceptions are passed to the `report` method on your `App\Exceptions\Handler` class (which extends `MacropaySolutions\Framework\Exceptions\Handler`). By default, the exception handler writes the exception to the logger resolved from the container (`Psr\Log\LoggerInterface`).

To customize how an exception is logged, you may define a `report` method directly on that specific exception class.

<a name="the-report-helper"></a>
### The `report` Helper

To report an exception without interrupting request execution, call the `report` global helper function:

```php
public function process(string $data): bool
{
    try {
        return $this->parseData($data);
    } catch (\Throwable $e) {
        \report($e);

        return false;
    }
}
```

<a name="exception-log-context"></a>
### Exception Log Context

To add custom contextual data to log entries for a specific exception, define a `context` method on the exception class. The returned array will automatically be passed into log records:

```php
namespace App\Exceptions;

use Exception;

class InvalidOrderException extends Exception
{
    /**
     * Create a new exception instance.
     */
    public function __construct(
        protected int $orderId
    ) {
        parent::__construct('Invalid order payload.');
    }

    /**
     * Get the exception's context information for logging.
     *
     * @return array<string, mixed>
     */
    public function context(): array
    {
        return [
            'order_id' => $this->orderId,
        ];
    }
}
```

<a name="ignoring-exceptions-by-type"></a>
### Ignoring Exceptions by Type

To prevent specific types of exceptions from being logged, list their class names in the `$dontReport` property on your `App\Exceptions\Handler` class:

```php
namespace App\Exceptions;

use App\Exceptions\InvalidOrderException;
use MacropaySolutions\Framework\Exceptions\Handler as ExceptionHandler;

class Handler extends ExceptionHandler
{
    /**
     * A list of the exception types that should not be reported.
     *
     * @var array<int, class-string<\Throwable>>
     */
    protected $dontReport = [
        InvalidOrderException::class,
    ];
}
```

The base handler automatically ignores internal HTTP exceptions, validation errors, and missing model exceptions (including `ModelNotFoundException` and `RecordsNotFoundException`).

<a name="rendering-exceptions"></a>
### Rendering Exceptions

By default, the framework handler converts caught exceptions into an HTTP or JSON response based on the incoming request headers.

The base exception handler automatically maps internal database and security exceptions to appropriate HTTP status codes:

* `ModelNotFoundException` &rarr; `404 Not Found`
* `RecordsNotFoundException` &rarr; `404 Not Found`
* `SuspiciousOperationException` &rarr; `404 Not Found`
* `AuthorizationException` &rarr; `403 Forbidden`
* `TokenMismatchException` &rarr; `419 Page Expired`

<a name="reportable-and-renderable-exceptions"></a>
### Reportable and Renderable Exceptions

Instead of handling exception logic inside `App\Exceptions\Handler`, you may define `report` and `render` methods directly on custom exception classes:

```php
namespace App\Exceptions;

use Exception;
use MacropaySolutions\Kernel\Http\Request;
use MacropaySolutions\Kernel\Http\Response;

class InvalidOrderException extends Exception
{
    /**
     * Custom report method. Returning false allows standard logging to continue.
     */
    public function report(): bool
    {
        \app('log')->warning('Order processing failed for specific customer');

        return true;
    }

    /**
     * Custom render method into an HTTP response.
     */
    public function render(Request $request): Response
    {
        return \response()->json([
            'status' => 'error',
            'message' => $this->getMessage(),
        ], 400);
    }
}
```

You may also implement the `MacropaySolutions\Kernel\Contracts\Support\Responsable` interface on your exception class to convert it into a response:

```php
namespace App\Exceptions;

use Exception;
use MacropaySolutions\Kernel\Contracts\Support\Responsable;
use Symfony\Component\HttpFoundation\Response;

class CustomApiException extends Exception implements Responsable
{
    public function toResponse($request): Response
    {
        return \response()->json([
            'error' => $this->getMessage(),
        ], 422);
    }
}
```

<a name="http-exceptions"></a>
## HTTP Exceptions

<a name="the-abort-helper"></a>
### The `abort` Helper

To throw an HTTP exception and stop processing immediately, call the `abort` global helper function:

```php
\abort(404, 'The requested resource was not found.');
```

Passing `404` throws a `Symfony\Component\HttpKernel\Exception\NotFoundHttpException`. Any other HTTP status code throws a `Symfony\Component\HttpKernel\Exception\HttpException`.

<a name="http-response-formatting"></a>
### HTTP Response Formatting

#### JSON API Requests

When `$request->expectsJson()` is true, the handler returns a structured JSON response:

With `APP_DEBUG=true`:
```json
{
    "message": "The requested resource was not found.",
    "exception": "Symfony\\Component\\HttpKernel\\Exception\\NotFoundHttpException",
    "file": "/var/www/app/Http/Controllers/PostController.php",
    "line": 42,
    "trace": [...]
}
```

With `APP_DEBUG=false`:
```json
{
    "message": "The requested resource was not found."
}
```

#### Non-JSON Requests

When `$request->expectsJson()` is false and `APP_DEBUG=false`, the exception handler returns a lightweight, centered HTML layout containing the status code and text (e.g., `<h1>404: Not Found</h1>` or `<h1>500: Server Error</h1>`).
