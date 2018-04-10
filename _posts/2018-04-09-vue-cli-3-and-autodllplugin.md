---
layout: post
title:  "Speed your builds way, way up in vue-cli 3.0 with AutoDllPlugin"
date:   2018-04-09 22:45:00
tags: vue vue-cli performance autodllplugin
---

vue-cli 3.0 is in beta, and it's pretty great! Our team wanted a zero-conf build tool for our vue projects so bad that a coworker actually [created one](http://vue-build.com/).  Now the community will be standardizing on one, and since it's built on top of webpack the weird, fussy parts of our existing build process can be ported over without too much fuss.

My favorite 3.0 feature, however, is off by default -- and if your builds last longer than a few seconds, you should probably know about it, too. vue-cli 3.0 comes with built-in support for the [autodll-webpack-plugin](https://github.com/asfktz/autodll-webpack-plugin).

If you're *really* short on time, here's the quick version:

1. In your `vue.config.js`, do this:

{% highlight js %}
module.exports = {
  // other config goes here...
  dll: true
}
{% endhighlight %}

Seriously, just that. Your next build will be about the same speed as usual. Subsequent builds will be faster than usual -- a lot faster. You can do even better, though...

<!-- more -->

That was so easy, I can actually spare 5 minutes to read more of this blog post if it can speed up my builds even more
---------------------------------------------------------

Awesome!

So let's talk a bit more about what happened here: the AutoDllPlugin wraps webpack's DllPlugin, which lets you prebuild dependencies that don't change very often into a separate bundle. DllPlugin requires a whole separate webpack config -- AutoDllPlugin will generate that webpack config (and when appropriate, run it) for you. vue-cli 3.0 goes one step further, and _automatically configures AutoDllPlugin_.

AutoDllPlugin needs a list of the modules you want to pre-build, and it compiles them, with all their dependendies, into a separate bundle. If you put `dll: true` in your `vue.config.js`, vue-build will just read your `package.json`, pick out everything in `dependencies`, and use that as its list.

This actually works really, really well. But it's not perfect. Consider this package:

{% highlight js %}
// some-package/index.js

import './a'
{% endhighlight %}

{% highlight js %}
// some-package/a.js

export default function somethingUseful () { }
{% endhighlight %}

{% highlight js %}
// some-package/b.js

import Highcharts from 'highcharts'
import Backbone from 'backbone'
import * as _ from 'lodash'
import Vue from 'vue'

export default {
  // imagine there's a lot of code here
}
{% endhighlight %}

If `some-package` is in your `package.json`, vue-cli will pass `'some-package'` to AutoDllPlugin, which will import `index.js` and its dependency `a.js`. It *won't* import `b.js`.

And if you use `b` in your project (say, with `import b from 'some-package/b'`), that's a real problem. `b` is huge -- look at those dependencies -- and now the whole thing is getting compiled into your app's entry chunk. Every. Single. Build.

You can pass your own list of modules to AutoDllPlugin instead, so a naive solution would be:

{% highlight js %}
  //vue.config.js

  module.exports = {
    // other config goes here...
    dll: ['some-package/b']
  }
{% endhighlight %}

But now, `some-package/b` is the _only_ dependency we're pre-building. Also not ideal, and you probably don't want to hand-specify every dependency in your project. Here's what I did:

{% highlight js %}
  //vue.config.js
  const packageJson = require('./package')
  const allPackageDeps = Object.keys(packageJson.dependencies)

  module.exports = {
    // other config goes here...
    dll: allPackageDeps.concat([
      'some-package/b',
      // more dependencies can go here
    ])
  }
{% endhighlight %}

This gets you the best of both worlds, automatically including both the base dependencies from `package.json` and more specific modules that vue-cli might overlook on its own. If you have a file in your project whose main purpose is to import a bunch of modules for another library (say, to import highcharts and import/register bunch of highcharts plugins), you can just add _that_ file to your `dll` -- all the child dependencies will get picked up automatically.

Once I played with this a bit, my builds went from 75 seconds to 15. Here's hoping it helps you, too.
