# New Route Methods introduced in Laravel 5.5

[Laravel 5.5](https://laravel-news.com/laravel-5-5) shipped a couple of convenient shortcuts to the Laravel `Router` class that eliminates the need for creating a controller or closure only to return a simple view or redirect. If you missed them in the release notes, let’s look at them briefly, they are sure to simplify your code and remove a couple of files.

## The `Route::view` method

The `Route::view` method eliminates the need for routes that only need a view returned. Instead of using a controller or a closure, you can define a URI and a path to a view file:

    // resources/views/pages/about.blade.php
    Route::view('/about', 'pages.about');

You can also pass in an array of variables that will be passed to the view:

    Route::view('/about', 'pages.about', ['year' => date('Y')]);

## The `Route::redirect` Method

The `Route::redirect` method also eliminates the need to create a controller or a closure only to return a redirect response:

    Route::redirect('/old-about', '/about');

The third default argument, if not passed, is a `301` redirect. However, you can pass the third argument for a different status code. For example, if you want to create a `307 Temporary Redirect`, it would look like this:

    Route::redirect('/old-about', '/about', 307);

## More Info

Laravel 5.5 is chalk-full of great new features; you can learn more by visiting our [coverage of Laravel 5.5](https://laravel-news.com/category/laravel-5.5) and the [official release notes](https://laravel.com/docs/5.5/releases).

This appeared first on [Laravel News](https://laravel-news.com)

Source: [https://laravel-news.com/laravel-5-5-router-view-and-redirect](https://laravel-news.com/laravel-5-5-router-view-and-redirect)