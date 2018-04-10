---
layout: post
title:  "Introducing code_monkey"
date:   2015-09-13 2:39:00
tags: python static-analysis refactoring code-monkey
---

A while back, I was working on an extremely tedious refactor on a huge Python codebase. I figure most people have hit something like this: a little too complicated for a find/replace, but painfully boring to write out by hand.

I realized pretty quickly that I could describe the edits I wanted to make algorithmically -- in that context, they were really, really simple. So I went looking for some scriptable Python refactoring tools, and found... well... not much. [Rope](https://github.com/python-rope/rope), the most well-developed refactoring library for Python, still felt a little too heavy and complicated for the kind of quick one-offs I wanted to write. So, I took a stab at creating my own.

Singer/songwriter/genius Jonathan Coulton wrote a [great song](https://www.youtube.com/watch?v=v4Wy7gRGgeA) honoring those who slog through boring, tedious coding work. So, I thought, what better name for a refactoring library than [code_monkey](https://github.com/davidwallacejackson/code_monkey)?

<!-- more -->

code_monkey sat dormant for a long time, but I needed his help for another new project (which I hope to be posting about soon!), so I decided to dust him off, update him, and finally give him a proper release.

So, it does what, exactly?
--------------------------

If you've got a weird refactoring problem (e.g., rename all instances of "foo" to "bar", but only on odd-numbered lines of methods of subclasses of BazQuux, unless, of course, it's during a full moon[^1]) , the information you want is probably tied up in Python's [abstract syntax tree](https://docs.python.org/2/library/ast.html). It's not the easiest thing to use, though, and the documentation is pretty sparse -- in fact, there's an entire [independent project](https://greentreesnakes.readthedocs.org/en/latest/) dedicated just to explaining `ast`.

It'd be nice if you could get at this data in a more straightforward way, and that's the first problem code_monkey solves. I'd be remiss if not to mention here that I'm building on the work of Logilab's [astroid](http://www.astroid.org/), which is an expanded version of the Python AST with a similar API but more data. Thank you, Logilab!

Anyway, here's some code:

{% highlight python %}
    from code_monkey.node_query import project_query
    from code_monkey.edit import ChangeSet

    q = project_query('/path/to/my/project')

    #these are all the Model classes in our project (hopefully)
    model_classes = q.flatten().classes().path_contains('.models.')
{% endhighlight %}


code_monkey's strategy for exposing the AST is to wrap it in chainable queries (think jQuery). `project_query()`, `flatten()`, `classes()`, and `path_contains()` all return `NodeQuery` objects, and NodeQueries don't change once they're created. They're also indexable, so we can do `model_classes[0]` to grab a single `Node` representing one of the classes we've selected.

Nodes keep track of things like:

* what file the code -- in this case, a single class -- is in (`Node.fs_path`)
* what the class' "dotpath" is, e.g. my_project.my_app.models.Todo (`Node.path`)
* what line the class starts on (`Node.start_line`)
* what line the body of the class starts on (`Node.body_start_line`)

and so forth.

In the code above, we're using some of the information through the querying API. We create a query with a `ProjectNode` in it representing our whole project (that's `project_query()`), then we "flatten" it to contain all the children of everything in the project, recursively, all the way down to the variable assignment level. After that, we filter out everything that's not a class definition, and search through what's left using the dotpath.

The whole query API is [here](http://code-monkey.readthedocs.org/en/latest/querying.html).

That's great and all, but weren't we trying to *change* the code?
-----------------------------------------------------------------

Yes! Definitely! Getting there.

Once you've got a bunch of interesting Nodes, you can create a `ChangeSet` to alter them:

{% highlight python %}
    #continuing from above...

    from code_monkey.edit import ChangeSet
    changeset = ChangeSet()

    for class_node in model_classes:
        changeset.add(class_node.change.inject_at_body_line(0,
            '    new_property = models.CharField(max_length=128)\n'))

    print(changeset.diff())
{% endhighlight %}

So what's going on here is that every Node has a `.change` property with a bunch of methods to generate `Change` objects. A `Change` object says, basically, "write this code into this file, at this position" (or potentially, "overwrite this part of the file with this other code"). You can make Changes by hand, but you probably don't want to, because it's tedious. Let code_monkey do that! It's his job.

All those Change objects go into a ChangeSet, which can either generate a diff (as we saw above), or apply them to the filesystem (that'd be `changeset.commit()`). Any refactor complicated enough to merit this sort of treatment should probably be reviewed as a diff before you apply it.

It's possible to create Changes that overlap, so if you add conflicting Changes to the same ChangeSet, code_monkey will politely raise an exception. He might also mutter a few unflattering things about you behind your back, but hey -- given what he's putting up with on your behalf, he's probably entitled to a little grousing.

The API for creating Changes can be found [here](http://code-monkey.readthedocs.org/en/latest/editing.html).

So, yeah.
---------

If your curiosity is piqued, you can grab code_monkey off of pip. I'd welcome comments and criticism, of course. It's **super** alpha right now.


[^1]: If you want your code to behave differently on the full moon, there's actually a [package for that](https://pypi.python.org/pypi/astral/0.7.4). I'd tell you to use it responsibly, but really, who am I kidding? The prank potential is literally astronomical.
