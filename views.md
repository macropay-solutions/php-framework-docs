---
title: Views
description: Creating, rendering, and optimizing Blade views in PHP-Framework.
context: views
---

# Views

- [Introduction](#introduction)
  - [Writing Views in React / Vue](#writing-views-in-react-or-vue)
  - [Explicit View Opt-In Flow](#explicit-view-opt-in-flow)
- [Creating and Rendering Views](#creating-and-rendering-views)
  - [Nested View Directories](#nested-view-directories)
  - [Creating the First Available View](#creating-the-first-available-view)
  - [Determining if a View Exists](#determining-if-a-view-exists)
- [Passing Data to Views](#passing-data-to-views)
  - [Sharing Data With All Views](#sharing-data-with-all-views)
- [View Composers](#view-composers)
  - [View Creators](#view-creators)
- [Optimizing Views](#optimizing-views)

<a name="introduction"></a>
## Introduction

Of course, it's not practical to return entire HTML documents strings directly from your routes and controllers. Thankfully, views provide a convenient way to place all of our HTML in separate files.

Views separate your controller / application logic from your presentation logic and are stored in the `resources/views` directory. When using Framework, view templates are usually written using the [Blade templating language](/blade). A simple view might look something like this:

```blade
<!-- View stored in resources/views/greeting.blade.php -->

<html>
    <body>
        <h1>Hello, {{ $name }}</h1>
    </body>
</html>
```

Since this view is stored at `resources/views/greeting.blade.php`, we may return it using the global `view` helper from within a controller method:

    namespace App\Http\Controllers;

    class GreetingController
    {
        public function show()
        {
            return view('greeting', ['name' => 'James']);
        }
    }

> [!NOTE]  
> Looking for more information on how to write Blade templates? Check out the full [Blade documentation](/blade) to get started.

<a name="writing-views-in-react-or-vue"></a>
### Writing Views in React / Vue

Instead of writing their frontend templates in PHP via Blade, many developers have begun to prefer to write their templates using React or Vue. Framework makes this painless thanks to [Inertia](https://inertiajs.com/), a library that makes it a cinch to tie your React / Vue frontend to your Framework backend without the typical complexities of building an SPA.

<a name="explicit-view-opt-in-flow"></a>
### Explicit View Opt-In Flow

Because PHP-Framework is optimized for headless JSON APIs, the HTML View rendering engine is completely disabled by default to conserve memory. To utilize Blade views, you must actively toggle the engine on within the application lifecycle:

1. **Composer Realignment**: Open your `composer.json` file and remove the `"vendor/macropay-solutions/php-kernel/kernel/View/"` string from the `exclude-from-classmap` array.
2. **Container Activation**: Open `App\Application.php` and uncomment the view factory container bindings inside the `$availableBindings` array:

    ```php
    public $availableBindings = [
        // ...
        'view' => 'registerViewBindings',
        \MacropaySolutions\Kernel\Contracts\View\Factory::class => 'registerViewBindings',
        'view.finder' => 'registerViewBindings',
        'blade.compiler' => 'registerViewBindings',
        'view.engine.resolver' => 'registerViewBindings',
        \MacropaySolutions\Kernel\View\Engines\EngineResolver::class => 'registerViewBindings',
    ];
    ```

3. **Alias Activation**: While still inside `App\Application.php`, scroll down to the `registerContainerAliases` method and uncomment the corresponding view aliases in both the `$abstractAliases` and `$aliases` arrays.

Once uncommented, run `composer dump-autoload` in your terminal to rebuild the classmap.

<a name="creating-and-rendering-views"></a>
## Creating and Rendering Views

You may create a view by placing a file with the `.blade.php` extension in your application's `resources/views` directory or by using the `make:view` Run command:

```shell
php run make:view greeting
```

The `.blade.php` extension informs the framework that the file contains a [Blade template](/blade). Blade templates contain HTML as well as Blade directives that allow you to easily echo values, create "if" statements, iterate over data, and more.

Once you have created a view, you may return it from one of your application's controllers using the global `view` helper:

    public function show()
    {
        return view('greeting', ['name' => 'James']);
    }

Views may also be rendered explicitly by resolving the view factory directly from the service container using the `\app()` helper:

    public function show()
    {
        return \app('view')->make('greeting', ['name' => 'James']);
    }

As you can see, the first argument passed to the `view` helper corresponds to the name of the view file in the `resources/views` directory. The second argument is an array of data that should be made available to the view. In this case, we are passing the `name` variable, which is displayed in the view using [Blade syntax](/blade).

<a name="nested-view-directories"></a>
### Nested View Directories

Views may also be nested within subdirectories of the `resources/views` directory. "Dot" notation may be used to reference nested views. For example, if your view is stored at `resources/views/admin/profile.blade.php`, you may return it from your controller like so:

    return view('admin.profile', $data);

> [!WARNING]  
> View directory names should not contain the `.` character.

<a name="creating-the-first-available-view"></a>
### Creating the First Available View

Using the container's `first` method, you may create the first view that exists in a given array of views. This may be useful if your application or package allows views to be customized or overwritten:

    return \app('view')->first(['custom.admin', 'admin'], $data);

<a name="determining-if-a-view-exists"></a>
### Determining if a View Exists

If you need to determine if a view exists, you may use the view factory. The `exists` method will return `true` if the view exists:

    if (\app('view')->exists('admin.profile')) {
        // ...
    }

<a name="passing-data-to-views"></a>
## Passing Data to Views

As you saw in the previous examples, you may pass an array of data to views to make that data available to the view:

    return view('greetings', ['name' => 'Victoria']);

When passing information in this manner, the data should be an array with key / value pairs. After providing data to a view, you can then access each value within your view using the data's keys, such as `<?php echo $name; ?>`.

As an alternative to passing a complete array of data to the `view` helper function, you may use the `with` method to add individual pieces of data to the view. The `with` method returns an instance of the view object so that you can continue chaining methods before returning the view:

    return view('greeting')
                ->with('name', 'Victoria')
                ->with('occupation', 'Astronaut');

<a name="sharing-data-with-all-views"></a>
### Sharing Data With All Views

Occasionally, you may need to share data with all views that are rendered by your application. You may do so using the `share` method. Typically, you should place calls to the `share` method within a service provider's `boot` method. You are free to add them to the `App\Providers\AppServiceProvider` class or generate a separate service provider to house them:

    <?php

    namespace App\Providers;

    use MacropaySolutions\Kernel\Support\ServiceProvider;

    class AppServiceProvider extends ServiceProvider
    {
        /**
         * Register any application services.
         */
        public function register(): void
        {
            // ...
        }

        /**
         * Bootstrap any application services.
         */
        public function boot(): void
        {
            \app('view')->share('key', 'value');
        }
    }

<a name="view-composers"></a>
## View Composers

View composers are callbacks or class methods that are called when a view is rendered. If you have data that you want to be bound to a view each time that view is rendered, a view composer can help you organize that logic into a single location. View composers may prove particularly useful if the same view is returned by multiple routes or controllers within your application and always needs a particular piece of data.

Typically, view composers will be registered within one of your application's [service providers](/providers). In this example, we'll assume that we have created a new `App\Providers\ViewServiceProvider` to house this logic.

We'll use the view factory's `composer` method to register the view composer. Framework does not include a default directory for class based view composers, so you are free to organize them however you wish. For example, you could create an `app/View/Composers` directory to house all of your application's view composers:

    <?php

    namespace App\Providers;

    use App\View\Composers\ProfileComposer;
    use MacropaySolutions\Kernel\Support\ServiceProvider;
    use MacropaySolutions\Kernel\View\View;

    class ViewServiceProvider extends ServiceProvider
    {
        /**
         * Register any application services.
         */
        public function register(): void
        {
            // ...
        }

        /**
         * Bootstrap any application services.
         */
        public function boot(): void
        {
            // Using class based composers...
            \app('view')->composer('profile', ProfileComposer::class);

            // Using closure based composers...
            \app('view')->composer('welcome', function (View $view) {
                // ...
            });

            \app('view')->composer('dashboard', function (View $view) {
                // ...
            });
        }
    }

> [!WARNING]  
> Remember, if you create a new service provider to contain your view composer registrations, you will need to add the service provider to the `providers` array in the `config/app.php` configuration file.

Now that we have registered the composer, the `compose` method of the `App\View\Composers\ProfileComposer` class will be executed each time the `profile` view is being rendered. Let's take a look at an example of the composer class:

    <?php

    namespace App\View\Composers;

    use App\Repositories\UserRepository;
    use MacropaySolutions\Kernel\View\View;

    class ProfileComposer
    {
        /**
         * Create a new profile composer.
         */
        public function __construct(
            protected UserRepository $users,
        ) {}

        /**
         * Bind data to the view.
         */
        public function compose(View $view): void
        {
            $view->with('count', $this->users->count());
        }
    }

As you can see, all view composers are resolved via the [service container](/container), so you may type-hint any dependencies you need within a composer's constructor.

<a name="attaching-a-composer-to-multiple-views"></a>
#### Attaching a Composer to Multiple Views

You may attach a view composer to multiple views at once by passing an array of views as the first argument to the `composer` method:

    use App\View\Composers\MultiComposer;

    \app('view')->composer(
        ['profile', 'dashboard'],
        MultiComposer::class
    );

The `composer` method also accepts the `*` character as a wildcard, allowing you to attach a composer to all views:

    use MacropaySolutions\Kernel\View\View;

    \app('view')->composer('*', function (View $view) {
        // ...
    });

<a name="view-creators"></a>
### View Creators

View "creators" are very similar to view composers; however, they are executed immediately after the view is instantiated instead of waiting until the view is about to render. To register a view creator, use the `creator` method:

    use App\View\Creators\ProfileCreator;

    \app('view')->creator('profile', ProfileCreator::class);

<a name="optimizing-views"></a>
## Optimizing Views

By default, Blade template views are compiled on demand. When a request is executed that renders a view, Framework will determine if a compiled version of the view exists. If the file exists, Framework will then determine if the uncompiled view has been modified more recently than the compiled view. If the compiled view either does not exist, or the uncompiled view has been modified, Framework will recompile the view.

Compiling views during the request may have a small negative impact on performance, so Framework provides the `view:cache` Run command to precompile all the views utilized by your application. For increased performance, you may wish to run this command as part of your deployment process:

```shell
php run view:cache
```

You may use the `view:clear` command to clear the view cache:

```shell
php run view:clear
```
