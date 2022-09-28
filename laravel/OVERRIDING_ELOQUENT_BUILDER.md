# Overriding model's Eloquent Builder
## Step 1. Create class, which extends Eloquent Builder

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

Then proceed by creating class for a specific model, e.g. `HolidayQueryBuilder`, which is used for `Holiday` model.

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

## Step 2: Override model's **newEloquentBuilder()**

Override `Holiday` model's `newEloquentBuilder()` in order to make it use newly created `HolidayQueryBuilder`.

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

## Step 3: Add model's **query()** type hint

To `Holiday` model's PHPDoc add `query()` method annotation, in which specify that return type is `HolidayQueryBuilder`.
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

## Step 4. Add querying methods
After completing steps 1-3 the `Holiday` model is set up to use `HolidayQueryBuilder`, so now it's time to proceed with creating methods for querying.

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

# Usage examples
## Perfectly type hinted
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

## No type hints at all
``` php
Holiday::repeating(true)
    ->relevantOn(Carbon::yesterday())
    ->each(function (Holiday $holiday) {
        dispatch(new ProlongHoliday($holiday));
    });
```
This code snippet will have the same result as the previus one, but without calling `query()` IDE won't know that following calls are made on an instance of `HolidayQueryBuilder`, so it won't show hints.
<br>
