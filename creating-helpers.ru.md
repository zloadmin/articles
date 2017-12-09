# Creating Your Own PHP Helpers in a Laravel Project

Laravel provides many excellent [helper functions](https://laravel.com/docs/5.5/helpers) that are convenient for doing things like working with arrays, file paths, strings, and routes, among other things like the beloved `dd()` function.

You can also define your own set of helper functions for your Laravel applications and PHP packages, by using Composer to import them automatically.

If you are new to Laravel or PHP, let’s walk through how you might go about creating your own helper functions that automatically get loaded by Laravel.

## Creating a Helpers file in a Laravel App

The first scenario you might want to include your helper functions is within the context of a Laravel application. Depending on your preference, you can organize the location of your helper file(s) however you want, however, here are a few suggested locations:

*   `app/helpers.php`
*   `app/Http/helpers.php`

I prefer to keep mine in `app/helpers.php` in the root of the application namespace.

### Autoloading

To use your PHP helper functions, you need to load them into your program at runtime. In the early days of my career, it wasn’t uncommon to see this kind of code at the top of a file:

    require_once ROOT . '/helpers.php';

PHP functions cannot be autoloaded. However, we have a much better solution through Composer than using `require` or `require_once`.

If you create a new Laravel project, you will see an `autoload` and `autoload-dev` keys in the `composer.json` file:

    "autoload": {
        "classmap": [
            "database/seeds",
            "database/factories"
        ],
        "psr-4": {
            "App\\": "app/"
        }
    },
    "autoload-dev": {
        "psr-4": {
            "Tests\\": "tests/"
        }
    },

If you want to add a helpers file, composer has a `files` key (which is an array of file paths) that you can define inside of `autoload`:

    "autoload": {
        "files": [
            "app/helpers.php"
        ],
        "classmap": [
            "database/seeds",
            "database/factories"
        ],
        "psr-4": {
            "App\\": "app/"
        }
    },

Once you add a new path to the `files` array, you need to dump the autoloader:

    composer dump-autoload

Now on every request the helpers.php file will be loaded automatically because Laravel requires Composer’s autoloader in `public/index.php`:

    require __DIR__.'/../vendor/autoload.php';

### Defining Functions

Defining functions in your helpers class is the easy part, although, there are a few caveats. All of the Laravel helper files are wrapped in a check to avoid function definition collisions:

    if (! function_exists('env')) {
        function env($key, $default = null) {
            // ...
        }
    }

This can get tricky, because you can run into situations where you are using a function definition that you did not expect based on which one was defined first.

I prefer to use `function_exists` checks in my application helpers, but if you are defining helpers within the context of your application, you _could_ forgo the `function_exists` check.

By skipping the check, you’d see collisions any time your helpers are redefining functions, which could be useful.

In practice, collisions don’t tend to happen as often as you’d think, and you should make sure you’re defining function names that aren’t overly generic. You can also prefix your function names to make them less likely to collide with other dependencies.

### Helper Example

I like the Rails path and URL helpers that you get for free when defining a resourceful route. For example, a `photos` resource route would expose route helpers like `new_photo_path`, edit_photo_path`, etc.

When I use resource routing in Laravel, I like to add a few helper functions that make defining routes in my templates easier. In my implementation, I like to have a URL helper function that I can pass an Eloquent model and get a resource route back using conventions that I define, such as:

    create_route($model);
    edit_route($model);
    show_route($model);
    destroy_route($model);

Here’s how you might define a `show_route` in your `app/helpers.php` file (the others would look similar):

    if (! function_exists('show_route')) {
        function show_route($model, $resource = null)
        {
            $resource = $resource ?? plural_from_model($model);

            return route("{$resource}.show", $model);
        }
    }

    if (! function_exists('plural_from_model')) {
        function plural_from_model($model)
        {
            $plural = Str::plural(class_basename($model));

            return Str::kebab($plural);
        }
    }

The `plural_from_model()` function is just some reusable code that the helper route functions use to predict the route resource name based on a naming convention that I prefer, which is a kebab-case plural of a model.

For example, here’s an example of the resource name derived from the model:

    $model = new App\LineItem;
    plural_from_model($model);
    => line-items

    plural_from_model(new App\User);
    => users

Using this convention you’d define the resource route like so in `routes/web.php`:

    Route::resource('line-items', 'LineItemsController');
    Route::resource('users', 'UsersController');

And then in your blade templates, you could do the following:

    <a href="{{ show_route($lineItem) }}">
        {{ $lineItem->name }}
    </a>

Which would produce something like the following HTML:

    <a href="http://localhost/line-items/1">
        Line Item #1
    </a>

## Packages

Your Composer packages can also use a helpers file for any helper functions you want to make available to projects consuming your package.

You will take the same approach in the package’s `composer.json` file, defining a `files` key with an array of your helper files.

It’s imperative that you add `function_exists()` checks around your helper functions so that projects using your code don’t break due to naming collisions.

You should choose proper function names that are unique to your package, and consider using a short prefix if you are afraid your function name is too generic.

## Learn More

Check out Composer’s [autoloading](https://getcomposer.org/doc/04-schema.md#autoload) documentation to learn more about including files, and general information about autoloading classes.

Another recommended resource is learning about all the nifty [Laravel helpers](https://laravel.com/docs/5.5/helpers) available in the framework and learning how they work by checking out the source code for the [Illuminate\Foundation helpers](https://github.com/laravel/framework/blob/5.5/src/Illuminate/Foundation/helpers.php) and the [Illuminate\Support helpers](https://github.com/laravel/framework/blob/5.5/src/Illuminate/Support/helpers.php).

This appeared first on [Laravel News](https://laravel-news.com)

Original: [https://laravel-news.com/creating-helpers](https://laravel-news.com/creating-helpers)