---
title: Run Console
description: Guide to writing, registering, and executing Run console commands in PHP Framework.
context: run
---

# Run Console

- [Introduction](#introduction)
- [Writing Commands](#writing-commands)
  - [Generating Commands](#generating-commands)
  - [Command Structure](#command-structure)
  - [Isolatable Commands](#isolatable-commands)
- [Defining Input Expectations](#defining-input-expectations)
  - [Arguments](#arguments)
  - [Options](#options)
  - [Input Arrays](#input-arrays)
  - [Input Descriptions](#input-descriptions)
  - [Prompting for Missing Input](#prompting-for-missing-input)
- [Command I/O](#command-io)
  - [Retrieving Input](#retrieving-input)
  - [Prompting for Input](#prompting-for-input)
  - [Writing Output](#writing-output)
- [Registering Commands](#registering-commands)
- [Programmatically Executing Commands](#programmatically-executing-commands)
  - [Calling Commands From Other Commands](#calling-commands-from-other-commands)
- [Signal Handling](#signal-handling)
- [Stub Customization](#stub-customization)
- [Events](#events)
- [Built-in Commands](#built-in-commands)
- [Prohibited Commands](#prohibited-commands)

<a name="introduction"></a>
## Introduction

Run is the command line interface included with Framework. Run exists at the root of your application as the `run` script and provides a number of helpful commands that can assist you while you build your application. To view a list of all available Run commands, you may use the `list` command:

```shell
php run list
```

Every command also includes a "help" screen which displays and describes the command's available arguments and options. To view a help screen, precede the name of the command with `help`:

```shell
php run help migrate
```

## Writing Commands

In addition to the commands provided with Run, you may build your own custom commands. Commands are typically stored in the `app/Console/Commands` directory; however, you are free to choose your own storage location as long as your commands can be loaded by Composer.

<a name="generating-commands"></a>
### Generating Commands

To create a new command, you may use the `make:command` Run command. This command will create a new command class in the `app/Console/Commands` directory. Don't worry if this directory does not exist in your application - it will be created the first time you run the `make:command` Run command:

```shell
php run make:command SendEmails
```

<a name="command-structure"></a>
### Command Structure

After generating your command, you should define appropriate values for the `signature` and `description` properties of the class. These properties will be used when displaying your command on the `list` screen. The `signature` property also allows you to define [your command's input expectations](#defining-input-expectations). The `handle` method will be called when your command is executed. You may place your command logic in this method.

Let's take a look at an example command. Note that we are able to request any dependencies we need via the command's `handle` method. The Framework [service container](/container) will automatically inject all dependencies that are type-hinted in this method's signature:

    <?php

    namespace App\Console\Commands;

    use App\Models\User;
    use App\Support\DripEmailer;
    use MacropaySolutions\Kernel\Console\Command;

    class SendEmails extends Command
    {
        /**
         * The name and signature of the console command.
         *
         * @var string
         */
        protected $signature = 'mail:send {user}';

        /**
         * The console command description.
         *
         * @var string
         */
        protected $description = 'Send a marketing email to a user';

        /**
         * Execute the console command.
         */
        public function handle(DripEmailer $drip): void
        {
            $drip->send(User::getQuery()->find($this->argument('user')));
        }
    }

> [!NOTE]  
> For greater code reuse, it is good practice to keep your console commands light and let them defer to application services to accomplish their tasks. In the example above, note that we inject a service class to do the "heavy lifting" of sending the e-mails.

<a name="isolatable-commands"></a>
### Isolatable Commands

> [!WARNING]  
> To utilize this feature, your application must be using the `memcached`, `redis`, `dynamodb`, `database`, `file`, or `array` cache driver as your application's default cache driver. In addition, all servers must be communicating with the same central cache server.

Sometimes you may wish to ensure that only one instance of a command can run at a time. To accomplish this, you may implement the `MacropaySolutions\Kernel\Contracts\Console\Isolatable` interface on your command class:

    <?php

    namespace App\Console\Commands;

    use MacropaySolutions\Kernel\Console\Command;
    use MacropaySolutions\Kernel\Contracts\Console\Isolatable;

    class SendEmails extends Command implements Isolatable
    {
        // ...
    }

When a command is marked as `Isolatable`, Framework will automatically add an `--isolated` option to the command. When the command is invoked with that option, Framework will ensure that no other instances of that command are already running. Framework accomplishes this by attempting to acquire an atomic lock using your application's default cache driver. If other instances of the command are running, the command will not execute; however, the command will still exit with a successful exit status code:

```shell
php run mail:send 1 --isolated
```

If you would like to specify the exit status code that the command should return if it is not able to execute, you may provide the desired status code via the `isolated` option:

```shell
php run mail:send 1 --isolated=12
```

<a name="lock-id"></a>
#### Lock ID

By default, Framework will use the command's name to generate the string key that is used to acquire the atomic lock in your application's cache. However, you may customize this key by defining an `isolatableId` method on your Run command class, allowing you to integrate the command's arguments or options into the key:

```php
/**
 * Get the isolatable ID for the command.
 */
public function isolatableId(): string
{
    return $this->argument('user');
}
```

<a name="lock-expiration-time"></a>
#### Lock Expiration Time

By default, isolation locks expire after the command is finished. Or, if the command is interrupted and unable to finish, the lock will expire after one hour. However, you may adjust the lock expiration time by defining an `isolationLockExpiresAt` method on your command:

```php
use DateTimeInterface;
use DateInterval;

/**
 * Determine when an isolation lock expires for the command.
 */
public function isolationLockExpiresAt(): DateTimeInterface|DateInterval
{
    return now()->addMinutes(5);
}
```

<a name="defining-input-expectations"></a>
## Defining Input Expectations

When writing console commands, it is common to gather input from the user through arguments or options. Framework makes it very convenient to define the input you expect from the user using the `signature` property on your commands. The `signature` property allows you to define the name, arguments, and options for the command in a single, expressive, route-like syntax.

<a name="arguments"></a>
### Arguments

All user supplied arguments and options are wrapped in curly braces. In the following example, the command defines one required argument: `user`:

    /**
     * The name and signature of the console command.
     *
     * @var string
     */
    protected $signature = 'mail:send {user}';

You may also make arguments optional or define default values for arguments:

    // Optional argument...
    'mail:send {user?}'

    // Optional argument with default value...
    'mail:send {user=foo}'

<a name="options"></a>
### Options

Options, like arguments, are another form of user input. Options are prefixed by two hyphens (`--`) when they are provided via the command line. There are two types of options: those that receive a value and those that don't. Options that don't receive a value serve as a boolean "switch". Let's take a look at an example of this type of option:

    /**
     * The name and signature of the console command.
     *
     * @var string
     */
    protected $signature = 'mail:send {user} {--force}';

In this example, the `--force` switch may be specified when calling the Run command. If the `--force` switch is passed, the value of the option will be `true`. Otherwise, the value will be `false`:

```shell
php run mail:send 1 --force
```

<a name="options-with-values"></a>
#### Options With Values

Next, let's take a look at an option that expects a value. If the user must specify a value for an option, you should suffix the option name with a `=` sign:

    /**
     * The name and signature of the console command.
     *
     * @var string
     */
    protected $signature = 'mail:send {user} {--status=}';

In this example, the user may pass a value for the option like so. If the option is not specified when invoking the command, its value will be `null`:

```shell
php run mail:send 1 --status=default
```

You may assign default values to options by specifying the default value after the option name. If no option value is passed by the user, the default value will be used:

    'mail:send {user} {--status=default}'

<a name="option-shortcuts"></a>
#### Option Shortcuts

To assign a shortcut when defining an option, you may specify it before the option name and use the `|` character as a delimiter to separate the shortcut from the full option name:

    'mail:send {user} {--S|status}'

When invoking the command on your terminal, option shortcuts should be prefixed with a single hyphen and no `=` character should be included when specifying a value for the option:

```shell
php run mail:send 1 -Sdefault
```

<a name="input-arrays"></a>
### Input Arrays

If you would like to define arguments or options to expect multiple input values, you may use the `*` character. First, let's take a look at an example that specifies such an argument:

    'mail:send {user*}'

When calling this method, the `user` arguments may be passed in order to the command line. For example, the following command will set the value of `user` to an array with `1` and `2` as its values:

```shell
php run mail:send 1 2
```

This `*` character can be combined with an optional argument definition to allow zero or more instances of an argument:

    'mail:send {user?*}'

<a name="option-arrays"></a>
#### Option Arrays

When defining an option that expects multiple input values, each option value passed to the command should be prefixed with the option name:

    'mail:send {--id=*}'

Such a command may be invoked by passing multiple `--id` arguments:

```shell
php run mail:send --id=1 --id=2
```

<a name="input-descriptions"></a>
### Input Descriptions

You may assign descriptions to input arguments and options by separating the argument name from the description using a colon. If you need a little extra room to define your command, feel free to spread the definition across multiple lines:

    /**
     * The name and signature of the console command.
     *
     * @var string
     */
    protected $signature = 'mail:send
      {user : The ID of the user}
      {--pretend : Simulate sending the email without actually delivering it}';

<a name="prompting-for-missing-input"></a>
### Prompting for Missing Input

If your command contains required arguments, the user will receive an error message when they are not provided. Alternatively, you may configure your command to automatically prompt the user when required arguments are missing by implementing the `PromptsForMissingInput` interface:

    <?php

    namespace App\Console\Commands;

    use MacropaySolutions\Kernel\Console\Command;
    use MacropaySolutions\Kernel\Contracts\Console\PromptsForMissingInput;

    class SendEmails extends Command implements PromptsForMissingInput
    {
        /**
         * The name and signature of the console command.
         *
         * @var string
         */
        protected $signature = 'mail:send {user}';

        // ...
    }

If Framework needs to gather a required argument from the user, it will automatically ask the user for the argument by intelligently phrasing the question using either the argument name or description. If you wish to customize the question used to gather the required argument, you may implement the `promptForMissingArgumentsUsing` method, returning an array of questions keyed by the argument names:

    /**
     * Prompt for missing input arguments using the returned questions.
     *
     * @return array
     */
    protected function promptForMissingArgumentsUsing()
    {
        return [
            'user' => 'Which user ID should receive the mail?',
        ];
    }

You may also provide placeholder text by using a tuple containing the question and placeholder:

    return [
        'user' => ['Which user ID should receive the mail?', 'E.g. 123'],
    ];

If you would like complete control over the prompt, you may provide a closure that should prompt the user and return their answer:

    use App\Models\User;
    use function MacropaySolutions\Prompts\search;

    // ...

    return [
        'user' => fn () => search(
            label: 'Search for a user:',
            placeholder: 'E.g. Surname Name',
            options: fn ($value) => strlen($value) > 0
                ? User::getQuery()->where('name', 'like', "%{$value}%")->pluck('name', 'id')->all()
                : []
        ),
    ];

> [!NOTE]  
> The comprehensive [Framework Prompts](/prompts) documentation includes additional information on the available prompts and their usage.

If you wish to prompt the user to select or enter [options](#options), you may include prompts in your command's `handle` method. However, if you only wish to prompt the user when they have also been automatically prompted for missing arguments, then you may implement the `afterPromptingForMissingArguments` method:

    use Symfony\Component\Console\Input\InputInterface;
    use Symfony\Component\Console\Output\OutputInterface;
    use function MacropaySolutions\Prompts\confirm;

    // ...

    /**
     * Perform actions after the user was prompted for missing arguments.
     *
     * @param  \Symfony\Component\Console\Input\InputInterface  $input
     * @param  \Symfony\Component\Console\Output\OutputInterface  $output
     * @return void
     */
    protected function afterPromptingForMissingArguments(InputInterface $input, OutputInterface $output)
    {
        $input->setOption('force', confirm(
            label: 'Would you like to force the mail?',
            default: $this->option('force')
        ));
    }

<a name="command-io"></a>
## Command I/O

<a name="retrieving-input"></a>
### Retrieving Input

While your command is executing, you will likely need to access the values for the arguments and options accepted by your command. To do so, you may use the `argument` and `option` methods. If an argument or option does not exist, `null` will be returned:

    /**
     * Execute the console command.
     */
    public function handle(): void
    {
        $userId = $this->argument('user');
    }

If you need to retrieve all the arguments as an `array`, call the `arguments` method:

    $arguments = $this->arguments();

Options may be retrieved just as easily as arguments using the `option` method. To retrieve all the options as an array, call the `options` method:

    // Retrieve a specific option...
    $status = $this->option('status');

    // Retrieve all options as an array...
    $options = $this->options();

<a name="prompting-for-input"></a>
### Prompting for Input

> [!NOTE]  
> [Framework Prompts](/prompts) is a PHP package for adding beautiful and user-friendly forms to your command-line applications, with browser-like features including placeholder text and validation.

In addition to displaying output, you may also ask the user to provide input during the execution of your command. The `ask` method will prompt the user with the given question, accept their input, and then return the user's input back to your command:

    /**
     * Execute the console command.
     */
    public function handle(): void
    {
        $name = $this->ask('What is your name?');

        // ...
    }

The `ask` method also accepts an optional second argument which specifies the default value that should be returned if no user input is provided:

    $name = $this->ask('What is your name?', 'Surname');

The `secret` method is similar to `ask`, but the user's input will not be visible to them as they type in the console. This method is useful when asking for sensitive information such as passwords:

    $password = $this->secret('What is the password?');

<a name="asking-for-confirmation"></a>
#### Asking for Confirmation

If you need to ask the user for a simple "yes or no" confirmation, you may use the `confirm` method. By default, this method will return `false`. However, if the user enters `y` or `yes` in response to the prompt, the method will return `true`.

    if ($this->confirm('Do you wish to continue?')) {
        // ...
    }

If necessary, you may specify that the confirmation prompt should return `true` by default by passing `true` as the second argument to the `confirm` method:

    if ($this->confirm('Do you wish to continue?', true)) {
        // ...
    }

<a name="auto-completion"></a>
#### Auto-Completion

The `anticipate` method can be used to provide auto-completion for possible choices. The user can still provide any answer, regardless of the auto-completion hints:

    $name = $this->anticipate('What is your name?', ['Surname', 'Dayle']);

Alternatively, you may pass a closure as the second argument to the `anticipate` method. The closure will be called each time the user types an input character. The closure should accept a string parameter containing the user's input so far, and return an array of options for auto-completion:

    $name = $this->anticipate('What is your address?', function (string $input) {
        // Return auto-completion options...
    });

<a name="multiple-choice-questions"></a>
#### Multiple Choice Questions

If you need to give the user a predefined set of choices when asking a question, you may use the `choice` method. You may set the array index of the default value to be returned if no option is chosen by passing the index as the third argument to the method:

    $name = $this->choice(
        'What is your name?',
        ['Surname', 'Dayle'],
        $defaultIndex
    );

In addition, the `choice` method accepts optional fourth and fifth arguments for determining the maximum number of attempts to select a valid response and whether multiple selections are permitted:

    $name = $this->choice(
        'What is your name?',
        ['Surname', 'Dayle'],
        $defaultIndex,
        $maxAttempts = null,
        $allowMultipleSelections = false
    );

<a name="writing-output"></a>
### Writing Output

To send output to the console, you may use the `line`, `info`, `comment`, `question`, `warn`, and `error` methods. Each of these methods will use appropriate ANSI colors for their purpose. For example, let's display some general information to the user. Typically, the `info` method will display in the console as green colored text:

    /**
     * Execute the console command.
     */
    public function handle(): void
    {
        // ...

        $this->info('The command was successful!');
    }

To display an error message, use the `error` method. Error message text is typically displayed in red:

    $this->error('Something went wrong!');

You may use the `line` method to display plain, uncolored text:

    $this->line('Display this on the screen');

You may use the `newLine` method to display a blank line:

    // Write a single blank line...
    $this->newLine();

    // Write three blank lines...
    $this->newLine(3);

<a name="tables"></a>
#### Tables

The `table` method makes it easy to correctly format multiple rows / columns of data. All you need to do is provide the column names and the data for the table and Framework will automatically calculate the appropriate width and height of the table for you:

    use App\Models\User;

    $this->table(
        ['Name', 'Email'],
        User::getQuery()->all(['name', 'email'])->toArray()
    );

<a name="progress-bars"></a>
#### Progress Bars

For long running tasks, it can be helpful to show a progress bar that informs users how complete the task is. Using the `withProgressBar` method, Framework will display a progress bar and advance its progress for each iteration over a given iterable value:

    use App\Models\User;

    $users = $this->withProgressBar(User::getQuery()->all(), function (User $user) {
        $this->performTask($user);
    });

Sometimes, you may need more manual control over how a progress bar is advanced. First, define the total number of steps the process will iterate through. Then, advance the progress bar after processing each item:

    $users = App\Models\User::getQuery()->all();

    $bar = $this->output->createProgressBar(count($users));

    $bar->start();

    foreach ($users as $user) {
        $this->performTask($user);

        $bar->advance();
    }

    $bar->finish();

> [!NOTE]  
> For more advanced options, check out the [Symfony Progress Bar component documentation](https://symfony.com/doc/current/components/console/helpers/progressbar.html).

<a name="registering-commands"></a>
## Registering Commands

All of your console commands are registered within your application's `App\Console\Kernel` class. To register your commands, you must manually add the command's class name to the `$commands` property within your kernel.

When the console application boots, all the commands listed in this property (along with framework defaults like `ScheduleRunCommand`) will be resolved and registered:

    protected $commands = [
        \App\Console\Commands\SendEmails::class,
    ];

To optimize performance, you can use the `commands:cache` command, which creates a `bootstrap/cache/commands.php` file that facilitates lazy loading of your registered commands.

<a name="programmatically-executing-commands"></a>
## Programmatically Executing Commands

Sometimes you may wish to execute a Run command outside of the CLI. For example, you might want to fire a command directly from a controller. You can accomplish this by resolving the console kernel via dependency injection and using its `call` method.

The `call` method accepts either the command's signature name or class name as its first argument, and an array of command parameters as its second argument. Upon execution, the command's exit code will be returned:

    namespace App\Controllers;

    use MacropaySolutions\Kernel\Contracts\Console\Kernel;

    class MailController
    {
        public function send(string $user, Kernel $kernel)
        {
            $exitCode = $kernel->call('mail:send', [
                'user' => $user,
            ]);

            // ...
        }
    }

Alternatively, you may pass the entire Run command to the `call` method as a single string. If you are in a context where you cannot easily inject the kernel, you may resolve it directly from the application container:

    \app(\MacropaySolutions\Kernel\Contracts\Console\Kernel::class)->call('mail:send 1');

<a name="passing-array-values"></a>
#### Passing Array Values

If your command defines an option that accepts an array, you may pass an array of values to that option:

    \app(\MacropaySolutions\Kernel\Contracts\Console\Kernel::class)->call('mail:send', [
        '--id' => [5, 13]
    ]);

<a name="passing-boolean-values"></a>
#### Passing Boolean Values

If you need to specify the value of an option that does not accept string values, such as the `--force` flag on the `migrate:refresh` command, you should pass `true` or `false` as the value of the option:

    $exitCode = \app(\MacropaySolutions\Kernel\Contracts\Console\Kernel::class)->call('migrate:refresh', [
        '--force' => true,
    ]);

<a name="queueing-commands"></a>
#### Queueing Commands

> [!WARNING]  
> **Queueing Commands is Unsupported:** Queueing console commands for background execution is not supported in PHP Framework. If you need to execute tasks in the background, you should dispatch a queued Job or Storable Array Callable instead or just use ->runInBackground() in scheduler.

<a name="calling-commands-from-other-commands"></a>
### Calling Commands From Other Commands

Sometimes you may wish to call other commands from an existing Run command. You may do so using the `call` method, which accepts the command name and an array of command arguments/options:

    /**
     * Execute the console command.
     */
    public function handle(): void
    {
        $this->call('mail:send', [
            'user' => 1
        ]);

        // ...
    }

If you would like to call another console command and suppress all of its output, you may use the `callSilently` method. The `callSilently` method has the same signature as the `call` method:

    $this->callSilently('mail:send', [
        'user' => 1
    ]);

<a name="signal-handling"></a>
## Signal Handling

As you may know, operating systems allow signals to be sent to running processes. For example, the `SIGTERM` signal is how operating systems ask a program to terminate. If you wish to listen for signals in your Run console commands and execute code when they occur, you may use the `trap` method:

    /**
     * Execute the console command.
     */
    public function handle(): void
    {
        $this->trap(SIGTERM, fn () => $this->shouldKeepRunning = false);

        while ($this->shouldKeepRunning) {
            // ...
        }
    }

To listen for multiple signals at once, you may provide an array of signals to the `trap` method:

    $this->trap([SIGTERM, SIGQUIT], function (int $signal) {
        $this->shouldKeepRunning = false;

        dump($signal); // SIGTERM / SIGQUIT
    });

<a name="stub-customization"></a>
## Stub Customization

The Run console's `make` commands are used to create a variety of classes, such as controllers, jobs, migrations, and tests. These classes are generated using "stub" files that are populated with values based on your input. However, you may want to make small changes to files generated by Run. To accomplish this, you may use the `stub:publish` command to publish the most common stubs to your application so that you can customize them:

```shell
php run stub:publish
```

The published stubs will be located within a `stubs` directory in the root of your application. Any changes you make to these stubs will be reflected when you generate their corresponding classes using Run's `make` commands.

<a name="events"></a>
## Events

Run dispatches three events when running commands: `MacropaySolutions\Kernel\Console\Events\RunStarting`, `MacropaySolutions\Kernel\Console\Events\CommandStarting`, and `MacropaySolutions\Kernel\Console\Events\CommandFinished`. The `RunStarting` event is dispatched immediately when Run starts running. Next, the `CommandStarting` event is dispatched immediately before a command runs. Finally, the `CommandFinished` event is dispatched once a command finishes executing.

## Built-in Commands

```shell
php run commands:cache
```
Will create a bootstrap/cache/commands.php that facilitates commands lazy loading.

Many development commands, including generators (like `make:command` or `make:factory`), schema utilities, and diagnostics, are registered and provided directly by the `macropay-solutions/php-kernel-dev` package.

These are the built-in commands depending on the composer flag `--no-dev`:

```php

<?php return array (
  'migrate' => 'command.migrate',
  'migrate:fresh' => 'command.migrate.fresh',
  'migrate:install' => 'command.migrate.install',
  'migrate:refresh' => 'command.migrate.refresh',
  'migrate:reset' => 'command.migrate.reset',
  'migrate:rollback' => 'command.migrate.rollback',
  'migrate:status' => 'command.migrate.status',
  'make:migration' => 'command.migrate.make',
  'autowiring:cache' => 'command.autowiring.cache',
  'autowiring:clear' => 'command.autowiring.clear',
  'event:cache' => 'command.event.cache',
  'event:clear' => 'command.event.clear',
  'view:cache' => 'command.view.cache',
  'view:clear' => 'command.view.clear',
  'cache:clear' => 'command.cache.clear',
  'cache:forget' => 'command.cache.forget',
  'auth:clear-resets' => 'command.auth.resets.clear',
  'commands:cache' => 'command.commands.cache',
  'commands:clear' => 'command.commands.clear',
  'db:seed' => 'command.seed',
  'schedule:finish' => 'command.schedule.finish',
  'schedule:run' => 'MacropaySolutions\Kernel\\Console\\Scheduling\\ScheduleRunCommand',
  'schedule:work' => 'command.schedule.work',
  'db:wipe' => 'command.wipe',
  'schema:dump' => 'command.schema.dump',
  'cache:table' => 'command.cache.table',
  'queue:failed-table' => 'command.queue.failed-table',
  'queue:batches-table' => 'command.queue.batches-table',
  'queue:table' => 'command.queue.table',
  'make:seeder' => 'command.seeder.make',
  'about' => 'MacropaySolutions\\KernelDev\\Foundation\\Console\\AboutCommand',
  'make:cast' => 'MacropaySolutions\\KernelDev\\Foundation\\Console\\CastMakeCommand',
  'channel:list' => 'MacropaySolutions\\KernelDev\\Foundation\\Console\\ChannelListCommand',
  'make:channel' => 'MacropaySolutions\\KernelDev\\Foundation\\Console\\ChannelMakeCommand',
  'config:show' => 'MacropaySolutions\\KernelDev\\Foundation\\Console\\ConfigShowCommand',
  'make:command' => 'MacropaySolutions\\KernelDev\\Foundation\\Console\\ConsoleMakeCommand',
  'make:controller' => 'MacropaySolutions\\KernelDev\\Routing\\Console\\ControllerMakeCommand',
  'docs' => 'MacropaySolutions\\KernelDev\\Foundation\\Console\\DocsCommand',
  'env:encrypt' => 'MacropaySolutions\\KernelDev\\Foundation\\Console\\EnvironmentEncryptCommand',
  'event:list' => 'MacropaySolutions\\KernelDev\\Foundation\\Console\\EventListCommand',
  'make:event' => 'MacropaySolutions\\KernelDev\\Foundation\\Console\\EventMakeCommand',
  'make:exception' => 'MacropaySolutions\\KernelDev\\Foundation\\Console\\ExceptionMakeCommand',
  'make:factory' => 'MacropaySolutions\\KernelDev\\Database\\Console\\Factories\\FactoryMakeCommand',
  'make:job' => 'MacropaySolutions\\KernelDev\\Foundation\\Console\\JobMakeCommand',
  'key:generate' => 'MacropaySolutions\\KernelDev\\Foundation\\Console\\KeyGenerateCommand',
  'lang:publish' => 'MacropaySolutions\\KernelDev\\Foundation\\Console\\LangPublishCommand',
  'make:listener' => 'MacropaySolutions\\KernelDev\\Foundation\\Console\\ListenerMakeCommand',
  'make:mail' => 'MacropaySolutions\\KernelDev\\Foundation\\Console\\MailMakeCommand',
  'make:middleware' => 'MacropaySolutions\\KernelDev\\Routing\\Console\\MiddlewareMakeCommand',
  'make:model' => 'MacropaySolutions\\KernelDev\\Foundation\\Console\\ModelMakeCommand',
  'make:notification' => 'MacropaySolutions\\KernelDev\\Foundation\\Console\\NotificationMakeCommand',
  'notifications:table' => 'MacropaySolutions\\KernelDev\\Notifications\\Console\\NotificationTableCommand',
  'make:observer' => 'MacropaySolutions\\KernelDev\\Foundation\\Console\\ObserverMakeCommand',
  'make:policy' => 'MacropaySolutions\\KernelDev\\Foundation\\Console\\PolicyMakeCommand',
  'make:provider' => 'MacropaySolutions\\KernelDev\\Foundation\\Console\\ProviderMakeCommand',
  'make:request' => 'MacropaySolutions\\KernelDev\\Foundation\\Console\\RequestMakeCommand',
  'make:resource' => 'MacropaySolutions\\KernelDev\\Foundation\\Console\\ResourceMakeCommand',
  'make:rule' => 'MacropaySolutions\\KernelDev\\Foundation\\Console\\RuleMakeCommand',
  'schedule:list' => 'MacropaySolutions\\KernelDev\\Console\\Scheduling\\ScheduleListCommand',
  'schedule:test' => 'MacropaySolutions\\KernelDev\\Console\\Scheduling\\ScheduleTestCommand',
  'make:scope' => 'MacropaySolutions\\KernelDev\\Foundation\\Console\\ScopeMakeCommand',
  'serve' => 'MacropaySolutions\\KernelDev\\Foundation\\Console\\ServeCommand',
  'session:table' => 'MacropaySolutions\\KernelDev\\Session\\Console\\SessionTableCommand',
  'db:show' => 'MacropaySolutions\\KernelDev\\Database\\Console\\ShowCommand',
  'model:show' => 'MacropaySolutions\\KernelDev\\Database\\Console\\ShowModelCommand',
  'db:table' => 'MacropaySolutions\\KernelDev\\Database\\Console\\TableCommand',
  'make:test' => 'MacropaySolutions\\KernelDev\\Foundation\\Console\\TestMakeCommand',
  'make:view' => 'MacropaySolutions\\KernelDev\\Foundation\\Console\\ViewMakeCommand',
  'config:cache' => 'App\\Console\\Commands\\ConfigCacheCommand',
  'config:clear' => 'App\\Console\\Commands\\ConfigClearCommand',
  'route:cache' => 'App\\Console\\Commands\\RouteCacheCommand',
  'route:clear' => 'App\\Console\\Commands\\RouteClearCommand',
);
```
```php

<?php return array (
  'migrate' => 'command.migrate',
  'migrate:install' => 'command.migrate.install',
  'migrate:rollback' => 'command.migrate.rollback',
  'migrate:status' => 'command.migrate.status',
  'autowiring:cache' => 'command.autowiring.cache',
  'autowiring:clear' => 'command.autowiring.clear',
  'event:cache' => 'command.event.cache',
  'event:clear' => 'command.event.clear',
  'view:cache' => 'command.view.cache',
  'view:clear' => 'command.view.clear',
  'cache:clear' => 'command.cache.clear',
  'cache:forget' => 'command.cache.forget',
  'auth:clear-resets' => 'command.auth.resets.clear',
  'commands:cache' => 'command.commands.cache',
  'commands:clear' => 'command.commands.clear',
  'schedule:finish' => 'command.schedule.finish',
  'schedule:run' => 'MacropaySolutions\Kernel\\Console\\Scheduling\\ScheduleRunCommand',
  'schedule:work' => 'command.schedule.work',
  'config:cache' => 'App\\Console\\Commands\\ConfigCacheCommand',
  'config:clear' => 'App\\Console\\Commands\\ConfigClearCommand',
  'route:cache' => 'App\\Console\\Commands\\RouteCacheCommand',
  'route:clear' => 'App\\Console\\Commands\\RouteClearCommand',
);

```


## Prohibited Commands

`\MacropaySolutions\Kernel\Console\Prohibitable` trait prohibits by default any command that uses it from running in production, unless coded otherwise via `CommandFQN::prohibit(false);`

    FreshCommand   \MacropaySolutions\Kernel\Database\Console\Migrations
    ResetCommand   \MacropaySolutions\Kernel\Database\Console\Migrations
    RefreshCommand   \MacropaySolutions\Kernel\Database\Console\Migrations
    SeedCommand   \MacropaySolutions\Kernel\Database\Console\Seeds