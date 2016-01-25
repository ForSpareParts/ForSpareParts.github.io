---
layout: post
title:  "Actions Up, Data Sideways"
date:   2016-01-24 21:36:00
tags: ember architecture components routes react relay
---

The rallying cry for Ember 2.0 has been "Data Down, Actions Up," indicating a
pattern wherein parents (routes/controllers, or high-level components) pass
data "down" to child components, while children "request" changes from their
parents by triggering actions. Doing things this way keeps parents from
worrying about what their children do with data[^1], and children form worrying
about how data is represented in their parents[^2].

Expressed another way, DDAU is about keeping a nice, clean, abstractable API
between parents and children. And that's great, so long as that API stays
simple, but, well...

<!-- more -->

Let's invent a case study.

A Case Study
------------

You are building a data dashboard. Your users need to see what their users do
all day, so they want a bunch of tables and graphs representing how people
spend time using their product. As a programmer, your inclination would
naturally be to model each of these data representations as a separate route.
Your users balk at this. They need to "compare data" and "use space
efficiently." They want the app to be "intuitive" and "usable." Philistines.

Well, *fine*. You're smart, you'll figure it out. `RSVP.hash` is a thing:

{% highlight js %}
//routes/dashboard.js
import Ember from 'ember';
import DS from 'ember-data';

export default Ember.Route.extend({
  model(params) {
    return Ember.RSVP.hash({
      aDataset: this.store.findAll('some-model'),
      anotherDataset: this.store.findAll('another-model'),
      moarData: this.store.findAll('return-of-model'),
      stillMore: this.store.findAll('revenge-of-model'),
      yepEvenMore: this.store.findAll('son-of-model')
    })
  }
});
{% endhighlight %}

{% highlight handlebars %}
<!-- templates/dashboard.hbs -->
{% raw %}
{{some-chart data=model.aDataset}}
{{another-table data=model.anotherDataset}}
{{moar-table data=model.moarData}}
{{still-more-chart data=model.stillMore}}
{{yep-even-more-table data=model.yepEvenMore}}
{% endraw %}
{% endhighlight %}

Your components can just refer to data. Everything's fine. Until somebody
points out that `still-more-chart` needs to be able to configure what data it
receives from the server. So...

{% highlight js %}
//routes/dashboard.js
import Ember from 'ember';
import DS from 'ember-data';

export default Ember.Route.extend({
  model(params) {
    //imagine that the controller defines a `stillMoreDataSize` query
    //parameter that's set by the `setStillMoreQuery` action
    let smQuery = {size: params.stillMoreDataSize}
    return Ember.RSVP.hash({
      aDataset: this.store.findAll('some-model'),
      anotherDataset: this.store.findAll('another-model'),
      moarData: this.store.findAll('return-of-model'),
      stillMore: this.store.query('revenge-of-model', smQuery),
      yepEvenMore: this.store.findAll('son-of-model')
    })
  }
});
{% endhighlight %}

{% highlight handlebars %}
<!-- templates/dashboard.hbs -->
{% raw %}
{{some-chart data=model.aDataset}}
{{another-table data=model.anotherDataset}}
{{moar-table data=model.moarData}}
{{still-more-chart data=model.stillMore setQuery=(action 'setStillMoreQuery')}}
{{yep-even-more-table data=model.yepEvenMore}}
{% endraw %}
{% endhighlight %}

Awesome. You can do that for `some-chart`, too, right? But with, like, three
parameters?

{% highlight js %}
//routes/dashboard.js
import Ember from 'ember';
import DS from 'ember-data';

