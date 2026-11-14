---
title: URL Generation
description: Generating URLs for your application routes and assets in PHP Framework.
context: urls
---
# URL Generation

- [Introduction](#introduction)
- [The Basics](#the-basics)
  - [Generating URLs](#generating-urls)
  - [Accessing the Current URL](#accessing-the-current-url)
- [URLs for Named Routes](#urls-for-named-routes)
  - [Wildcard Routes](#urls-for-wildcard-routes)

<a name="introduction"></a>
## Introduction

Framework provides several helpers to assist you in generating URLs for your application. These helpers are primarily helpful when building links in your templates and API responses, or when generating redirect responses to another part of your application.

<a name="the-basics"></a>
## The Basics

<a name="generating-urls"></a>
### Generating URLs

The `url` helper may be used to generate arbitrary URLs for your application. The generated URL will automatically use the scheme (HTTP or HTTPS) and host from the current request being handled by the application:

    $post = App\Models\Post::query()->find(1);

    echo url("/posts/{$post->id}");

    // http://example.com/posts/1

<a name="accessing-the-current-url"></a>
### Accessing the Current URL

The `url` service resolves to a `MacropaySolutions\Framework\Routing\UrlGenerator` instance, allowing you to access information about the current URL:

    // Get the current URL without the query string...
    echo app('url')->current();

    // Get the current URL including the query string...
    echo app('url')->full();

Each of these methods may also be accessed via the `url` service:

    echo app('url')->current();

<a name="urls-for-named-routes"></a>
## URLs for Named Routes

The `route` helper may be used to generate URLs to [named routes](/routing#named-routes). Named routes allow you to generate URLs without being coupled to the actual URL defined on the route. Therefore, if the route's URL changes, no changes need to be made to your calls to the `route` function. For example, imagine your application contains a route defined like the following:

    $router->get('/post/{post}', [
        'as' => 'post.show', 'uses' => 'PostController@show'
    ]);

To generate a URL to this route, you may use the `route` helper like so:

    echo route('post.show', ['post' => 1]);

    // http://example.com/post/1

Of course, the `route` helper may also be used to generate URLs for routes with multiple parameters:

    $router->get('/post/{post}/comment/{comment}', [
        'as' => 'comment.show', 'uses' => 'CommentController@show'
    ]);

    echo route('comment.show', ['post' => 1, 'comment' => 3]);

    // http://example.com/post/1/comment/3

Any additional array elements that do not correspond to the route's definition parameters will be added to the URL's query string:

    echo route('post.show', ['post' => 1, 'search' => 'rocket']);

    // http://example.com/post/1?search=rocket

> [!NOTE]  
> Route parameter values are URL-encoded (`rawurlencode`), so a value containing `/`, `?`, `#` or `..` stays inside its parameter: `route('post.show', ['post' => 'a/b?x=1'])` produces `/post/a%2Fb%3Fx%3D1`. The router decodes each parameter, so the controller receives the original value. Do not encode the values yourself, or they will be encoded twice.

<a name="urls-for-wildcard-routes"></a>
#### Wildcard Routes

A route whose parameter may contain slashes, such as `docs/{path:.+}`, can also be generated with `route`, but the slashes in the value are encoded: `route('docs.show', ['path' => 'guide/intro'])` produces `/docs/guide%2Fintro`. The route matches it and the controller receives `guide/intro`.

For a readable URL, build the path from its segments with the `url` helper instead. Each extra segment is encoded on its own and the segments are joined with `/`:

    $router->get('/docs/{path:.+}', , [
        'as' => 'doc.show', 'uses' => 'DocsController@show'
    ]);

    echo url('docs', ['guide', 'intro']);

    // http://example.com/docs/guide/intro

The `url` helper uses the current request's host. For a route restricted to another [domain](/routing#domain-routing), pass the full URL; the segments are appended before any query string:

    echo url('https://api.example.com/docs', ['guide', 'intro']);

    // https://api.example.com/docs/guide/intro

The `route` helper already generates absolute URLs for domain routes, with the value encoded as a single parameter.

> [!WARNING]  
> Segments that contain only `.` or `..` are not encoded, and the browser resolves them: `url('docs', ['..', 'admin'])` produces `/docs/../admin`, which the browser requests as `/admin`. When the segments come from user input, drop the empty, `.` and `..` segments first:
>
> ```php
> $segments = \array_filter(
>     \explode('/', $path),
>     static fn(string $segment): bool => '' !== $segment && '.' !== $segment && '..' !== $segment
> );
>
> echo url('docs', $segments);
> ```
