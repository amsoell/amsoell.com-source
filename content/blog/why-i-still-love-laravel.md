---
title: "Why I (still) Love Laravel"
date: 2016-11-07
description: "More ways Laravel makes my life easier and applications more awesome"
tags: [ "development", "backend", "php", "laravel" ]
---

Earlier this year I reviewed [some of the things that I love most about Laravel](../why-i-love-laravel), the incredibly popular PHP application framework. Features like the Eloquent database abstraction layer, built-in unit testing, and Git-friendly database schema tracking through Migrations are some of the fundamental things that make the choice to use Laravel something of a no-brainer. In the time since that post I have continued to leverage Laravel for client projects as well as some products of my own, and I wanted to take the time to talk about some more built-in features of Laravel that make applications fast, full-featured, and scalable.


## Queues

Laravel is more than capable of running a complete web application, but oftentimes I leverage Laravel as a backend engine and API alongside a complimentary front-end application built on a modern JavaScript framework like Vuejs or React. Given that Laravel's role in this equation is to be a super responsive workhorse, it's important that tasks that don't need to be run immediately be offloaded to a time when the resources are there to handle them. Laravel has a [powerful and flexible queue system](https://laravel.com/docs/queues) built right into it that helps accomplish exactly that.

A great example of this is sending email. Application-generated email is important, but given the choice between improved application responsiveness and a second or two delay in delivering email, it's an easy decision. I'm getting ready to ship a new product that generates _a lot_ of email, and I've made heavy use of queues to make sure the API stays responsive and that emails are only sent out when the system resources are there to handle it. Here's what the code looks like.

{{< highlight php >}}
Mail::queue(['mail.out.html', 'mail.out.plain'], $data, function ($mailable) use ($data) {
	// Add the appropriate headers
	$headers = $mailable->getSwiftMessage()->getHeaders();
	$headers->remove('Message-ID');
	$headers->addTextHeader('Message-ID', $data['message_id']);
	$headers->addTextHeader('Reply-To', $data['inbox']['email']);
	$headers->addTextHeader('Return-Path', $data['inbox']['email']);

	// Set the mail details
	$mailable
		->to($data['to'], $data['to_name'])
		->from($data['inbox']['email'], $data['from_name'])
		->subject($data['subject']);
}, 'outbound');
{{< / highlight >}}

The important lines here at the first and the last. If you were sending this email immediately, you would make use of the `Mail::send` method. In order to queue the message, all you need to do is instead use the `Mail::queue` method. It's just that easy. By doing that your email will be packaged up and passed along to your queued task manager, whether it's Beanstalkd, Redis, or a even a custom Laravel controller, and run when the system has the appropriate bandwidth to handle it.

It's also worth noting the last line, the optional fourth paramter to the `Mail::queue` method which is set to `outbound`. By default, queued tasks will be categorized into a "default" queue, and all tasks will be run in sequential order as the system has resources to handle them. You can, however, chunk your queued tasks into separate "buckets" and prioritize how they're processed. In one application, for example, there are four "buckets:" outbound, inbound, user-notify, and default. This ensures that customer-bound outgoing email is given the highest priority over processing incoming messages, user notifications, and other tasks.

But queues don't stop at mail; You can queue literally any task you like by wrapping it in a job and dispatching it to your queue. For example, in this upcoming application incoming email is received and run it through a controller called `ProcessInboundItem`. This does some important data processing, but it doesn't need to run the instant that the email is received, so instead of running it directly it is packaged up and sent to the queue for later processing. It looks like this.

{{< highlight php >}}
$job = (new ProcessInboundItem($request->all()))->onQueue('inbound');
$this->dispatch($job);
{{< / highlight >}}

That's all there is to it. The task is now in the `inbound` queue bucket and will be processed when they system is ready for it. That usually means a few seconds, but it's good to know that if the server is under heavy load these tasks can sit in the queue for minutes so the server can give all of its attention handling incoming API requests.

## Policies

