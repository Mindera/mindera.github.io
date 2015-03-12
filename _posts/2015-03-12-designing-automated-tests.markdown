---
layout: post
title:  "Designing beautiful automated browser tests"
date:   2015-03-12 11:32:46
author: "Bruno Palos"
tags: tests pageobject browser latte protractor javascript angular automation api fluent pattern webdriver
categories: tech tests
---
During the last two years I worked with development teams that write their own automated tests. That’s right, developers who also write automated tests. However, sometimes developers tend to think that automating is boring and the lack of discipline when writing automated tests becomes a real problem. That problem reflects in the quality of the tests. That becomes evident when the tests are failing “for no apparent reason”. The majority of the times, the reasons are badly written tests or negligence.

I think there are two things to do for developers start writing automated tests with (almost) the same motivation as writing code: convince them that automated tests should be written by the development team and make it a fun task to do.

Well, the first thing is optional and is a matter for another post, but I’ll help you with the second thing.

For this post we assume that we are testing [AngularJS][angularjs] applications using [Protractor][protractor], which is a testing tool that allows us to automate browsers using [WebDriverJS][webdriverjs] - that’s right JavaScript for writing beautiful tests. Seams a paradox? Continue reading.

### The PageObject Pattern
If you work with browser automation you probably know the PageObject Pattern. You can read an excellent post about it by Martin Fowler [here][fowler-po].

The basic idea is to have page elements from the browser represented as objects in your tests and to have them structured in the same hierarchical way. Also, you can model your objects based on the way the user interacts with them. Designing the page objects with an API for the interactions and assertions, besides adding an abstraction layer that hides details the tests shouldn’t know, it eases page objects maintenance and improves code readability. It will also increase your team’s productivity and code re-use is maximised.

### Candy for the tests
I always liked to use fluent interfaces. They are particular useful with functional programming for manipulating streams of data were one can chain a sequence of filters or transformations in the same line of code. Well, we could write another topic just on this matter because manipulating streams with excess can in fact make the code unreadable, awkward and very inefficient. But, assuming that we use fluent interfaces properly, it can turn the code more intelligible, more compact and it’s easier to test.

For the browser automation concerns, we aren’t going to manipulating streams of data - at least in the majority of the use cases - we just want to simulate user actions, changing the application state and check for results on that state. We aspire to describe our tests in a clean and easy way.

Let’s get to the point: why not using BDD-like interface for writing our tests using a fluent idiom? First, we could inspire ourselves on the [BDD way of describing acceptance criteria][bdd-acc]:

{% highlight gherkin %}
Given some precondition
And another precondition
When user performs some action
And performs another action
Then some outcome should happen
And another outcome should happen
{% endhighlight %}

So, let’s translate it to pseudo-code using a fluent idiom:

{% highlight javascript %}
given.somePrecondition()
.and.anotherPrecondition()
.when.userPerformsAction()
.and.performsAnotherAction()
.then.itShouldHappenOutcome()
.and.itShouldHappenAnotherOutcome();
{% endhighlight %}

Looks good? Let’s move on then!

### Designing the interface
We will adapt the PageObject Pattern to have a fluent API that describes the business needs related to that particular page object. However, when we need to interact with other page object, we need to change the context. Let’s create a new keyword for telling the test how to interact with another page object. The `on` keyword seems to fit, but let’s try it with pseudo-code:

{% highlight javascript %}
on.aPageObject
.given.somePrecondition()
.and.anotherPrecondition()
.when.userPerformsAction()
.and.performsAnotherAction()
.then.itShouldHappenOutcome()
.and.itShouldHappenAnotherOutcome();
{% endhighlight %}

{% highlight javascript %}
on.anotherPageObject
.given.yetAnotherSomePrecondition()
.when.userPerformsYetAnotherAction()
.then.itShouldHappenYetAnotherOutcome();
{% endhighlight %}

It’s not the “last cookie in the box” in terms of lexical choice, but it’s okay. Concretely, our interface would become something like the following code written in JavaScript:

