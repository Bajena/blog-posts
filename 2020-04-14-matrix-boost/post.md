---
title: Ruby on C(ocaine) 💉 Intro to C extensions for newbies.
published: false
description: Building C extensions in Ruby isn't really that hard. Let me show you how I built my first one.
tags: ruby tutorial
---

A few months ago I've read a book by Pat Shaugnessy called [Ruby Under Microscope](http://patshaughnessy.net/ruby-under-a-microscope). It taught me a lot about Ruby's internals and inspired to dive a bit deeper than normally and try building an extension to the language.

![enter image description here](http://patshaughnessy.net/assets/images/RUM_coverfront.png)

As most of Rubyists know Ruby is a language originally written in C language by [Yukihiro Matsumoto]([https://en.wikipedia.org/wiki/Yukihiro_Matsumoto](https://en.wikipedia.org/wiki/Yukihiro_Matsumoto)). The C implementation is called MRI (Matz's Ruby Interpreter) and it supports writing plugins that can "talk" to any C program of choice. I'd like to show you how I built my first extension, that it really isn't that difficult and potentially can bring you huge benefits in terms of performance.

Before we start, keep in mind that apart from MRI there's a bunch of other Ruby language implementations, e.g. [jRuby]([https://github.com/jruby/jruby](https://github.com/jruby/jruby)) written in Java, [Rubinius]([https://github.com/rubinius/rubinius/) written in... Ruby and [many others]([https://github.com/codicoscepticos/ruby-implementations](https://github.com/codicoscepticos/ruby-implementations)), so the C extension that I'm presenting in the article will most probably not work with other platforms than MRI.

## Ladies and gentlemen, I present you the MatrixBoost library 👏
I didn't want to create YAHW (Yet Another Hello World) extension, so I spent quite some time thinking what exactly should I implement, but finally after some brainstorming I came up with an idea. I've noticed that the [Matrix](https://ruby-doc.org/stdlib-2.5.1/libdoc/matrix/rdoc/Matrix.html) gem which is a part of Ruby's standard library is completely written in Ruby, so I decided to write a gem that'd reimplement some of the common matrix operations in pure C. Such library should come in handy for people whose programs heavily rely on Ruby's Matrix library and perform lots of matrix operations.

That's how I started working on [MatrixBoost](https://github.com/Bajena/matrix_boost) library. I decided that my gem will incorporate [https://github.com/nhomble/yasML](https://github.com/nhomble/yasML) which is a very simple matrix implementation written in C.

The main idea is that:
1. The C extension will convert a Ruby matrix into yasML C matrix
2. yasML will perform the desired matrix operation
3. C extension will convert the result C matrix back to the format understandable by Ruby

In the next paragraphs you'll see what were the necessary steps to achieve it.

## Step 1: Set up the gem
First thing that needs to be done is to create all the boilerplate required for a Ruby gem. You can e.g. follow up this [guide](https://guides.rubygems.org/make-your-own-gem/) or copy-paste from [MatrixBoost](https://github.com/Bajena/matrix_boost) repository.

In your `gemspec` file make sure to add `ext` folder to the list of required paths, like this: `spec.require_paths = ["lib", "ext"]`. It'll be necessary for the gem to load the extension code we're going to add.

Next insert the C library code in the `ext/<gem_name>` folder. In my case it was [ext/matrix_boost/matrices.c](https://github.com/Bajena/matrix_boost/blob/master/ext/matrix_boost/matrices.c) and [ext/matrix_boost/matrices.h](https://github.com/Bajena/matrix_boost/blob/master/ext/matrix_boost/matrices.h).

## Step 2: Prepare `extconf.rb`
In `ext` directory create a file called `extconf.rb` with the following content:
```ruby
require "mkmf"

create_makefile "matrix_boost/extension"
```

When you execute the `extconf.rb` file then the `mkmf` library (part of the Ruby extension build system) will prepare a [Makefile](https://en.wikipedia.org/wiki/Make_%28software%29) required to compile the C part of the gem.

The `extconf.rb` in my snippet is the simplest possible config, but of course if needed it can be extended to do things like generating header files, checking other necessary C dependencies or configure C compiler options.

You can check out what  `mkmf` is capable of [here](https://github.com/ruby/ruby/blob/master/lib/mkmf.rb).

## Step 3: Write Ruby <-> C glue code
Here's the most interesting and the most difficult part. We'll need to write a C file which'll serve as an entry point from the Ruby code. It'll add a Ruby module with functions that'll convert a Ruby arrays into C arrays, perform desired matrix operations and convert the results back into a Ruby array.

## Step 4: Compile the extension

- rakefile


Reason
- Ruby under microscope
- Learn how ruby internals work
-

The idea
- While browsing ruby's std library I've noticed that the Matrix library is written in pure ruby

List of materials

