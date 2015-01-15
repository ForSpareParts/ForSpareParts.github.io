---
layout: post
title:  "Promise, but verify"
date:   2015-01-09 13:03:00
tags: ember testing async promises
---

I've been writing tests for my Ember app recently, and had a bit of trouble
figuring out how to test asynchronous behavior in my app. Cory Forsyth's [Demystifying Ember Async Testing](http://coryforsyth.com/2014/07/10/demystifing-ember-async-testing/)
proved very helpful in understanding how Ember's built-in async test helpers
work, but it stops just short of telling you how to hook into that behavior
yourself. The following is my attempt to (very briefly) answer the question:

In Ember, how do I write a test that depends on the resolution of a Promise?
<!-- more -->

For the impatient, here's the **tl;dr**:

In any test:

    Ember.run(function() {
      promiseReturningFunction():
    });

    wait();

    andThen(function() {
      //the result of promiseReturningFunction() has now resolved
    })

Why/how does this work? As Cory explains in his article, Ember maintains a chain
of promises throughout the whole test -- which powers its standard helpers like
`click()`, and `fillIn()`. In order for a promise to be resolved, though, it has
to get executed inside of the Ember run loop. Normally that loop's going all the
time, but it's disabled in testing mode to speed up the tests. So, we:

1. Wrap our promise inside of `Ember.run()`, which executes a single 'tick' of
the run loop. That allows it to be resolved by Ember's Promise implementation.

2. Call `wait()`, which prevents a test from finishing until all promises have
been resolved. Ember's standard async test helpers all call `wait()` themselves.

We can make this a little cleaner by building Ember test helpers for common use
cases. Here's one I made to wrap *chai-as-promised*'s `isRejected` assertion:

tests/helpers/is-rejected.js:

    import Ember from 'ember';

    export default Ember.Test.registerAsyncHelper('isRejected',
      function(app, promiseReturningFunction) {
        Ember.run(function() {
          assert.isRejected(promiseReturningFunction());
        });

        return wait();
      });

tests/unit/models/connection.js:

    it('refuses to be created or deleted', function() {
      
      //create:
      var store = this.store();

      isRejected(function() {
        return store.createRecord('connection').save();
      });


      //delete:
      isRejected(function() {
        return store.find('connection', 1)
        .then(function(connection){
          return connection.destroyRecord();
        });
      });
    });

I was honestly a bit concerned about the implicit promise chain in Ember tests
at first -- it seems like dangerously obtuse behavior -- but I've tried writing
tests that don't depend on it, and they're miserably verbose. Fortunately, (like
most of Ember) the magic behavior is actually pretty straightforward once you
understand how it works.
