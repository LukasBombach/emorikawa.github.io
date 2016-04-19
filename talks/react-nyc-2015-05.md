---
layout: default
title: React NYC — Scaling Flux
---

# React NYC: Scaling Flux

May 28, 2015

### Description

Flux is a great for managing data across your React application; however,
when apps grow in complexity Flux can become unruly. Stores quickly
develop messy dependencies with each other and it becomes extremely
difficult to reason about your application. While developing a highly
extensible email client we found an extremely effective way to manage
Store complexity was to consolidate data into singular Centralized Data
Store. It became a single source of truth that made our app easier to
reason about, more robust, and scalable.

This talk was orginally given at [React NYC](http://www.meetup.com/NYC-Javascript-React-Group/events/221427467/).

### Slides

<script async class="speakerdeck-embed" data-id="dd9ca71e4a4644db9ca91fd113fe9274" data-ratio="1.77777777777778" src="//speakerdeck.com/assets/embed.js"></script>

### Transcript

**Slide 2**

Hi, my name is Evan I&rsquo;m a React engineer at <a href="https://nylas.com/">Nylas</a> building a <a href="https://nylas.com/blog/splitting-the-atom">highly extensible email client</a> in React and Flux. This, like most apps, started simple, but over the past several months it has grown into tens of thousands of lines of javascript spread across dozens of stores and components.

**Slide 3**

Tonight I&rsquo;m going to be talking about a better way to handle Flux in large, complex applications.

We found that when you have a ton of interconnected stores keeping a clean declarative pattern quickly gets hard. Our solution was to centralize all of our data in one top-level store and make stores hierarchical.

But first, let&rsquo;s introduce Flux. For those of you that have built an app with Flux before the next couple slides should be review. But to get us all on the same page, let me show you how this works with a simple example.

**Slide 4**

Let&rsquo;s say we wanted to build a list of Email Messages with React and Flux.

**Slide 5**

This is Flux. It is a design pattern. It is a cleaner, more declarative way to manage your data. There are many libraries you may have heard of that offer implementations of Flux. For example, Fluxxor, Reflux, and Facebook&rsquo;s flux.js are some. For the sake of this talk we&rsquo;ll only be referring to the design principle. 

**Slide 6**

At the core of Flux, and this talk, are Stores. Stores hold your data. In our example it holds a bunch of messages.

Stores are not Models in the traditional MVC sense.

The first thing to remember about the Flux design pattern is that stores are Singletons. There&rsquo;s only one. The MessageStore&rsquo;s job is to cache and aggregate my messages in meaningful ways, vend ideally immutable copies of that data through public getters, and listen for Actions that may cause data to change.

**Slide 7**

This is the React component. Any time the MessageStore changes, it notifies Component. The component then gets fresh data from The MessageStore and renders it.

**Slide 8**

If a user wants to interact with the component, like deleting a message, the Component will fire an Action that&rsquo;s dispatched through the central Dispatcher and caught by the Store.

**Slide 9**

Upon receiving the Action, the store will update its own internal data, then &ldquo;trigger&rdquo; to let everyone know the data has changed.

Upon trigger, the Component fetches new data and the cycle is complete.

**Slide 10**

This is the key pattern that keeps the view declaratively bound to its data. More importantly it ensures that the view never gets out of sync. The view always has an accurate representation of the data. If the data were to ever change, the MessageStore would trigger. When the message store triggers, the view re-fetches its latest data.

This &ldquo;Action to trigger to refresh pattern&rdquo; is the central dogma of Flux and guides the creation of simple applications. However we&rsquo;ve found it starts to get messier and messier the more complex an app gets.

**Slide 11**

If you have a relatively small amount of data or your data is always completely independent of each other then congratulations! You can stop listening to this talk now. Unfortunately in large, complex applications this is unavoidable.

Let&rsquo;s look at what happens to our initially simple Message List example as our requirements become more complex.

**Slide 12**

The first thing we want to do is update the current list of messages whenever the actively selected thread changes. Instead of having an imperative method to swap out the thread, I&rsquo;m going to wire up the MessageList to be declaratively linked to a ThreadStore. This way whenever the ThreadStore changes with the newly selected thread, the MessageList will update with the correct data.

Having Stores listening to other stores is a common design pattern in Flux. Once you have interdependent data, this pattern will emerge. In this example it&rsquo;s relatively manageable and clean, but we&rsquo;ve found it quickly gets messier.

**Slide 13**

Great. Now I can display the list of email messages in a single conversation thread. Next I want to be able to reply to those messages, and I want to display the drafts that I&rsquo;m working on in-line with the rest of the conversation. One reasonable way to do this is to have my MessageStore listen to changes in a singleton DraftStore. Whenever a new draft is created or destroyed the DraftStore updates. When the DraftStore updates, the MessageStore will fetch the appropriate data and then update. When the MessageStore updates, the View will re-fetch the new data and re-render. The net result is that the message list will now declaratively represent the state of the DraftStore AND the MessageStore.

**Slide 14**

Next I add in a websocket connection to my app to get live updates from my server. This is great because when I get a new email message it can instantly show up at the bottom of my message list. If this were jQuery it&rsquo;d be easy. We&rsquo;d simply get the update then call `append ` to the MessageList. However, this would break our declarative pattern and cause a new set of headaches down the road that emerge in jQuery spaghetti code. Instead I&rsquo;m going to listen for whenever the websocket changes and declaratively fetch new information from a cache of new data.

**Slide 15**

And another requirement comes in. We now want to support multiple languages and have the concept of a Translation module that will help us display the messages properly based on the users currently selected language. Once again, we listen to another store, this time a TranslationStore, to declaratively get the appropriate data for the message list.

Let&rsquo;s take a step back and look at where we&rsquo;ve gotten ourselves.

We have a MessageStore that&rsquo;s now wired up to the current state of several other stores. That was just one Store!

**Slide 16**

Those stores in turn might also be wired up to more stores.

Now we have a problem. This is hard to reason about.

When a change happens somewhere on the system where will it will propagate to. There might be hidden circular references. These chains may be arbitrarily long. Worst of all the ordering of triggers now suddenly matters!

**Slide 17**

This problem is not new. Facebook&rsquo;s own Flux website has this hairy bit of forewarning about the nature of complex applications:

&ldquo;As an application grows, dependencies across different stores are a near certainty. Store A will inevitably need Store B to update itself first, so that Store A can know how to update itself. We need the dispatcher to be able to invoke the callback for Store B, and finish that callback, before moving forward with Store A. To declaratively assert this dependency, a store needs to be able to say to the dispatcher, &quot;I need to wait for Store B to finish processing this action.&quot; The dispatcher provides this functionality through its waitFor() method.&rdquo;

In my opinion the fact that `waitsFor` even exists is a bad smell. Using `waitsFor` starts to introduce brittle code and is another example of the issues that arise with Store interdependency.

**Slide 18**

Instead of getting ourselves into interdependent hell, we found one, initially scary, but eventually elegant solution to fix a lot of these problems.

**Slide 19**

We centralized all of our data in one top-level store.

This is the global, singular, DataStore. It houses all of the data and is crucially the single source of truth in the application.

It makes the dependency diagram go from this &mdash;&nbsp;to this.

No more cycles. No more arrow spaghetti. No more redundant listeners. This diagram is much easier to reason about, and scales better. It decouples previously dependent stores from each other and leads to a cleaner, more isolated system.

Most of the stores in the app listen to the DataStore. Whenever the DataStore triggers, the listening stores re-fetch their data from this central repository and, if necessary, trigger themselves.

When the stores re-fetch their data from the central DataStore, they can now pull together whatever disparate data they need to fulfill their request. This is how we easily satisfy stores with composite data. Instead of fetching data from a myriad of sources, stores can get it from one place.

**Slide 20**

To further illustrate how this helps us, let&rsquo;s look back at a notoriously annoying problem with Flux: Getting new websocket data into the app.

**Slide 21**

Before we had the centralized DataStore, our MessageStore had to explicitly listen to the websocket. Before the DataStore, just about every store had to individually listen to the websocket. This was a lot of duplicated code and contributed to the dependency mess we saw earlier. Furthermore, if the socket streamed lots of different data, there was a routing problem to get the right type of data to the right Store.

**Slide 22**

Now with a centralized DataStore, the websocket can simply dump all of its composite data right into this repository. The minute that happens, the DataStore will trigger and all listening Stores will correctly fetch the appropriate composite data from the DataStore.

Changing state in the app is easy - data can be written to the data store from anywhere, and changes propagate to the other stores, and out to the React components.

When we describe this data store as being a singular piece of global. shared. mutable. state. we&rsquo;d commonly get this reaction:

**Slide 23**

Anyone who has written parallel code knows how horrible it can be when an object you&rsquo;re referencing suddenly, silently, and inconsistently changes its state from underneath you. This tends to be the source of the worst form of Heisenbug and there are tons of coding patterns and even programming languages designed to avoid this one problem.

**Slide 24**

The DataStore is global, it is mutable, and it is shared. But it is not evil. One of the biggest reasons it is not evil is because of the immediate triggering mechanism built into the Flux pattern. The minute anything in the DataStore changes, a synchronous trigger propagates throughout the system telling each store to refresh. Instead of passing the data through that trigger, the refresh-based mechanism helps ensure that Stores are getting fresh copies of the data they need, even if those refresh-mechanisms are asynchronous. Furthermore, it&rsquo;s good practice to make the fetched data immutable. In fact there are React systems like Om, that make it mandatory to fetch immutable data.

A centralized DataStore is also extremely robust to mutating changes. Since everything else in the app, from the views, to the stores, declaratively derives from this data, if you wish to change something in the DataStore, your wish will be granted.

**Slide 25**

A Store can mutate the dataStore.

**Slide 26**

The Websocket can mutate the DataStore.

**Slide 27**

You can even open up the console and change some data manually and know it will safely propagate its way through the app.

**Slide 28**

And because of the trigger, refresh mechanisms of listening stores, everything stays in sync.

**Slide 29**

One of the lasting implications of moving everything to a DataStore like this is that it helped us think about our app in a much more structured and hierarchical way.

**Slide 30**

As our app grew, and views required composite data, thinking about what Stores I needed to hold what data became challenging.

What we realized that each store was just a cached subset of its parent. The data always stays in sync because of the Flux trigger &amp; refresh pattern. With all the data getting aggregated at the root.

**Slide 31**

The DataStore is a cached subset of all of the data on the API.

**Slide 32**

The MessageStore is a cached subset of some of the data in the DataStore. I might have another store that&rsquo;s a cached subset of that.

**Slide 33**

Finally the state in a React Component holds a cached subset of data in stores.

**Slide 34**

At each level in the hierarchy, we&rsquo;re defining a declarative contract to guarantee that a certain subset of composite data will be at this point at any given time. We then rely on the trigger/fetch mechanism of the Flux pattern to ensure that this is always true.

With this hierarchy, any data entered from the top will propagate down to exactly the stores and components that need it and will always stay in sync.

**Slide 35**

One final bonus of a centralized DataStore is that the state of the entire app is in one, savable, place. You could persist it to local storage, you can keep it on a backend, or you can save it for diagnosing crashes. In fact, in the application we&rsquo;re building our DataStore is a full blown local SQLite Database instance. We furthermore wrote our own ActiveRecord-like interface to retrieve and set data into our Database. The Database is our single source of truth, and the only place we have to worry about changing to reflect new states in the application.

**Slide 36**

A centralized data store is one of the more effective ways we found to reign in the complexity of our increasingly growing application. It&rsquo;s drastically help simplify the reasoning of our app, kept things decoupled, and led us to think about our data in a hierarchical fashion.

Even if you don&rsquo;t immediately restructure everything you&rsquo;re building now, I hope that as your apps grow in complexity and data starts to get intertwined, you&rsquo;ll remember this as a way to keep your app sane. And, of course, if you&rsquo;re interested on working on complex applications like this, we&rsquo;re always hiring and I&rsquo;d love to talk to you.

**Slide 37**

Thank you very much!

**Slides 38 - 42**

Bonus slides. Code implementation of a Store using Coffeescript and Reflux

**Slides 43 - 48**

Bonus slides. Code implementation of a Component using Coffeescript and Reflux

### Recorded Video:

A [member of the audience](https://twitter.com/noragoldman) recorded the only complete video of the event.

<iframe width="720" height="405" src="https://www.youtube.com/embed/DWzeZhXWQd8" frameborder="0" allowfullscreen></iframe>

### Photos:

![Evan Morikawa NYC Flux](/assets/images/Evan-Morikawa-React-NYC.jpg)
