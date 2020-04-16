---
title: Ruby on C(ocaine) 💉 Intro to C extensions for newbies.
published: false
description: Building C extensions in Ruby isn't really that hard. Let me show you how I built my first one.
tags: ruby tutorial
---

A few months ago I've read a book by Pat Shaugnessy called [Ruby Under Microscope](http://patshaughnessy.net/ruby-under-a-microscope). It taught me a lot about Ruby's internals and inspired to dive a bit deeper than normally and try building an extension to the language.

![enter image description here](http://patshaughnessy.net/assets/images/RUM_coverfront.png)

As most of Rubyists know Ruby is a language originally written in C language by [Yukihiro Matsumoto]([https://en.wikipedia.org/wiki/Yukihiro_Matsumoto](https://en.wikipedia.org/wiki/Yukihiro_Matsumoto)). The C implementation is called MRI (Matz's Ruby Interpreter) and it supports writing plugins that can "talk" to any C program of choice. I'd like to show you how I built my first extension and that it really isn't that difficult and potentially can bring you huge benefits in terms of performance.

Before we start, keep in mind that apart from MRI there's a bunch of other Ruby language implementations, e.g. [jRuby]([https://github.com/jruby/jruby](https://github.com/jruby/jruby)) written in Java, [Rubinius]([https://github.com/rubinius/rubinius/) written in... Ruby and [many others]([https://github.com/codicoscepticos/ruby-implementations](https://github.com/codicoscepticos/ruby-implementations)), so the C extension that I'm presenting in the article will most probably not work with other platforms than MRI.

## Ladies and gentlemen, I present you the MatrixBoost library 👏
I didn't want to create YAHW (Yet Another Hello World) extension, so I spent quite some time thinking what exactly should I implement, but finally after some brainstorming I came up with an idea. I've noticed that the [Matrix](https://ruby-doc.org/stdlib-2.5.1/libdoc/matrix/rdoc/Matrix.html) gem which is a part of Ruby's standard library is completely written in Ruby, so I decided to write a gem that'd reimplement some of the common matrix operations in pure C. Such library should come in handy for people whose programs heavily rely on Ruby's Matrix library and perform lots of matrix operations.

That's how I started working on [MatrixBoost](https://github.com/Bajena/matrix_boost) library.





Reason
- Ruby under microscope
- Learn how ruby internals work
-

The idea
- While browsing ruby's std library I've noticed that the Matrix library is written in pure ruby

List of materials