Really, what makes Laravel such a joy to work with isn't _necessarily_ the features baked in, but it's the nuance and the way in which they're used. Take user permissions for example. Any developer worth their salt can code a multi-layered access control system without too much trouble. [Laravel's Policy system](https://laravel.com/docs/authorization) gives you a big head start on it, and actually makes adding policy rules to your controller readable. For example, take this snippet from an application that defined the policy for allowing a user to post a comment.

{{< highlight php >}}
// app/Policies/CommentPolicy.php

class CommentPolicy {
	use HandlesAuthorization;

	public function store(User $user, Comment $comment) {
		return $user->hasOrganization($comment->thread->inbox->organization);
	}

	...
}
{{< / highlight >}}

The `CommentPolicy::store` method takes in a User object and a Comment object and does a quick check to see if the user is allowed to post that comment. Pretty simple stuff. But the way this is tied in on the controller is what makes Laravel's Policies a pleasure to work with.

{{< highlight php >}}
// app/Http/Controllers/CommentController.php

if ($user->can('store', $comment)) {
	$comment->save();
} else {
    ...
}
{{< / highlight >}}

When you're reading through your code, it's _painfully_ obvious what these lines do. That's what so great about Laravel's Policies, and really nearly every aspect of the way the framework is designed.

## Scout

[Laravel Scout](https://laravel.com/docs/scout) is one of the brand new features that was introduced in Laravel 5.3, and it brings full-text searching directly into your Eloquent models. Through a series of drivers, you can now quickly and easily make any fields fully searchable without writing painful raw database queries. I'm using it to make email messages searchable. Here's what the model looks like.

{{< highlight php >}}
// app/Message.php

use Laravel\Scout\Searchable;

class Message extends BaseModel {
    use Searchable;

    public function toSearchableArray() {
        return [
    		'id'         => $this->id,
    		'from_name'  => $this->from_name,
    		'from_email' => $this->from_email,
    		'subject'    => $this->subject,
    		'body_plain' => $this->body_plain,
    		'created_at' => $this->created_at
        ];
    }

    ...
}
{{< / highlight >}}

Once you've [installed and configured your Scout driver](https://laravel.com/docs/scout#installation) there are only two things you need to do to make your models fully searchable. First, add the `Searchable` trait to your class, and then define which attributes should be indexed with the `toSearchableArray` method. That's it. Once that's done the [Laravel model observers](https://laravel.com/docs/eloquent#observers) will automatically keep your indices up-to-date by watching as your objects are created, updated and deleted. To actually perform the search, it's as simple as using the `search` method on the model.

{{< highlight php >}}
// app/Http/Controllers/MessageController.php

class MessageController extends Controller {

    public function index(Request $request) {
        if ($request->has('search')) {
            $messages = Message::search($request->get('search'))->get();
        } else {
            ...
        }
    }

    ...
}
{{< / highlight >}}

And, of course, the indexing process can be offloaded to your queues so you can be sure your application will stay super responsive.

## Notifications

Even five years ago, the idea of a "user notification" typically meant one thing: an email. Modern applications built today, though, need to think about notifying users where it's most convient for them. That might mean email, but it might also mean text messaging, or a post in a Slack channel. It could mean a Twitter mention, triggering a custom webhook, or firing off a desktop notification—the possibilities are quite limitless. So I'm absolutely in love with another feature introduced in Laravel 5.3, and that's the concept of [Notification abstraction](https://www.laravel.com/docs/notifications). Rather than hard-coding the logic of sending an email in one of your controllers, you can instead build up a general "notification" object and send it off. Continuing from the earlier example, here's how it notifies users when an inbound email has been received that needs their attention.

{{< highlight php >}}
// app/jobs/ProcessInboundItem.php
class ProcessInboundItem implements ShouldQueue {

    public function handle() {
        foreach ($this->item->thread->inbox->organization->users as $user) {
            $user->notify((new MessageReceived($this->item))->onQueue('user-notify'));
        }
    }

    ...
}
{{< / highlight >}}

You'll notice there isn't a single mention of an "email" here at all. We're merely telling the application that the specified users should be notified; It's up to the notification controller to figure out how that's done, whether it's based on a user preference setting or based on the content of the message. You can even hard-code it to only send emails, but by using Laravel Notifications you have flexilibity to expand it when you add more notification channels in the future.

(By the way, did you notice the `onQueue()` method tacked on after the `notify()` method? The ability to queue tasks for later processing are _everywhere_ in Laravel!)

## More coming all the time

As if Laravel weren't powerful enough, the developers behind the framework continue to work tirelessly to bring us new releases all the time. And these aren't just bug fixes or minor maintenance releases; Taylor and his team have always worked to add powerful new updates to make developing in Laravel faster, easier and more powerful. I'm especially interested in this update that was teased out last week.

{{<tweet 794539869393580040>}}

As with my last Laravel review, I've barely scratched the surface and there are so many more amazing features that make Laravel a dream to work with. Working with Laravel has helped us deliver incredibly powerful and fast applications, and I can't wait to see what the framework has in store for us in the future.
