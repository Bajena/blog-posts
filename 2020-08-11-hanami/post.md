---
title: Hanami: Thoughts of a Rails developer
published: false
description:
tags: hanami, rails, ruby
---

# Intro
I remember hearing about [Hanami](http://hanamirb.org/) framework for the first time. It was a few years ago, during a talk on [Wroclove.rb](https://wrocloverb.com/) conference. It didn't really catch my attention back then as I was still a newbie in the Ruby world. I was 100% focused on learning Rails and didn't wanna cognitively overload my brain with concepts from yet another framework.

Now, after all these years spent working with Rails I got a little burnt out and thought that it's a hight time to get my hands dirty with something completely new. Having the old wroclove.rb talk in mind I decided to learn Hanami. My favorite way of learning is by doing, so I started a pet project called [flashcard-genius](https://github.com/Bajena/flashcard-genius). The purpose of the app is helping me with learning Italian words. It lets users create, share and memorize sets of flashcards.

After around 2 months of more and less frequent after-hours development I managed to create a prototype (can be found at http://flashcard-genius.com). Even though it's a basic server-renderered CRUD-like application it already forced me to learn a full variety of Hanami concepts and overcome multiple roadblocks.

In this post I'd like to share with you some of my impressions after these few weeks of working with Hanami.

![image](https://user-images.githubusercontent.com/5732023/97406654-d7b7c380-18f9-11eb-9256-89dac0360af7.png)

# Why should you use Hanami at all?
Hanami was started by [Luca Guidi](https://twitter.com/jodosha) as a counterproposal to Rails. The main idea was to build a secure, feature-rich web framework that follows clean architecture principles and relies on "battle tested" libraries like [Sequel](http://sequel.jeremyevans.net/), [Rack](http://rack.github.io/) and [dry-rb](https://dry-rb.org/gems/) gems.

Sounds tempting, right? Well, to me it did. Especially because our beloved Rails are often criticized for having become too complex and breaking some of the best practices (e.g. it mixes domain and data-access layers).

One potential benefit of Hanami might be also memory usage. Hanami may be much less memory-consuming and faster (even ~5x) (https://rpanachi.com/2016/03/28/from-rails-to-hanami-part1-container-architecture-model-views-assets)

# What I liked?

## The setup
Setting up a Hanami application was really simple. The [getting started](https://guides.hanamirb.org/introduction/getting-started/) page explains how to:
- setup a project
- create a first route with matching controller action, view and template
- prepare basic tests

That good first impression was really important for me, because seeing things work out-of-the box motivated me to continue learning more complex concepts in order to achieve the desired functionalities for my app.

## Hanami project is a "microlith"
https://guides.hanamirb.org/architecture/overview/

Hanami can have multiple apps per project - it helps with later extraction. Also the domain has its own separate "lib" folder.

## Entities, repositories, validation objects and interactors instead of fat models
Hanami attempts to prevent developers from making the same mistakes as they tend to do in Rails.

- Instead of allowing usage of database-querying methods directly on the model Hanami provides special Repository classes. At the beginning it feels like useless boilerplate, but your future self will be greatful for that.
- Instead of lifecycle callbacks (`after_create`, `after_build`, etc.) which often end up being untestable and unmaintainable Hanami encourages you to use service-objects called [Interactors](https://guides.hanamirb.org/architecture/interactors/).
- [Models](https://guides.hanamirb.org/models/overview/) in Hanami  by default provide you automagically only with accessors to the corresponding table in the database. This makes them lightweight compared to the active record model that creates tons of automatically generated methods (like `username`, `username_changed?`, `username_did_change?`, etc.).
- Authors of Hanami came to the conclusion that validations shouldn't be a part of the model, because the way how we validate data is highly dependent on the use case. So, instead of providing validation mechanisms on the model they decided to create a `Hanami::Validation` [mixin](https://guides.hanamirb.org/validations/overview/) (based on the great gem called [dry-validation](https://github.com/dry-rb/dry-validation)). Controller actions in Hanami have them included out-of-the-box, but you're free to create your own validator objects. If you'd like to learn more about why model validations are an anti-pattern I recommend you to watch this [video](https://www.youtube.com/watch?v=nOUPIa7tWpA) by Piotr Solnica.

## Data access and manipulation
As I mentioned before in order to query/manipulate the data in Hanami you have to use repository classes. Hanami developers decided to incorporate another top-notch gem called [Sequel](https://github.com/jeremyevans/sequel) for database communication so you can use all the goodies provided by Jeremy Evans.

One of the things I liked in Hanami repositories is that when I declare an association, that repository does NOT get any extra methods to its public interface. This is because Hanami wants to prevent to bloat in repositories by adding methods that are often never used.

There are also a few niceties that I haven't used yet but may come in handy in the future:
- You can [manipulate many records at once](https://guides.hanamirb.org/repositories/overview/#custom-commands) without writing raw SQL queries.
- There's a support for [database constraints](https://guides.hanamirb.org/migrations/create-table/#constraints)

## No global view helpers
Templates are backed by View classes that can implement the methods that in Rails would be implemented by global helpers. This prevents the ERB templates from being bloated with logic which is quite hard to test.

## Other
- Hanami is based on [Rack](https://github.com/rack/rack) which makes it easy to use popular Rack middleware (error trackers, rate limiters etc.)
- Hanami comes with a Rails-like console command which turns out to be really useful
- In general Hanami is a pretty mature library that provides most of the features you'd expect from a modern web framework (asset management, DB migration system, mailers, security first, etc).

# What I disliked

## Small ecosystem
That's obvious - Hanami is a much smaller project than Rails, so there's much less gems, guides dedicated to Hanami on the web. Also many 3rd party services have SDKs prepared specially for Rails so in order to run them with Hanami sometimes you have to spend some additional time.

## Guides are sometimes too basic
Guides cover only very basic use cases. A lot of knowledge is spread on discourse forum/gitter chat/stack overflow. Docs could be improved a bit. They could e.g. guide how to prepare a simple CRUD. Often I had to read some random posts to understand how to achieve sth.

Also the fact that some great gems are incorporated into Hanami sometimes is a curse, because you have to understand what APIs are based on what gems and then dig into the gem's code/documentation to understand how to achieve the thing that you want.

## Using the repositories is painful sometimes
Maybe I haven't spent enough time with Hanami repositories and Sequel yet, but I didn't really see a simple way of DRYing the parts of queries, so I ended up writing query methods having many parts in common.

I also remember struggling with writing a repository method that'd return an additional column based on `GROUP BY` clause. It took me quite a while to solve it. It'd be cool if the docs included a bit more complex usage examples like this one.

# Summary
I must admit that even though I really liked using Hanami I'd probably not suggest starting/converting any bigger app to Hanami unless you want to spend lots of time on forums/chats and solve problems that had already been solved in Rails ecosystem.

Still, I really admire all the ideas and work put by the creators of Hanami and I'm still gonna develop my side project with that framework. Thank you guys!
