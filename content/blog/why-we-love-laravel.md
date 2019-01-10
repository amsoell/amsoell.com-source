---
title: Why We Love Laravel
date: 2016-01-28
description: "We do a lot of our backend application development in PHP, and preferably with the Laravel framework. What do we love so much about Laravel? We're glad you asked."
tags: [ "development", "backend", "php", "laravel" ]
---

For the longest time I was a language purist. Every line of code I wrote was done by hand using nothing but a simple text editor and a PHP reference book. Inline tag completion? _That's cheating!_ Starting templates? _Shortcuts are for the weak!_ And programming frameworks? Why would I tie myself to an abstraction when I can write "real code?"

This was the early 2000's though, and things have come a long way since then. We've moved through CakePHP to CodeIgnitor to modern, truly impressive frameworks like ExpressionEngine, Craft, and now Laravel. I've dabbled in most of them, but it wasn't until Laravel that I truly felt at home in a PHP framework. Since devoting our PHP developer efforts on Laravel I've begun enjoying writing PHP-based project again, and I'd like to tell you a few reasons why.

## Eloquent: Laravel's database abstration layer

Laravel is far from unique in this one, but their database abstraction layer is just so damn versatile. If you're unfamiliar with database abstraction, it essentially allows you to write your application without making any assumptions about the specific platform that the database is built on. It could be MySQL, Postgres, MariaDB, or even Microsoft SQL Server. Hell, you could run your site off of CSV files if you had the right middleware in place. Once the application is written, it makes very little difference which database platform you build it on, making upgrades and migrations trivial.

Secondly, and perhaps even more impressively, writing your database queries is so much cleaner and self-documenting when you're using Eloquent's methods than trying to pick apart a SQL statement. For example, instead of the following fairly tame SQL statement:

```sql
SELECT * FROM questions LEFT JOIN answers ON questions.id=answers.question_id WHERE scheduled<=NOW() ORDER BY questions.scheduled LIMIT 1
```

becomes even more readable:

```php
Question::with('answers')->where('scheduled','<=', date('Y-m-d H:i:s'))->orderBy('scheduled', 'desc')->first()
```

## Built-in Application Testing

Many, many developers look at testing their applications as a chore; They write down some basic tests, thinking about how _they think_ the user ought to be interacting with the application, and as long as those cases work they consider an application good enough to ship. It's clear, though, that Laravel recognizes how important a solid, structured, and thorough test plan is to a complex web application, and they've included support for testing right out of the gate.

Creating application tests in the Laravel framework is super easy, thanks again to the artisan tool. `artisan make:test UserTest` will create a brand new test class right in the "tests" folder where you can add unit tests for the corresponding class. The tests themselves are even written in very plain-to-understand syntax:

```php
class UserTest extends TestCase {
    public function testLoginPage() {
        $this->visit('/user/login')
            ->see('User login')
            ->type('amsoell', 'username')
            ->type('supersecret', 'password')
            ->press('Login')
            ->seePageIs('/user/dashboard');
    }
}
```

More than simply testing our user-facing pages, we can also use Laravel's built-in unit testing to make sure an API is working as expected:

```php
class ApiTest extends TestCase {
    public function testUserIndex() {
        $this->json('GET', '/api/v1/user')
            ->seeJsonStructure([
                'success',
                'users' => [
                    'id', 'username', 'email', 'created_at', 'updated_at'
                ]
            ]);
    }
}
```

If we write our tests concurrently with writing our routes and controllers (you _are_ writing your tests while you write your routes and controllers, right??) we can test our entire application at any time by running `phpunit` from the terminal. Laravel's testing capabilities go [so much further](https://laravel.com/docs/5.2/testing), but this gives you a good idea of where it starts.

## Migrations

