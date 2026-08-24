---
title: Obvious Mutators & Casting
description: Guide to transforming data using accessors, mutators, and attribute casting in PHP Framework.
context: obvious-mutators
---

# Obvious: Mutators & Casting

- [Introduction](#introduction)
- [Accessors and Mutators](#accessors-and-mutators)
  - [Defining Accessors](#defining-accessors)
  - [Defining Mutators](#defining-mutators)
- [Attribute Casting](#attribute-casting)
  - [Enum Casting](#enum-casting)

<a name="introduction"></a>
## Introduction

Accessors, mutators, and attribute casting allow you to transform Obvious attribute values when you retrieve or set them on model instances. For example, you may want to use the [Framework encrypter](/encryption) to encrypt a value while it is stored in the database, and then automatically decrypt the attribute when you access it on an Obvious model. Or, you may want to convert a JSON string that is stored in your database to an array when it is accessed via your Obvious model.

<a name="accessors-and-mutators"></a>
## Accessors and Mutators

Framework provides a high-performance architecture for accessors and mutators using Segregated Maps. These maps use O(1) static lookups to resolve closures, completely bypassing the overhead of dynamic method calls and reflection.

> [!WARNING]
> **Strict Return Type Enforcement:** Accessors and mutators are strictly restricted to scalar primitives (`int`, `string`, `null`) and `\BackedEnum` instances.

<a name="defining-accessors"></a>
### Defining Accessors

An accessor transforms an Obvious attribute value when it is accessed. To define accessors, override the `segregatedAccessorsMap` method on your model. This method should return an array where the keys are attribute names and the values are closures:

    <?php

    namespace App\Models;

    use MacropaySolutions\Kernel\Database\Obvious\Model;

    class User extends Model
    {
        /**
         * Define the segregated accessors for the model.
         */
        protected function segregatedAccessorsMap(): array
        {
            return [
                'first_name' => fn(?string $value): ?string => $value !== null ? \ucfirst($value) : null,
                'full_name' => fn(): string => $this->a->first_name . ' ' . $this->a->last_name,
                'status' => fn(?string $value): ?ServerStatus => $value !== null ? ServerStatus::from($value) : null,
            ];
        }
    }

As you can see, the original value of the column is passed to the accessor closure, allowing you to manipulate and return the computed value. To access the value of the accessor, you simply use the `a` attribute accessor on a model instance:

    use App\Models\User;

    $user = User::query()->find(1);

    $firstName = $user->a->first_name;
    $fullName = $user->a->full_name;

> [!NOTE]  
> Closures in segregated maps are automatically bound to the model instance (`$this`). Do not use static closures!

<a name="defining-mutators"></a>
### Defining Mutators

A mutator transforms an Obvious attribute value when it is set. To define mutators, override the `segregatedMutatorsMap` method. These closures receive the value being set and interact directly with the model's internal attributes array:

    <?php

    namespace App\Models;

    use MacropaySolutions\Kernel\Database\Obvious\Model;

    class User extends Model
    {
        /**
         * Define the segregated mutators for the model.
         */
        protected function segregatedMutatorsMap(): array
        {
            return [
                'first_name' => function (?string $value): void {
                    $this->attributes['first_name'] = $value !== null ? \strtolower($value) : null;
                },
                'status' => fn(?string|ServerStatus $value): void => \is_string($value) ? ServerStatus::from($value) : null,
            ];
        }
    }

To use our mutator, we only need to set the `first_name` attribute via the `a` accessor on an Obvious model:

    use App\Models\User;

    $user = User::query()->find(1);

    $user->a->first_name = 'Sally';

<a name="attribute-casting"></a>
## Attribute Casting


Attribute casting provides functionality similar to accessors and mutators without requiring you to define any additional methods on your model. Instead, your model's `$casts` property provides a convenient method of converting attributes to common data types.

> [!WARNING]
> **Strict Primitive Casts Only:** To eliminate dynamic type churn, state-synchronization flaws, and array diffing overhead on `save()`, `$casts` strictly permits only primitive types (`int`, `string`) and `\BackedEnum` class-strings. Complex objects, Carbon instances, arrays, JSON objects, and custom class casting are forbidden in storage arrays.

The `$casts` property should be an array where the key is the attribute name and the value is the primitive type or `\BackedEnum` FQN:

<div class="content-list" markdown="1">

- `int`
- `string`
- `\App\Enums\YourBackedEnum::class`

</div>

To demonstrate attribute casting, let's cast the `role_id` attribute to an integer:

    <?php

    namespace App\Models;

    use MacropaySolutions\Kernel\Database\Obvious\Model;

    class User extends Model
    {
        /**
         * The attributes that should be cast.
         *
         * @var array
         */
        protected $casts = [
            'role_id' => 'int',
        ];
    }

    $user = App\Models\User::query()->find(1);

    if ($user->a->role_id === 1) {
        // ...
    }

If you need to add a new, temporary cast at runtime, you may use the `mergeCasts` method. These cast definitions will be added to any of the casts already defined on the model:

    $user->mergeCasts([
        'is_admin' => 'int',
    ]);

> [!WARNING]  
> Attributes that are `null` will not be cast. In addition, you should never define a cast (or an attribute) that has the same name as a relationship or assign a cast to the model's primary key.

### Enum Casting

Obvious also allows you to cast your attribute values to PHP [Enums](https://www.php.net/manual/en/language.enumerations.backed.php). To accomplish this, you may specify the attribute and enum you wish to cast in your model's `$casts` property array:

    use App\Enums\ServerStatus;

    /**
     * The attributes that should be cast.
     *
     * @var array
     */
    protected $casts = [
        'status' => ServerStatus::class,
    ];

Once you have defined the cast on your model, the specified attribute will be automatically cast to and from an enum when you interact with the attribute:

    if ($server->a->status == ServerStatus::Provisioned) {
        $server->a->status = ServerStatus::Ready;

        $server->save();
    }

### Query Time Casting

Sometimes you may need to apply casts while executing a query, such as when selecting a raw value from a table. For example, consider the following query:

    use App\Models\Post;
    use App\Models\User;

    $users = User::query()->select([
        'users.*',
        'last_posted_at' => Post::query()->selectRaw('MAX(created_at)')
                ->whereColumn('user_id', 'users.id')
    ])->get();

The `last_posted_at` attribute on the results of this query will be a simple string. It would be wonderful if we could apply a `string` cast to this attribute when executing the query. Thankfully, we may accomplish this using the `withCasts` method:

    $users = User::query()->select([
        'users.*',
        'last_posted_at' => Post::query()->selectRaw('MAX(created_at)')
                ->whereColumn('user_id', 'users.id')
    ])->withCasts([
        'last_posted_at' => 'string'
    ])->get();