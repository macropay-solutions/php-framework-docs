---
title: Mocking
description: Mocking objects and intercepting events, and jobs in PHP Framework tests.
context: mocking
---
# Mocking

- [Introduction](#introduction)
- [Mocking Objects](#mocking-objects)
- [Mocking Container Bindings](#mocking-container-bindings)
    - [Container Spies](#container-spies)
- [Interacting With Time](#interacting-with-time)

<a name="introduction"></a>
## Introduction

When testing Framework applications, you may wish to "mock" certain aspects of your application so they are not actually executed during a given test. For example, when testing a controller that dispatches an event, you may wish to mock the event listeners so they are not actually executed during the test. This allows you to only test the controller's HTTP response without worrying about the execution of the event listeners since the event listeners can be tested in their own test case.

Framework provides helpful methods for mocking events, jobs, and container bindings out of the box. These helpers primarily provide a convenience layer over Mockery so you do not have to manually make complicated Mockery method calls.

<a name="mocking-objects"></a>
## Mocking Objects

When mocking an object that is going to be injected into your application via Framework's [service container](/container), you will need to bind your mocked instance into the container as an `instance` binding. This will instruct the container to use your mocked instance of the object instead of constructing the object itself:

    use App\Service;
    use Mockery;
    use Mockery\MockInterface;

    public function test_something_can_be_mocked(): void
    {
        $this->instance(
            Service::class,
            Mockery::mock(Service::class, function (MockInterface $mock) {
                $mock->shouldReceive('process')->once();
            })
        );
    }

In order to make this more convenient, you may use the `mock` method that is provided by Framework's base test case class. For example, the following example is equivalent to the example above:

    use App\Service;
    use Mockery\MockInterface;

    $mock = $this->mock(Service::class, function (MockInterface $mock) {
        $mock->shouldReceive('process')->once();
    });

You may use the `partialMock` method when you only need to mock a few methods of an object. The methods that are not mocked will be executed normally when called:

    use App\Service;
    use Mockery\MockInterface;

    $mock = $this->partialMock(Service::class, function (MockInterface $mock) {
        $mock->shouldReceive('process')->once();
    });

Similarly, if you want to [spy](http://docs.mockery.io/en/latest/reference/spies.html) on an object, Framework's base test case class offers a `spy` method as a convenient wrapper around the `Mockery::spy` method. Spies are similar to mocks; however, spies record any interaction between the spy and the code being tested, allowing you to make assertions after the code is executed:

    use App\Service;

    $spy = $this->spy(Service::class);

    // ...

    $spy->shouldHaveReceived('process');

<a name="mocking-container-bindings"></a>
## Mocking Container Bindings

When testing, you may often want to mock a call to a Framework container binding that occurs in one of your controllers. For example, consider the following controller action:

    <?php

    namespace App\Http\Controllers;


    class UserController extends Controller
    {
        /**
         * Retrieve a list of all users of the application.
         */
        public function index(): array
        {
            $value = app('cache')->get('key');

            return [
                // ...
            ];
        }
    }

We can mock the call to the `cache` service. Since services are resolved and managed by the Framework [service container](/container), they have high testability. For example, let's mock our call to the `cache` service's `get` method:

    <?php

    namespace Tests\Feature;

    use Tests\TestCase;

    class UserControllerTest extends TestCase
    {
        public function test_get_index(): void
        {
            $mock = \Mockery::mock();
            $mock->shouldReceive('get')
                        ->once()
                        ->with('key')
                        ->andReturn('value');
            app()->instance('cache', $mock);

            $response = $this->get('/users');

            // ...
        }
    }

<a name="container-spies"></a>
### Container Spies

If you would like to [spy](http://docs.mockery.io/en/latest/reference/spies.html) on a container binding, you may use Mockery. Spies are similar to mocks; however, spies record any interaction between the spy and the code being tested, allowing you to make assertions after the code is executed:


    public function test_values_are_be_stored_in_cache(): void
    {
        $spy = \Mockery::spy();
        app()->instance('cache', $spy);

        $response = $this->get('/');

        $response->assertStatus(200);

        app('cache')->shouldHaveReceived('put')->once()->with('name', 'Surname', 10);
    }

<a name="interacting-with-time"></a>
## Interacting With Time

When testing, you may occasionally need to modify the time returned by helpers such as `now` or `MacropaySolutions\Kernel\Support\Carbon::now()`. Thankfully, Framework's base feature test class includes helpers that allow you to manipulate the current time:

    use MacropaySolutions\Kernel\Support\Carbon;

    public function test_time_can_be_manipulated(): void
    {
        // Travel into the future...
        $this->travel(5)->milliseconds();
        $this->travel(5)->seconds();
        $this->travel(5)->minutes();
        $this->travel(5)->hours();
        $this->travel(5)->days();
        $this->travel(5)->weeks();
        $this->travel(5)->years();

        // Travel into the past...
        $this->travel(-5)->hours();

        // Travel to an explicit time...
        $this->travelTo(now()->subHours(6));

        // Return back to the present time...
        $this->travelBack();
    }

You may also provide a closure to the various time travel methods. The closure will be invoked with time frozen at the specified time. Once the closure has executed, time will resume as normal:

    $this->travel(5)->days(function () {
        // Test something five days into the future...
    });
    
    $this->travelTo(now()->subDays(10), function () {
        // Test something during a given moment...
    });

The `freezeTime` method may be used to freeze the current time. Similarly, the `freezeSecond` method will freeze the current time but at the start of the current second:

    use MacropaySolutions\Kernel\Support\Carbon;

    // Freeze time and resume normal time after executing closure...
    $this->freezeTime(function (Carbon $time) {
        // ...
    });

    // Freeze time at the current second and resume normal time after executing closure...
    $this->freezeSecond(function (Carbon $time) {
        // ...
    })

As you would expect, all the methods discussed above are primarily useful for testing time sensitive application behavior, such as locking inactive posts on a discussion forum:

    use App\Models\Thread;
    
    public function test_forum_threads_lock_after_one_week_of_inactivity()
    {
        $thread = Thread::factory()->create();
        
        $this->travel(1)->week();
        
        $this->assertTrue($thread->isLockedByInactivity());
    }
