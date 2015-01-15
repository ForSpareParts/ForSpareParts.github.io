---
layout: post
title:  "Ember: binding a read-only component property to a parent scope"
date:   2015-01-02 12:00:00
tags: ember irc component debugging properties
---

I spent a couple days trying to figure out why updates weren't firing on one of
my computed properties. Hopefully this saves somebody else the headache.

I had a component for my project that looked something like this:
<!-- more -->

    {% raw %}//components/my-component.js
    export default Ember.TextField.extend({
      selectedModel: function() {
        return this.get('content').findBy('name', this.get('value'))
      }.property('value')
    });{% endraw %}

used in a template sort of like this:

    {% raw %}{{ my-component
      content=listOfABunchOfModels
      selectedModel=someProperty }}{% endraw %}

Intuitively, every time 'value' (the value of the text field) updates,
someProperty (which is bound to 'selectedModel') should be updated. Instead,
the 'selectedModel' property function was only called once, when the page
loaded. This confused and upset me. After a couple days of banging my head
against the wall, here's what actually worked:

    //components/my-component.js
    export default Ember.TextField.extend({
      currentModel: function() {
        return this.get('content').findBy('name', this.get('value'))
      }.property('value'),

      setSelected: function() {
        this.set('selectedModel', this.get('currentModel'));
      }.observes('currentModel')
    });

With the template:

    {% raw %}{{ my-component
      content=listOfABunchOfModels
      selectedModel=someProperty }}{% endraw %}

After I did this, the property fired every time the text field changed, and
selectedModel was updated as I expected. Near as I can tell, here's why:

Setting `selectedModel=someProperty` in the template is sort of like doing:

    myComponentInstance.set('selectedModel', someProperty);

Calling .set() on a read-only computed property breaks that computed property --
afterward, it won't fire when its dependencies update. That's why we need the
observer: it translates the read-only computed property to a normal property
that can be both read from and written to.