export default Ember.Route.extend({
  model(params) {
    //imagine that the controller defines a `stillMoreDataSize` query
    //parameter that's set by the `setStillMoreQuery` action
    let smQuery = {size: params.stillMoreDataSize}
    let aDatasetQuery = {
      sortBy: params.aDSort,
      filterOn: params.aDFilter,
      limit: params.aDLimit
    };

    return Ember.RSVP.hash({
      aDataset: this.store.query('some-model', aDatasetQuery),
      anotherDataset: this.store.findAll('another-model'),
      moarData: this.store.findAll('return-of-model'),
      stillMore: this.store.query('revenge-of-model', smQuery),
      yepEvenMore: this.store.findAll('son-of-model')
    })
  }
});
{% endhighlight %}

{% highlight handlebars %}
<!-- templates/dashboard.hbs -->
{% raw %}
{{some-chart data=model.aDataset
  setSort=(action 'setADSort')
  setFilter=(action 'setADFilter')
  setLimit=(action 'setADLimit')}}
{{another-table data=model.anotherDataset}}
{{moar-table data=model.moarData}}
{{still-more-chart data=model.stillMore setQuery=(action 'setStillMoreQuery')}}
{{yep-even-more-table data=model.yepEvenMore}}
{% endraw %}
{% endhighlight %}

OK, but now we need to wrap those first two charts in another component.

{% highlight js %}
//routes/dashboard.js
import Ember from 'ember';
import DS from 'ember-data';

export default Ember.Route.extend({
  model(params) {
    //imagine that the controller defines a `stillMoreDataSize` query
    //parameter that's set by the `setStillMoreQuery` action
    let smQuery = {size: params.stillMoreDataSize}
    let aDatasetQuery = {
      sortBy: params.aDSort,
      filterOn: params.aDFilter,
      limit: params.aDLimit
    };

    return Ember.RSVP.hash({
      aDataset: this.store.query('some-model', aDatasetQuery),
      anotherDataset: this.store.findAll('another-model'),
      moarData: this.store.findAll('return-of-model'),
      stillMore: this.store.query('revenge-of-model', smQuery),
      yepEvenMore: this.store.findAll('son-of-model')
    })
  }
});
{% endhighlight %}

{% highlight handlebars %}
<!-- templates/dashboard.hbs -->
{% raw %}
{{grouping-component
  data=model
  setSort=(action 'setADSort')
  setFilter=(action 'setADFilter')
  setLimit=(action 'setADLimit')}}

{{moar-table data=model.moarData}}
{{still-more-chart data=model.stillMore setQuery=(action 'setStillMoreQuery')}}
{{yep-even-more-table data=model.yepEvenMore}}

<!-- templates/components/grouping-component.hbs -->
{{some-chart data=model.aDataset
  setSort=setSort
  setFilter=setFilter
  setLimit=setLimit}}
{{another-table data=model.anotherDataset}}
{% endraw %}
{% endhighlight %}

But, then-

{% highlight js %}
//routes/dashboard.js
import Ember from 'ember';
import DS from 'ember-data';

export default Ember.Route.extend({
  model(params) {
    //imagine that the controller defines a `stillMoreDataSize` query
    //parameter that's set by the `setStillMoreQuery` action
    let smQuery = {size: params.stillMoreDataSize}
    let aDatasetQuery = {
      sortBy: params.aDSort,
      filterOn: params.aDFilter,
      limit: params.aDLimit
    };

    let anotherQuery;
    if (params.anotherThing) {
      anotherQuery = {
        filterOn: 'foo'
      }
    }
    else {
      anotherQuery = {
        filterOn: 'bar'
      }
    }

    return Ember.RSVP.hash({
      aDataset: this.store.query('some-model', aDatasetQuery),
      anotherDataset: this.store.query('another-model', anotherQuery),
      moarData: this.store.findAll('return-of-model'),
      stillMore: this.store.query('revenge-of-model', smQuery),
      yepEvenMore: this.store.findAll('son-of-model')
    })
  }
});
{% endhighlight %}

{% highlight handlebars %}
<!-- templates/dashboard.hbs -->
{% raw %}
{{grouping-component
  data=model
  setSort=(action 'setADSort')
  setFilter=(action 'setADFilter')
  setLimit=(action 'setADLimit')
  setAnotherThing=(action 'setAnotherThing')}}