{% highlight javascript %}
var api = {
  on: {
    anotherPageObject: anotherPageObjectInstance;
  },
  given: {
    yetAnotherSomePrecondition: function () {
      // Do some stuff
      return this;
    }
  },
  when: {
    userPerformsYetAnotherAction: function () {
      // Do some stuff
      return this;
	}
  },
  then: {
    itShouldHappenYetAnotherOutcome: function () {
      //Do some stuff
      return this;
    }
  },
};
{% endhighlight %}

Notice the `return this` sentence on every function of the code. This is basically what triggers our fluent API to reference itself and to be infinitely chainable. It’s possible to chain multiple functions from the same block because the `this` keyword inside a context object uses implicit binding, which means that the this references the block were it’s used. For example, the `this` inside the given object is the given object itself.

One of the most important pieces of our puzzle is how to chain the different *on-and-given-when-then* blocks is still missing, but that is simply solved by the following snippet:

{% highlight javascript %}
fluidify: function(apiIngredients) { 
  var fluidAPI = { 
    and: fluidAPI, 
    on: apiIngredients.on, 
    given: apiIngredients.given, 
    when: apiIngredients.when, 
    then: apiIngredients.then 
  };  

  fluidAPI.on.and = fluidAPI; 
  fluidAPI.given.and = fluidAPI; 
  fluidAPI.then.and = fluidAPI.then; 
  fluidAPI.when.then = fluidAPI.then; 
  fluidAPI.when.and = fluidAPI.when; 
  fluidAPI.and = fluidAPI;  

  return fluidAPI; 
}
{% endhighlight %}

This is the core of the page objects’ API. Having cyclic references in our API allow us to write the tests with the proposed idiom. Notice how the `and` is added to each API block and how the `then` is referenced on `when`. All of those cyclic references surely will create a lot of problems if you need to serialise your page objects, since we are using properties as our keyword to chain the tests steps, but there’s a way to fix it and we’ll get back to this later.

### Anatomy of a page object
For start joining some pieces together, let’s define the anatomy of our page object. We already have an interface, so it’s possible to have a skeleton of what our page object will be, even if we didn’t yet defined how to chain and and how to relate the page objects in hierarchy.

Let’s use the module pattern, which is convenient because we are using Protractor, which runs over [NodeJS][nodejs].

{% highlight javascript %}
var loginForm = function(parentContext) {
// Public API
var api = {
on: {},
  given: {
    emptyForm: function () {
      usernameBox().clear();
      passwordBox().clear();
      return this;
    }
  },
  when: {
    loginIsSuccessfull: function (username, password) {
    usernameBox().sendKeys(username);
      passwordBox().sendKeys(password);
      loginButton().click();
        return this;
      }
    },
    then: {
      itShouldCloseForm: function () {
        expect(context().isPresent()).toBeFalsy();
        return this;
      }
    }
  };
  return api;

  // Private functions

  function context() {
    parentContext.element(by.css('.login-form'));
  }

  function usernameBox() {
    context().element(by.css('[name="username"]'));
  }

  function passwordBox() {
    context().element(by.css('[name="password"]'));
  }

  function loginButton() {
    context().element(by.css('[name="loginBtn"]'));
  }
};
module.exports = loginForm();
{% endhighlight %}

Wow! We have a lot of new things here! Let’s try to scrutinise the example.

Has you can see, we defined our fluent interface as the API of our *LoginForm* page object. The private functions are related with the internals of our page object and they help organize implementation and to avoid leaking its details to the tests.

Note how the code is organised in small abstractions. These abstractions could be, as the example shows, elements from the page object - the ones that don’t worth to implement as independent page objects - like input boxes, buttons, etc. This way, the code is more readable and better prepared for refactors. This is really an important point because, during development of web applications, selectors could change multiple times, more than the page elements structure, so we should be prepared for changes. By organising the internals of your page objects this way, you should be able to change the selectors in one place instead of changing all the related parts in the code.

Besides the API, there’s a detail that should be always part of you page objects: the page object’s own selector. In the example, the selector is defined inside the `context` function. You probably wondered what’s the parameter passed to the *LoginForm* constructor, the *parentContext*. That’s the context of the parent page object and this is what makes the *LoginForm* reusable across multiple page objects - by context I mean the [ElementFinder][elementfinder] of the parent. That makes possible to have *LoginForm* to be found only inside the parent’s scope and not inside the whole document (as shown in the example), which also streamlines the hierarchical structure of the page objects tree. This behaviour also applies to the *LoginForm*’s internal elements, which are searched only inside the *LogiForm*’s context instead of being selected in the whole page. Check how simple the CSS selectors become using this strategy, they just need to be relative to the parent’s context: the *LoginForm* selector is relative to the parent page object and the *LoginForm*’s internal elements are selected relative to it.

