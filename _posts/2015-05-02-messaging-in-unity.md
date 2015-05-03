---
layout: post
title:  "Promise, but verify"
date:   2015-05-02 20:23:00
tags: unity messaging ld32
---

Messaging in Unity (without SendMessage)
========================================

[Ludum Dare](http://ludumdare.com/compo/) is a 48-hour solo game jam. Naturally, that means quick, dirty work. It is absolutely, positively, not the right time to be looking for elegant solutions to vexing technical design problems that could be solved temporarily with a handwave. You'd have to be absolutely insane to confront architecture woes on a two-day deadline. Very bad idea.

You can probably tell where this is going.

<!-- more -->

Suffice it to say, I'm stubborn and I hate Bad Code. I've been doing [a lot of Unity work lately](http://www.got-rekt.com), and I had decided prior to Ludum Dare that I'd take any opportunity -- even a *bad* opportunity -- to work on improving my Unity project organization. And improbably, I actually worked at least one thing out.

It may be that everybody else was already doing this and I'm late to the party. Or it may be that it's a terrible idea, and I just haven't realized it yet. But so far, it looks like a decent way to hack around some of Unity's architecture limitations without working too hard. So, without further ado:

The Problem
-----------

In Unity, game code is generally organized into Components which can be attached to GameObjects in the editor. Unity's architecture (and its documentation) unfortunately encourage tight coupling between game components by encouraging components to look each other up by class, and then refer to each other directly (e.g., a player controller class might look up a cannon on the player object, and then call a method on the cannon to fire it).

That's workable in a small project, but quickly becomes unweildly as you scale up. Some components might trigger behavior with far reaching implications, causing other components to change state, spawn new objects, perform some work over time, and so forth. That requires all the components involved to have references to each other, and if code in one component changes, it usually motivates changes in the others.

The more you have this problem, the less benefit there seems to be in separating things out. If the cannon can't fire without the controller, and the controller always expects a cannon to be there, you might as well make them the same component, right?

A couple of 1000-line-plus kitchen sink "controller" classes later, it's obvious that this was a terrible mistake. You *do* want things separated -- but it turns out that looking up other components with `GetComponent()` isn't separated *enough*. You need components that know way less about each other, so they can each do their job and get on with life.


The Solution
------------

One way to solve this problem (or at least, some of it) is with some kind of messaging system. If you're not familiar with this approach, the basic idea is to organize your code around the idea of sending messages between parts of the program, rather than calling methods. An event says that "something happened," and that's it -- other parts of the program might be listening for that event and do something in response, or they might not. Either way, the code that raised the event doesn't care: once the event is sent off, its job is done. It doesn't know or care what happened as a result (if it needs to know, it'll probably get notified in turn by another event).

You may be aware that Unity includes a messaging system of sorts, by way of the `GameObject.SendMessage()` method. SendMessage lets you specify a method name and call the named method with a single argument on any components that contain it on another GameObject. That's better than a direct method call, but not by much -- its singular argument is untyped and it runs the risk of triggering unwanted behavior (if, say, you called SendMessage("DoStuff") but forgot that you have components with a DoStuff() method that's not related to this message and which you don't want called).

There really ought to be a better way, and I think I've found it. More recent versions of Unity include an UnityEvent class that you can use to build custom messages that take up to four strongly-typed arguments. UnityEvents are also bound explicitly to whichever methods they'll trigger. Detailed information is in the [documentation](http://docs.unity3d.com/ScriptReference/Events.UnityEvent.html), but in brief, usage looks like this:

    public class SomeEvent : UnityEvent<int, string> { }

    //later, in another class...

    SomeEvent evnt = new SomeEvent();

    void SetupListeners()
    {
        evnt.AddListener(SomeHandler);
    }

    void SomeHandler(int intArg, string stringArg)
    {
        Debug.Log("The int value is: " + intArg.ToString());
        Debug.Log("The string value is: " + stringArg);
    }

SetupListeners() "subscribes" (or "binds") SomeHandler to evnt. If you were to first call SetupListeners and then do this...

    evnt.Invoke(42, "hello!");

you would see two logging messages: `The int value is: 42` and `The string value is: hello!`. When you call Invoke(), the event calls *every method subscribed to it* using Invoke's arguments. Take note of two things: First, there could be any number of methods subscribed to evnt, or none at all. Second, and just as important: the code that calls Invoke doesn't need to know about them.

Hopefully, this is already suggesting some interesting possibilities to you. When I learned about it, my first reaction was to wonder how different classes would share event objects to subscribe to and invoke. I've got a solution to that problem that I really like -- but this post is getting rather long, so I'll leave it for next time.
