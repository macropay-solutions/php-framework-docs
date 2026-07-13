---
title: Testing
description: Comprehensive guide to testing in PHP-Framework, including stateless JSON APIs, parallel execution, and Obvious model factories.
context: testing
---

# Testing

- [Introduction](#introduction)
- [Environment](#environment)
- [Creating Tests](#creating-tests)
- [Application Testing](#application-testing)
	- [Authentication](#authentication)
	- [Custom HTTP Requests](#custom-http-requests)
- [Working With Databases](#working-with-databases)
	- [Resetting The Database After Each Test](#resetting-the-database-after-each-test)
	- [Model Factories](#model-factories)
- [Mocking](#mocking)
	- [Mocking Events](#mocking-events)
	- [Mocking Jobs](#mocking-jobs)
- [Running Tests](#running-tests)
	- [Running Tests in Parallel](#running-tests-in-parallel)
	- [Reporting Test Coverage](#reporting-test-coverage)
	- [Profiling Tests](#profiling-tests)

<a name="introduction"></a>
## Introduction

Framework is built with testing in mind. In fact, support for testing with PHPUnit is included out of the box and a `phpunit.xml` file is already set up for your application. The framework also ships with convenient helper methods that allow you to expressively test your applications.

By default, your application's `tests` directory contains two directories: `Feature` and `Unit`. Unit tests are tests that focus on a very small, isolated portion of your code. In fact, most unit tests probably focus on a single method. Tests within your "Unit" test directory do not boot your Framework application and therefore are unable to access your application's database or other framework services.

Feature tests may test a larger portion of your code, including how several objects interact with each other or even a full HTTP request to a JSON endpoint. **Generally, most of your tests should be feature tests. These types of tests provide the most confidence that your system as a whole is functioning as intended.**

An `ExampleTest.php` file is provided in both the `Feature` and `Unit` test directories. After installing a new Framework application, execute the `vendor/bin/phpunit` or `php run test` commands to run your tests.

<a name="environment"></a>
## Environment

When running tests, Framework will automatically set the [configuration environment](/configuration#environment-configuration) to `testing` because of the environment variables defined in the `phpunit.xml` file. Framework also automatically configures the session and cache to the `array` driver so that no session or cache data will be persisted while testing.

You are free to define other testing environment configuration values as necessary. The `testing` environment variables may be configured in your application's `phpunit.xml` file, but make sure to clear your configuration cache using the `config:clear` Run command before running your tests!

> **Note:** Because PHP-Framework is optimized for performance-tuned, headless APIs, memory-heavy components like Sessions, Cookies, and HTML Views are completely disabled by default. If your tests require them, you must explicitly opt-in by modifying your `exclude-from-classmap` rules and uncommenting the relevant bindings in `App\Application.php`.

<a name="the-env-testing-environment-file"></a>
#### The `.env.testing` Environment File

In addition, you may create a `.env.testing` file in the root of your project. This file will be used instead of the `.env` file when running PHPUnit tests or executing Run commands with the `--env=testing` option.

<a name="the-creates-application-trait"></a>
#### The `CreatesApplication` Trait

Framework includes a `CreatesApplication` trait that is applied to your application's base `TestCase` class. This trait contains a `createApplication` method that bootstraps the Framework application before running your tests. It's important that you leave this trait at its original location as some features, such as Framework's parallel testing feature, depend on it.

<a name="creating-tests"></a>
## Creating Tests

To create a new test case, use the `make:test` Run command. By default, tests will be placed in the `tests/Feature` directory:

    php run make:test UserTest

If you would like to create a test within the `tests/Unit` directory, you may use the `--unit` option when executing the `make:test` command:

    php run make:test UserTest --unit

If you would like to create a [Pest PHP](https://pestphp.com) test, you may provide the `--pest` option to the `make:test` command:

    php run make:test UserTest --pest
    php run make:test UserTest --unit --pest

Once the test has been generated, you may define test methods as you normally would using [PHPUnit](https://phpunit.de). To run your tests, execute the `vendor/bin/phpunit` or `php run test` command from your terminal:

    <?php

    namespace Tests\Unit;

    use PHPUnit\Framework\TestCase;

    class ExampleTest extends TestCase
    {
        /**
         * A basic test example.
         */
        public function test_basic_test(): void
        {
            $this->assertTrue(true);
        }
    }

> [!WARNING]  
> If you define your own `setUp` / `tearDown` methods within a test class, be sure to call the respective `parent::setUp()` / `parent::tearDown()` methods on the parent class. Typically, you should invoke `parent::setUp()` at the start of your own `setUp` method, and `parent::tearDown()` at the end of your `tearDown` method.

<a name="application-testing"></a>
## Application Testing

Framework provides a very fluent API for making HTTP requests to your application and examining the output.

<a name="authentication"></a>
### Authentication

The `actingAs` helper method provides a simple way to authenticate a given user as the current user:

    <?php

    class ExampleTest extends TestCase
    {
        public function testApplication()
        {
            $user = factory('App\Models\User')->create();

            $this->actingAs($user)
                 ->get('/user');
        }
    }

<a name="custom-http-requests"></a>
### Custom HTTP Requests

If you would like to make a custom HTTP request into your application and get the full `MacropaySolutions\Kernel\Http\Response` object, you may use the `call` method:

    public function testApplication()
    {
        $response = $this->call('GET', '/');

        $this->assertEquals(200, $response->status());
    }

If you are making `POST`, `PUT`, or `PATCH` requests you may pass an array of input data with the request. Of course, this data will be available in your routes and controller via the Request instance:

    $response = $this->call('POST', '/user', ['name' => 'Name']);

#### JSON Validation Errors

If you are using the `assertJsonValidationErrors` method, you should pass `null` as the response key that contains the response's error message. Unlike Kernel, Framework returns errors at the root of the returned JSON object:

    public function testApplication()
    {
        $response = $this->call('POST', '/user', ['name' => null]);

        $response->assertJsonValidationErrors('name', $responseKey = null);
    }

<a name="working-with-databases"></a>
## Working With Databases

Framework also provides a variety of helpful tools to make it easier to test your database driven applications. First, you may use the `seeInDatabase` helper to assert that data exists in the database matching a given set of criteria. For example, if we would like to verify that there is a record in the `users` table with the `email` value of `sally@example.com`, we can do the following:

    public function testDatabase()
    {
        // Make call to application...

        $this->seeInDatabase('users', ['email' => 'sally@foo.com']);
    }

Of course, the `seeInDatabase` method and other helpers like it are for convenience. You are free to use any of PHPUnit's built-in assertion methods to supplement your tests.

<a name="resetting-the-database-after-each-test"></a>
### Resetting The Database After Each Test

It is often useful to reset your database after each test so that data from a previous test does not interfere with subsequent tests.

#### Using Migrations

One option is to rollback the database after each test and migrate it before the next test. Framework provides a simple `DatabaseMigrations` trait that will automatically handle this for you. Simply use the trait on your test class:

    <?php

    use MacropaySolutions\KernelDev\Foundation\Testing\DatabaseMigrations;
    use MacropaySolutions\KernelDev\Foundation\Testing\DatabaseTransactions;

    class ExampleTest extends TestCase
    {
        use DatabaseMigrations;

        /**
         * A basic functional test example.
         *
         * @return void
         */
        public function testBasicExample()
        {
            $this->get('/foo');
        }
    }

#### Using Transactions

Another option is to wrap every test case in a database transaction. Again, Framework provides a convenient `DatabaseTransactions` trait that will automatically handle this:

    <?php

    use MacropaySolutions\KernelDev\Foundation\Testing\DatabaseMigrations;
    use MacropaySolutions\KernelDev\Foundation\Testing\DatabaseTransactions;

    class ExampleTest extends TestCase
    {
        use DatabaseTransactions;

        /**
         * A basic functional test example.
         *
         * @return void
         */
        public function testBasicExample()
        {
            $this->get('/foo');
        }
    }

<a name="model-factories"></a>
### Model Factories

When testing, it is common to need to insert a few records into your database before executing your test. Instead of manually specifying the value of each column when you create this test data, Framework allows you to define a default set of attributes for each of your [Obvious models](/obvious) using model factories.

To get started, take a look at the `database/factories/UserFactory.php` file in your application. Out of the box, this file contains the following factory definition:

    <?php
    
    namespace Database\Factories;
    
    use App\Models\User;
    use MacropaySolutions\Kernel\Database\Obvious\Factories\Factory;
    
    class UserFactory extends Factory
    {
        /**
         * The name of the factory's corresponding model.
         *
         * @var string
         */
        protected $model = User::class;
    
        /**
         * Define the model's default state.
         *
         * @return array
         */
        public function definition()
        {
            return [
                'name' => $this->faker->name,
                'email' => $this->faker->unique()->safeEmail,
            ];
        }
    }

As you can see, in their most basic form, factories are classes that extend Framework's base factory class and define a `model` property and `definition` method. The `definition` method returns the default set of attribute values that should be applied when creating a model using the factory.

Via the `faker` property, factories have access to the [Faker](https://github.com/fzaninotto/Faker) PHP library, which allows you to conveniently generate various kinds of random data for testing.

#### Factory States

State manipulation methods allow you to define discrete modifications that can be applied to your model factories in any combination. For example, your `User` model might have a `suspended` state that modifies one of its default attribute values. You may define your state transformations using the base factory's `state` method. You may name your state method anything you like. After all, it's just a typical PHP method:

    /**
     * Indicate that the user is suspended.
     *
     * @return \MacropaySolutions\Kernel\Database\Obvious\Factories\Factory
     */
    public function suspended()
    {
        return $this->state([
            'account_status' => 'suspended',
        ]);
    }

If your state transformation requires access to the other attributes defined by the factory, you may pass a callback to the `state` method. The callback will receive the array of raw attributes defined for the factory:

    /**
     * Indicate that the user is suspended.
     *
     * @return \MacropaySolutions\Kernel\Database\Obvious\Factories\Factory
     */
    public function suspended()
    {
        return $this->state(function (array $attributes) {
            return [
                'account_status' => 'suspended',
            ];
        });
    }

#### Factory Callbacks

Factory callbacks are registered using the `afterMaking` and `afterCreating` methods and allow you to perform additional tasks after making or creating a model. You should register these callbacks by defining a `configure` method on the factory class. This method will automatically be called by Kernel when the factory is instantiated:

    namespace Database\Factories;

    use App\Models\User;
    use MacropaySolutions\Kernel\Database\Obvious\Factories\Factory;
    use MacropaySolutions\Kernel\Support\Str;

    class UserFactory extends Factory
    {
        /**
         * The name of the factory's corresponding model.
         *
         * @var string
         */
        protected $model = User::class;

        /**
         * Configure the model factory.
         *
         * @return $this
         */
        public function configure()
        {
            return $this->afterMaking(function (User $user) {
                //
            })->afterCreating(function (User$user) {
                //
            });
        }

        // ...
    }

<a name="mocking"></a>
## Mocking

<a name="mocking-events"></a>
### Mocking Events

If you are making heavy use of Framework's event system, you may wish to silence or mock certain events while testing. For example, if you are testing user registration, you probably do not want all of a `UserRegistered` event's handlers firing, since these may send "welcome" e-mails, etc.

Framework provides a convenient `expectsEvents` method that verifies the expected events are fired, but prevents any handlers for those events from running:

    <?php

    class ExampleTest extends TestCase
    {
        public function testUserRegistration()
        {
            $this->expectsEvents('App\Events\UserRegistered');

            // Test user registration code...
        }
    }

If you would like to prevent all event handlers from running, you may use the `withoutEvents` method:

    <?php

    class ExampleTest extends TestCase
    {
        public function testUserRegistration()
        {
            $this->withoutEvents();

            // Test user registration code...
        }
    }

<a name="mocking-jobs"></a>
### Mocking Jobs

Sometimes, you may wish to simply test that specific jobs are dispatched by your controllers when making requests to your application. This allows you to test your routes / controllers in isolation - set apart from your job's logic. Of course, you can then test the job itself in a separate test class.

Framework provides a convenient `expectsJobs` method that will verify that the expected jobs are dispatched, but the job itself will not be executed:

    <?php

    class ExampleTest extends TestCase
    {
        public function testPurchasePodcast()
        {
            $this->expectsJobs('App\Jobs\PurchasePodcast');

            // Test purchase podcast code...
        }
    }

> **Note:** This method only detects jobs that are dispatched via the `dispatch` global helper function or the `$this->dispatch` method from a route or controller. It does not detect jobs that are sent directly to `Queue::push`. Always use `app('queue')` or `dispatch()` to queue your jobs.

<a name="running-tests"></a>
## Running Tests

As mentioned previously, once you've written tests, you may run them using `phpunit`:

    ./vendor/bin/phpunit

In addition to the `phpunit` command, you may use the `test` Run command to run your tests. The Run test runner provides verbose test reports in order to ease development and debugging:

    php run test

Any arguments that can be passed to the `phpunit` command may also be passed to the Run `test` command:

    php run test --testsuite=Feature --stop-on-failure

<a name="running-tests-in-parallel"></a>
### Running Tests in Parallel

By default, Framework and PHPUnit execute your tests sequentially within a single process. However, you may greatly reduce the amount of time it takes to run your tests by running tests simultaneously across multiple processes. To get started, you should install the `brianium/paratest` Composer package as a "dev" dependency. Then, include the `--parallel` option when executing the `test` Run command:

    composer require brianium/paratest --dev

    php run test --parallel

By default, Framework will create as many processes as there are available CPU cores on your machine. However, you may adjust the number of processes using the `--processes` option:

    php run test --parallel --processes=4

> [!WARNING]  
> When running tests in parallel, some PHPUnit options (such as `--do-not-cache-result`) may not be available.

<a name="parallel-testing-and-databases"></a>
#### Parallel Testing and Databases

As long as you have configured a primary database connection, Framework automatically handles creating and migrating a test database for each parallel process that is running your tests. The test databases will be suffixed with a process token which is unique per process. For example, if you have two parallel test processes, Framework will create and use `your_db_test_1` and `your_db_test_2` test databases.

By default, test databases persist between calls to the `test` Run command so that they can be used again by subsequent `test` invocations. However, you may re-create them using the `--recreate-databases` option:

    php run test --parallel --recreate-databases

<a name="parallel-testing-hooks"></a>
#### Parallel Testing Hooks

Occasionally, you may need to prepare certain resources used by your application's tests so they may be safely used by multiple test processes.

To safely initialize parallel testing without polluting your production code or causing errors when running optimization tasks via `--no-dev`, testing-specific service providers must never be registered inside application service providers. Instead, register the provider and declare its hooks directly inside your application's base `TestCase` or environment bootstrapping configuration layer (e.g., `tests/TestCase.php`):

    <?php

    namespace Tests;

    use PHPUnit\Framework\TestCase as BaseTestCase;

    abstract class TestCase extends BaseTestCase
    {
        /**
         * Bootstraps the application and isolates parallel testing logic.
         *
         * @return \MacropaySolutions\Framework\Application
         */
        public function createApplication()
        {
            $app = require __DIR__ . '/../bootstrap/app.php';

            // Safely register the testing provider exclusively during test suite runs
            $app->register(\MacropaySolutions\KernelDev\Testing\ParallelTestingServiceProvider::class);

            // Resolve the manager to safely attach parallel task workflows
            $parallelTesting = $app->make(\MacropaySolutions\KernelDev\Testing\ParallelTesting::class);

            $parallelTesting->setUpProcess(function (int $token) {
                // ...
            });

            $parallelTesting->setUpTestCase(function (int $token, BaseTestCase $testCase) {
                // ...
            });

            // Executed when a test database is created...
            $parallelTesting->setUpTestDatabase(function (string $database, int $token) {
                app('run')->call('db:seed');
            });

            $parallelTesting->tearDownTestCase(function (int $token, BaseTestCase $testCase) {
                // ...
            });

            $parallelTesting->tearDownProcess(function (int $token) {
                // ...
            });

            return $app;
        }
    }

<a name="accessing-the-parallel-testing-token"></a>
#### Accessing the Parallel Testing Token

If you would like to access the current parallel process "token" from any other location in your application's test code, you may resolve it via the global helper. This token is a unique, string identifier for an individual test process and may be used to segment resources across parallel test processes. For example, Framework automatically appends this token to the end of the test databases created by each parallel testing process:

    $token = \app(\MacropaySolutions\KernelDev\Testing\ParallelTesting::class)->token();

<a name="reporting-test-coverage"></a>
### Reporting Test Coverage

> [!WARNING]  
> This feature requires [Xdebug](https://xdebug.org) or [PCOV](https://pecl.php.net/package/pcov).

When running your application tests, you may want to determine whether your test cases are actually covering the application code and how much application code is used when running your tests. To accomplish this, you may provide the `--coverage` option when invoking the `test` command:

    php run test --coverage

<a name="enforcing-a-minimum-coverage-threshold"></a>
#### Enforcing a Minimum Coverage Threshold

You may use the `--min` option to define a minimum test coverage threshold for your application. The test suite will fail if this threshold is not met:

    php run test --coverage --min=80.3

<a name="profiling-tests"></a>
### Profiling Tests

The Run test runner also includes a convenient mechanism for listing your application's slowest tests. Invoke the `test` command with the `--profile` option to be presented with a list of your ten slowest tests, allowing you to easily investigate which tests can be improved to speed up your test suite:

    php run test --profile