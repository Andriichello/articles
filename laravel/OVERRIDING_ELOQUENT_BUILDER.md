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
<br><br/>

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
<br><br/>

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