---
title: One app, two frameworks - comparison between Ruby on Rails and Hanami
published: false
description:
tags:
---

1. Intro about each framework
- History, pros, cons, twitter links to authors
- For Hanami & trailblazer - similarities/differences

Other options: Trailblazer? Padrino?

1. Set up was rather simple. Everything worked out of the box.
1. Hanami can have multiple apps per project - it helps with later extraction. Also the domain has its own separate "lib" folder.
1. Set up database
2. Create tables
Hanami (sequel) supports database constraints. Should you use it?
3. Data access - Repositories vs AR
Hanami: More flexible querying API
- When we declare an association, that repository does NOT get any extra method to its public interface. This because Hanami wants to prevent to bloat in repositories by adding methods that are often never used.
Bonus: Hanami supports creating many records at once
- It was my first contact with Dry-rb and Sequel so I might now know the ways to achieve it, but I didn't really see a simple way of DRYing the parts of queries, so I ended up writing methods which I know will only be used in a single context. I'm afraid this might end
4. Entities - dry types, good separation
5. Hanami console
6. No global helpers :tada: named routes have own helper method. Templates are backed by View classes that can implement the methods that in Rails would be implemented by global helpers.
7. Docs could be improved a bit. They could e.g. guide how to prepare a simple CRUD. Often I had to read some random posts to understand how to achieve sth.
8. Validations are great, no model validations as in AR
9. Service objects are encouraged by use of Interactors
10. Forms:
- By default when a edit/new form is rendered with errors the form prioritizes the data in "params". It saves some boilerplate time.
- In order to achieve dynamic nested forms I had to use a hack.
11. I liked having separate classes for each controller action
12. Hanami may be much less memory-consuming and faster (even ~5x) (https://rpanachi.com/2016/03/28/from-rails-to-hanami-part1-container-architecture-model-views-assets)

Summary:
- Hanami is a mature framework
- Even though I'm tired of Rails I'd probably not suggest starting/converting any bigger app to hanami. Maybe if your company's already using microservices and you want some refreshment you could try implementing a simple microservice in Hanami.