{{moar-table data=model.moarData}}
{{still-more-chart data=model.stillMore setQuery=(action 'setStillMoreQuery')}}
{{yep-even-more-table data=model.yepEvenMore}}

<!-- templates/components/grouping-component.hbs -->
{{some-chart data=model.aDataset
  setSort=setSort
  setFilter=setFilter
  setLimit=setLimit}}
{{another-table data=model.anotherDataset
  setAnotherThing=setAnotherThing}}
{% endraw %}
{% endhighlight %}

Good job, and now can you just-

But you can't. You absolutely can't, there's just no way, because your files
aren't the ten-line examples above, they're real code and they've turned
into something out of an H.P. Lovecraft story. Your components are nested
five levels deep, and they take a lot of parameters, mostly actions bound for
lower components. Your model hook doesn't fit on a page anymore. The actions
hash in your controller is way the hell out of control. And God help you when
you try to write tests.


A Possible Solution
-------------------

Where did we start, again? We wanted to keep parents and children out of each
others' business; to let them each do their thing and talk along a few clean
lines of abstraction. How did we end up in this tightly-coupled mess?

I'd argue that it was by overdoing DDAU. Where the DDAU pattern works well,
the parent provides the child with some direction, and leaves implementation
details to the child's discretion. Similarly, the child doesn't trouble the
parent with actions unless something's happened that is important to the
parent.

The architecture above isn't like that. The child can't do anything without
the parent's cooperation, and most of the parent's code is dedicated to
accomodating the child. But there is an answer. We can break the letter of
the law to preserve its spirit. We can fetch data from the component itself.

Doing so is strong medicine, and we should probably apply it with caution. Not
every component should fetch its own data, and those that do may still need
other data from their parents. But there is value in the idea of a component
that can talk to the server, particularly where it preserves the idea of
components being self-sufficient.

So, when and how should components fetch their own data? I'd like to suggest
that Ember has, perhaps inadvertently, answered this question for us. Consider
routes, which:

* can be nested inside of another visual context,
* are considered the visual representation of some particular piece of data
  (their `model`),
* are responsible for fetching that data themselves, and, as a bonus,
* are aware of their own loading state

This is a good pattern. Routes are self-contained, easy to write, and easy to
reason about. I think that when components really need to fetch their own
data, they tend to feel like routes. And if we're smart about it, they can
look and act like routes, too. Almost.

Our components shouldn't affect our URLs, but the rest of the analogy holds.
They retrieve some piece of data, and they represent it. Until the data comes
back, they think of themselves as "loading."

Here's a slightly modified version of what we came up with: a mixin that gives
asynchronous data-loading behavior to a component. It's pretty short:

{% highlight js %}
//mixins/components/async-data.js
export default Ember.Mixin.create({
  //this is injected as a convenience, since subclasses will probably need it
  //to implement `dataPromise`
  store: Ember.inject.service('store'),

  /**
   * Return a Promise that resolves to the data you need for your component.
   *
   * This is usually a computed property -- you may need new data when other
   * component properties change.
   *
   * Until this promise resolves, `isLoading` will be `true`.
   */
  dataPromise: undefined,
  dataLoadError: null,

  dataPromiseObserver: Ember.observer('dataPromise', function() {
    let resolvingPromise = this.get('dataPromise');
    this.set('dataLoadError', null);

    if (!resolvingPromise) {
      this.set('isLoading', false);
      return;
    }

    this.set('isLoading', true);
    resolvingPromise
      .then((data) => {
        let currentPromise = this.get('dataPromise');

        //the 'dataPromise' property might have changed since we chained this
        //then() call, so let's make sure this data is still what we want:
        if (resolvingPromise === currentPromise) {
          this.set('data', data);
          this.set('isLoading', false);
        }})
      .catch((err) => {
        this.set('dataLoadError', err);
        this.set('isLoading', false);
      });
  }).on('init'),

});
{% endhighlight %}

