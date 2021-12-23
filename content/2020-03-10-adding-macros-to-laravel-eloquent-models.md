+++
title = "Adding macros to Laravel Eloquent models"
path = "blog/post/adding-macros-to-laravel-eloquent-models"

[taxonomies]
categories = ["blog"]
tags = ["development", "laravel", "package"]
+++

Maybe it's not the most elegant solution. Maybe even it isn't the right one. But I needed to explore this option, test it's viability and decide afterwards.
<!-- more -->

It's a controversial topic. More often than not, it's not seen as a good programming practice. It surely has implications in code debug and *testability*.
On the other hand, some people defend this approach citing the SOLID principles and other justifications like producing more generic implementations.

What is it that I'm are talking about?

## Adding methods to a class at runtime
Specifically, I will dive into how I managed to add methods to [Laravel](https://laravel.com) [Eloquent](https://laravel.com/docs/eloquent) [models](https://laravel.com/docs/7.x/eloquent#defining-models) dinamically, at runtime, and how I ended up packaging it for reusability.
This is achieved using Laravel's [Macroable](https://laravel.com/api/7.x/Illuminate/Support/Traits/Macroable.html) trait which some internal classes make use of to allow developers to extend their functionality.

## Into the **why**
I'm working on developing a platform for a client. This platform is a really generic one. The client plans on developing once, and selling to multiple businesses. So it needs to be really generic and adaptable to various scenarios.
But, the **core** of the platform will be always the same, shared through all it's instances.

After defining what this common **core** should include, investigation on how to implement the desired extensibility started. And, because the **core** is shared between implementations, I wanted to find a solution that allowed me to add functionality to the core without modifying it's code.

The idea of creating modules that add functionality (methods, relationships, endpoints) to the existing core was very appealing. In a Laravel application, some of these things are really easy to implement through packages (like routes, for example), but others require some more trickery.

What ended up being quite difficult was adding methods to core-defined models without adding traits to them afterwards, or extending the classes. That's why I ended up making a [package](https://github.com/javoscript/laravel-macroable-models), for anyone to use, with the solution I found.

## The **how**
Because of Laravel Eloquent models not using the `Macroable` trait to allow for runtime-method-adding ([and probably never will](https://github.com/laravel/ideas/issues/1473)) a different approach was needed.

As some people [already](https://medium.com/@zarehesmaiel/dynamic-relation-by-macro-988d638b6e51) [discovered](https://laracasts.com/discuss/channels/eloquent/using-laravel-macros-on-eloquent-models) [before](https://github.com/laravel/cashier-mollie/issues/107) me, Laravel's `Illuminate\Database\Eloquent\Builder` class uses the `Macroable` trait and, indirectly through it, you can add methods to Eloquent models at runtime. But all of these implementations where not generic, so a class-by-class statement was necessary.

The solution proposed in those links is of the form:

```php
<?php

// This example adds the `categories` function to the `Post` model
// Taken from https://medium.com/@zarehesmaiel/dynamic-relation-by-macro-988d638b6e51

Builder::macro(‘categories’, function() {
    $model = $this->getModel();
    if($model instanceof Post) {
         return $model->belongsToMany(Category::class);
    }
    unset(static::$macros['categories']);
    return $model->categories();
 });
```

Two issues exist in that implementation:
- The same block has to be repeated for every macro added to a model
- The logic of the macro itself is tangled with the logic of the "add this macro" functionality, reducing readability

Some inspiration was also taken from some other existing packages that solve similar problems:
* imanghafoori1/eloquent-relativity: [github](https://github.com/imanghafoori1/eloquent-relativity)
* spatie/laravel-collection-macros: [github](https://github.com/spatie/laravel-collection-macros)
* spatie/macroable: [github](https://github.com/spatie/macroable)

Be sure to check them out to see how they implemented their solutions!

### So, **how** does this package solve this?

It registers a `singleton` (of the `Javoscript\MacroableModels\MacroableModels` class) in it's `Service Provider` that basically keeps track of all the added macros to the `Builder` class, and to which `model` they should be attached.

In the singleton, an array of the following structure gets populated with the macro names, the models to which the macro corresponds to and the function itself (using PHP's [Closure](https://www.php.net/manual/en/class.closure.php) class):

```php
<?php

$macros = [
    "macroOne" => [
        "ModelOne" => Closure() {#3362 …2},
        "ModelTwo" => Closure() {#3376 …2},
    ],
    "macroTwo" => [
        "ModelOne" => Closure() {#3366 …2},
    ]
]
```

Because the macros are actually being added to the `Builder` class, both functions defined with the name `macroOne` in the example above should be added into the same macro (the `Builder` class can't have two methods with the same name and different logic).

So, the whole array corresponding to the `macroOne` key is passed to the macro being added to the `Builder` class and the macro itself fetches the corresponding `Closure` and executes it if it exists.

Whenever a new method is added (or removed) through the package, this is the function in charge of doing what I explained just above:

```php
<?php

private function syncMacros($name)
{
    $models = $this->macros[$name];
    Builder::macro($name, function(...$args) use ($name, $models){
        $class = get_class($this->getModel());

        if (! isset($models[$class])) {
            throw new \BadMethodCallException("Call to undefined method ${class}::${name}()");
        }

        $closure = \Closure::bind($models[$class], $this->getModel());
        return call_user_func($closure, ...$args);
    });
}
```

> psst! The package is open source.. go check out the code by yourself! ([link](https://github.com/javoscript/laravel-macroable-models))

Some additional goodies were added for a better developer experience:

#### Multiple parameters support
You can define macros with any number and kind of parameters. For example:

```php
<?php

// Macro with no parameters
MacroableModels::addMacro(App\User::class, 'sayHi', function() {
    return 'Hi!';
})

// Macro with one parameter
MacroableModels::addMacro(App\User::class, 'saySomething', function(String $something) {
    return $something;
})


$user = App\User::first();

$user->sayHi();
// >>> "Hi!"

$user->saySomething("Bye!");
// >>> "Bye!"
```

#### Context binding
As you normally would when defining methods in a model, you have access to the `$this` variable, and it works as you would expect!

```php
<?php

MacroableModels::addMacro(App\User::class, 'getId', function() {
    return $this->id;
});

App\User::first()->getId();
// >>> 1
```

#### Some convenient methods
Useful methods for querying existent macros and to what models they were attached:
- `removeMacro`
- `getAllMacros`
- `modelHasMacro`
- `modelsThatImplement`
- `macrosForModel`

## Conclusion
It's time to test if this is the right approach to my problem... But it'll be much easier to implement now that I have it packaged up and uploaded to [packagist](https://packagist.org/packages/javoscript/laravel-macroable-models).

If you have any question, suggestions or feature request, drop by the [github project](https://github.com/javoscript/laravel-macroable-models) and don't hesitate to open an issue! =)
