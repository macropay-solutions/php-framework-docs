---
title: Database
description: Getting started with databases, configuration, SQL queries, transactions, and testing in PHP Framework.
context: database
---

# Database

- [Introduction](#introduction)
    - [Configuration](#configuration)
    - [Read and Write Connections](#read-and-write-connections)
- [Basic Usage & Running Queries](#running-queries)
    - [Using Multiple Database Connections](#using-multiple-database-connections)
    - [Listening for Query Events](#listening-for-query-events)
    - [Monitoring Cumulative Query Time](#monitoring-cumulative-query-time)
    - [Obvious ORM](#obvious-orm)
- [Database Transactions](#database-transactions)
- [Migrations](#migrations)
- [Connecting to the Database CLI](#connecting-to-the-database-cli)
- [Inspecting Your Databases](#inspecting-your-databases)
- [Monitoring Your Databases](#monitoring-your-databases)
- [Database Testing](#database-testing)
    - [Resetting the Database After Each Test](#resetting-the-database-after-each-test)
    - [Model Factories](#model-factories)
    - [Running Seeders](#running-seeders)
    - [Available Assertions](#available-assertions)

<a name="introduction"></a>
## Introduction

Almost every modern web application interacts with a database. PHP-Framework makes interacting with databases extremely simple across a variety of supported databases using raw SQL, a fluent query builder, and the Obvious ORM. Currently, PHP-Framework provides first-party support for five databases:

<div class="content-list" markdown="1">

- MariaDB 10.10+
- MySQL 5.7+
- PostgreSQL 11.0+
- SQLite 3.8.8+
- SQL Server 2017+

</div>

<a name="configuration"></a>
### Configuration

The configuration for PHP-Framework's database services is located in your application's `config/database.php` configuration file. In this file, you may define all of your database connections, as well as specify which connection should be used by default. Most of the configuration options within this file are driven by the values of your application's environment variables (`DB_*`).

<a name="sqlite-configuration"></a>
#### SQLite Configuration

SQLite databases are contained within a single file on your filesystem. You can create a new SQLite database using the `touch` command in your terminal: `touch database/database.sqlite`. After the database has been created, you may easily configure your environment variables to point to this database by placing the absolute path to the database in the `DB_DATABASE` environment variable:

    DB_CONNECTION=sqlite
    DB_DATABASE=/absolute/path/to/database.sqlite

To enable foreign key constraints for SQLite connections, you should set the `DB_FOREIGN_KEYS` environment variable to `true`:

    DB_FOREIGN_KEYS=true

<a name="mssql-configuration"></a>
#### Microsoft SQL Server Configuration

To use a Microsoft SQL Server database, you should ensure that you have the `sqlsrv` and `pdo_sqlsrv` PHP extensions installed as well as any dependencies they may require such as the Microsoft SQL ODBC driver.

<a name="configuration-using-urls"></a>
#### Configuration Using URLs

Typically, database connections are configured using multiple configuration values such as `host`, `database`, `username`, `password`, etc. Each of these configuration values has its own corresponding environment variable.

Some managed database providers provide a single database "URL" that contains all the connection information for the database in a single string. An example database URL may look something like the following:

    mysql://root:password@127.0.0.1/forge?charset=UTF-8

For convenience, PHP-Framework supports these URLs as an alternative to configuring your database with multiple configuration options. If the `url` (or corresponding `DATABASE_URL` environment variable) configuration option is present, it will be used to extract the database connection and credential information.

<a name="read-and-write-connections"></a>
### Read and Write Connections

Sometimes you may wish to use one database connection for SELECT statements, and another for INSERT, UPDATE, and DELETE statements. PHP-Framework makes this a breeze, and the proper connections will always be used whether you are using raw queries, the query builder, or the Obvious ORM.

To see how read / write connections should be configured, let's look at this example:

    'mysql' => [
        'read' => [
            'host' => [
                '192.168.1.1',
                '196.168.1.2',
            ],
        ],
        'write' => [
            'host' => [
                '196.168.1.3',
            ],
        ],
        'sticky' => true,
        'driver' => 'mysql',
        'database' => 'database',
        'username' => 'root',
        'password' => '',
        'charset' => 'utf8mb4',
        'collation' => 'utf8mb4_unicode_ci',
        'prefix' => '',
    ],

Note that three keys have been added to the configuration array: `read`, `write` and `sticky`. The `read` and `write` keys have array values containing a single key: `host`. The rest of the database options for the `read` and `write` connections will be merged from the main `mysql` configuration array.

When multiple values exist in the `host` configuration array, a database host will be randomly chosen for each request.

<a name="the-sticky-option"></a>
#### The `sticky` Option

The `sticky` option is an *optional* value that can be used to allow the immediate reading of records that have been written to the database during the current request cycle. If the `sticky` option is enabled and a "write" operation has been performed against the database during the current request cycle, any further "read" operations will use the "write" connection.

<a name="running-queries"></a>
## Basic Usage & Running Queries

<a name="running-a-select-query"></a>
#### Running a Select Query

To run a basic SELECT query, you may use the `select` method on the database component:

    <?php

    namespace App\Http\Controllers;

    use MacropaySolutions\Framework\Routing\Controller;

    class UserController extends Controller
    {
        public function index(): Response
        {
            $users = \app('db')->select('select * from users where active = ?', [1]);
            
            return response()->json($users);
        }
    }

The first argument passed to the `select` method is the SQL query, while the second argument is any parameter bindings that need to be bound to the query. Parameter binding provides protection against SQL injection.

<a name="selecting-scalar-values"></a>
#### Selecting Scalar Values

Sometimes your database query may result in a single, scalar value. You may retrieve this value directly using the `scalar` method:

    $burgers = app('db')->scalar(
        "select count(case when food = 'burger' then 1 end) as burgers from menu"
    );

<a name="selecting-multiple-result-sets"></a>
#### Selecting Multiple Result Sets

If your application calls stored procedures that return multiple result sets, you may use the `selectResultSets` method:

    [$options, $notifications] = app('db')->selectResultSets(
        "CALL get_user_options_and_notifications(?)", $request->user()->id
    );

<a name="using-named-bindings"></a>
#### Using Named Bindings

Instead of using `?` to represent your parameter bindings, you may execute a query using named bindings:

    $results = app('db')->select('select * from users where id = :id', ['id' => 1]);

<a name="running-an-insert-statement"></a>
#### Running an Insert Statement

To execute an `insert` statement, you may use the `insert` method. This method accepts the SQL query as its first argument and bindings as its second argument:

    app('db')->insert('insert into users (id, name) values (?, ?)', [1, 'Marc']);

<a name="running-an-update-statement"></a>
#### Running an Update Statement

The `update` method should be used to update existing records in the database. The number of rows affected by the statement is returned by the method:

    $affected = app('db')->update(
        'update users set votes = 100 where name = ?',
        ['Anita']
    );

<a name="running-a-delete-statement"></a>
#### Running a Delete Statement

The `delete` method should be used to delete records from the database. Like `update`, the number of rows affected will be returned by the method:

    $deleted = app('db')->delete('delete from users');

<a name="running-a-general-statement"></a>
#### Running a General Statement

Some database statements do not return any value. For these types of operations, you may use the `statement` method:

    app('db')->statement('drop table users');

<a name="running-an-unprepared-statement"></a>
#### Running an Unprepared Statement

Sometimes you may want to execute an SQL statement without binding any values. You may use the `unprepared` method to accomplish this:

    app('db')->unprepared('update users set votes = 100 where name = "Dries"');

> [!WARNING]  
> Since unprepared statements do not bind parameters, they may be vulnerable to SQL injection. You should never allow user-controlled values within an unprepared statement.

<a name="using-multiple-database-connections"></a>
### Using Multiple Database Connections

If your application defines multiple connections in your `config/database.php` configuration file, you may access each connection via the `connection` method:

    $users = app('db')->connection('sqlite')->select(/* ... */);

You may access the raw, underlying PDO instance of a connection using the `getPdo` method on a connection instance:

    $pdo = app('db')->connection()->getPdo();

<a name="listening-for-query-events"></a>
### Listening for Query Events

If you would like to specify a closure that is invoked for each SQL query executed by your application, you may use the `listen` method. This method can be useful for logging queries or debugging. You may register your query listener closure in the `boot` method of a service provider:

    <?php

    namespace App\Providers;

    use MacropaySolutions\Kernel\Database\Events\QueryExecuted;
    use MacropaySolutions\Kernel\Support\ServiceProvider;

    class AppServiceProvider extends ServiceProvider
    {
        public function boot(): void
        {
            app('db')->listen(function (QueryExecuted $query) {
                // $query->sql;
                // $query->bindings;
                // $query->time;
            });
        }
    }

<a name="monitoring-cumulative-query-time"></a>
### Monitoring Cumulative Query Time

PHP-Framework can invoke a closure or callback of your choice when it spends too much time querying the database during a single request. Provide a query time threshold (in milliseconds) and closure to the `whenQueryingForLongerThan` method in the `boot` method of a service provider:

    public function boot(): void
    {
        app('db')->whenQueryingForLongerThan(500, function (Connection $connection, QueryExecuted $event) {
            // Notify development team...
        });
    }

<a name="obvious-orm"></a>
#### Obvious ORM

Of course, you may easily use the full Obvious ORM with PHP-Framework. To learn how to use Obvious, check out the [Obvious ORM documentation](/obvious).

<a name="database-transactions"></a>
## Database Transactions

You may use the `transaction` method to run a set of operations within a database transaction. If an exception is thrown within the transaction closure, the transaction will automatically be rolled back and the exception is re-thrown. If the closure executes successfully, the transaction will automatically be committed:

    app('db')->transaction(function () {
        app('db')->update('update users set votes = 1');
        app('db')->delete('delete from posts');
    });

<a name="handling-deadlocks"></a>
#### Handling Deadlocks

The `transaction` method accepts an optional second argument which defines the number of times a transaction should be retried when a deadlock occurs. Once these attempts have been exhausted, an exception will be thrown:

    app('db')->transaction(function () {
        app('db')->update('update users set votes = 1');
        app('db')->delete('delete from posts');
    }, 5);

<a name="manually-using-transactions"></a>
#### Manually Using Transactions

If you would like to begin a transaction manually and have complete control over rollbacks and commits, you may use the `beginTransaction` method:

    app('db')->beginTransaction();

You can rollback the transaction via the `rollBack` method:

    app('db')->rollBack();

Lastly, you can commit a transaction via the `commit` method:

    app('db')->commit();

<a name="migrations"></a>
## Migrations

For further information on how to create database tables and run migrations, check out the standalone [migrations documentation](/migrations).

<a name="connecting-to-the-database-cli"></a>
## Connecting to the Database CLI

If you would like to connect to your database's CLI, you may use the `db` command:

    php run db

If needed, you may specify a database connection name to connect to a database connection that is not the default connection:

    php run db mysql

<a name="inspecting-your-databases"></a>
## Inspecting Your Databases

Using the `db:show` and `db:table` commands, you can get valuable insight into your database and its associated tables. To see an overview of your database, including its size, type, number of open connections, and a summary of its tables:

    php run db:show

<a name="monitoring-your-databases"></a>
## Monitoring Your Databases

Using the `db:monitor` command, you can instruct PHP-Framework to dispatch an `MacropaySolutions\Kernel\Database\Events\DatabaseBusy` event if your database is managing more than a specified number of open connections.

    php run db:monitor --databases=mysql,pgsql --max=100

---

<a name="database-testing"></a>
## Database Testing

PHP-Framework provides a variety of helpful tools and assertions to make it easier to test your database-driven applications. In addition, model factories and seeders make it painless to create test database records using your application's Obvious models and relationships.

<a name="resetting-the-database-after-each-test"></a>
### Resetting the Database After Each Test

To reset your database after each of your tests so that data from a previous test does not interfere with subsequent tests, use the `RefreshDatabase` trait:

    <?php

    namespace Tests\Feature;

    use MacropaySolutions\KernelDev\Testing\RefreshDatabase;
    use Tests\TestCase;

    class ExampleTest extends TestCase
    {
        use RefreshDatabase;

        public function test_basic_example(): void
        {
            $response = $this->get('/');
            // ...
        }
    }

The `RefreshDatabase` trait does not migrate your database if your schema is up to date. Instead, it will only execute the test within a database transaction.

If you would like to totally reset the database, you may use the `DatabaseMigrations` or `DatabaseTruncation` traits from the `MacropaySolutions\KernelDev\Foundation\Testing` namespace. However, both of these options are significantly slower than the `RefreshDatabase` trait.

<a name="model-factories"></a>
### Model Factories

When testing, you may need to insert a few records into your database before executing your test. PHP-Framework allows you to define a set of default attributes for each of your Obvious models using model factories. Once you have defined a model factory, you may utilize the factory within your test to create models:

    use App\Models\User;

    public function test_models_can_be_instantiated(): void
    {
        $user = User::factory()->create();
        // ...
    }

<a name="running-seeders"></a>
### Running Seeders

If you would like to use database seeders to populate your database during a feature test, you may invoke the `seed` method. By default, the `seed` method will execute the `DatabaseSeeder`, which should execute all of your other seeders:

    public function test_orders_can_be_created(): void
    {
        // Run the DatabaseSeeder...
        $this->seed();

        // Run a specific seeder...
        $this->seed(\Database\Seeders\OrderStatusSeeder::class);
    }

<a name="available-assertions"></a>
### Available Assertions

PHP-Framework provides several database assertions for your PHPUnit feature tests.

#### assertDatabaseCount

Assert that a table in the database contains the given number of records:

    $this->assertDatabaseCount('users', 5);

#### assertDatabaseHas

Assert that a table in the database contains records matching the given key / value query constraints:

    $this->assertDatabaseHas('users', [
        'email' => 'sally@example.com',
    ]);

#### assertDatabaseMissing

Assert that a table in the database does not contain records matching the given key / value query constraints:

    $this->assertDatabaseMissing('users', [
        'email' => 'sally@example.com',
    ]);

#### assertSoftDeleted

The `assertSoftDeleted` method may be used to assert a given Obvious model has been "soft deleted":

    $this->assertSoftDeleted($user);

#### assertNotSoftDeleted

The `assertNotSoftDeleted` method may be used to assert a given Obvious model hasn't been "soft deleted":

    $this->assertNotSoftDeleted($user);

#### assertModelExists

Assert that a given model exists in the database:

    $this->assertModelExists($user);

#### assertModelMissing

Assert that a given model does not exist in the database:

    $this->assertModelMissing($user);

#### expectsDatabaseQueryCount

The `expectsDatabaseQueryCount` method may be invoked at the beginning of your test to specify the total number of database queries that you expect to be run during the test:

    $this->expectsDatabaseQueryCount(5);