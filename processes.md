---
title: Processes
description: Invoking and managing external asynchronous processes in PHP Framework.
context: processes
---
# Processes

- [Introduction](#introduction)
- [Invoking Processes](#invoking-processes)
    - [Process Options](#process-options)
    - [Process Output](#process-output)
    - [Pipelines](#process-pipelines)
- [Asynchronous Processes](#asynchronous-processes)
    - [Process IDs and Signals](#process-ids-and-signals)
    - [Asynchronous Process Output](#asynchronous-process-output)
- [Concurrent Processes](#concurrent-processes)
    - [Naming Pool Processes](#naming-pool-processes)
    - [Pool Process IDs and Signals](#pool-process-ids-and-signals)
- [Testing](#testing)
    - [Faking Processes](#faking-processes)
    - [Faking Specific Processes](#faking-specific-processes)
    - [Faking Process Sequences](#faking-process-sequences)
    - [Faking Asynchronous Process Lifecycles](#faking-asynchronous-process-lifecycles)
    - [Available Assertions](#available-assertions)
    - [Preventing Stray Processes](#preventing-stray-processes)

<a name="introduction"></a>
## Introduction

Framework provides an expressive, minimal API around the [Symfony Process component](https://symfony.com/doc/current/components/process.html), allowing you to conveniently invoke external processes from your Framework application. Framework's process features are focused on the most common use cases and a wonderful developer experience.

<a name="invoking-processes"></a>
## Invoking Processes

To invoke a process, you may use the `run` and `start` methods offered by the `app(\MacropaySolutions\Kernel\Process\Factory::class)`. The `run` method will invoke a process and wait for the process to finish executing, while the `start` method is used for asynchronous process execution. We'll examine both approaches within this documentation. First, let's examine how to invoke a basic, synchronous process and inspect its result:

```php
$result = app(\MacropaySolutions\Kernel\Process\Factory::class)->run('ls -la');

return $result->output();
```

Of course, the `MacropaySolutions\Kernel\Contracts\Process\ProcessResult` instance returned by the `run` method offers a variety of helpful methods that may be used to inspect the process result:

```php
$result = app(\MacropaySolutions\Kernel\Process\Factory::class)->run('ls -la');

$result->successful();
$result->failed();
$result->exitCode();
$result->output();
$result->errorOutput();
```

<a name="throwing-exceptions"></a>
#### Throwing Exceptions

If you have a process result and would like to throw an instance of `MacropaySolutions\Kernel\Process\Exceptions\ProcessFailedException` if the exit code is greater than zero (thus indicating failure), you may use the `throw` and `throwIf` methods. If the process did not fail, the process result instance will be returned:

```php
$result = app(\MacropaySolutions\Kernel\Process\Factory::class)->run('ls -la')->throw();

$result = app(\MacropaySolutions\Kernel\Process\Factory::class)->run('ls -la')->throwIf($condition);
```

<a name="process-options"></a>
### Process Options

Of course, you may need to customize the behavior of a process before invoking it. Thankfully, Framework allows you to tweak a variety of process features, such as the working directory, timeout, and environment variables.

<a name="working-directory-path"></a>
#### Working Directory Path

You may use the `path` method to specify the working directory of the process. If this method is not invoked, the process will inherit the working directory of the currently executing PHP script:

```php
$result = app(\MacropaySolutions\Kernel\Process\Factory::class)->path(__DIR__)->run('ls -la');
```

<a name="input"></a>
#### Input

You may provide input via the "standard input" of the process using the `input` method:

```php
$result = app(\MacropaySolutions\Kernel\Process\Factory::class)->input('Hello World')->run('cat');
```

<a name="timeouts"></a>
#### Timeouts

By default, processes will throw an instance of `MacropaySolutions\Kernel\Process\Exceptions\ProcessTimedOutException` after executing for more than 60 seconds. However, you can customize this behavior via the `timeout` method:

```php
$result = app(\MacropaySolutions\Kernel\Process\Factory::class)->timeout(120)->run('bash import.sh');
```

Or, if you would like to disable the process timeout entirely, you may invoke the `forever` method:

```php
$result = app(\MacropaySolutions\Kernel\Process\Factory::class)->forever()->run('bash import.sh');
```

The `idleTimeout` method may be used to specify the maximum number of seconds the process may run without returning any output:

```php
$result = app(\MacropaySolutions\Kernel\Process\Factory::class)->timeout(60)->idleTimeout(30)->run('bash import.sh');
```

<a name="environment-variables"></a>
#### Environment Variables

Environment variables may be provided to the process via the `env` method. The invoked process will also inherit all the environment variables defined by your system:

```php
$result = app(\MacropaySolutions\Kernel\Process\Factory::class)->forever()
            ->env(['IMPORT_PATH' => __DIR__])
            ->run('bash import.sh');
```

If you wish to remove an inherited environment variable from the invoked process, you may provide that environment variable with a value of `false`:

```php
$result = app(\MacropaySolutions\Kernel\Process\Factory::class)->forever()
            ->env(['LOAD_PATH' => false])
            ->run('bash import.sh');
```

<a name="tty-mode"></a>
#### TTY Mode

The `tty` method may be used to enable TTY mode for your process. TTY mode connects the input and output of the process to the input and output of your program, allowing your process to open an editor like Vim or Nano as a process:

```php
app(\MacropaySolutions\Kernel\Process\Factory::class)->forever()->tty()->run('vim');
```

<a name="process-output"></a>
### Process Output

As previously discussed, process output may be accessed using the `output` (stdout) and `errorOutput` (stderr) methods on a process result:

```php
$result = app(\MacropaySolutions\Kernel\Process\Factory::class)->run('ls -la');

echo $result->output();
echo $result->errorOutput();
```

However, output may also be gathered in real-time by passing a closure as the second argument to the `run` method. The closure will receive two arguments: the "type" of output (`stdout` or `stderr`) and the output string itself:

```php
$result = app(\MacropaySolutions\Kernel\Process\Factory::class)->run('ls -la', function (string $type, string $output) {
    echo $output;
});
```

Framework also offers the `seeInOutput` and `seeInErrorOutput` methods, which provide a convenient way to determine if a given string was contained in the process' output:

```php
if (app(\MacropaySolutions\Kernel\Process\Factory::class)->run('ls -la')->seeInOutput('framework')) {
    // ...
}
```

<a name="disabling-process-output"></a>
#### Disabling Process Output

If your process is writing a significant amount of output that you are not interested in, you can conserve memory by disabling output retrieval entirely. To accomplish this, invoke the `quietly` method while building the process:

```php
$result = app(\MacropaySolutions\Kernel\Process\Factory::class)->quietly()->run('bash import.sh');
```

<a name="process-pipelines"></a>
### Pipelines

Sometimes you may want to make the output of one process the input of another process. This is often referred to as "piping" the output of a process into another. The `pipe` method provided by the `app(\MacropaySolutions\Kernel\Process\Factory::class)`s makes this easy to accomplish. The `pipe` method will execute the piped processes synchronously and return the process result for the last process in the pipeline:

```php
use MacropaySolutions\Kernel\Process\Pipe;


$result = app(\MacropaySolutions\Kernel\Process\Factory::class)->pipe(function (Pipe $pipe) {
    $pipe->command('cat example.txt');
    $pipe->command('grep -i "framework"');
});

if ($result->successful()) {
    // ...
}
```

If you do not need to customize the individual processes that make up the pipeline, you may simply pass an array of command strings to the `pipe` method:

```php
$result = app(\MacropaySolutions\Kernel\Process\Factory::class)->pipe([
    'cat example.txt',
    'grep -i "framework"',
]);
```

The process output may be gathered in real-time by passing a closure as the second argument to the `pipe` method. The closure will receive two arguments: the "type" of output (`stdout` or `stderr`) and the output string itself:

```php
$result = app(\MacropaySolutions\Kernel\Process\Factory::class)->pipe(function (Pipe $pipe) {
    $pipe->command('cat example.txt');
    $pipe->command('grep -i "framework"');
}, function (string $type, string $output) {
    echo $output;
});
```

Framework also allows you to assign string keys to each process within a pipeline via the `as` method. This key will also be passed to the output closure provided to the `pipe` method, allowing you to determine which process the output belongs to:

```php
$result = app(\MacropaySolutions\Kernel\Process\Factory::class)->pipe(function (Pipe $pipe) {
    $pipe->as('first')->command('cat example.txt');
    $pipe->as('second')->command('grep -i "framework"');
})->start(function (string $type, string $output, string $key) {
    // ...
});
```

<a name="asynchronous-processes"></a>
## Asynchronous Processes

While the `run` method invokes processes synchronously, the `start` method may be used to invoke a process asynchronously. This allows your application to continue performing other tasks while the process runs in the background. Once the process has been invoked, you may utilize the `running` method to determine if the process is still running:

```php
$process = app(\MacropaySolutions\Kernel\Process\Factory::class)->timeout(120)->start('bash import.sh');

while ($process->running()) {
    // ...
}

$result = $process->wait();
```

As you may have noticed, you may invoke the `wait` method to wait until the process is finished executing and retrieve the process result instance:

```php
$process = app(\MacropaySolutions\Kernel\Process\Factory::class)->timeout(120)->start('bash import.sh');

// ...

$result = $process->wait();
```

<a name="process-ids-and-signals"></a>
### Process IDs and Signals

The `id` method may be used to retrieve the operating system assigned process ID of the running process:

```php
$process = app(\MacropaySolutions\Kernel\Process\Factory::class)->start('bash import.sh');

return $process->id();
```

You may use the `signal` method to send a "signal" to the running process. A list of predefined signal constants can be found within the [PHP documentation](https://www.php.net/manual/en/pcntl.constants.php):

```php
$process->signal(SIGUSR2);
```

<a name="asynchronous-process-output"></a>
### Asynchronous Process Output

While an asynchronous process is running, you may access its entire current output using the `output` and `errorOutput` methods; however, you may utilize the `latestOutput` and `latestErrorOutput` to access the output from the process that has occurred since the output was last retrieved:

```php
$process = app(\MacropaySolutions\Kernel\Process\Factory::class)->timeout(120)->start('bash import.sh');

while ($process->running()) {
    echo $process->latestOutput();
    echo $process->latestErrorOutput();

    sleep(1);
}
```

Like the `run` method, output may also be gathered in real-time from asynchronous processes by passing a closure as the second argument to the `start` method. The closure will receive two arguments: the "type" of output (`stdout` or `stderr`) and the output string itself:

```php
$process = app(\MacropaySolutions\Kernel\Process\Factory::class)->start('bash import.sh', function (string $type, string $output) {
    echo $output;
});

$result = $process->wait();
```

<a name="concurrent-processes"></a>
## Concurrent Processes

Framework also makes it a breeze to manage a pool of concurrent, asynchronous processes, allowing you to easily execute many tasks simultaneously. To get started, invoke the `pool` method, which accepts a closure that receives an instance of `MacropaySolutions\Kernel\Process\Pool`.

Within this closure, you may define the processes that belong to the pool. Once a process pool is started via the `start` method, you may access the [collection](/collections) of running processes via the `running` method:

```php
use MacropaySolutions\Kernel\Process\Pool;


$pool = app(\MacropaySolutions\Kernel\Process\Factory::class)->pool(function (Pool $pool) {
    $pool->path(__DIR__)->command('bash import-1.sh');
    $pool->path(__DIR__)->command('bash import-2.sh');
    $pool->path(__DIR__)->command('bash import-3.sh');
})->start(function (string $type, string $output, int $key) {
    // ...
});

while ($pool->running()->isNotEmpty()) {
    // ...
}

$results = $pool->wait();
```

As you can see, you may wait for all the pool processes to finish executing and resolve their results via the `wait` method. The `wait` method returns an array accessible object that allows you to access the process result instance of each process in the pool by its key:

```php
$results = $pool->wait();

echo $results[0]->output();
```

Or, for convenience, the `concurrently` method may be used to start an asynchronous process pool and immediately wait on its results. This can provide particularly expressive syntax when combined with PHP's array destructuring capabilities:

```php
[$first, $second, $third] = app(\MacropaySolutions\Kernel\Process\Factory::class)->concurrently(function (Pool $pool) {
    $pool->path(__DIR__)->command('ls -la');
    $pool->path(app_path())->command('ls -la');
    $pool->path(storage_path())->command('ls -la');
});

echo $first->output();
```

<a name="naming-pool-processes"></a>
### Naming Pool Processes

Accessing process pool results via a numeric key is not very expressive; therefore, Framework allows you to assign string keys to each process within a pool via the `as` method. This key will also be passed to the closure provided to the `start` method, allowing you to determine which process the output belongs to:

```php
$pool = app(\MacropaySolutions\Kernel\Process\Factory::class)->pool(function (Pool $pool) {
    $pool->as('first')->command('bash import-1.sh');
    $pool->as('second')->command('bash import-2.sh');
    $pool->as('third')->command('bash import-3.sh');
})->start(function (string $type, string $output, string $key) {
    // ...
});

$results = $pool->wait();

return $results['first']->output();
```

<a name="pool-process-ids-and-signals"></a>
### Pool Process IDs and Signals

Since the process pool's `running` method provides a collection of all invoked processes within the pool, you may easily access the underlying pool process IDs:

```php
$processIds = $pool->running()->each->id();
```

And, for convenience, you may invoke the `signal` method on a process pool to send a signal to every process within the pool:

```php
$pool->signal(SIGUSR2);
```

<a name="testing"></a>
## Testing

Many Framework services provide functionality to help you easily and expressively write tests, and Framework's process service is no exception. The `app(\MacropaySolutions\Kernel\Process\Factory::class)`'s `fake` method allows you to instruct Framework to return stubbed / dummy results when processes are invoked.

<a name="faking-processes"></a>
### Faking Processes

To explore Framework's ability to fake processes, let's imagine a route that invokes a process:

```php
$router->get('/import', 'ImportController@import');

class ImportController extends Controller
{
    public function import() {
        app(\MacropaySolutions\Kernel\Process\Factory::class)->run('bash import.sh');
        return response()->json(['status' => 'Import complete!']);
    }
}
```

When testing this route, we can instruct Framework to return a fake, successful process result for every invoked process by calling the `fake` method on the `app(\MacropaySolutions\Kernel\Process\Factory::class)` with no arguments. In addition, we can even [assert](#available-assertions) that a given process was "run":

```php
<?php

namespace Tests\Feature;

use MacropaySolutions\Kernel\Process\PendingProcess;
use MacropaySolutions\Kernel\Contracts\Process\ProcessResult;

use Tests\TestCase;

class ExampleTest extends TestCase
{
    public function test_process_is_invoked(): void
    {
        app(\MacropaySolutions\Kernel\Process\Factory::class)->fake();

        $response = $this->get('/import');

        // Simple process assertion...
        app(\MacropaySolutions\Kernel\Process\Factory::class)->assertRan('bash import.sh');

        // Or, inspecting the process configuration...
        app(\MacropaySolutions\Kernel\Process\Factory::class)->assertRan(function (PendingProcess $process, ProcessResult $result) {
            return $process->command === 'bash import.sh' &&
                   $process->timeout === 60;
        });
    }
}
```

As discussed, invoking the `fake` method on the `app(\MacropaySolutions\Kernel\Process\Factory::class)` will instruct Framework to always return a successful process result with no output. However, you may easily specify the output and exit code for faked processes using the `app(\MacropaySolutions\Kernel\Process\Factory::class)`'s `result` method:

```php
app(\MacropaySolutions\Kernel\Process\Factory::class)->fake([
    '*' => app(\MacropaySolutions\Kernel\Process\Factory::class)->result(
        output: 'Test output',
        errorOutput: 'Test error output',
        exitCode: 1,
    ),
]);
```

<a name="faking-specific-processes"></a>
### Faking Specific Processes

As you may have noticed in a previous example, the `app(\MacropaySolutions\Kernel\Process\Factory::class)` allows you to specify different fake results per process by passing an array to the `fake` method.

The array's keys should represent command patterns that you wish to fake and their associated results. The `*` character may be used as a wildcard character. Any process commands that have not been faked will actually be invoked. You may use the `app(\MacropaySolutions\Kernel\Process\Factory::class)`'s `result` method to construct stub / fake results for these commands:

```php
app(\MacropaySolutions\Kernel\Process\Factory::class)->fake([
    'cat *' => app(\MacropaySolutions\Kernel\Process\Factory::class)->result(
        output: 'Test "cat" output',
    ),
    'ls *' => app(\MacropaySolutions\Kernel\Process\Factory::class)->result(
        output: 'Test "ls" output',
    ),
]);
```

If you do not need to customize the exit code or error output of a faked process, you may find it more convenient to specify the fake process results as simple strings:

```php
app(\MacropaySolutions\Kernel\Process\Factory::class)->fake([
    'cat *' => 'Test "cat" output',
    'ls *' => 'Test "ls" output',
]);
```

<a name="faking-process-sequences"></a>
### Faking Process Sequences

If the code you are testing invokes multiple processes with the same command, you may wish to assign a different fake process result to each process invocation. You may accomplish this via the `app(\MacropaySolutions\Kernel\Process\Factory::class)`'s `sequence` method:

```php
app(\MacropaySolutions\Kernel\Process\Factory::class)->fake([
    'ls *' => app(\MacropaySolutions\Kernel\Process\Factory::class)->sequence()
                ->push(app(\MacropaySolutions\Kernel\Process\Factory::class)->result('First invocation'))
                ->push(app(\MacropaySolutions\Kernel\Process\Factory::class)->result('Second invocation')),
]);
```

<a name="faking-asynchronous-process-lifecycles"></a>
### Faking Asynchronous Process Lifecycles

Thus far, we have primarily discussed faking processes which are invoked synchronously using the `run` method. However, if you are attempting to test code that interacts with asynchronous processes invoked via `start`, you may need a more sophisticated approach to describing your fake processes.

For example, let's imagine the following route which interacts with an asynchronous process:

```php

$router->get('/import', function () {
    $process = app(\MacropaySolutions\Kernel\Process\Factory::class)->start('bash import.sh');

    while ($process->running()) {
        \app('log')->info($process->latestOutput());
        \app('log')->info($process->latestErrorOutput());
    }

    return 'Done';
});
```

To properly fake this process, we need to be able to describe how many times the `running` method should return `true`. In addition, we may want to specify multiple lines of output that should be returned in sequence. To accomplish this, we can use the `app(\MacropaySolutions\Kernel\Process\Factory::class)`'s `describe` method:

```php
app(\MacropaySolutions\Kernel\Process\Factory::class)->fake([
    'bash import.sh' => app(\MacropaySolutions\Kernel\Process\Factory::class)->describe()
            ->output('First line of standard output')
            ->errorOutput('First line of error output')
            ->output('Second line of standard output')
            ->exitCode(0)
            ->iterations(3),
]);
```

Let's dig into the example above. Using the `output` and `errorOutput` methods, we may specify multiple lines of output that will be returned in sequence. The `exitCode` method may be used to specify the final exit code of the fake process. Finally, the `iterations` method may be used to specify how many times the `running` method should return `true`.

<a name="available-assertions"></a>
### Available Assertions

As [previously discussed](#faking-processes), Framework provides several process assertions for your feature tests. We'll discuss each of these assertions below.

<a name="assert-process-ran"></a>
#### assertRan

Assert that a given process was invoked:

```php


app(\MacropaySolutions\Kernel\Process\Factory::class)->assertRan('ls -la');
```

The `assertRan` method also accepts a closure, which will receive an instance of a process and a process result, allowing you to inspect the process' configured options. If this closure returns `true`, the assertion will "pass":

```php
app(\MacropaySolutions\Kernel\Process\Factory::class)->assertRan(fn ($process, $result) =>
    $process->command === 'ls -la' &&
    $process->path === __DIR__ &&
    $process->timeout === 60
);
```

The `$process` passed to the `assertRan` closure is an instance of `MacropaySolutions\Kernel\Process\PendingProcess`, while the `$result` is an instance of `MacropaySolutions\Kernel\Contracts\Process\ProcessResult`.

<a name="assert-process-didnt-run"></a>
#### assertDidntRun

Assert that a given process was not invoked:

```php


app(\MacropaySolutions\Kernel\Process\Factory::class)->assertDidntRun('ls -la');
```

Like the `assertRan` method, the `assertDidntRun` method also accepts a closure, which will receive an instance of a process and a process result, allowing you to inspect the process' configured options. If this closure returns `true`, the assertion will "fail":

```php
app(\MacropaySolutions\Kernel\Process\Factory::class)->assertDidntRun(fn (PendingProcess $process, ProcessResult $result) =>
    $process->command === 'ls -la'
);
```

<a name="assert-process-ran-times"></a>
#### assertRanTimes

Assert that a given process was invoked a given number of times:

```php


app(\MacropaySolutions\Kernel\Process\Factory::class)->assertRanTimes('ls -la', times: 3);
```

The `assertRanTimes` method also accepts a closure, which will receive an instance of a process and a process result, allowing you to inspect the process' configured options. If this closure returns `true` and the process was invoked the specified number of times, the assertion will "pass":

```php
app(\MacropaySolutions\Kernel\Process\Factory::class)->assertRanTimes(function (PendingProcess $process, ProcessResult $result) {
    return $process->command === 'ls -la';
}, times: 3);
```

<a name="preventing-stray-processes"></a>
### Preventing Stray Processes

If you would like to ensure that all invoked processes have been faked throughout your individual test or complete test suite, you can call the `preventStrayProcesses` method. After calling this method, any processes that do not have a corresponding fake result will throw an exception rather than starting an actual process:

    

    app(\MacropaySolutions\Kernel\Process\Factory::class)->preventStrayProcesses();

    app(\MacropaySolutions\Kernel\Process\Factory::class)->fake([
        'ls *' => 'Test output...',
    ]);

    // Fake response is returned...
    app(\MacropaySolutions\Kernel\Process\Factory::class)->run('ls -la');

    // An exception is thrown...
    app(\MacropaySolutions\Kernel\Process\Factory::class)->run('bash import.sh');
