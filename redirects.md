---
title: HTTP Redirects
description: Generating redirect responses and handling stateless routing logic in PHP Framework.
context: redirects
---
# HTTP Redirects

- [Creating Redirects](#creating-redirects)
- [Redirecting To Named Routes](#redirecting-named-routes)
- [Redirecting With Flashed Session Data](#redirecting-with-flashed-session-data)

<a name="creating-redirects"></a>
## Creating Redirects

Redirect responses are instances of the `MacropaySolutions\Kernel\Http\RedirectResponse` class, and contain the proper headers needed to redirect the user to another URL. There are several ways to generate a `RedirectResponse` instance. The simplest method is to use the global `redirect` helper:

    $router->get('/dashboard', 'DashboardController@index');

    public function index(): RedirectResponse
    {
        return redirect('/home/dashboard');
    }

<a name="redirecting-named-routes"></a>
## Redirecting To Named Routes

When you call the `redirect` helper with no parameters, an instance of `MacropaySolutions\Framework\Http\Redirector` is returned, allowing you to call any method on the `Redirector` instance. For example, to generate a `RedirectResponse` to a named route, you may use the `route` method:

    return redirect()->route('login');

If your route has parameters, you may pass them as the second argument to the `route` method:

    // For a route with the following URI: profile/{id}

    return redirect()->route('profile', ['id' => 1]);

<a name="redirecting-with-flashed-session-data"></a>
## Redirecting With Flashed Session Data

Redirecting to a new URL and [flashing data to the session](/session#flash-data) are usually done at the same time. Typically, this is done after successfully performing an action when you flash a success message to the session. For convenience, you may create a `RedirectResponse` instance and flash data to the session in a single, fluent method chain:

    $router->post('/user/profile', 'ProfileController@update');

    public function update(): RedirectResponse
    {
        return redirect('/dashboard')->with('status', 'Profile updated!');
    }

You may use the `withInput` method provided by the `RedirectResponse` instance to flash the current request's input data to the session before redirecting the user to a new location. Once the input has been flashed to the session, you may easily [retrieve it](/validation#manually-flashing-errors-and-input) during the next request:

    return redirect('/dashboard')->withInput();

After the user is redirected, you may display the flashed message from the [session](/session). For example, using [Template syntax](/template):

    @if (session('status'))
        <div class="alert alert-success">
            {{ session('status') }}
        </div>
    @endif
