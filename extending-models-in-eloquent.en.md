![](https://tighten.co/assets/img/blog/painted_model_inheritance.png)



We’ve come a long way since the days of hand-writing SQL queries in our web apps. Tools like Laravel’s [Eloquent ORM](https://laravel.com/docs/master/eloquent) allow us to interact with databases at a higher level, freeing us from lower level details like query syntax and security.

When starting out with Eloquent, it’s natural to reach for familiar operations like `where` and `join`. For more advanced users, features like scopes, accessors, and mutators offer more expressive alternatives to the query-building patterns of old.

Let’s explore another alternative that can be used as a stand-in for repetitive `where` statements and local scopes. This technique involves creating new Eloquent models that extend other models. By extending another model, you inherit the full functionality of the parent model, while retaining the ability to add custom methods, scopes, event listeners, etc. This is commonly referred to as “Single Table Inheritance,” but I prefer to just call it “Model Inheritance”.

## An Example

Most web apps have the concept of an "administrator." Administrators are typically users with elevated permissions and access to restricted areas of the application. To make the distinction between a normal user and an admin user, statements like the one below emerge as a common pattern:

    $admins = User::where('is_admin', true)->get();

When a specific `where` statement becomes a pattern throughout your app, it is often beneficial to replace it with a [local scope](https://laravel.com/docs/master/eloquent#local-scopes). By implementing an `isAdmin` scope on the `User` model, we can write a more expressive and reusable Eloquent statement:

    $admins = User::isAdmin()->get();

    // Implementation:
    class User extends Model
    {
        public function scopeIsAdmin($query)
        {
            $query->where('is_admin', true);
        }
    }

Let's take this abstraction one step further using model inheritance. By extending the `User` model and adding a global scope, we achieve the exact same result as before, but now with an entirely new entity in our application. This entity (`Admin`) can now be home to custom methods, scopes, and other meaningful functionality.

    $admins = Admin::all();

    // Implementation:
    class Admin extends User
    {
        protected $table = 'users';

        public static function boot()
        {
            parent::boot();

            static::addGlobalScope(function ($query) {
                $query->where('is_admin', true);
            });
        }
    }

> Note: `protected $table = ‘users’` is necessary for queries to work properly. Eloquent uses a model’s class name to determine the name of the database table. Therefore, it assumes a table name of “admins” instead of “users,” resulting in a `Base table or view not found` error.

Once you have an `Admin` model, it is easier and cleaner to separate admin-specific functionality from your `User` class. For example:

#### Notifications

Simple operations like sending notifications to all administrators become simpler with the new `Admin` model.

    Notification::send(Admin::all(), NewSignUp($user));

#### Guard Clauses

Any time an operation on the `User` model is restricted to admins only, a guard clause is typically needed to ensure authorization.

    // Guard clause
    if ($admin = User::find($id)->is_admin !== true) {
          throw new Exception;
    }

    $admin->impersonate($user);

Because of `Admin`’s global scope, the guard clause becomes unnecessary when the `impersonate` method is called on the `Admin` class.

    Admin::findOrFail($id)->impersonate($user);

#### Model Factories

In a testing context, you may need to create `User` models with admin privileges using [model factories](https://laravel.com/docs/master/database-testing#writing-factories) like the example below.

    $admin = factory(User::class)->create(['is_admin' => true]);

    // User factory implementation
    $factory->define(User::class, function () {
        return [
            ...
              'is_admin' => false,
        ];
    });

We can improve this statement by introducing a [model factory state](https://laravel.com/docs/master/database-testing#factory-states) to encapsulate what defines a user as an admin.

    $admin = factory(User::class)->states('admin')->create();

    // Admin state implementation
    $factory->state(User::class, 'admin', function () {
        return ['is_admin' => true];
    });

This is surely an improvement, however, the factory still returns an instance of the `User` model. By defining an entirely new factory for `Admin`, we get the same permissions while returning an instance of `Admin`.

    $admin = factory(Admin::class)->create();

    // Admin factory implementation
    $factory->define(Admin::class, function () {
        return ['is_admin' => true]
              + factory(User::class)->raw();
    });

## One Gotcha: relationships don’t work

Similar to how Eloquent evaluates table names, a model’s class name is used to determine foreign keys and pivot tables. Therefore, accessing relationships from the `Admin` model is problematic.

    Admin::first()->posts;
    // Throws: Unknown column 'posts.admin_id'

    // Failing implementation:
    class Admin extends User {
        //
    }

    class User extends Model {
        public function posts() {
              return $this->hasMany(Post::class);
        }
    }

Eloquent can’t process this relationship because it assumes each `Post` has an `admin_id` field instead of a `user_id` field. We can fix this by explicitly passing the `user_id` foreign key on the `User` model:

    // Working implementation:
    class Admin extends User {
        //
    }

    class User extends Model {
        public function posts() {
              return $this->hasMany(Post::class, 'user_id');
        }
    }

The same problem exists with many-to-many relationships. Eloquent assumes the pivot table name matches the current model’s class name:

    Admin::first()->tags;
    // Throws: Table 'admin_tag' doesn't exist

    // Failing implementation:
    class Admin extends User {
        //
    }

    class User extends Model {
        public function tags() {
              return $this->belongsToMany(Tag::class);
        }
    ...

Again, we can solve this problem by explicitly passing the pivot table name and foreign key:

    // Working implementation:
    class Admin extends User {
        //
    }

    class User extends Model {
        public function tags() {
              return $this->belongsToMany(Tag::class, 'user_tag', 'user_id');
        }
    ...

Although explicitly defining foreign keys and pivot table names will allow our `Admin` model to access relationships defined on the `User` model, this is less than ideal. The existence of these seemingly unnecessary definitions is not apparent anywhere in the codebase.

However, you can create a `HasParentModel` trait that automatically handles this issue. From Eloquent’s perspective, it substitutes the current model’s class name with the class name of the parent model. Check out an example on [GitHub](https://gist.github.com/calebporzio/a63b165b500d491a0c250eb5853e5d94).

> There's a lot more that you can do to make Laravel handle single-table inheritance well. We've created a package that makes it easy to add to your Laravel apps and are preparing to release it any day now. Make sure to watch our [Twitter](http://twitter.com/tightenco/) for the announcement!

Let’s take a look at our new `Admin` model that utilizes this trait:

    use App\Abilities\HasParentModel;

    class Admin extends User
    {
        use HasParentModel;
          // Notice we no longer need: protected $table = 'users'

        public static function boot()
        {
            parent::boot();

            static::addGlobalScope(function ($query) {
                $query->where('is_admin', true);
            });
        }
    }

Now our `User` model’s relationships can go back to relying on Eloquent’s sensible defaults.

    // Working implementation:
    class User extends Model
    {
          public function posts() {
              return $this->hasMany(Post::class);
        }

        public function tags() {
              return $this->belongsToMany(Tag::class);
        }
    }

The `HasParentModel` trait cleans up our models and lets developers know there is something special going on under the hood.

## Wrapping things up

We’ve identified common Eloquent patterns and cleaned them up using model inheritance. This technique helps us create better-named and encapsulated entities within our application. Remember, model inheritance can be applied to any Eloquent model, not just `Users` and `Admins`. The possibilities are endless!

Go be creative, have fun, and share what you’ve learned along the way. Let us know how you're using this technique for fun or in the wild! (Tweet [@calebporzio](https://twitter.com/calebporzio) and [@tightenco](https://twitter.com/tightenco))

Enjoy!

Source: [https://tighten.co/blog/extending-models-in-eloquent](https://tighten.co/blog/extending-models-in-eloquent)