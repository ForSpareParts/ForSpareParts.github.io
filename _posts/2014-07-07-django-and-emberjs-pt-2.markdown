---
layout: post
title:  "Django and Ember.js, pt. 2"
date:   2014-07-07 00:56:50
tags: django ember python
---

In my [previous post](http://forspareparts.svbtle.com/django-and-emberjs-pt-1), I covered setting up a Django project that would correctly serve an Ember-based frontend built with ember-cli. Here, I'm going to set up some Django models and wire them to Ember using Ember Data.

There are a few possible approaches here. Ember Data is highly customizable, so if we had an existing API it would be possible to create an Ember Data adapter to communicate with it. This is a fresh project, though, and since Django is far less opinionated than Ember, I'd prefer to create a Django API that conforms as closely as possible to Ember Data's default API spec.

I'm going to use [django-rest-framework](http://www.django-rest-framework.org/) to build my API. If you're unfamiliar with django-rest-framework, you'll probably want to at least check out their [quickstart guide](http://www.django-rest-framework.org/tutorial/quickstart) to give the rest of this post some context. Details on the JSON formatting that Ember Data expects can be found [in the Ember Data docs](http://emberjs.com/guides/models/connecting-to-an-http-server/).

So here's a sample models.py, defining a Widget model:

{% highlight python %}

from django.contrib.auth.models import User
from django.db import models

class Widget(models.Model):
    name = models.TextField()
    is_awesome = models.BooleanField(default=True)

    user = models.ForeignKey(User)

{% endhighlight %}

Some django-rest-framework boilerplate to serialize it (serializers.py)...

{% highlight python %}

from django.contrib.auth.models import User

from rest_framework import serializers

from backend.base.models import Widget

class WidgetSerializer(serializers.ModelSerializer):
    class Meta:
        model = Widget
        fields = ('id', 'name', 'is_awesome', 'user')

class UserSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = ('id', 'username', 'email')

{% endhighlight %}

... and serve it (views.py):

{% highlight python %}

from django.contrib.auth.models import User

from backend.base.models import Widget
from backend.base.serializers import UserSerializer, WidgetSerializer
from backend.django_ember_utils.views import EmberModelViewSet


class WidgetViewSet(EmberModelViewSet):
    queryset = Widget.objects.all()
    serializer_class = WidgetSerializer

class UserViewSet(EmberModelViewSet):
    queryset = User.objects.all()
    serializer_class = UserSerializer

{% endhighlight %}

And finally, a urls file to expose the ViewSets:

{% highlight python %}

from django.conf.urls import patterns, url, include
from rest_framework import routers
from backend.base import views

router = routers.DefaultRouter()
router.register(r'users', views.UserViewSet)
router.register(r'widgets', views.WidgetViewSet)

# Wire up our API using automatic URL routing.
# Additionally, we include login URLs for the browseable API.
urlpatterns = patterns('',
    url(r'^', include(router.urls)),
    url(r'^api-auth/', include('rest_framework.urls', namespace='rest_framework'))
)

{% endhighlight %}


One makemigrations and migrate later, we've got a working API. Here's our /widgets:

{% highlight json %}

[
    {
        "id": 1,
        "name": "some widget",
        "is_awesome": true,
        "user": 1},
    {
        "id": 2,
        "name": "old widget",
        "is_awesome": false,
        "user": 1
    }
]

{% endhighlight %}

A good start, but it doesn't match the API spec. So, to fix that, here's a little subclass of django-rest-framework's ModelViewSet:

{% highlight python %}

from rest_framework import viewsets

class EmberModelViewSet(viewsets.ModelViewSet):
    '''As ModelViewSet, but wraps data with the name of the model. That is, for
    a model called Widget:

    {
        'widget': {
            'some_field': 3
        }
    }

    or, for a list:

    {
        'widgets': [
            {
                'some_field: 3
            },
            {
                'some_field': 4
            }]
    }

    The plural name can be set explicitly by giving a value for
    model_plural_name. Otherwise, it will be self.get_model_name() + 's'.
    '''

    model_plural_name = None

    def get_model(self):
        '''Get the Django Model with which this ViewSet is associated. If the
        model is set directly, we use that, otherwise we derive it from the
        queryset.'''

        if self.model:
            return self.model

        return self.queryset.model

    def get_model_name(self):
        '''Get the lower-case, underscored singular form of the Django model's
        name.'''
        return self.get_model()._meta.model_name

    def get_model_plural_name(self):
        if self.__class__.model_plural_name:
            return self.__class__.model_plural_name

        return self.get_model_name() + 's'


    #for outgoing requests, we run the super method first, then modify the data
    #before we send it out
    def list(self, request, *args, **kwargs):
        response = super(EmberModelViewSet, self).list(
            request, *args, **kwargs)

        model_plural_name = self.get_model_plural_name()
        response.data = {
            model_plural_name: response.data
        }

        return response

    def retrieve(self, request, *args, **kwargs):
        response = super(EmberModelViewSet, self).retrieve(
            request, *args, **kwargs)

        model_name = self.get_model_name()
        response.data = {
            model_name: response.data
        }

        return response


    #for incoming requests, it's the opposite: strip the wrapper off of the data
    #on the request, then pass it along to the super method.
    def create(self, request, *args, **kwargs):
        model_name = self.get_model_name()
        request._data = request.DATA[model_name]

        return super(EmberModelViewSet, self).create(
            request, *args, **kwargs)

    def update(self, request, *args, **kwargs):
        model_name = self.get_model_name()
        request._data = request.DATA[model_name]

        return super(EmberModelViewSet, self).update(
            request, *args, **kwargs)


    def destroy(self, request, *args, **kwargs):
        model_name = self.get_model_name()
        request._data = request.DATA[model_name]

        return super(EmberModelViewSet, self).destroy(
            request, *args, **kwargs)

{% endhighlight %}

EmberModelViewSet acts just like ModelViewSet, except that it sends and receives data that matches Ember Data's API spec. So, we just stash that class somewhere (I've made a new app, django_ember_utils), and change our views.py file like so:

{% highlight python %}

from django.contrib.auth.models import User

from backend.base.models import Widget
from backend.base.serializers import UserSerializer, WidgetSerializer
from backend.django_ember_utils.views import EmberModelViewSet


class WidgetViewSet(EmberModelViewSet):
    queryset = Widget.objects.all()
    serializer_class = WidgetSerializer

class UserViewSet(EmberModelViewSet):
    queryset = User.objects.all()
    serializer_class = UserSerializer

{% endhighlight %}

Now, if we make the same request, we'll get:

{% highlight json %}

{
    "widgets": [
        {
            "id": 1,
            "name": "some widget",
            "is_awesome": true,
            "user": 1
        },
        {
            "id": 2,
            "name": "old widget",
            "is_awesome": false,
            "user": 1
        }
    ]
}

{% endhighlight %}

Perfect. Now, we can wire the Widget model into Ember using the default adapter.

Well, almost.

It turns out there are still two problems with this approach, both fixable. The first is that Ember Data does not, by default, append a slash to its API paths, whereas Django prefers trailing slashes on all URLs. It's usually trivial to address this problem in Django -- just set `APPEND_SLASH = True`, and Django redirects users to the correct path. But Ember Data considers any redirect a failed reqest, so we'll override the adapter, very lightly, to deal with that. This goes in frontend/adapters/application.js:

{% highlight javascript %}

import Ember from 'ember';
import DS from 'ember-data';

export default DS.RESTAdapter.extend({
  buildURL: function(type, id) {
    var url = this._super(type, id);

    //make sure we have a trailing slash!
    if (url.slice(-1) !== '/') {
      url = url + '/';
    }

    return url;
  },

  pathForType: function(type) {
    var originalPath = this._super(type);
    return Ember.String.dasherize(originalPath);
  }
});

{% endhighlight %}

Easy enough. The second problem is that having the API and the pages share the same paths (i.e. '/posts' takes you to a page if you request HTML, or the API if you request JSON) triggers weird caching behavior in Chrome. In certain circumstances, a user going 'back' or 'forward' sees a blank page instead of the site itself.

To my frustration, after several hours of grappling with this problem, I never figured out *why* it happens. I did, however, discover that it's easily remedied by giving your API a separate path on the server. Just add `namespace: 'api'` to the adapter, and use `url(r'^api/', include(router.urls))` to register your API routes with Django. Incidentally, if anyone can shed more light on the issue, please let me know.

So, as before, Ember and Django butt heads a bit -- but you can make them play nice with a little effort. If I learn anything else interesting, I'll update this post or create another one.