Version control makes building complex applications with a large team of developers a lot simpler. Database development, however, has largely lived outside of the version control ecosystem, making it difficult to track who has the latest version of the data schema. Laravel solves this through its [migrations system](https://laravel.com/docs/5.2/migrations). Rather than build your database structure right in the database, you build a series of data model definitions directly inside your Laravel application. For example:

```php
Schema::create('questions', function (Blueprint $table) {
  $table->increments('id');
  $table->string('question', 255);
  $table->text('detail');
  $table->string('image', 50)->nullable();
  $table->datetime('scheduled')->nullable();
  $table->datetime('posted')->nullable();
  $table->timestamps();
});
```

It's pretty clear what this does: It defines a simple table with an id field, a few extra properties, and some timestamps. But by putting this inside your application, you achieve two important goals:

1. Your database structure is now part of the code, which means it is part of your version control strategy.
2. You can rebuild your database as-needed from the command line with a few keystrokes.

Deploying to a new platform? A simple `artisan migrate` and the database is completely built out. You can even go a step further by taking advantage of Laravel's [seeding hooks](https://laravel.com/docs/5.2/seeding) to fill up the database with the right starting-point data.

## Scheduled task support

Scheduled tasks are another area where, many times, you have important code that is built and maintained completely outside of the main application structure. Laravel brings scheduled tasks into the application itself through its [task scheduling system](https://laravel.com/docs/5.2/scheduling). A single cron entry is added to the system's crontab after which all tasks can be defined and managed right in the application, making full use of established controllers, data models, and configuration settings.

A scheduled task in a Laravel application is just this simple:

```php
protected function schedule(Schedule $schedule) {
    $schedule->call(function() {
        Contest::where('active', true)->update(['active' => false]);
        Contest::where('scheduled', date('Y-m-d'))->limit(1)->update(['active' => true]);
    })->dailyAt('00:15');
}
```

It doesn't get much easier than that. The scheduled task is right inside the application where it can leverage existing database models, controllers and methods.

## Routing

Routing! This one should be absolutely be higher on the list. With Laravel it is insanely easy to define your [application routes](https://laravel.com/docs/5.2/routing) in a single unified location. You can make them as simple or as complex as you like. If you just have a couple of routes that tie into your controllers, it can be as simple as defining the route and indicating which controller method should handle it:

```php
Route::get('about', 'ContentController@showAbout');
Route::get('faq', 'ContentController@showFAQ');
Route::get('contact', 'ContentController@showContact');
```

However, you're building a complex RESTful API, you can also take advantage of more complex routing definitions:

```php
Route::resource('user', 'UserController');
Route::resource('user.order', 'UserOrderController');
Route::resource('user.order.item', 'UserOrderItemController');
```

With these three lines, we've established three API endpoints that _each_ respond to standard GET, PUT, POST, and DELETE HTTP requests, passing the information along to the indicated controller's methods. Even better, you can create the corresponding controllers using artisan:

```bash
artisan controller:make UserController
artisan controller:make UserOrderController
artisan controller:make UserOrderItemController
```

The result of this is three brand new controller files, completely stubbed out with the appropriate `index`, `create`, `store`, `show`, `edit`, `update`, and `destroy` methods ready to be fleshed out.

Additionally, through Laravel's flexible middleware system, we can even wrap all of these requests in a directive that ensures that only authenticated users can access the routes, right here in the same file:

```php
Route::group(array('middleware' => 'auth'), function() {
    Route::resource('user', 'UserController');
    Route::resource('user.order', 'UserOrderController');
    Route::resource('user.order.item', 'UserOrderItemController');
});
```

This is really just the beginning of what Laravel’s routing system can do for us; There is so much more, including built-in CSRF protection, subdomain routing, and [custom middleware](https://laravel.com/docs/5.2/middleware).

## Scratching the surface

Really, I could go on and on ([Authentication](https://laravel.com/docs/5.2/authentication)! [Stripe integration](https://laravel.com/docs/5.2/billing)! [Blade templating](https://laravel.com/docs/5.2/blade)!) but I'll stop here for now. Needless to say, Laravel has made writing robust web applications fun again. If you've found writing PHP applications tedious and repetitive, give Laravel a chance on your next application.