### Page Objects hierarchy
This topic should be part of the page object anatomy but I decided to create a whole new section about this because this is probably the most tricky part of implementing the test as proposed on this post.
Implementing page objects inheritance will not be as easy as chaining page objects’ prototypes, because we have a fluent API that is infinitely chainable. We can’t just define the prototype of our API and chain it with a parent page object because we define the *on-given-when-then* structure as properties of the page object’s API. So, to have a inheritance-like feature in our page objects, we need to find a way for each block from the *on-given-when-then* to inherit behaviour from the API of the parent page object. Something like:

{% highlight javascript %}
var pageObjectA = function() {
  var api = {
    on: {},
    given: {},
    when: {
      somethingHappensInA: function () { … }
    },
    then: {}
  }
  return api;
};
module.exports = pageObjectA();
{% endhighlight %}

{% highlight javascript %}
var pageObjectB = function() {
  var api = {
    on: {},
    given: {},
    when: {
      somethingHappensInB: function () { … }
    },
    then: {}
  }
  // Omitted some magic code that makes B to extend A
  return extendedApi;
};
module.exports = pageObjectB();
{% endhighlight %}

{% highlight javascript %}
pageObjectB
.when.somethingHappensInB()
.and.somethingHappensInA();
{% endhighlight %}

The *PageObjectB* inherited the `when block from *PageObjectA*. How’s that possible?

Let’s forget prototypes for a minute. Lets think on what we want here: we want *PageObjectB* to be able to execute the code inside *PageObjectA*. If we want just that, we can just create a *mixin* from *PageObjectA* and *PageObjectB*. Sure, this have a lot of limitations, but if we think we would like to keep our code simple, having such kind of limitations can even be beneficial. Is also helps us implement our code less tangled and loosely coupled by not having code that should be in the parent page object.

Let’s stick to that, let’s mix our page objects together and be able to have *mixins* of multiple page objects. First, we need we need to implement a function to mix page objects together. Wait, we don’t need to do that! We could use [lodash] library, which have the `merge` function that can merge a lot of objects together. Perfect!

Our *PageObjectB* could be written in the following way, where `pageObject` is an abstract factory that hides the mechanism of merging objects and chains the fluent API following the rules we defined yearlier:

{% highlight javascript %}
var pageObjectB = function() {
  var api = {
    // Omitted API implementation
  }
  var extendedApi = pageObject.create(api, pageObjectA);
  return extendedApi;
};
module.exports = pageObjectB();
{% endhighlight %}

The next step is to implement the abstract factory for creating our page objects transparently.

### Look no further, Latte Page Object is here!

We developed a small library that do all the hard work for you, it’s called [Latte Pageobject][latte]. It’s open source and you can use it and contribute.

*Latte Pageobject* builds the fluent interface for you and merges all the parent page objects into a single and ready-to-use page object. It also merges custom blocks you might add to your page object API, like some getters that expose the state of your page object.

One of the major problems of this idiom is when you need to serialise your page objects. Serialisation of raw page objects can cause your call stack to reach the limit, since it’s dealing with a lot of cyclic references. *Latte Pageobject* implements interceptors over the *and-on-given-when-then* properties and the final instantiated page object don’t expose those properties directly. Instead, it exposes the interceptor’s functions.

As an old colleague of mine use to say: “What a nice trick!!”.

[angularjs]:      https://angularjs.org
[protractor]:     http://angular.github.io/protractor
[webdriverjs]:    https://code.google.com/p/selenium/wiki/WebDriverJs
[fowler-po]:      http://martinfowler.com/bliki/PageObject.html
[bdd-acc]:        http://en.wikipedia.org/wiki/Behavior-driven_development#Behavioural_specifications
[nodejs]:         https://nodejs.org
[elementfinder]:  http://angular.github.io/protractor/#/api?view=ElementFinder
[lodash]:         http://lodash.com
[latte]:          https://github.com/Mindera/latte-pageobject