Our `AsyncDataComponentMixin` has the following public API:

 * `store`: the injected Ember Data Store
 * `dataPromise`: implement to retrieve your data -- it should return a Promise.
 * `data`: when your data resolves, it will be here
 * `isLoading`: true until your data resolves
 * `dataLoadError`: if dataPromise fails, its rejection reason (null otherwise)

So to use this mixin, we just implement `dataPromise`. When our data comes
back, the mixin will put it in the `data` property (which we can observe).
Until then, `isLoading` will be `true`, so we can display a nice loading state
to the user. In other words: `dataPromise` is like our `model()` hook, `data`
is like the controller's `model` property. And here's what it might look like
in action, cleaning up a bit of that horrible code from before...

{% highlight js %}
//components/another-table.js
import Ember;
import AsyncDataComponentMixin from 'our-app/mixins/components/async-data';

export default Ember.Component.extend({
  sortBy: 'value',
  filterOn: 'otherValue',
  limit: 10,

  dataPromise: Ember.computed('sortBy', 'filterOn', 'limit', function() {
    return this.get('store').query('another-model', {
      sortBy: this.get('sortBy')
      filterOn: this.get('filterOn'),
      limit: this.get('limit')
    });
  })
});
{% endhighlight %}

Our `another-table` component can set `sortBy`, `filterOn`, and `limit`
itself, without notifying its parent -- and since they don't affect anything
outside the component, we can feel good about that. When they change,
`dataPromise`, being a computed property, changes with them. When `isLoading`
is false, and `data` has resolved, the component can use `data` in its
template. Until then, if it's so inclined, it can display a loading state. In
this, we get a clear functional advantage over handling all this data at the
route level: each component can have its own, independent loading state,
whereas the route can have only one.

It's worth noting, too, that `dataPromise` (being a computed property) can
*also* depend on information that came from the parent. This lets the parent
control the component in a way that's more in line with the principles of
DDAU. Imagine that we want our component to only fetch data that's related to
the main model of the page: that model can be passed in from the parent, and
used in our `dataPromise` to build part of our query. We still don't burden
the parent with the child's data -- instead, the parent gives the component
just enough information to do its job, and gets out of the way. Just like it's
supposed to work.


Does This Actually Hold Up?
---------------------------

So far, signs point to yes. We built one of our app's most complicated pages
on this architecture, and it went smoothly. With the concept proven, we've
been using it extensively in a major refactor. Our app does a lot of
data-visualization, and accordingly we've found use for the data-fetching
component pattern in many places. At least for me, it consistently makes the
code involved easier to reason about.

It also echoes an upcoming pattern in Ember itself: that of the routable
component. Routable components will be components bound to one piece of (usually
asynchronous) data, just like the components described above. The big
difference is that our components fetch data themselves, rather than relying
on a route to do it. In both cases, the data and the component are
intrinsically linked.

Elsewhere in the frontend world, we can see similar ideas expressed by React's
Relay framework. Relay sticks data queries right next to components, just
like we do. In the [blog post announcing Relay and GraphQL](https://facebook.github.io/react/blog/2015/02/20/introducing-relay-and-graphql.html), Greg
Hurrell argues that colocating queries and UI code lets developers "reason
about what a component is doing by looking at it in isolation." He also
mentions that it allows components to move anywhere in the render hierarchy
without triggering a "cascade of modifications" to other code. On both counts,
I'm inclined to agree.

Along with our early successes, the Relay and routable components parallels
are giving me confidence we're on the right track here. And hopefully, some of
you reading this will also be inspired to give it a shot. If you do, please
let me know. I'm curious how it fits into (or doesn't fit into) what
everybody else is working on.


[^1]: Well, they'll worry anyway; that's what parents do. But at least this way, their concerns will be largely unjustified.

[^2]: In this metaphor, is an unexpected change to a bound value analogous to a parent dropping by a child's place unannounced?
