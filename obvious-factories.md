---
title: Obvious Factories
description: Guide to defining, creating, and using model factories for testing and seeding in PHP Framework.
context: obvious-factories
---

# Obvious: Factories

- [Introduction](#introduction)
- [Defining Model Factories](#defining-model-factories)
    - [Generating Factories](#generating-factories)
    - [Factory States](#factory-states)
    - [Factory Callbacks](#factory-callbacks)
- [Creating Models Using Factories](#creating-models-using-factories)
    - [Instantiating Models](#instantiating-models)
    - [Persisting Models](#persisting-models)
    - [Sequences](#sequences)
- [Factory Relationships](#factory-relationships)
    - [Has Many Relationships](#has-many-relationships)
    - [Belongs To Relationships](#belongs-to-relationships)
    - [Many to Many Relationships](#many-to-many-relationships)
    - [Defining Relationships Within Factories](#defining-relationships-within-factories)
    - [Recycling an Existing Model for Relationships](#recycling-an-existing-model-for-relationships)

<a name="introduction"></a>
## Introduction

When testing your application or seeding your database, you may need to insert a few records into your database. Instead of manually specifying the value of each column, Framework allows you to define a set of default attributes for each of your [Obvious models](/obvious) using model factories.

To see an example of how to write a factory, take a look at the `database/factories/UserFactory.php` file in your application. This factory is included with all new Framework applications and contains the following factory definition:

    namespace Database\Factories;

    use MacropaySolutions\Kernel\Support\Str;
    use MacropaySolutions\KernelDev\Database\Obvious\Factories\Factory;

    class UserFactory extends Factory
    {
        /**
         * Define the model's default state.
         *
         * @return array<string, mixed>
         */
        public function definition(): array
        {
            return [
                'name' => fake()->name(),
                'email' => fake()->unique()->safeEmail(),
                'email_verified_at' => now(),
                'password' => '$2y$10$92IXUNpkjO0rOQ5byMi.Ye4oKoEa3Ro9llC/.og/at2.uheWG/igi', // password
                'remember_token' => Str::random(10),
            ];
        }
    }

As you can see, in their most basic form, factories are classes that extend Framework's base factory class and define a `definition` method. The `definition` method returns the default set of attribute values that should be applied when creating a model using the factory.

Via the `fake` helper, factories have access to the [Faker](https://github.com/FakerPHP/Faker) PHP library, which allows you to conveniently generate various kinds of random data for testing and seeding.

> [!NOTE]  
> You can set your application's Faker locale by adding a `faker_locale` option to your `config/app.php` configuration file.

<a name="defining-model-factories"></a>
## Defining Model Factories

<a name="generating-factories"></a>
### Generating Factories

To create a factory, execute the `make:factory` [Run command](/run):

```shell
php run make:factory PostFactory
```

The new factory class will be placed in your `database/factories` directory.

<a name="factory-and-model-discovery-conventions"></a>
#### Model and Factory Discovery Conventions

Factories belong exclusively to the dev environment (`MacropaySolutions\KernelDev\Database\Obvious\Factories\Factory`) and are instantiated directly via `UserFactory::new()` to prevent production trait pollution.

If your factory does not follow standard naming conventions, define a `$model` property on the factory class:

    use App\Administration\Flight;
    use MacropaySolutions\KernelDev\Database\Obvious\Factories\Factory;

    class FlightFactory extends Factory
    {
        /**
         * The name of the factory's corresponding model.
         *
         * @var class-string<\MacropaySolutions\Kernel\Database\Obvious\Model>
         */
        protected $model = Flight::class;
    }

<a name="factory-states"></a>
### Factory States

State manipulation methods allow you to define discrete modifications that can be applied to your model factories in any combination. For example, your `Database\Factories\UserFactory` factory might contain a `suspended` state method that modifies one of its default attribute values.

State transformation methods typically call the `state` method provided by Framework's base factory class. The `state` method accepts a closure which will receive the array of raw attributes defined for the factory and should return an array of attributes to modify:

    use MacropaySolutions\KernelDev\Database\Obvious\Factories\Factory;

    /**
     * Indicate that the user is suspended.
     */
    public function suspended(): Factory
    {
        return $this->state(function (array $attributes) {
            return [
                'account_status' => 'suspended',
            ];
        });
    }

<a name="trashed-state"></a>
#### "Trashed" State

If your Obvious model can be [soft deleted](/obvious#soft-deleting), you may invoke the built-in `trashed` state method to indicate that the created model should already be "soft deleted". You do not need to manually define the `trashed` state as it is automatically available to all factories:

    use Database\Factories\UserFactory;

    $user = UserFactory::new()->trashed()->create();

<a name="factory-callbacks"></a>
### Factory Callbacks

Factory callbacks are registered using the `afterMaking` and `afterCreating` methods and allow you to perform additional tasks after making or creating a model. You should register these callbacks by defining a `configure` method on your factory class. This method will be automatically called by Framework when the factory is instantiated:

    namespace Database\Factories;

    use App\Models\User;
    use MacropaySolutions\KernelDev\Database\Obvious\Factories\Factory;

    class UserFactory extends Factory
    {
        /**
         * Configure the model factory.
         */
        public function configure(): static
        {
            return $this->afterMaking(function (User $user) {
                // ...
            })->afterCreating(function (User $user) {
                // ...
            });
        }

        // ...
    }

You may also register factory callbacks within state methods to perform additional tasks that are specific to a given state:

    use App\Models\User;
    use MacropaySolutions\KernelDev\Database\Obvious\Factories\Factory;

    /**
     * Indicate that the user is suspended.
     */
    public function suspended(): Factory
    {
        return $this->state(function (array $attributes) {
            return [
                'account_status' => 'suspended',
            ];
        })->afterMaking(function (User $user) {
            // ...
        })->afterCreating(function (User $user) {
            // ...
        });
    }

<a name="creating-models-using-factories"></a>
## Creating Models Using Factories

<a name="instantiating-models"></a>
### Instantiating Models

To instantiate models without persisting them to the database, invoke `new()` on your factory class followed by `make()`:

    use Database\Factories\UserFactory;

    $user = UserFactory::new()->make();

You may create a collection of many models using the `count` method:

    $users = UserFactory::new()->count(3)->make();

<a name="applying-states"></a>
#### Applying States

You may also apply any of your [states](#factory-states) to the models. If you would like to apply multiple state transformations to the models, you may simply call the state transformation methods directly:

    $users = UserFactory::new()->count(5)->suspended()->make();

<a name="overriding-attributes"></a>
#### Overriding Attributes

If you would like to override some of the default values of your models, you may pass an array of values to the `make` method. Only the specified attributes will be replaced while the rest of the attributes remain set to their default values as specified by the factory:

    $user = UserFactory::new()->make([
        'name' => 'Abigail Name',
    ]);

Alternatively, the `state` method may be called directly on the factory instance to perform an inline state transformation:

    $user = UserFactory::new()->state([
        'name' => 'Abigail Name',
    ])->make();

> [!NOTE]  
> [Mass assignment protection](/obvious#mass-assignment) is automatically disabled when creating models using factories.

<a name="persisting-models"></a>
### Persisting Models

The `create` method instantiates model instances and persists them to the database using Obvious's `save` method:

    use Database\Factories\UserFactory;

    // Create a single User instance...
    $user = UserFactory::new()->create();

    // Create three User instances...
    $users = UserFactory::new()->count(3)->create();

You may override the factory's default model attributes by passing an array of attributes to the `create` method:

    $user = UserFactory::new()->create([
        'name' => 'Abigail',
    ]);

<a name="sequences"></a>
### Sequences

Sometimes you may wish to alternate the value of a given model attribute for each created model. You may accomplish this by defining a state transformation as a sequence. For example, you may wish to alternate the value of an `admin` column between `Y` and `N` for each created user:

    use Database\Factories\UserFactory;
    use MacropaySolutions\KernelDev\Database\Obvious\Factories\Sequence;

    $users = UserFactory::new()
                    ->count(10)
                    ->state(new Sequence(
                        ['admin' => 'Y'],
                        ['admin' => 'N'],
                    ))
                    ->create();

In this example, five users will be created with an `admin` value of `Y` and five users will be created with an `admin` value of `N`.

If necessary, you may include a closure as a sequence value. The closure will be invoked each time the sequence needs a new value:

    use MacropaySolutions\KernelDev\Database\Obvious\Factories\Sequence;

    $users = UserFactory::new()
                    ->count(10)
                    ->state(new Sequence(
                        fn(Sequence $sequence) => ['role' => UserRoles::query()->all()->random()],
                    ))
                    ->create();

Within a sequence closure, you may access the `$index` or `$count` properties on the sequence instance that is injected into the closure. The `$index` property contains the number of iterations through the sequence that have occurred thus far, while the `$count` property contains the total number of times the sequence will be invoked:

    $users = UserFactory::new()
                    ->count(10)
                    ->sequence(fn(Sequence $sequence) => ['name' => 'Name '.$sequence->index])
                    ->create();

For convenience, sequences may also be applied using the `sequence` method, which simply invokes the `state` method internally. The `sequence` method accepts a closure or arrays of sequenced attributes:

    $users = UserFactory::new()
                    ->count(2)
                    ->sequence(
                        ['name' => 'First User'],
                        ['name' => 'Second User'],
                    )
                    ->create();

<a name="factory-relationships"></a>
## Factory Relationships

<a name="has-many-relationships"></a>
### Has Many Relationships

Next, let's explore building Obvious model relationships using Framework's fluent factory methods. First, let's assume our application has an `App\Models\User` model and an `App\Models\Post` model. Also, let's assume that the `User` model defines a `hasMany` relationship with `Post`. We can create a user that has three posts using the `has` method provided by the Framework's factories. The `has` method accepts a factory instance:

    use Database\Factories\PostFactory;
    use Database\Factories\UserFactory;

    $user = UserFactory::new()
                ->has(PostFactory::new()->count(3))
                ->create();

By convention, when passing a `Post` model to the `has` method, Framework will assume that the `User` model must have a `posts` method that defines the relationship. If necessary, you may explicitly specify the name of the relationship that you would like to manipulate:

    $user = UserFactory::new()
                ->has(PostFactory::new()->count(3), 'posts')
                ->create();

Of course, you may perform state manipulations on the related models. In addition, you may pass a closure based state transformation if your state change requires access to the parent model:

    $user = UserFactory::new()
                ->has(
                    PostFactory::new()
                            ->count(3)
                            ->state(function (array $attributes, User $user) {
                                return ['user_type' => $user->a->type];
                            })
                )
                ->create();

<a name="has-many-relationships-using-magic-methods"></a>
#### Using Magic Methods

For convenience, you may use Framework's magic factory relationship methods to build relationships. For example, the following example will use convention to determine that the related models should be created via a `posts` relationship method on the `User` model:

    $user = UserFactory::new()
                ->hasPosts(3)
                ->create();

When using magic methods to create factory relationships, you may pass an array of attributes to override on the related models:

    $user = UserFactory::new()
                ->hasPosts(3, [
                    'published' => false,
                ])
                ->create();

You may provide a closure based state transformation if your state change requires access to the parent model:

    $user = UserFactory::new()
                ->hasPosts(3, function (array $attributes, User $user) {
                    return ['user_type' => $user->a->type];
                })
                ->create();

<a name="belongs-to-relationships"></a>
### Belongs To Relationships

Now that we have explored how to build "has many" relationships using factories, let's explore the inverse of the relationship. The `for` method may be used to define the parent model that factory created models belong to. For example, we can create three `App\Models\Post` model instances that belong to a single user:

    use Database\Factories\PostFactory;
    use Database\Factories\UserFactory;

    $posts = PostFactory::new()
                ->count(3)
                ->for(UserFactory::new()->state([
                    'name' => 'Jessica Archer',
                ]))
                ->create();

If you already have a parent model instance that should be associated with the models you are creating, you may pass the model instance to the `for` method:

    $user = UserFactory::new()->create();

    $posts = PostFactory::new()
                ->count(3)
                ->for($user)
                ->create();

<a name="belongs-to-relationships-using-magic-methods"></a>
#### Using Magic Methods

For convenience, you may use Framework's magic factory relationship methods to define "belongs to" relationships. For example, the following example will use convention to determine that the three posts should belong to the `user` relationship on the `Post` model:

    $posts = PostFactory::new()
                ->count(3)
                ->forUser([
                    'name' => 'Jessica Archer',
                ])
                ->create();

<a name="many-to-many-relationships"></a>
### Many to Many Relationships

Like [has many relationships](#has-many-relationships), "many to many" relationships may be created using the `has` method:

    use Database\Factories\RoleFactory;
    use Database\Factories\UserFactory;

    $user = UserFactory::new()
                ->has(RoleFactory::new()->count(3))
                ->create();

<a name="pivot-table-attributes"></a>
#### Pivot Table Attributes

If you need to define attributes that should be set on the pivot / intermediate table linking the models, you may use the `hasAttached` method. This method accepts an array of pivot table attribute names and values as its second argument:

    use App\Models\User;
    use Database\Factories\RoleFactory;
    use Database\Factories\UserFactory;

    $user = UserFactory::new()
                ->hasAttached(
                    RoleFactory::new()->count(3),
                    ['active' => true]
                )
                ->create();

You may provide a closure based state transformation if your state change requires access to the related model:

    $user = UserFactory::new()
                ->hasAttached(
                    RoleFactory::new()
                        ->count(3)
                        ->state(function (array $attributes, User $user) {
                            return ['name' => $user->a->name.' Role'];
                        }),
                    ['active' => true]
                )
                ->create();

If you already have model instances that you would like to be attached to the models you are creating, you may pass the model instances to the `hasAttached` method. In this example, the same three roles will be attached to all three users:

    $roles = RoleFactory::new()->count(3)->create();

    $user = UserFactory::new()
                ->count(3)
                ->hasAttached($roles, ['active' => true])
                ->create();

<a name="many-to-many-relationships-using-magic-methods"></a>
#### Using Magic Methods

For convenience, you may use Framework's magic factory relationship methods to define many to many relationships. For example, the following example will use convention to determine that the related models should be created via a `roles` relationship method on the `User` model:

    $user = UserFactory::new()
                ->hasRoles(1, [
                    'name' => 'Editor'
                ])
                ->create();

<a name="defining-relationships-within-factories"></a>
### Defining Relationships Within Factories

To define a relationship within your model factory, you will typically assign a new factory instance to the foreign key of the relationship. This is normally done for the "inverse" relationships such as `belongsTo` relationships. For example, if you would like to create a new user when creating a post, you may do the following:

    use Database\Factories\UserFactory;

    /**
     * Define the model's default state.
     *
     * @return array<string, mixed>
     */
    public function definition(): array
    {
        return [
            'user_id' => UserFactory::new(),
            'title' => fake()->title(),
            'content' => fake()->paragraph(),
        ];
    }

If the relationship's columns depend on the factory that defines it you may assign a closure to an attribute. The closure will receive the factory's evaluated attribute array:

    /**
     * Define the model's default state.
     *
     * @return array<string, mixed>
     */
    public function definition(): array
    {
        return [
            'user_id' => UserFactory::new(),
            'user_type' => function (array $attributes) {
                return User::query()->find($attributes['user_id'])->a->type;
            },
            'title' => fake()->title(),
            'content' => fake()->paragraph(),
        ];
    }

<a name="recycling-an-existing-model-for-relationships"></a>
### Recycling an Existing Model for Relationships

If you have models that share a common relationship with another model, you may use the `recycle` method to ensure a single instance of the related model is recycled for all the relationships created by the factory.

For example, imagine you have `Airline`, `Flight`, and `Ticket` models, where the ticket belongs to an airline and a flight, and the flight also belongs to an airline. When creating tickets, you will probably want the same airline for both the ticket and the flight, so you may pass an airline instance to the `recycle` method:

    TicketFactory::new()
        ->recycle(AirlineFactory::new()->create())
        ->create();

You may find the `recycle` method particularly useful if you have models belonging to a common user or team.

The `recycle` method also accepts a collection of existing models. When a collection is provided to the `recycle` method, a random model from the collection will be chosen when the factory needs a model of that type:

    TicketFactory::new()
        ->recycle($airlines)
        ->create();
