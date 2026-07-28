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

<a name="introduction"></a>
## Introduction

Framework provides several helpers to assist you in generating URLs for your application. These helpers are primarily helpful when building links in your templates and API responses, or when generating redirect responses to another part of your application.

<a name="the-basics"></a>
## The Basics

<a name="generating-urls"></a>
### Generating URLs

The `url` helper may be used to generate arbitrary URLs for your application. The generated URL will automatically use the scheme (HTTP or HTTPS) and host from the current request being handled by the application:

    $post = App\Models\Post::find(1);

    echo url("/posts/{$post->id}");

    // http://example.com/posts/1

<a name="accessing-the-current-url"></a>
### Accessing the Current URL

If no path is provided to the `url` helper, a `MacropaySolutions\Framework\Routing\UrlGenerator` instance is returned, allowing you to access information about the current URL:

    // Get the current URL without the query string...
    echo url()->current();

    // Get the current URL including the query string...
    echo url()->full();

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

<a name="obvious-models"></a>
#### Obvious Models

You will often be generating URLs using the route key (typically the primary key) of [Obvious models](/obvious). For this reason, you may pass Obvious models as parameter values. The `route` helper will automatically extract the model's route key:

    echo route('post.show', ['post' => $post]);