# Table of contents
[Overriding model's Eloquent Builder](#overriding)
- [Step 1. Create class, which extends Eloquent Builder](#overriding-step-1)
- [Step 2: Override model's **newEloquentBuilder()**](#overriding-step-2)
- [Step 3: Add model's **query()** type hint](#overriding-step-3)
- [Step 4. Create querying methods](#overriding-step-4)

[Usage examples](#usage)
- [Perfect type hints](#usage-perfect-hints)
- [No type hints at all](#usage-no-hints)
- [Relations](#usage-relations)
- [Scopes](#usage-scopes)
    - [In model class](#usage-scopes-in-model)
    - [In query builder class](#usage-scopes-in-builder)

[Nested queries](#nested-queries)
- [Creating methods for nested querying](#creating-wrapped-where)
- [Using newly created methods for nested querying](#using-wrapped-where)

[How can it make your life easier?](#how-it-helps-you)
- [Limiting visibility of records based on **User**](#limiting-visibility-based-on-user)

<br><br>

# Overriding model's Eloquent Builder <a name="overriding"></a>
## Step 1. Create class, which extends Eloquent Builder <a name="overriding-step-1"></a>

First create a class, which will be the base for all future query builders.

``` php
use Illuminate\Database\Eloquent\Builder as EloquentBuilder;

/**
 * Class BaseQueryBuilder.
 */
class BaseQueryBuilder extends EloquentBuilder
{
    // common methods querying or setting up the queries
}
```

Then proceed by creating class for a specific model, e.g. `HolidayQueryBuilder`, which is used for `Holiday`.

``` php
/**
 * Class HolidayQueryBuilder.
 */
class HolidayQueryBuilder extends BaseQueryBuilder
{
   // methods and properties...
}
```
<br>

## Step 2: Override model's **newEloquentBuilder()** <a name="overriding-step-2"></a>

Override `Holiday`'s `newEloquentBuilder()` in order to make it use newly created `HolidayQueryBuilder`.

``` php
use Illuminate\Database\Query\Builder as DatabaseBuilder;

/**
 * Class Holiday.
 */
class Holiday extends Model
{
    // properties and methods...

    /**
     * @param DatabaseBuilder $query
     *
     * @return HolidayQueryBuilder
     */
    public function newEloquentBuilder($query): HolidayQueryBuilder
    {
        return new HolidayQueryBuilder($query);
    }
}
```
<br>

## Step 3: Add model's **query()** type hint <a name="overriding-step-3"></a>

To `Holiday`'s PHPDoc add `query()` method annotation, in which specify that return type is `HolidayQueryBuilder`.
``` php 
@method static HolidayQueryBuilder query()
```

``` php
/**
 * Class Holiday.
 *
 * @method static HolidayQueryBuilder query()
 */
class Holiday extends Model
{
    // properties and methods...

    /**
     * @param DatabaseBuilder $query
     *
     * @return HolidayQueryBuilder
     */
    public function newEloquentBuilder($query): HolidayQueryBuilder
    {
        return new HolidayQueryBuilder($query);
    }
}
```
<br>

## Step 4. Create querying methods <a name="overriding-step-4"></a>
After completing steps 1-3 the `Holiday` is set up to use `HolidayQueryBuilder`, so now it's time to proceed with creating methods for querying.

``` php
/**
 * Class HolidayQueryBuilder.
 */
class HolidayQueryBuilder extends BaseQueryBuilder
{
    /**
     * Only holidays that are repeating.
     *
     * @param bool $repeating
     *
     * @return static
     */
    public function repeating(bool $repeating): static
    {
        $this->where('repeating', $repeating);

        return $this;
    }

    /**
     * Only holidays that are relevant on a given date.
     *
     * @param CarbonInterface $date
     *
     * @return static
     */
    public function relevantOn(CarbonInterface $date): static
    {
        $this->where('date', '=', $date);

        return $this;
    }
}
```
<br>

# Usage examples <a name="usage"></a>
## Perfect type hints <a name="usage-perfect-hints"></a>
``` php
Holiday::query()
    ->repeating(true)
    ->relevantOn(Carbon::yesterday())
    ->each(function (Holiday $holiday) {
        dispatch(new ProlongHoliday($holiday));
    });
```
After calling `query()` IDE will know that following calls are made on an instance of `HolidayQueryBuilder`, so it will show a list of available methods, such as:
``` php 
repeating(bool $repeating): HolidayQueryBuilder
```
``` php 
relevantOn(CarbonInterface $date): HolidayQueryBuilder
```
<br>

## No type hints at all <a name="usage-no-hints"></a>
``` php
Holiday::repeating(true)
    ->relevantOn(Carbon::yesterday())
    ->each(function (Holiday $holiday) {
        dispatch(new ProlongHoliday($holiday));
    });
```
This code snippet will have the same result as the previus one, but without calling `query()` IDE won't know that following calls are made on an instance of `HolidayQueryBuilder`, so it won't show hints.
<br><br>

## Relations <a name="usage-relations"></a>

``` php
class Restaurant extends BaseModel
{
    /**
     * Holidays associated with the model.
     *
     * @return BelongsToMany
     */
    public function holidays(): BelongsToMany
    {
        return $this->belongsToMany(Holiday::class, 'restaurant_holiday');
    }
}
```
Important thing to note is that calling `holidays()` on an instance of `Restaurant` will return object of type `BelongsToMany`, which in turn has `HolidayQueryBuilder` as base query builder. In tinker it will look something like this:

``` php
>>> $restaurant->holidays()
=> Illuminate\Database\Eloquent\Relations\BelongsToMany {}
```
``` php
>>> $restaurant->holidays()->getQuery()
=> App\Queries\HolidayQueryBuilder {}
```

This means that the following code block is correct, since `HolidayQueryBuilder`'s methods can be used on `holidays()` relation. 
``` php
$holidays = $restaurant->holidays()
    ->relevantOn(now())
    ->get();
```
But IDE won't show `relevantOn()` method in hints, since it has no idea that `holidays()` relation has `HolidayQueryBuilder` as base query builder. Specifying `BelongsToMany|HolidayQueryBuilder` union as return type will fix that.
``` php
/**
 * Holidays associated with the model.
 *
 * @return BelongsToMany|HolidayQueryBuilder
 */
public function holidays(): BelongsToMany|HolidayQueryBuilder
```
 <br>


## Scopes <a name="usage-scopes"></a>
## In model class <a name="usage-scopes-in-model"></a>
First lets define `relevant` scope on the `Holiday`.
``` php
/**
 * Class Holiday.
 *
 * @method static HolidayQueryBuilder query()
 */
class Holiday extends Model
{
    /** 
     * Only holidays that are relevant starting from now.
     * 
     * @param HolidayQueryBuilder $builder
     * 
     * @return void
     */
    public function scopeRelevant(HolidayQueryBuilder $builder): void {
        $builder->where('date', '>=', now());
    }

    /**
     * @param DatabaseBuilder $query
     *
     * @return HolidayQueryBuilder
     */
    public function newEloquentBuilder($query): HolidayQueryBuilder
    {
        return new HolidayQueryBuilder($query);
    }
}
```
There are three ways, in which `relevant` scope can be applied, namely:
1. **Static call on class**
``` php
>>> Holiday::relevant()->toSql()
=> "select * from `holidays` where `date` >= ?"
```
2. **Direct call on query builder instance**
``` php
>>> Holiday::query()->relevant()->toSql()
=> "select * from `holidays` where `date` >= ?"
```
3. **By specifying scope name in a call to `scopes()` on query builder instance**
``` php
>>> Holiday::query()->scopes('relevant')->toSql()
=> "select * from `holidays` where `date` >= ?"
```
<br>

## In query builder class <a name="usage-scopes-in-builder"></a>
Lets move `scopeRelevant()` definition from `Holiday` to `HolidayQueryBuilder` and see what will change.

``` php
/**
 * Class HolidayQueryBuilder.
 */
class HolidayQueryBuilder extends BaseQueryBuilder
{
    /** 
     * Only holidays that are relevant starting from now.
     * 
     * @param HolidayQueryBuilder $builder
     * 
     * @return void
     */
    public function scopeRelevant(HolidayQueryBuilder $builder): void {
        $builder->where('date', '>=', now());
    }
}
```

There were three ways, in which `relevant` scope could be applied when it was defined in `Holiday`. Lets test them when `scopeRelevant()` is defined in `HolidayQueryBuilder`:
1. **Static call on class**
``` php
>>> Holiday::relevant()->toSql()
=> BadMethodCallException with message 'Call to undefined method App\Models\Holiday::relevant()'
```
2. **Direct call on query builder instance**
``` php
>>> Holiday::query()->relevant()->toSql()
=> BadMethodCallException with message 'Call to undefined method App\Queries\HolidayQueryBuilder::relevant()'
```
3. **By specifying scope name in a call to `scopes()` on query builder instance**
``` php
>>> Holiday::query()->scopes('relevant')->toSql()
=> "select * from `holidays` where `date` >= ?"
```
Scopes, defined in query builders, can be applied by specifying name in a `scopes()` call, but not by calling it directly on query builder instance or statically on model class.

To make scopes on dedicated query builder behave the same way as the ones defined on model we need to add methods, which replicate scope calls. It will look something like this:

``` php
/**
 * Class HolidayQueryBuilder.
 */
class HolidayQueryBuilder extends BaseQueryBuilder
{
    /** 
     * Only holidays that are relevant starting from now.
     * 
     * @param HolidayQueryBuilder $builder
     * 
     * @return void
     */
    public function scopeRelevant(HolidayQueryBuilder $builder): void {
        $builder->where('date', '>=', now());
    }

    /** 
     * Only holidays that are relevant starting from now.
     * 
     * @return static
     */
    public function relevant(): static {
        $this->scopes('relevant');

        return $this;
    }
}
```

After adding `relevant()` method to `HolidayQueryBuilder` all three ways of applying `relevant` scope will be available, namely:
1. **Static call on class**
``` php
>>> Holiday::relevant()->toSql()
=> "select * from `holidays` where `date` >= ?"
```
2. **Direct call on query builder instance**
``` php
>>> Holiday::query()->relevant()->toSql()
=> "select * from `holidays` where `date` >= ?"
```
3. **By specifying scope name in a call to `scopes()` on query builder instance**
``` php
>>> Holiday::query()->scopes('relevant')->toSql()
=> "select * from `holidays` where `date` >= ?"
```
<br>

# Nested queries <a name="nested-queries"></a>
By default callback specified in `whereNested()` will have first parameter of `Illuminate\Database\Query\Builder` type, so methods defined in the dedicated query builder will not be available.

If there is a need to use dedicated query builder in nested callbacks, then you’ll have to create new methods for it or update existing ones (the problem is that those methods are located in `Illuminate\Database\Query\Builder`, but we are extending `Illuminate\Database\Eloquent\Builder` and this brings us a lot of problems due to method calls forwarding).
<br><br>

## Creating methods for nested querying <a name="creating-wrapped-where"></a>
``` php
use Illuminate\Database\Eloquent\Builder as EloquentBuilder;

/**
 * Class BaseQueryBuilder.
 */
class BaseQueryBuilder extends EloquentBuilder
{
    /**
     * Create a new query instance for wrapped where condition.
     *
     * @return static
     */
    public function forWrappedWhere(): static
    {
        /** @var static $builder */
        $builder = $this->model::query();
        $builder->withoutGlobalScopes();

        return $builder;
    }

    /**
     * Add a wrapped where statement to the query.
     *
     * @param Closure $callback
     * @param string $boolean
     *
     * @return static
     */
    public function whereWrapped(Closure $callback, string $boolean = 'and'): static
    {
        call_user_func($callback, $query = $this->forWrappedWhere());
        $this->addNestedWhereQuery($query->getQuery(), $boolean);

        return $this;
    }

    /**
     * Add a wrapped orWhere statement to the query.
     *
     * @param Closure $callback
     *
     * @return static
     */
    public function orWhereWrapped(Closure $callback): static
    {
        return $this->whereWrapped($callback, 'or');
    }
}
```
Calling `forWrappedWhere()` will return a new instance of model's dedicated query builder and it's important to note that global scopes are removed from the returned query builder to avoid unpredicteble side effects. 
<br><br>

## Using newly created methods for nested querying <a name="using-wrapped-where"></a>
Callback, which is being specified in `whereWrapped()` and `orWhereWrapped()`, has first parameter of dedicated query builder type, so all of the methods defined in it will be available to use.

``` php
Holiday::query()
    ->whereWrapped(function (HolidayQueryBuilder $builder) {
        // conditions, which should be nested            
    }) 
```
<br>

# How can it make your life easier? <a name="how-it-helps-you"></a>
## Limiting visibility of records based on `User` <a name="limiting-visibility-based-on-user"></a>
One of the cool ways to use dedicated query builder is to define querying methods for limiting records to only those that are available for given user or on given request. It’s really helpful with CRUD operations such as index.

First lets create `IndexableInterface`. 
``` php
/**
 * Interface IndexableInterface.
 */
interface IndexableInterface
{
    /**
     * Apply index query conditions.
     *
     * @param User $user
     *
     * @return BaseQueryBuilder
     */
    public function index(User $user): BaseQueryBuilder;
}
```

Then implement `IndexableInterface` in `BaseQueryBuilder` or on specific dedicated query builder class.

``` php
use Illuminate\Database\Eloquent\Builder as EloquentBuilder;

/**
 * Class BaseQueryBuilder.
 */
class BaseQueryBuilder extends EloquentBuilder implements IndexableInterface
{
    /**
     * Apply index query conditions.
     *
     * @param User $user
     *
     * @return static
     */
    public function index(User $user): static
    {
        return $this;
    }

    // common methods and properties...
}
```

Lets take `CustomerQueryBuilder` as an example. It makes sure that staff members can get a list of all customers and at the same time other users can only get their own record.

``` php
/**
 * Class CustomerQueryBuilder.
 */
class CustomerQueryBuilder extends BaseQueryBuilder
{
    /**
     * Apply index query conditions.
     *
     * @param User $user
     *
     * @return static
     */
    public function index(User $user): static
    {
        if ($user->isStaff()) {
            return $this;
        }

        $this->where('id', $user->customer_id);

        return $this;
    }
}
```
