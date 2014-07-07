---
layout: post
title:  "Django and Ember.js, pt. 1"
date:   2014-07-06 15:06:36
tags: django ember python
---

The Quick Version:
-------------------------
The steps to getting an Ember-friendly Django setup are:

1. Register your Ember build directory ('build' by default in ember-cli) in STATICFILES_DIRS, and remove 'django.contrib.staticfiles' from INSTALLED_APPS.
2. Create and register a middleware class that serves index.html for all html requests, and serves other static files from the site root. You can use the snippet below that begins with *class EmberStaticFilesMiddleware* for that.
3. Override Django's default 'runserver' management command inside of your project to rebuild files automatically with 'ember serve'. You can use the
snippet below that begins with *import os* for that.

Walkthrough and Rationale:
------------------------------------
At work, we're planning an ambitious new application. One of the requests we've heard from our stakeholders is to give this new app a slicker, more 'modern' feel -- and to that end, we've been investigating the best options for client-side web development.

So far, Ember looks like it's the best fit for our use case: it's a very robust, well-documented framework with a lot of mindshare, and it can work with any backend that provides a RESTful interface. In our case, we're looking to integrate Ember with Django, which has been our shop's primary technology so far.

There's a lot written about how to use Ember with Rails (not surprising, given Ember's origins in the Rails community), and with Node (up-and-coming Ember project tool ember-cli actually uses an Express server as its default backend). I've found relatively little about integrating Ember with Django, so I thought it'd be interesting to track our progress here.

For starters, here's our project stub (assuming you've already got pip and npm installed):

{% highlight bash %}

mkdir django-ember-demo
cd django-ember-demo

#the django part:
mkvirtualenv django-ember-demo
pip install https://www.djangoproject.com/download/1.7.b4/tarball/
django-admin startproject backend
pip freeze >> backend/requirements.txt    

#the Ember part:
npm install -g ember-cli
ember new frontend

{% endhighlight %}

So far, so good: that gives us this directory structure:

    django-ember-demo
        backend/
            backend/
                ...
            manage.py
        frontend/
            app/
            ...
            node_modules/

ember-cli will build our project for us, but we need to make sure that Django knows how to find it, and where to serve it. Because we're creating a single-page app, *all page requests* get the same HTML, CSS, and JS delivered -- only JSON requests actually get handled by Django. So, for a development environment, here's how we handle that:

In our settings.py:

{% highlight python %}

BASE_DIR = os.path.dirname(os.path.dirname(__file__))
DJANGO_ROOT = BASE_DIR
EMBER_ROOT = os.path.join(
    os.path.dirname(DJANGO_ROOT),
    'frontend')
EMBER_STATIC_ROOT = os.path.join(EMBER_ROOT, "build")

...

STATICFILES_DIRS = (
    EMBER_STATIC_ROOT,
)

{% endhighlight %}

Now Django knows that ember-cli's '/build' directory contains the static build of our app. We could build it by hand to a custom location with ember build --output-path dir_name, but this way Django will watch the same location where the project gets automatically built by ember serve.

Ordinarily, we'd use Django's 'staticfiles' app to serve our static content in development. Unfortunately, staticfiles expects all of the sites static files to be accessible under a specific path, like /static (i.e. /static/mypage.html, /static/styles.css). That's not going to work for our ember-cli project, which expects to find static files at the root (/mypage.html, /styles.css), and configuring the staticfiles app to serve from the root will break the rest of the app. We're going to need to build a smarter way to serve static files in development, calling out functionality from staticfiles by hand. Here's a middleware that I came up with to do just that:


{% highlight python %}

class EmberStaticFilesMiddleware(object):

    def process_request(self, request):

        if settings.DEBUG:
            accept_header = request.META.get('HTTP_ACCEPT')

            #JSON requests get passed on to the rest of the app
            if 'application/json' in accept_header:
                return None

            #HTML requests get the index.html page:        
            if 'html' in accept_header:
                return serve(
                    request,
                    'index.html')

            #otherwise, we'll try to serve a static file (this should 404 if the
            #file isn't found)
            return serve(
                request,
                request.path)

        return

{% endhighlight %}

We can put that in any module we like, according to our preferred project structure. Then in settings.py, we add it to MIDDLEWARE_CLASSES ('path.to.module.EmberStaticFilesMiddleware'), and remove 'django.contrib.staticfiles' from INSTALLED_APPS. After that, we can access index.html from anywhere on our site, just as we could with the Express server that ships with ember-cli.

The next step will be to make runserver invoke 'ember serve'. Ember serve is mostly intended to start a development server for your ember content, but it has the side effect of causing ember files to get built into the 'build' directory, every time they're changed. At present, ember-cli doesn't have a way to do this without starting the Express server, but that's OK: the Express server shouldn't use the same port as Django, so we won't even notice it's there. If future versions of ember-cli add a 'watch' function, we can easily switch to that later.

Here's an override for Django's runserver that can go in any app in your project, based on a similar snippet for grunt I found [here](http://lincolnloop.com/blog/simplifying-your-django-frontend-tasks-grunt/). I created an app called 'base' for shared functionality, so this went in 'backend/base/management/commands/runserver.py':

{% highlight python %}

import os
import subprocess
import atexit
import signal

from django.conf import settings
from django.core.management.commands.runserver import Command as DjangoRunServer


class Command(DjangoRunServer):

    def inner_run(self, *args, **options):
        self.start_ember()
        return super(Command, self).inner_run(*args, **options)

    def start_ember(self):
        self.stdout.write('>>> Starting ember')
        self.ember_process = subprocess.Popen(
            ['ember', 'serve'],
            stdin=subprocess.PIPE,
            stdout=self.stdout,
            stderr=self.stderr,
            cwd=settings.EMBER_ROOT
        )

        self.stdout.write(
            '>>> ember process on pid {0}'.format(self.ember_process.pid))

        def kill_ember_process(pid):
            self.stdout.write('>>> Closing ember process')
            os.kill(pid, signal.SIGTERM)

        atexit.register(kill_ember_process, self.ember_process.pid)

{% endhighlight %}

Now, whenever we run './manage.py runserver', our site will both serve the correct files for Ember and automatically rebuild those files as the source in the frontend directory changes.

This setup's been more of a hassle than I anticipated, but hopefully this post saves somebody else the headache. Next up, I'm going to create a few models and wire them to Ember Data. There are a few approaches for this; I'll write again when I've got something working.
