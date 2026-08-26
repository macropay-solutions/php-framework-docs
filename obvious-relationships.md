---
title: Obvious Relationships
description: Guide to defining, querying, and managing database relationships in PHP Framework.
context: obvious-relationships
---

# Obvious: Relationships

- [Introduction](#introduction)
- [Defining Relationships](#defining-relationships)
  - [One to One](#one-to-one)
  - [One to Many](#one-to-many)
  - [One to Many (Inverse) / Belongs To](#one-to-many-inverse)
  - [Has One of Many](#has-one-of-many)
  - [Has One Through](#has-one-through)
  - [Has Many Through](#has-many-through)
- [Many to Many Relationships](#many-to-many)
  - [Retrieving Intermediate Table Columns](#retrieving-intermediate-table-columns)
  - [Filtering Queries via Intermediate Table Columns](#filtering-queries-via-intermediate-table-columns)
  - [Ordering Queries via Intermediate Table Columns](#ordering-queries-via-intermediate-table-columns)
  - [Defining Custom Intermediate Table Models](#defining-custom-intermediate-table-models)
- [Querying Relations](#querying-relations)
  - [Relationship Methods vs. Dynamic Properties](#relationship-methods-vs-dynamic-properties)
  - [Querying Relationship Existence](#querying-relationship-existence)
  - [Querying Relationship Absence](#querying-relationship-absence)
- [Aggregating Related Models](#aggregating-related-models)
  - [Counting Related Models](#counting-related-models)
  - [Other Aggregate Functions](#other-aggregate-functions)
- [Eager Loading](#eager-loading)
  - [Constraining Eager Loads](#constraining-eager-loads)
  - [Lazy Eager Loading](#lazy-eager-loading)
  - [Preventing Lazy Loading](#preventing-lazy-loading)
- [Inserting and Updating Related Models](#inserting-and-updating-related-models)
  - [The `save` Method](#the-save-method)
  - [The `create` Method](#the-create-method)
  - [Belongs To Relationships](#updating-belongs-to-relationships)
  - [Many to Many Relationships](#updating-many-to-many-relationships)
- [Touching Parent Timestamps](#touching-parent-timestamps)

<a name="introduction"></a>
## Introduction

Database tables are often related to one another. For example, a blog post may have many comments or an order could be related to the user who placed it. Obvious makes managing and working with these relationships easy, and supports a variety of common relationships:

<div class="content-list" markdown="1">

- [One To One](#one-to-one)
- [One To Many](#one-to-many)
- [Many To Many](#many-to-many)
- [Has One Through](#has-one-through)
- [Has Many Through](#has-many-through)

</div>

<a name="defining-relationships"></a>
## Defining Relationships

Obvious relationships are defined as methods on your Obvious model classes **OR as closures**. Since relationships also serve as powerful [query builders](/queries), defining relationships provides powerful method chaining and querying capabilities. For example, we may chain additional query constraints on this `posts` relationship:

    $user->r->posts()->where('active', 1)->get();

Relationships can be defined using the `segregatedRelationsDefinitionMap()` method on your model. This decouples relation definitions from model methods and avoids clashing with column names, properties, or native methods:

    protected function segregatedRelationsDefinitionMap(): array
    {
        return [
            'relName' => fn(): HasOne => $this->hasOne(Model::class, 'model_id', 'id'),
            // Reuse the segregated relation inside another segregated relation:
            'relNameScoped' => fn(): HasOne => $this->relName()->where('col', '=', 'text'),
            'relNameScoped2' => fn(): HasOne => $this->callSegregatedRelation('relName')->where('col', '=', 'text'),
            // Reuse the method relation:
            'relNameAsMethod' => fn(...$args): mixed => $this->relNameAsMethod(...$args),
            // AVOID THESE:
            'relNameAsMethodBad1' => $this->relNameAsMethod(...), // CRASHES: Hard-binds closure scope to the first booted model
            'relNameAsMethodBad2' => [$this, 'relNameAsMethod'], // is not a Closure
            'relNameAsMethodBad3' => fn(): HasOne => [$this, 'relNameAsMethod'](),
            // DO NOT USE IT LIKE THIS!:
            'relNameAsMethodBad4' => fn(): HasOne => $this->relNameAsMethod(...)(), // executes the relation inside the map.
        ];
    }

Inside the closures, you may invoke relation helpers directly on `$this` (e.g., `$this->hasMany()`, `$this->belongsTo()`).

If you have a method that has the same name with a column from DB or with a method from the Model, you can work with it like this:

    $model->a->attributeAndRelationName; // calls Model::getAttributeValue('attributeAndRelationName')
    $model->r->attributeAndRelationName; // calls Model::getRelationValue('attributeAndRelationName')
    $model->r->attributeAndRelationName() // calls Model::callSegregatedRelation('attributeAndRelationName')
        ->where('active', 1)->get(); // or getResults()
    $model->callSegregatedRelation('attributeAndRelationName')->where('active', 1)->get(); // or getResults()

`a` stands for attributes and `r` for relations.

> [!NOTE]
> DO NOT store `a` or `r` objects in variables because they contain only `\WeakReference` of the model.

The `Model::isRelation` and `Model::callSegregatedRelation` methods route relationship calls strictly through the segregated relations map.

External libs like php-rest-wizard will still rely on the methods like behaviour so it is a good idea to keep the relation names different from the methods of the Model because, even if the relation is not defined as a method, it will behave like it through the Model::__call magic method.

> [!NOTE]
> Defining the relations as methods will still work, and they will be auto-promoted into this segregated logic on "touch" (access as property without the `()` or isRelation call but, if the method is called, it will not be auto-promoted).
> The promotion happens ONLY ONCE per Model class, and is bound to each Model instance when used.

If all the method relations are used in a request cycle before calling:

    $model->segregatedRelationList(); // no reflection involved

it will return a list with all.

To promote beforehand all method relations to this new segregation logic, you can call:

    $model->segregatedRelationList(discoverMethods: true); // reflection involved

> [!NOTE] This will discover ONLY methods that are not manually added in `segregatedRelationsDefinitionMap` and that DEFINE a return type instance of Relation!

If you want to use reflection on the method itself, the \ReflectionFunction can be retrieved via:

    $model->getSegregatedRelationReflectionFunction('methodName') // returns null|\ReflectionFunction

This will also auto-promote the method to this segregated logic. PHP attributes can be used on the method/callback and read this way.

**The reflection usage is OPT IN!** If the developer does not call the above methods, no reflection is involved.

But, before diving too deep into using relationships, let's learn how to define each type of relationship supported by Obvious.

<a name="one-to-one"></a>
### One to One

A one-to-one relationship is a very basic type of database relationship. For example, a `User` model might be associated with one `Phone` model. To define this relationship, we will place a `phone` method on the `User` model. The `phone` method should call the `hasOne` method and return its result. The `hasOne` method is available to your model via the model's `MacropaySolutions\Kernel\Database\Obvious\Model` base class:

    <?php

    namespace App\Models;

    use MacropaySolutions\Kernel\Database\Obvious\Model;
    use MacropaySolutions\Kernel\Database\Obvious\Relations\HasOne;

    class User extends Model
    {
        /**
         * Get the phone associated with the user.
         */
        public function phone(): HasOne
        {
            return $this->hasOne(Phone::class, 'user_id', 'id');
        }
    }

The first argument passed to the `hasOne` method is the name of the related model class. Once the relationship is defined, we may retrieve the related record using Obvious's dynamic properties. Dynamic properties allow you to access relationship methods as if they were properties defined on the model:

    $phone = User::query()->find(1)->r->phone;

All key parameters (`$foreignKey` and `$localKey`) are required to maintain strict execution speed and prevent runtime guessing overhead.

<a name="one-to-one-defining-the-inverse-of-the-relationship"></a>
#### Defining the Inverse of the Relationship

So, we can access the `Phone` model from our `User` model. Next, let's define a relationship on the `Phone` model that will let us access the user that owns the phone. We can define the inverse of a `hasOne` relationship using the `belongsTo` method:

    <?php

    namespace App\Models;

    use MacropaySolutions\Kernel\Database\Obvious\Model;
    use MacropaySolutions\Kernel\Database\Obvious\Relations\BelongsTo;

    class Phone extends Model
    {
        /**
         * Get the user that owns the phone.
         */
        public function user(): BelongsTo
        {
            return $this->belongsTo(User::class, 'user_id', 'id', 'user');
        }
    }

<a name="one-to-many"></a>
### One to Many

A one-to-many relationship is used to define relationships where a single model is the parent to one or more child models. For example, a blog post may have an infinite number of comments. Like all other Obvious relationships, one-to-many relationships are defined by defining a method on your Obvious model:

    <?php

    namespace App\Models;

    use MacropaySolutions\Kernel\Database\Obvious\Model;
    use MacropaySolutions\Kernel\Database\Obvious\Relations\HasMany;

    class Post extends Model
    {
        /**
         * Get the comments for the blog post.
         */
        public function comments(): HasMany
        {
            return $this->hasMany(Comment::class, 'post_id', 'id');
        }
    }

Once the relationship method has been defined, we can access the [collection](/obvious-collections) of related comments by accessing the `comments` property. Remember, since Obvious provides "dynamic relationship properties", we can access relationship methods as if they were defined as properties on the model:

    use App\Models\Post;

    $comments = Post::query()->find(1)->r->comments;

    foreach ($comments as $comment) {
        // ...
    }

Since all relationships also serve as query builders, you may add further constraints to the relationship query by calling the `comments` method and continuing to chain conditions onto the query:

    $comment = Post::query()->find(1)->r->comments()
                        ->where('title', 'foo')
                        ->first();

<a name="one-to-many-inverse"></a>
### One to Many (Inverse) / Belongs To

Now that we can access all of a post's comments, let's define a relationship to allow a comment to access its parent post. To define the inverse of a `hasMany` relationship, define a relationship method on the child model which calls the `belongsTo` method:

    <?php

    namespace App\Models;

    use MacropaySolutions\Kernel\Database\Obvious\Model;
    use MacropaySolutions\Kernel\Database\Obvious\Relations\BelongsTo;

    class Comment extends Model
    {
        /**
         * Get the post that owns the comment.
         */
        public function post(): BelongsTo
        {
            return $this->belongsTo(Post::class, 'post_id', 'id', 'post');
        }
    }

Once the relationship has been defined, we can retrieve a comment's parent post by accessing the `post` "dynamic relationship property":

    use App\Models\Comment;

    $comment = Comment::query()->find(1);

    return $comment->r->post->a->title;

Always provide the required foreign key and owner key parameters when defining a `belongsTo` relationship:

    /**
     * Get the post that owns the comment.
     */
    public function post(): BelongsTo
    {
        return $this->belongsTo(Post::class, 'post_id', 'id');
    }

<a name="default-models"></a>
#### Default Models

The `belongsTo`, `hasOne`, and `hasOneThrough` relationships allow you to define a default model that will be returned if the given relationship is `null`. This pattern is often referred to as the [Null Object pattern](https://en.wikipedia.org/wiki/Null_Object_pattern) and can help remove conditional checks in your code. In the following example, the `user` relation will return an empty `App\Models\User` model if no user is attached to the `Post` model:

    /**
     * Get the author of the post.
     */
    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class, 'user_id', 'id')->withDefault();
    }

To populate the default model with attributes, you may pass an array or closure to the `withDefault` method:

    /**
     * Get the author of the post.
     */
    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class, 'user_id', 'id')->withDefault([
            'name' => 'Guest Author',
        ]);
    }

    /**
     * Get the author of the post.
     */
    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class, 'user_id', 'id')->withDefault(function (User $user, Post $post) {
            $user->name = 'Guest Author';
        });
    }

<a name="querying-belongs-to-relationships"></a>
#### Querying Belongs To Relationships

When querying for the children of a "belongs to" relationship, you may manually build the `where` clause to retrieve the corresponding Obvious models:

    use App\Models\Post;

    $posts = Post::query()->where('user_id', $user->a->id)->get();

However, you may find it more convenient to use the `whereBelongsTo` method, which will automatically determine the proper relationship and foreign key for the given model:

    $posts = Post::query()->whereBelongsTo($user)->get();

You may also provide a [collection](/obvious-collections) instance to the `whereBelongsTo` method. When doing so, Framework will retrieve models that belong to any of the parent models within the collection:

    $users = User::query()->where('vip', true)->get();

    $posts = Post::query()->whereBelongsTo($users)->get();

By default, Framework will determine the relationship associated with the given model based on the class name of the model; however, you may specify the relationship name manually by providing it as the second argument to the `whereBelongsTo` method:

    $posts = Post::query()->whereBelongsTo($user, 'author')->get();

<a name="has-one-of-many"></a>
### Has One of Many

Sometimes a model may have many related models, yet you want to easily retrieve the "latest" or "oldest" related model of the relationship. For example, a `User` model may be related to many `Order` models, but you want to define a convenient way to interact with the most recent order the user has placed. You may accomplish this using the `hasOne` relationship type combined with the `ofMany` methods:

```php
/**
 * Get the user's most recent order.
 */
public function latestOrder(): HasOne
{
    return $this->hasOne(Order::class, 'user_id', 'id')->latestOfMany();
}
```

Likewise, you may define a method to retrieve the "oldest", or first, related model of a relationship:

```php
/**
 * Get the user's oldest order.
 */
public function oldestOrder(): HasOne
{
    return $this->hasOne(Order::class, 'user_id', 'id')->oldestOfMany();
}
```

By default, the `latestOfMany` and `oldestOfMany` methods will retrieve the latest or oldest related model based on the model's primary key, which must be sortable. However, sometimes you may wish to retrieve a single model from a larger relationship using a different sorting criteria.

For example, using the `ofMany` method, you may retrieve the user's most expensive order. The `ofMany` method accepts the sortable column as its first argument and which aggregate function (`min` or `max`) to apply when querying for the related model:

```php
/**
 * Get the user's largest order.
 */
public function largestOrder(): HasOne
{
    return $this->hasOne(Order::class, 'user_id', 'id')->ofMany('price', 'max');
}
```

> [!WARNING]  
> Because PostgreSQL does not support executing the `MAX` function against UUID columns, it is not currently possible to use one-of-many relationships in combination with PostgreSQL UUID columns.

<a name="converting-many-relationships-to-has-one-relationships"></a>
#### Converting "Many" Relationships to Has One Relationships

Often, when retrieving a single model using the `latestOfMany`, `oldestOfMany`, or `ofMany` methods, you already have a "has many" relationship defined for the same model. For convenience, Framework allows you to easily convert this relationship into a "has one" relationship by invoking the `one` method on the relationship:

```php
/**
 * Get the user's orders.
 */
public function orders(): HasMany
{
    return $this->hasMany(Order::class, 'user_id', 'id');
}

/**
 * Get the user's largest order.
 */
public function largestOrder(): HasOne
{
    return $this->orders()->one()->ofMany('price', 'max');
}
```

<a name="advanced-has-one-of-many-relationships"></a>
#### Advanced Has One of Many Relationships

It is possible to construct more advanced "has one of many" relationships. For example, a `Product` model may have many associated `Price` models that are retained in the system even after new pricing is published. In addition, new pricing data for the product may be able to be published in advance to take effect at a future date via a `published_at` column.

So, in summary, we need to retrieve the latest published pricing where the published date is not in the future. In addition, if two prices have the same published date, we will prefer the price with the greatest ID. To accomplish this, we must pass an array to the `ofMany` method that contains the sortable columns which determine the latest price. In addition, a closure will be provided as the second argument to the `ofMany` method. This closure will be responsible for adding additional publish date constraints to the relationship query:

```php
/**
 * Get the current pricing for the product.
 */
public function currentPricing(): HasOne
{
    return $this->hasOne(Price::class, 'product_id', 'id')->ofMany([
        'published_at' => 'max',
        'id' => 'max',
    ], function (Builder $query) {
        $query->where('published_at', '<', now());
    });
}
```

<a name="has-one-through"></a>
### Has One Through

The "has-one-through" relationship defines a one-to-one relationship with another model. However, this relationship indicates that the declaring model can be matched with one instance of another model by proceeding _through_ a third model.

For example, in a vehicle repair shop application, each `Mechanic` model may be associated with one `Car` model, and each `Car` model may be associated with one `Owner` model. While the mechanic and the owner have no direct relationship within the database, the mechanic can access the owner _through_ the `Car` model. Let's look at the tables necessary to define this relationship:

    mechanics
        id - integer
        name - string

    cars
        id - integer
        model - string
        mechanic_id - integer

    owners
        id - integer
        name - string
        car_id - integer

Now that we have examined the table structure for the relationship, let's define the relationship on the `Mechanic` model:

    <?php

    namespace App\Models;

    use MacropaySolutions\Kernel\Database\Obvious\Model;
    use MacropaySolutions\Kernel\Database\Obvious\Relations\HasOneThrough;

    class Mechanic extends Model
    {
        /**
         * Get the car's owner.
         */
        public function carOwner(): HasOneThrough
        {
            return $this->hasOneThrough(Owner::class, Car::class, 'mechanic_id', 'car_id', 'id', 'id');
        }
    }

<a name="has-many-through"></a>
### Has Many Through

The "has-many-through" relationship provides a convenient way to access distant relations via an intermediate relation. For example, let's assume we are building a deployment platform. A `Project` model might access many `Deployment` models through an intermediate `Environment` model. Using this example, you could easily gather all deployments for a given project. Let's look at the tables required to define this relationship:

    projects
        id - integer
        name - string

    environments
        id - integer
        project_id - integer
        name - string

    deployments
        id - integer
        environment_id - integer
        commit_hash - string

Now that we have examined the table structure for the relationship, let's define the relationship on the `Project` model:

    <?php

    namespace App\Models;

    use MacropaySolutions\Kernel\Database\Obvious\Model;
    use MacropaySolutions\Kernel\Database\Obvious\Relations\HasManyThrough;

    class Project extends Model
    {
        public function deployments(): HasManyThrough
        {
            return $this->hasManyThrough(Deployment::class, Environment::class, 'project_id', 'environment_id', 'id', 'id');
        }
    }

<a name="many-to-many"></a>
## Many to Many Relationships

Many-to-many relations are slightly more complicated than `hasOne` and `hasMany` relationships. An example of a many-to-many relationship is a user that has many roles and those roles are also shared by other users in the application. For example, a user may be assigned the role of "Author" and "Editor"; however, those roles may also be assigned to other users as well. So, a user has many roles and a role has many users.

<a name="many-to-many-table-structure"></a>
#### Table Structure

To define this relationship, three database tables are needed: `users`, `roles`, and `role_user`. The `role_user` table is derived from the alphabetical order of the related model names and contains `user_id` and `role_id` columns. This table is used as an intermediate table linking the users and roles.

Remember, since a role can belong to many users, we cannot simply place a `user_id` column on the `roles` table. This would mean that a role could only belong to a single user. In order to provide support for roles being assigned to multiple users, the `role_user` table is needed. We can summarize the relationship's table structure like so:

    users
        id - integer
        name - string

    roles
        id - integer
        name - string

    role_user
        user_id - integer
        role_id - integer

<a name="many-to-many-model-structure"></a>
#### Model Structure

Many-to-many relationships are defined by writing a method that returns the result of the `belongsToMany` method. The `belongsToMany` method is provided by the `MacropaySolutions\Kernel\Database\Obvious\Model` base class that is used by all of your application's Obvious models. For example, let's define a `roles` method on our `User` model. The first argument passed to this method is the name of the related model class:

    <?php

    namespace App\Models;

    use MacropaySolutions\Kernel\Database\Obvious\Model;
    use MacropaySolutions\Kernel\Database\Obvious\Relations\BelongsToMany;

    class User extends Model
    {
        /**
         * The roles that belong to the user.
         */
        public function roles(): BelongsToMany
        {
             return $this->belongsToMany(Role::class, 'role_user', 'user_id', 'role_id', 'id', 'id');
        }
    }

Once the relationship is defined, you may access the user's roles using the `roles` dynamic relationship property:

    use App\Models\User;

    $user = User::query()->find(1);

    foreach ($user->r->roles as $role) {
        // ...
    }

Since all relationships also serve as query builders, you may add further constraints to the relationship query by calling the `roles` method and continuing to chain conditions onto the query:

    $roles = User::query()->find(1)->r->roles()->orderBy('name')->get();

<a name="many-to-many-defining-the-inverse-of-the-relationship"></a>
#### Defining the Inverse of the Relationship

To define the "inverse" of a many-to-many relationship, you should define a method on the related model which also returns the result of the `belongsToMany` method. To complete our user / role example, let's define the `users` method on the `Role` model:

    <?php

    namespace App\Models;

    use MacropaySolutions\Kernel\Database\Obvious\Model;
    use MacropaySolutions\Kernel\Database\Obvious\Relations\BelongsToMany;

    class Role extends Model
    {
        /**
         * The users that belong to the role.
         */
        public function users(): BelongsToMany
        {
            return $this->belongsToMany(User::class, 'role_user', 'role_id', 'user_id', 'id', 'id');
        }
    }

As you can see, the relationship is defined exactly the same as its `User` model counterpart with the exception of referencing the `App\Models\User` model. Since we're reusing the `belongsToMany` method, all the usual table and key customization options are available when defining the "inverse" of many-to-many relationships.

<a name="retrieving-intermediate-table-columns"></a>
### Retrieving Intermediate Table Columns

As you have already learned, working with many-to-many relations requires the presence of an intermediate table. Obvious provides some very helpful ways of interacting with this table. For example, let's assume our `User` model has many `Role` models that it is related to. After accessing this relationship, we may access the intermediate table using the `pivot` attribute on the models:

    use App\Models\User;

    $user = User::query()->find(1);

    foreach ($user->r->roles as $role) {
        echo $role->r->pivot->a->created_at;
    }

Notice that each `Role` model we retrieve is automatically assigned a `pivot` attribute. This attribute contains a model representing the intermediate table.

By default, only the model keys will be present on the `pivot` model. If your intermediate table contains extra attributes, you must specify them when defining the relationship:

    return $this->belongsToMany(Role::class, 'role_user', 'user_id', 'role_id', 'id', 'id')->withPivot('active', 'created_by');

If you would like your intermediate table to have `created_at` and `updated_at` timestamps that are automatically maintained by Obvious, call the `withTimestamps` method when defining the relationship:

    return $this->belongsToMany(Role::class, 'role_user', 'user_id', 'role_id', 'id', 'id')->withTimestamps();

> [!WARNING]  
> Intermediate tables that utilize Obvious's automatically maintained timestamps are required to have both `created_at` and `updated_at` timestamp columns.

<a name="customizing-the-pivot-attribute-name"></a>
#### Customizing the `pivot` Attribute Name

As noted previously, attributes from the intermediate table may be accessed on models via the `pivot` attribute. However, you are free to customize the name of this attribute to better reflect its purpose within your application.

For example, if your application contains users that may subscribe to podcasts, you likely have a many-to-many relationship between users and podcasts. If this is the case, you may wish to rename your intermediate table attribute to `subscription` instead of `pivot`. This can be done using the `as` method when defining the relationship:

    return $this->belongsToMany(Podcast::class, 'podcast_user', 'user_id', 'podcast_id', 'id', 'id')
                    ->as('subscription')
                    ->withTimestamps();

Once the custom intermediate table attribute has been specified, you may access the intermediate table data using the customized name:

    $users = User::query()->with('podcasts')->get();

    foreach ($users->flatMap(fn($user) => $user->r->podcasts) as $podcast) {
        echo $podcast->r->subscription->a->created_at;
    }

<a name="filtering-queries-via-intermediate-table-columns"></a>
### Filtering Queries via Intermediate Table Columns

You can also filter the results returned by `belongsToMany` relationship queries using the `wherePivot`, `wherePivotIn`, `wherePivotNotIn`, `wherePivotBetween`, `wherePivotNotBetween`, `wherePivotNull`, and `wherePivotNotNull` methods when defining the relationship:

    return $this->belongsToMany(Role::class, 'role_user', 'user_id', 'role_id', 'id', 'id')
                    ->wherePivot('approved', 1);

    return $this->belongsToMany(Role::class, 'role_user', 'user_id', 'role_id', 'id', 'id')
                    ->wherePivotIn('priority', [1, 2]);

    return $this->belongsToMany(Role::class, 'role_user', 'user_id', 'role_id', 'id', 'id')
                    ->wherePivotNotIn('priority', [1, 2]);

    return $this->belongsToMany(Podcast::class, 'podcast_user', 'user_id', 'podcast_id', 'id', 'id')
                    ->as('subscriptions')
                    ->wherePivotBetween('created_at', ['2020-01-01 00:00:00', '2020-12-31 00:00:00']);

    return $this->belongsToMany(Podcast::class, 'podcast_user', 'user_id', 'podcast_id', 'id', 'id')
                    ->as('subscriptions')
                    ->wherePivotNotBetween('created_at', ['2020-01-01 00:00:00', '2020-12-31 00:00:00']);

    return $this->belongsToMany(Podcast::class, 'podcast_user', 'user_id', 'podcast_id', 'id', 'id')
                    ->as('subscriptions')
                    ->wherePivotNull('expired_at');

    return $this->belongsToMany(Podcast::class, 'podcast_user', 'user_id', 'podcast_id', 'id', 'id')
                    ->as('subscriptions')
                    ->wherePivotNotNull('expired_at');

<a name="ordering-queries-via-intermediate-table-columns"></a>
### Ordering Queries via Intermediate Table Columns

You can order the results returned by `belongsToMany` relationship queries using the `orderByPivot` method. In the following example, we will retrieve all the latest badges for the user:

    return $this->belongsToMany(Badge::class, 'badge_user', 'user_id', 'badge_id', 'id', 'id')
                    ->where('rank', 'gold')
                    ->orderByPivot('created_at', 'desc');

<a name="defining-custom-intermediate-table-models"></a>
### Defining Custom Intermediate Table Models

If you would like to define a custom model to represent the intermediate table of your many-to-many relationship, you may call the `using` method when defining the relationship. Custom pivot models give you the opportunity to define additional behavior on the pivot model, such as methods and casts.

Custom many-to-many pivot models should extend the `MacropaySolutions\Kernel\Database\Obvious\Relations\Pivot` class. For example, we may define a `Role` model which uses a custom `RoleUser` pivot model:

    <?php

    namespace App\Models;

    use MacropaySolutions\Kernel\Database\Obvious\Model;
    use MacropaySolutions\Kernel\Database\Obvious\Relations\BelongsToMany;

    class Role extends Model
    {
        /**
         * The users that belong to the role.
         */
        public function users(): BelongsToMany
        {
            return $this->belongsToMany(User::class, RoleUser::class, 'role_id', 'user_id', 'id', 'id');
        }
    }

When defining the `RoleUser` model, you should extend the `MacropaySolutions\Kernel\Database\Obvious\Relations\Pivot` class:

    <?php

    namespace App\Models;

    use MacropaySolutions\Kernel\Database\Obvious\Relations\Pivot;

    class RoleUser extends Pivot
    {
        // ...
    }

> [!WARNING]  
> Pivot models may not use the `SoftDeletes` trait. If you need to soft delete pivot records consider converting your pivot model to an actual Obvious model.

<a name="custom-pivot-models-and-incrementing-ids"></a>
#### Custom Pivot Models and Incrementing IDs

If you have defined a many-to-many relationship that uses a custom pivot model, and that pivot model has an auto-incrementing primary key, you should ensure your custom pivot model class defines an `incrementing` property that is set to `true`.

    /**
     * Indicates if the IDs are auto-incrementing.
     *
     * @var bool
     */
    public $incrementing = true;

<a name="querying-relations"></a>
## Querying Relations

Since all Obvious relationships are defined via methods, you may call those methods to obtain an instance of the relationship without actually executing a query to load the related models. In addition, all types of Obvious relationships also serve as [query builders](/queries), allowing you to continue to chain constraints onto the relationship query before finally executing the SQL query against your database.

For example, imagine a blog application in which a `User` model has many associated `Post` models:

    <?php

    namespace App\Models;

    use MacropaySolutions\Kernel\Database\Obvious\Model;
    use MacropaySolutions\Kernel\Database\Obvious\Relations\HasMany;

    class User extends Model
    {
        /**
         * Get all the posts for the user.
         */
        public function posts(): HasMany
        {
            return $this->hasMany(Post::class);
        }
    }

You may query the `posts` relationship and add additional constraints to the relationship like so:

    use App\Models\User;

    $user = User::query()->find(1);

    $user->r->posts()->where('active', 1)->get();

You are able to use any of the Framework [query builder's](/queries) methods on the relationship, so be sure to explore the query builder documentation to learn about all the methods that are available to you.

<a name="chaining-orwhere-clauses-after-relationships"></a>
#### Chaining `orWhere` Clauses After Relationships

As demonstrated in the example above, you are free to add additional constraints to relationships when querying them. However, use caution when chaining `orWhere` clauses onto a relationship, as the `orWhere` clauses will be logically grouped at the same level as the relationship constraint:

    $user->r->posts()
            ->where('active', 1)
            ->orWhere('votes', '>=', 100)
            ->get();

The example above will generate the following SQL. As you can see, the `or` clause instructs the query to return _any_ post with greater than 100 votes. The query is no longer constrained to a specific user:

```sql
select *
from posts
where user_id = ? and active = 1 or votes >= 100
```

In most situations, you should use [logical groups](/queries#logical-grouping) to group the conditional checks between parentheses:

    use MacropaySolutions\Kernel\Database\Obvious\Builder;

    $user->r->posts()
            ->where(function (Builder $query) {
                return $query->where('active', 1)
                             ->orWhere('votes', '>=', 100);
            })
            ->get();

The example above will produce the following SQL. Note that the logical grouping has properly grouped the constraints and the query remains constrained to a specific user:

```sql
select *
from posts
where user_id = ? and (active = 1 or votes >= 100)
```

<a name="relationship-methods-vs-dynamic-properties"></a>
### Relationship Methods vs. Dynamic Properties

If you do not need to add additional constraints to an Obvious relationship query, you may access the relationship as if it were a property. For example, continuing to use our `User` and `Post` example models, we may access all of a user's posts like so:

    use App\Models\User;

    $user = User::query()->find(1);

    foreach ($user->r->posts as $post) {
        // ...
    }

Dynamic relationship properties perform "lazy loading", meaning they will only load their relationship data when you actually access them. Because of this, developers often use [eager loading](#eager-loading) to pre-load relationships they know will be accessed after loading the model. Eager loading provides a significant reduction in SQL queries that must be executed to load a model's relations.

<a name="querying-relationship-existence"></a>
### Querying Relationship Existence

When retrieving model records, you may wish to limit your results based on the existence of a relationship. For example, imagine you want to retrieve all blog posts that have at least one comment. To do so, you may pass the name of the relationship to the `has` and `orHas` methods:

    use App\Models\Post;

    // Retrieve all posts that have at least one comment...
    $posts = Post::query()->has('comments')->get();

You may also specify an operator and count value to further customize the query:

    // Retrieve all posts that have three or more comments...
    $posts = Post::query()->has('comments', '>=', 3)->get();

Nested `has` statements may be constructed using "dot" notation. For example, you may retrieve all posts that have at least one comment that has at least one image:

    // Retrieve posts that have at least one comment with images...
    $posts = Post::query()->has('comments.images')->get();

If you need even more power, you may use the `whereHas` and `orWhereHas` methods to define additional query constraints on your `has` queries, such as inspecting the content of a comment:

    use MacropaySolutions\Kernel\Database\Obvious\Builder;

    // Retrieve posts with at least one comment containing words like code%...
    $posts = Post::query()->whereHas('comments', function (Builder $query) {
        $query->where('content', 'like', 'code%');
    })->get();

    // Retrieve posts with at least ten comments containing words like code%...
    $posts = Post::query()->whereHas('comments', function (Builder $query) {
        $query->where('content', 'like', 'code%');
    }, '>=', 10)->get();

> [!WARNING]  
> Obvious does not currently support querying for relationship existence across databases. The relationships must exist within the same database.

<a name="inline-relationship-existence-queries"></a>
#### Inline Relationship Existence Queries

If you would like to query for a relationship's existence with a single, simple where condition attached to the relationship query, you may find it more convenient to use the `whereRelation` and `orWhereRelation` methods. For example, we may query for all posts that have unapproved comments:

    use App\Models\Post;

    $posts = Post::query()->whereRelation('comments', 'is_approved', false)->get();

Of course, like calls to the query builder's `where` method, you may also specify an operator:

    $posts = Post::query()->whereRelation(
        'comments', 'created_at', '>=', now()->subHour()
    )->get();

<a name="querying-relationship-absence"></a>
### Querying Relationship Absence

When retrieving model records, you may wish to limit your results based on the absence of a relationship. For example, imagine you want to retrieve all blog posts that **don't** have any comments. To do so, you may pass the name of the relationship to the `doesntHave` and `orDoesntHave` methods:

    use App\Models\Post;

    $posts = Post::query()->doesntHave('comments')->get();

If you need even more power, you may use the `whereDoesntHave` and `orWhereDoesntHave` methods to add additional query constraints to your `doesntHave` queries, such as inspecting the content of a comment:

    use MacropaySolutions\Kernel\Database\Obvious\Builder;

    $posts = Post::query()->whereDoesntHave('comments', function (Builder $query) {
        $query->where('content', 'like', 'code%');
    })->get();

You may use "dot" notation to execute a query against a nested relationship. For example, the following query will retrieve all posts that do not have comments; however, posts that have comments from authors that are not banned will be included in the results:

    use MacropaySolutions\Kernel\Database\Obvious\Builder;

    $posts = Post::query()->whereDoesntHave('comments.author', function (Builder $query) {
        $query->where('banned', 0);
    })->get();

<a name="aggregating-related-models"></a>
## Aggregating Related Models

<a name="counting-related-models"></a>
### Counting Related Models

Sometimes you may want to count the number of related models for a given relationship without actually loading the models. To accomplish this, you may use the `withCount` method. The `withCount` method will place a `{relation}_count` attribute on the resulting models:

    use App\Models\Post;

    $posts = Post::query()->withCount('comments')->get();

    foreach ($posts as $post) {
        echo $post->a->comments_count;
    }

By passing an array to the `withCount` method, you may add the "counts" for multiple relations as well as add additional constraints to the queries:

    use MacropaySolutions\Kernel\Database\Obvious\Builder;

    $posts = Post::query()->withCount(['votes', 'comments' => function (Builder $query) {
        $query->where('content', 'like', 'code%');
    }])->get();

    echo $posts[0]->a->votes_count;
    echo $posts[0]->a->comments_count;

You may also alias the relationship count result, allowing multiple counts on the same relationship:

    use MacropaySolutions\Kernel\Database\Obvious\Builder;

    $posts = Post::query()->withCount([
        'comments',
        'comments as pending_comments_count' => function (Builder $query) {
            $query->where('approved', false);
        },
    ])->get();

    echo $posts[0]->comments_count;
    echo $posts[0]->pending_comments_count;

<a name="deferred-count-loading"></a>
#### Deferred Count Loading

Using the `loadCount` method, you may load a relationship count after the parent model has already been retrieved:

    $book = Book::query()->first();

    $book->loadCount('genres');

If you need to set additional query constraints on the count query, you may pass an array keyed by the relationships you wish to count. The array values should be closures which receive the query builder instance:

    $book->loadCount(['reviews' => function (Builder $query) {
        $query->where('rating', 5);
    }])

<a name="relationship-counting-and-custom-select-statements"></a>
#### Relationship Counting and Custom Select Statements

If you're combining `withCount` with a `select` statement, ensure that you call `withCount` after the `select` method:

    $posts = Post::query()->select(['title', 'body'])
                    ->withCount('comments')
                    ->get();

<a name="other-aggregate-functions"></a>
### Other Aggregate Functions

In addition to the `withCount` method, Obvious provides `withMin`, `withMax`, `withAvg`, `withSum`, and `withExists` methods. These methods will place a `{relation}_{function}_{column}` attribute on your resulting models:

    use App\Models\Post;

    $posts = Post::query()->withSum('comments', 'votes')->get();

    foreach ($posts as $post) {
        echo $post->a->comments_sum_votes;
    }

If you wish to access the result of the aggregate function using another name, you may specify your own alias:

    $posts = Post::query()->withSum('comments as total_comments', 'votes')->get();

    foreach ($posts as $post) {
        echo $post->a->total_comments;
    }

> [!WARNING]
> **Return Type Change:** The `withExists` method no longer automatically casts the resulting attribute to a strict PHP boolean to maximize query performance. It returns the raw value from your database driver (e.g., `1` or `0`). Standard truthy checks will continue to work, but strict type comparisons (`=== true`) may fail with some DBs.

Like the `loadCount` method, deferred versions of these methods are also available. These additional aggregate operations may be performed on Obvious models that have already been retrieved:

    $post = Post::query()->first();

    $post->loadSum('comments', 'votes');

If you're combining these aggregate methods with a `select` statement, ensure that you call the aggregate methods after the `select` method:

    $posts = Post::query()->select(['title', 'body'])
                    ->withExists('comments')
                    ->get();

<a name="eager-loading"></a>
## Eager Loading

When accessing Obvious relationships as properties, the related models are "lazy loaded". This means the relationship data is not actually loaded until you first access the property. However, Obvious can "eager load" relationships at the time you query the parent model. Eager loading alleviates the "N + 1" query problem. To illustrate the N + 1 query problem, consider a `Book` model that "belongs to" to an `Author` model:

    <?php

    namespace App\Models;

    use MacropaySolutions\Kernel\Database\Obvious\Model;
    use MacropaySolutions\Kernel\Database\Obvious\Relations\BelongsTo;

    class Book extends Model
    {
        /**
         * Get the author that wrote the book.
         */
        public function author(): BelongsTo
        {
            return $this->belongsTo(Author::class);
        }
    }

Now, let's retrieve all books and their authors:

    use App\Models\Book;

    $books = Book::query()->all();

    foreach ($books as $book) {
        echo $book->r->author->a->name;
    }

This loop will execute one query to retrieve all the books within the database table, then another query for each book in order to retrieve the book's author. So, if we have 25 books, the code above would run 26 queries: one for the original book, and 25 additional queries to retrieve the author of each book.

Thankfully, we can use eager loading to reduce this operation to just two queries. When building a query, you may specify which relationships should be eager loaded using the `with` method:

    $books = Book::query()->with('author')->get();

    foreach ($books as $book) {
        echo $book->r->author->a->name;
    }

For this operation, only two queries will be executed - one query to retrieve all the books and one query to retrieve all the authors for all the books:

```sql
select * from books

select * from authors where id in (1, 2, 3, 4, 5, ...)
```

<a name="eager-loading-multiple-relationships"></a>
#### Eager Loading Multiple Relationships

Sometimes you may need to eager load several different relationships. To do so, just pass an array of relationships to the `with` method:

    $books = Book::query()->with(['author', 'publisher'])->get();

<a name="nested-eager-loading"></a>
#### Nested Eager Loading

To eager load a relationship's relationships, you may use "dot" syntax. For example, let's eager load all the book's authors and all the author's personal contacts:

    $books = Book::query()->with('author.contacts')->get();

Alternatively, you may specify nested eager loaded relationships by providing a nested array to the `with` method, which can be convenient when eager loading multiple nested relationships:

    $books = Book::query()->with([
        'author' => [
            'contacts',
            'publisher',
        ],
    ])->get();

<a name="eager-loading-specific-columns"></a>
#### Eager Loading Specific Columns

You may not always need every column from the relationships you are retrieving. For this reason, Obvious allows you to specify which columns of the relationship you would like to retrieve:

    $books = Book::query()->with('author:id,name,book_id')->get();

> [!WARNING]  
> When using this feature, you should always include the `id` column and any relevant foreign key columns in the list of columns you wish to retrieve.

<a name="eager-loading-by-default"></a>
#### Eager Loading by Default

Sometimes you might want to always load some relationships when retrieving a model. To accomplish this, you may define a `$with` property on the model:

    <?php

    namespace App\Models;

    use MacropaySolutions\Kernel\Database\Obvious\Model;
    use MacropaySolutions\Kernel\Database\Obvious\Relations\BelongsTo;

    class Book extends Model
    {
        /**
         * The relationships that should always be loaded.
         *
         * @var array
         */
        protected $with = ['author'];

        /**
         * Get the author that wrote the book.
         */
        public function author(): BelongsTo
        {
            return $this->belongsTo(Author::class);
        }

        /**
         * Get the genre of the book.
         */
        public function genre(): BelongsTo
        {
            return $this->belongsTo(Genre::class);
        }
    }

If you would like to remove an item from the `$with` property for a single query, you may use the `without` method:

    $books = Book::query()->without('author')->get();

If you would like to override all items within the `$with` property for a single query, you may use the `withOnly` method:

    $books = Book::query()->withOnly('genre')->get();

<a name="constraining-eager-loads"></a>
### Constraining Eager Loads

Sometimes you may wish to eager load a relationship but also specify additional query conditions for the eager loading query. You can accomplish this by passing an array of relationships to the `with` method where the array key is a relationship name and the array value is a closure that adds additional constraints to the eager loading query:

    use App\Models\User;
    use MacropaySolutions\Kernel\Contracts\Database\Obvious\Builder;

    $users = User::query()->with(['posts' => function (Builder $query) {
        $query->where('title', 'like', '%code%');
    }])->get();

In this example, Obvious will only eager load posts where the post's `title` column contains the word `code`. You may call other [query builder](/queries) methods to further customize the eager loading operation:

    $users = User::query()->with(['posts' => function (Builder $query) {
        $query->orderBy('created_at', 'desc');
    }])->get();

> [!WARNING]  
> The `limit` and `take` query builder methods may not be used when constraining eager loads.

<a name="constraining-eager-loads-with-relationship-existence"></a>
#### Constraining Eager Loads With Relationship Existence

You may sometimes find yourself needing to check for the existence of a relationship while simultaneously loading the relationship based on the same conditions. For example, you may wish to only retrieve `User` models that have child `Post` models matching a given query condition while also eager loading the matching posts. You may accomplish this using the `withWhereHas` method:

    use App\Models\User;

    $users = User::query()->withWhereHas('posts', function ($query) {
        $query->where('featured', true);
    })->get();

<a name="lazy-eager-loading"></a>
### Lazy Eager Loading

Sometimes you may need to eager load a relationship after the parent model has already been retrieved. For example, this may be useful if you need to dynamically decide whether to load related models:

    use App\Models\Book;

    $books = Book::query()->all();

    if ($someCondition) {
        $books->load('author', 'publisher');
    }

If you need to set additional query constraints on the eager loading query, you may pass an array keyed by the relationships you wish to load. The array values should be closure instances which receive the query instance:

    $author->load(['books' => function (Builder $query) {
        $query->orderBy('published_date', 'asc');
    }]);

To load a relationship only when it has not already been loaded, use the `loadMissing` method:

    $book->loadMissing('author');

<a name="preventing-lazy-loading"></a>
### Preventing Lazy Loading

As previously discussed, eager loading relationships can often provide significant performance benefits to your application. Therefore, if you would like, you may instruct Framework to always prevent the lazy loading of relationships. To accomplish this, you may invoke the `preventLazyLoading` method offered by the base Obvious model class. Typically, you should call this method within the `boot` method of your application's `AppServiceProvider` class.

The `preventLazyLoading` method accepts an optional boolean argument that indicates if lazy loading should be prevented. For example, you may wish to only disable lazy loading in non-production environments so that your production environment will continue to function normally even if a lazy loaded relationship is accidentally present in production code:

```php
use MacropaySolutions\Kernel\Database\Obvious\Model;

/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Model::preventLazyLoading(! $this->app->isProduction());
}
```

After preventing lazy loading, Obvious will throw a `MacropaySolutions\Kernel\Database\LazyLoadingViolationException` exception when your application attempts to lazy load any Obvious relationship.

You may customize the behavior of lazy loading violations using the `handleLazyLoadingViolationsUsing` method. For example, using this method, you may instruct lazy loading violations to only be logged instead of interrupting the application's execution with exceptions:

```php
Model::handleLazyLoadingViolationUsing(function (Model $model, string $relation) {
    $class = $model::class;

    info("Attempted to lazy load [{$relation}] on model [{$class}].");
});
```

<a name="inserting-and-updating-related-models"></a>
## Inserting and Updating Related Models

<a name="the-save-method"></a>
### The `save` Method

Obvious provides convenient methods for adding new models to relationships. For example, perhaps you need to add a new comment to a post. Instead of manually setting the `post_id` attribute on the `Comment` model you may insert the comment using the relationship's `save` method:

    use App\Models\Comment;
    use App\Models\Post;

    $comment = new Comment(['message' => 'A new comment.']);

    $post = Post::query()->find(1);

    $post->r->comments()->save($comment);

Note that we did not access the `comments` relationship as a dynamic property. Instead, we called the `comments` method to obtain an instance of the relationship. The `save` method will automatically add the appropriate `post_id` value to the new `Comment` model.

If you need to save multiple related models, you may use the `saveMany` method:

    $post = Post::query()->find(1);

    $post->r->comments()->saveMany([
        new Comment(['message' => 'A new comment.']),
        new Comment(['message' => 'Another new comment.']),
    ]);

The `save` and `saveMany` methods will persist the given model instances, but will not add the newly persisted models to any in-memory relationships that are already loaded onto the parent model. If you plan on accessing the relationship after using the `save` or `saveMany` methods, you may wish to use the `refresh` method to reload the model and its relationships:

    $post->r->comments()->save($comment);

    $post->refresh();

    // All comments, including the newly saved comment...
    $post->r->comments;

<a name="the-push-method"></a>
#### Recursively Saving Models and Relationships

If you would like to `save` your model and all of its associated relationships, you may use the `push` method. In this example, the `Post` model will be saved as well as its comments and the comment's authors:

    $post = Post::query()->find(1);

    $post->r->comments[0]->a->message = 'Message';
    $post->r->comments[0]->r->author->a->name = 'Author Name';

    $post->push();

The `pushQuietly` method may be used to save a model and its associated relationships without raising any events:

    $post->pushQuietly();

<a name="the-create-method"></a>
### The `create` Method

In addition to the `save` and `saveMany` methods, you may also use the `create` method, which accepts an array of attributes, creates a model, and inserts it into the database. The difference between `save` and `create` is that `save` accepts a full Obvious model instance while `create` accepts a plain PHP `array`. The newly created model will be returned by the `create` method:

    use App\Models\Post;

    $post = Post::query()->find(1);

    $comment = $post->r->comments()->create([
        'message' => 'A new comment.',
    ]);

You may use the `createMany` method to create multiple related models:

    $post = Post::query()->find(1);

    $post->r->comments()->createMany([
        ['message' => 'A new comment.'],
        ['message' => 'Another new comment.'],
    ]);

The `createQuietly` and `createManyQuietly` methods may be used to create a model(s) without dispatching any events:

    $user = User::query()->find(1);

    $user->r->posts()->createQuietly([
        'title' => 'Post title.',
    ]);
    
    $user->r->posts()->createManyQuietly([
        ['title' => 'First post.'],
        ['title' => 'Second post.'],
    ]);

You may also use the `findOrNew`, `firstOrNew`, `firstOrCreate`, and `updateOrCreate` methods to [create and update models on relationships](/obvious#upserts).

> [!NOTE]  
> Before using the `create` method, be sure to review the [mass assignment](/obvious#mass-assignment) documentation.

<a name="updating-belongs-to-relationships"></a>
### Belongs To Relationships

If you would like to assign a child model to a new parent model, you may use the `associate` method. In this example, the `User` model defines a `belongsTo` relationship to the `Account` model. This `associate` method will set the foreign key on the child model:

    use App\Models\Account;

    $account = Account::query()->find(10);

    $user->r->account()->associate($account);

    $user->save();

To remove a parent model from a child model, you may use the `dissociate` method. This method will set the relationship's foreign key to `null`:

    $user->r->account()->dissociate();

    $user->save();

<a name="updating-many-to-many-relationships"></a>
### Many to Many Relationships

<a name="attaching-detaching"></a>
#### Attaching / Detaching

Obvious also provides methods to make working with many-to-many relationships more convenient. For example, let's imagine a user can have many roles and a role can have many users. You may use the `attach` method to attach a role to a user by inserting a record in the relationship's intermediate table:

    use App\Models\User;

    $user = User::query()->find(1);

    $user->r->roles()->attach($roleId);

When attaching a relationship to a model, you may also pass an array of additional data to be inserted into the intermediate table:

    $user->r->roles()->attach($roleId, ['expires' => $expires]);

Sometimes it may be necessary to remove a role from a user. To remove a many-to-many relationship record, use the `detach` method. The `detach` method will delete the appropriate record out of the intermediate table; however, both models will remain in the database:

    // Detach a single role from the user...
    $user->r->roles()->detach($roleId);

    // Detach all roles from the user...
    $user->r->roles()->detach();

For convenience, `attach` and `detach` also accept arrays of IDs as input:

    $user = User::query()->find(1);

    $user->r->roles()->detach([1, 2, 3]);

    $user->r->roles()->attach([
        1 => ['expires' => $expires],
        2 => ['expires' => $expires],
    ]);

<a name="syncing-associations"></a>
#### Syncing Associations

You may also use the `sync` method to construct many-to-many associations. The `sync` method accepts an array of IDs to place on the intermediate table. Any IDs that are not in the given array will be removed from the intermediate table. So, after this operation is complete, only the IDs in the given array will exist in the intermediate table:

    $user->r->roles()->sync([1, 2, 3]);

You may also pass additional intermediate table values with the IDs:

    $user->r->roles()->sync([1 => ['expires' => true], 2, 3]);

If you would like to insert the same intermediate table values with each of the synced model IDs, you may use the `syncWithPivotValues` method:

    $user->r->roles()->syncWithPivotValues([1, 2, 3], ['active' => true]);

If you do not want to detach existing IDs that are missing from the given array, you may use the `syncWithoutDetaching` method:

    $user->r->roles()->syncWithoutDetaching([1, 2, 3]);

<a name="toggling-associations"></a>
#### Toggling Associations

The many-to-many relationship also provides a `toggle` method which "toggles" the attachment status of the given related model IDs. If the given ID is currently attached, it will be detached. Likewise, if it is currently detached, it will be attached:

    $user->r->roles()->toggle([1, 2, 3]);

You may also pass additional intermediate table values with the IDs:

    $user->r->roles()->toggle([
        1 => ['expires' => true],
        2 => ['expires' => true],
    ]);

<a name="updating-a-record-on-the-intermediate-table"></a>
#### Updating a Record on the Intermediate Table

If you need to update an existing row in your relationship's intermediate table, you may use the `updateExistingPivot` method. This method accepts the intermediate record foreign key and an array of attributes to update:

    $user = User::query()->find(1);

    $user->r->roles()->updateExistingPivot($roleId, [
        'active' => false,
    ]);

<a name="touching-parent-timestamps"></a>
## Touching Parent Timestamps

When a model defines a `belongsTo` or `belongsToMany` relationship to another model, such as a `Comment` which belongs to a `Post`, it is sometimes helpful to update the parent's timestamp when the child model is updated.

For example, when a `Comment` model is updated, you may want to automatically "touch" the `updated_at` timestamp of the owning `Post` so that it is set to the current date and time. To accomplish this, you may add a `touches` property to your child model containing the names of the relationships that should have their `updated_at` timestamps updated when the child model is updated:

    <?php

    namespace App\Models;

    use MacropaySolutions\Kernel\Database\Obvious\Model;
    use MacropaySolutions\Kernel\Database\Obvious\Relations\BelongsTo;

    class Comment extends Model
    {
        /**
         * All the relationships to be touched.
         *
         * @var array
         */
        protected $touches = ['post'];

        /**
         * Get the post that the comment belongs to.
         */
        public function post(): BelongsTo
        {
            return $this->belongsTo(Post::class);
        }
    }

> [!WARNING]  
> Parent model timestamps will only be updated if the child model is updated using Obvious's `save` method.
> Using $touches will be slower.
