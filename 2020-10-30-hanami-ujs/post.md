---
title: Hanami: Unobtrusive JavaScript
published: false
description:
tags: hanami ruby
---

# Intro
In [one](https://dev.to/bajena/hanami-thoughts-of-a-rails-developer-ik1) of my previous posts I wrote down the impressions about Hanami I've had while working on my side project, [Flashcard Genius](https://flashcard-genius.com). One of the things I mentioned there was that some features you'd expect from a framework are missing or underdocumented. One of such features I missed in Hanami was unobtrusive javascript.

Rails comes out-of-the-box with [helpers](https://guides.rubyonrails.org/working_with_javascript_in_rails.html) that allow  developers to do following things without writing a single JavaScript line:
- Create links to POST/DELETE/PATCH endpoints (`link_to "Delete article, @article, method: :delete`)
- Send forms remotely using AJAX just by adding `remote: true` to your form tag.
- Show simple confirmation dialog after clicking a link (`link_to "Dangerous zone", dangerous_zone_path, data: { confirm: 'Are you sure?' }`)
- Disable submit buttons while waiting for the form to be sent (`f.submit data: { "disable-with": "Saving..." }`)

In my application I had a need for remote DELETE links (removing flashcards, log out endpoint), so I started looking for solutions and... turns out there's a way to do it in Hanami thanks to a plugin called [hanami-ujs](https://github.com/hanami/ujs). In this post I'm gonna show you how I solved some problems using this library.

Fun fact related to hanami-ujs is that it's hosted
