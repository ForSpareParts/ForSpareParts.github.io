---
layout: post
title: "Messaging in Unity, pt. 2 (or, PubSub from scratch)"
date: 2015-05-22 17:00:00
tags: unity messaging pubsub ld32
---

In my [previous post](/2015/05/02/messaging-in-unity/) about using UnityEvents
to communicate between Unity classes, I mentioned a problem with using events:
the event objects need to be easily accessible from different classes. If you
hang an event off of the class that uses it most heavily (say, putting a "fire"
event on a Cannon), then you haven't really solved the tight-coupling problem at
all: you still need to find a Cannon whose fire event you want to trigger (or
subscribe to). At that point, you might as well just call the method.

Other frameworks and programming models have solved this problem by introducing
a message "bus" -- a central hub through which messages travel. By making the
message bus widely accessible, you create a single (very, very simple) point of
contact that disparate pieces of code can use to **publish** and **subscribe
to** events. I decided to knock together a naive message bus implementation of
my own. And given my [caffeine-addled, sleep-deprived state]
(http://ludumdare.com/compo/), it actually turned out pretty well.

<!-- more -->

Before I begin, a disclaimer (similar to the one in the last post). I'm not a C#
expert, or a Unity expert. This is something I figured out that seems to work
pretty well, and I'm offering it in the hopes it may be useful -- or that
somebody else might see it and suggest an even better alternative.


A simple PubSub system
----------------------

The way I wanted to use messages/events was, conceptually, pretty similar to
Unity's SendMessage: a GameObject serves as a "namespace" through which
attached Components can send messages to each other. But SendMessage:

* is slow
* is not type safe
* can only take a single argument
* by default, complains when a message isn't received by anything.

So what I wanted was "something like SendMessage, but not terrible." It occurred
to me that a simple component could serve as a message bus by simply holding on
to a bunch of UnityEvent objects and making them accessible to other code.
Implementing that concept is very straightforward.

First, let's declare some strongly-typed events:

{% gist ae318c1a534857f1d1ae Events.cs %}

As it turns out, you *have* to do this: UnityEvent is abstract, so you can't
just do `new UnityEvent<int>()`. Instead, you have to subclass UnityEvent into
something concrete -- fortunately, as Events.cs hopefully demonstrates, that
task is trivial.

Next, let's make a message bus:

{% gist ae318c1a534857f1d1ae BehaviourMessageBus.cs %}

Now, any component that's attached to the same object as a BehaviourMessageBus
can find it with GetComponent, then use AddListener and Invoke to communicate
with other components. Not a bad start, but it requires us to manually place a
BehaviourMessageBus component on every single prefab in our game that uses
messaging, and then look it up. We can do better.

{% gist ae318c1a534857f1d1ae BaseBehaviour.cs %}

BaseBehaviour serves as the base class for (almost) every other component in our
project. Note the Awake() method: now, every single component that subclasses
BaseBehaviour is (at least) guaranteed to have access to a message bus for its
own object. If a message bus doesn't already exist, one gets created.

Let's see the impact of this on a health/damage implementation:

{% gist ae318c1a534857f1d1ae Health.cs %}

The only difference between this and a more traditional implementation of a
health system is that, instead of a public method to deal damage, we implement
that logic as a listener on an event. How do we call it? Here's a simple
collision handler for a bullet:

{% gist ae318c1a534857f1d1ae BulletController.cs %}

This is no harder than calling, say, `.Damage(int)` on a non event-based health
class. But it has one big advantage: you don't need to know that the object
*can* take damage to deal damage to it. After all, the object you collide with
might have a MessageBus but no Health class. Maybe this object can't be damaged,
but should react somehow when somebody tries to damage it.

With this implementation, that's no problem: the bullet just
sends the damage event without worrying about the results. After that,
everything's handled (or not handled) by event listeners, and the bullet
doesn't need to think about it.

Of course, while we don't have to know whether or not we collided with an object
that can take damage, we *do* have to know whether that object has a message
bus. That's a better problem to have, I think (you know less about the colliding
object this way), but it's still not quite ideal.

If we were designing this engine from the ground up, we could write GameObjects
to **always** have a message bus. Because we're using Unity, we can't do that...

...but maybe we can fake it. [Extension methods](https://msdn.microsoft.com/en-us/library/bb383977.aspx)
to the rescue:

{% gist ae318c1a534857f1d1ae ExtensionMethods.cs %}

Now, we have an easy way to get a message bus for any GameObject (the second
extension lets us call that method on a component as a convenience, but it's
really just passing the call along to the GameObject). With that change, the
collision handler above becomes dead simple:

{% gist ae318c1a534857f1d1ae BulletControllerSimplified.cs %}

This really gets the idea across: we hit something, we deal damage to it.
Whether it subtracts the damage from its health, uses it to create a particle
effect, applies damage reduction -- we don't care. It's out of our hands at that
point. That, in turn, should make it really easy to think about what bullets do
(they deal damage) and what Health does (it reacts to damage). Health and
bullets don't know anything about each other, and they shouldn't have to. The
concept of damage is now the only thing they have in common.

This message bus implementation takes about a hundred lines of code, and so far,
it's been a huge help in writing concise, discreet classes for my project. I
also believe it'll be extremely useful for making Unity classes easier to unit
test, but that's another post.

Thoughts? Criticisms? I'd be very interested in any input. If you're interested
in seeing the project that inspired these posts, it's [on GitHub](https://github.com/ForSpareParts/EllDeeThirtyTwo) and can be played online [right here](http://commondatastorage.googleapis.com/itchio/html/53025/FINAL%20WEBGL%20BUILD/index.html)
