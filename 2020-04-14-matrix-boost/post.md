---
title: Ruby on C(ocaine) 💉 Intro to C extensions for newbies.
published: true
description: Building C extensions in Ruby isn't really that hard. Let me show you how I built my first one.
tags: ruby, tutorial
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

![image](https://user-images.githubusercontent.com/5732023/80178390-5f94dc00-85fe-11ea-956b-923e8c66fcbe.png)

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
Here's the most interesting and at the same time the most difficult part. We'll need to write a C file which'll serve as an entry point from the Ruby code. It'll add a Ruby module with functions that'll convert a Ruby arrays into C arrays, perform desired matrix operations and convert the results back into a Ruby array.

### Entry point - Init_extension
In order to initialize the extension Ruby will search for a method called `Init_extension(void)` in a file named  `ext/matrix_boost/extension.c`. This function will define a Ruby module with functions performing matrix operations, like `inv_matrix`, `mul_matrix`. These functions will be executed in our C extension.

Here's how a basic `extension.c` with `Init_extension` will look like:
```c
#include "ruby/ruby.h"
#include "matrices.h"

static VALUE inv_matrix(VALUE self, VALUE m) { // We'll implement it later }
static VALUE mul_matrix(VALUE self, VALUE m1, VALUE m2) { // We'll implement in later }

void Init_extension(void) {
  VALUE MatrixBoost = rb_define_module("MatrixBoost");
  VALUE NativeHelpers = rb_define_class_under(MatrixBoost, "NativeHelpers", rb_cObject);
  rb_define_singleton_method(NativeHelpers, "mul_matrix", mul_matrix, 2);
  rb_define_singleton_method(NativeHelpers, "inv_matrix", inv_matrix, 1);
}
```

This content of `Init_extension` is rather straightforward:
1. First line defines a `MatrixBoost` Ruby module.
2. Second line defines a Ruby class called `MatrixBoost::NativeHelpers`. Last argument defines a base class for our class. `rb_cObject` corresponds to the basic ruby `Object` class.
3. Third line defines a Ruby method called `mul_matrix` under `MatrixBoost::NativeHelpers` module. This function will execute C function called `mul_matrix`. The last argument indicates the number of arguments (`self` doesn't count!).
4. Fourth line defines a Ruby method called `inv_matrix` under `MatrixBoost::NativeHelpers` module. This function will execute C function called `inv_matrix` and has one argument.

### Ruby <-> C communication

Let's now take a look at `inv_matrix` function implementation to see how to Ruby objects can "talk" to the C matrix library:
```c
static VALUE inv_matrix(VALUE self, VALUE m) {
  Check_Type(m, T_ARRAY);

  Matrix *mc = rb_array_to_matrix(m);
  Matrix *inverted = matrix_invert(mc);

  matrix_destroy(mc);

  if (inverted) {
    VALUE result = matrix_to_rb_array(inverted);
    matrix_destroy(inverted);
    return result;
  } else {
    return Qnil;
  }
}
```

First, it calls `Check_Type` function from Ruby C API that'll verify that the `m` argument is a Ruby array. If the check fails, a following error will be raised:
![screely-1587664936955](https://user-images.githubusercontent.com/5732023/80133328-57588480-859d-11ea-8733-57bce2f6918a.png)

Next, it executes a function `rb_array_to_matrix` in order to convert the input ruby array to a `Matrix` type from `matrices.c` library and assign it to  `mc` variable. I'll show you the definition of `rb_array_to_matrix`  in a sec.

Having a `Matrix` struct we can use it to feed the `matrix_invert` function from the `matrices.c` library. The function will return a new `Matrix` struct which we'll assign to `inverted` variable.

Next thing we need to do is to call `matrix_destroy(mc)`  in order to free the memory after `mc` variable to prevent memory leaks. Remember, it's C, there's no Garbage Collection here ;)

If the inversion operation failed (returned a null result), we'll return `Qnil` which corresponds to Ruby's `nil` object.

If it succeeded, we'll copy the inverted matrix to a new Ruby array and free the memory after the matrix struct.

Notice that the return type of our function is `VALUE`. [`VALUE`](http://silverhammermba.github.io/emberb/c/#value) type is defined by MRI and is basically a *pointer* to a Ruby object.

### Translating Ruby arrays to C matrices (and opposite)
In previous paragraph we skipped the part about converting the arrays from Ruby to C and opposite. Let's now take a look at these functions and explain some of more interesting parts.

First, there's the `rb_array_to_matrix`, defined as follows:
```c
Matrix* rb_array_to_matrix(VALUE a) {
  long rows = RARRAY_LEN(a);
  VALUE first_row = rb_ary_entry(a, 0);
  Check_Type(first_row, T_ARRAY);
  long columns = RARRAY_LEN(first_row);

  Matrix *matrix = matrix_constructor(rows, columns);
  int row;
  int column;

  for (row = 0; row < rows; row++) {
    VALUE rb_row = rb_ary_entry(a, row);
    for (column = 0; column < columns; column++) {
      VALUE rb_row = rb_ary_entry(a, row);
      (matrix->numbers)[column][row] = NUM2DBL(rb_ary_entry(rb_row, column));
    }
  }

  return matrix;
}
```

In order to initialize the C matrix we need the number of rows and columns from the Ruby array. To get the length of a Ruby array from C level we'll use a method called `RARRAY_LEN` from Ruby's C API.

Next, we'll iterate over each cell of the array, getting the element values by calling `rb_ary_entry`  and converting the Ruby `Number`s to C `double`s by calling  the`NUM2DBL` function. The resulting number will be assign to a proper cell in the C matrix.

In `matrix_to_rb_array` we're doing pretty much the opposite:
```c
VALUE matrix_to_rb_array(Matrix *matrix) {
  VALUE result = rb_ary_new();
  int row;
  int column;

  for (row = 0; row < matrix->rows; row++) {
    VALUE rb_row = rb_ary_new();
    rb_ary_push(result, rb_row);
    for (column = 0; column < matrix->columns; column++) {
      rb_ary_push(rb_row, DBL2NUM((matrix->numbers)[column][row]));
    }
  }

  return result;
}
```

First, we initialize a new Ruby array by calling `rb_ary_new()`. We'll then fill it by copying the values from the C matrix.

In the `for` loop we're getting the value of each matrix's cell, converting it from `double` to Ruby `Number` by calling `DBL2NUM` (opposite to what we called in `rb_array_to_matrix` and appending the value to the current row by calling `rb_ary_push`.

## Step 4: Compile the extension
The extension's code is ready, but before we can use it in a Ruby program we'll have to compile the C code.

In order to do it we'll add a following rake task to our gem's `Rakefile`:
```ruby
task :compile do
  puts "Compiling extension"
  `cd ext/matrix_boost && make clean`
  `cd ext/matrix_boost && ruby extconf.rb`
  `cd ext/matrix_boost && make`
  puts "Done"
end
```

And run it by calling `bin/rake compile`.

## Step 5: Test it 🧪
Now, if we call `require "matrix_boost/extension"` in our gem we'll be able to use the C extension from the Ruby code.

We'll define our `lib/matrix_boost.rb` as follows:
```ruby
require "matrix_boost/extension"
require "matrix"

module MatrixBoost
  class << self
    # @param m [Matrix] Stdlib Matrix instance
    # @return [Matrix] Inverted matrix
    def invert(m)
      Matrix[*NativeHelpers.inv_matrix(m.to_a)]
    end
  end
end
```

Now in terminal run the console by calling `bin/console` and compare the results of [original](https://github.com/ruby/matrix/blob/master/lib/matrix.rb#L1175) Ruby method with `MatrixBoost.invert`.

![screely-1587670808096](https://user-images.githubusercontent.com/5732023/80142167-051e6000-85ab-11ea-892f-c25eca4657c8.png)

Even though the original method returned the values as `Rational`s and our gem returned `Float`s the results are equal 🎉

## Step 6: Benchmark it 🤓
Great, our extension is working as expected! Now let's benchmark it to see how much we really gained by performing the matrix operations in C.

Let's add a benchmark rake task that'll compute 1000000 inversions of a random 4-dimension matrix (most common size in 3d graphics).

```ruby
task :benchmark_inverse do
  require "matrix_boost"

  dim = 4
  n = 1000000

  puts "Benchmark inversion (dim = #{dim}, n = #{n})..."

  m = Matrix.build(dim) { rand }

  Benchmark.benchmark(Benchmark::CAPTION, 45, Benchmark::FORMAT, ">Ruby slower (%):") do |x|
    r = x.report("Ruby matrix inverse:") { n.times { m.inverse } }
    c = x.report("C matrix inverse:") { n.times { MatrixBoost.invert(m) } }

    [r / c]
  end
end
```

![screely-1587671538599](https://user-images.githubusercontent.com/5732023/80143181-b8d41f80-85ac-11ea-977b-5806fafcc523.png)

It turns out that C implementation is **~5.2x faster**! It should make a difference for programs heavily relying on matrix operations, like [ray tracers](https://en.wikipedia.org/wiki/Ray_tracing_%28graphics%29) (who writes ray tracers in Ruby is another question 🙈).

## Summary
Even though C language might be scary for some people compared to Ruby and though Ruby's C API is a bit messy and not very well documented I think it's worth giving it a shot and trying to write a simple C. Especially when a part of your program needs a performance Boost or if you want to integrate a C library into your codebase.

Full code for `matrix_boost` library can be found [here](https://github.com/Bajena/matrix_boost).

## Useful links
Knowledge about C extensions in Ruby is a bit scattered around the Internet, so I collected some links worth checking if you're struggling with writing your own extension:
- [https://guides.rubygems.org/gems-with-extensions/](https://guides.rubygems.org/gems-with-extensions/)
- [https://www.rubyguides.com/2018/03/write-ruby-c-extension/](https://www.rubyguides.com/2018/03/write-ruby-c-extension/)
- [http://clalance.blogspot.com/2011/01/writing-ruby-extensions-in-c-part-1.html](http://clalance.blogspot.com/2011/01/writing-ruby-extensions-in-c-part-1.html)
- [https://github.com/ruby/ruby/blob/master/doc/extension.rdoc](https://github.com/ruby/ruby/blob/master/doc/extension.rdoc)
- [https://blog.appsignal.com/2018/10/30/ruby-magic-building-a-ruby-c-extension-from-scratch.html](https://blog.appsignal.com/2018/10/30/ruby-magic-building-a-ruby-c-extension-from-scratch.html)
- [https://www.rubyguides.com/2019/05/ruby-ffi/](https://www.rubyguides.com/2019/05/ruby-ffi/)
- [http://silverhammermba.github.io/emberb/c/](http://silverhammermba.github.io/emberb/c/)
