---
layout: post
title:  "Django vs. ember-cli (and --proxy)"
date:   2015-01-25 18:57:00
tags: ember django proxy
---

Our new application at work is an Ember frontend backed by Django. To facilitate
communication between Django and Ember, we're using [rest_framework_ember](https://github.com/ngenworks/rest_framework_ember),
which coerces the default JSON format of Django REST Framework into the JSON
format expected by Ember. Early in development, I tried to serve my Ember app
while proxying API calls to Django, using:

    ember serve --proxy http://localhost:8000

and was surprised to get an error message (in Chrome) that looked like this:

    XMLHttpRequest cannot load http://localhost:8000/api/users/. No 'Access-Control-Allow-Origin'
    header is present on the requested resource. Origin 'http://localhost:4200'
    is therefore not allowed access.

The problem turns out to have to do with trailing slashes, but is a little more
subtle than it appears.

<!-- more -->

Django, by default, expects resources to use a trailing slash, while Ember Data
expects them not to. By itself, this isn't a terrible problem: Django will send
a 301 (PERMANENTLY REDIRECTED) code with a response to any request lacking a
trailing slash, pointing it in the direction of the real resource. Ember, being
a good netizen, reacts to the 301 as you'd expect -- it makes a new request to
the resource indicated by the 301.

Which is great, except if you're running with --proxy.

When Django 301s a request, it responds with a *fully qualified URL* that the
client should direct further requests to. So a request for ``/api/users`` will
be redirected to ``http://localhost:8000/api/users/``. ember-cli's API proxy mode
is not quite smart enough to modify the 301 to point at itself. So from the
client's perspective, it makes a request to ``http://localhost:4200/api/users``
and is told to go look at ``http://localhost:8000/api/users/`` instead. At which
point, cross-site-scripting attack prevention does its job, and shuts the whole
thing down.

So, what can we do about it? That depends on your Django project. The simple
solution is to set ``APPEND_TRAILING_SLASH = False`` in ``settings.py``, which
will disable the redirect behavior. This only works, however, if your URL patterns
will still match without the slash at the end. In other words, make sure you're
using

    url(r'^users$', ...)

and NOT

    url(r'^users/$', ...)

The second pattern needs to end a slash, or it won't match -- which means that
requests with ``APPEND_TRAILING_SLASH = False`` will just 404.

If you're using Django REST Framework with a Router, you'll need to tell DRF
(which generates URLs via the Router) *not* to use trailing slashes, i.e.:

    router = DefaultRouter(trailing_slashes=False)

Credit where it's due: rest_framework_ember's README *does* warn you that you'll
need to do this to get around Django's 301 behavior, but it doesn't mention that
a 301 might actually break your Ember app in development. That hadn't occurred to
me, so, here's an extra warning.
