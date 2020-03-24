---
title: Solving "invalid byte sequence in UTF-8" errors in ruby
published: false
description: Learn what "UTF-8 byte sequences" are, why they can be invalid and how to solve this problem in Ruby.
tags: ruby rails tutorial
---

If you've landed here it means you've been hit by this message in your program. In this post I'll quickly introduce you to what "UTF-8 byte sequences" are, why they can be invalid and how to solve this problem in Ruby.

### Short introduction to UTF-8 and other encodings
UTF-8 is, as explained in [Wikipedia](https://en.wikipedia.org/wiki/UTF-8), is a set codepoints (in simple words: numbers representing characters). Every character in UTF-8 is a *sequence* of 1 up to 4 bytes.

Apart from UTF-8 there are also other encodings like *ISO-8859-1* or *Windows-1252* - you may have seen these names before in your programming career. These encodings cover a big set of characters, including special latin characters etc.

Now, even though UTF-8 covers a huge set of characters as well it is not 100% compatible with the above mentioned encodings. Take a look at the following picture:
 - Both UTF-8 and ISO-8859-1 are ASCII compatible - the include the same codepoints for digits and latin alphabet
 - UTF-8 includes characters not present in ISO-8859-1, like the rocket emoji 🚀
 - Both UTF-8 and ISO-8859-1 include "Å" characters, but these letters are defined using different codepoints - *c385* in UTF-8 and *c5* in ISO-8859-1

![enter image description here](https://user-images.githubusercontent.com/5732023/77397158-5cab8f00-6da5-11ea-85a4-55deac96f280.png)
*Encodings compatibility*

#### Why does an UTF-8 invalid byte sequence error happen?
Ruby's default encoding since 2.0 is UTF-8. This means that Ruby will treat any string you input as an UTF-8 encoded string unless you tell it explicitly that it's encoded differently.

Let's use the `Å` character from the introductory diagram to present this problem.
Imagine you have a file `file.txt` containing a following string: `"vandflyver \xC5rhus"`.  As you already know `C5` codepoint corresponds to `Å` in *ISO-8859-1* and isn't present in UTF-8 encoding. Ruby however doesn't know that the original encoding of the file is ISO-8859-1 and will by default interpret it as UTF-8.

So, the following operation will result in the infamous "UTF-8 Invalid byte sequence":
```ruby
irb(main):079:0> File.write("file.txt", "vandflyver \xC5rhus")
=> 16
irb(main):080:0> open("file.txt", "r") { |io| io.read.split }
Traceback (most recent call last):
        7: from /Users/bajena/.rbenv/versions/2.6.1/bin/irb:23:in `<main>'
        6: from /Users/bajena/.rbenv/versions/2.6.1/bin/irb:23:in `load'
        5: from /Users/bajena/.rbenv/versions/2.6.1/lib/ruby/gems/2.6.0/gems/irb-1.0.0/exe/irb:11:in `<top (required)>'
        4: from (irb):80
        3: from (irb):80:in `open'
        2: from (irb):80:in `block in irb_binding'
        1: from (irb):80:in `split'
ArgumentError (invalid byte sequence in UTF-8)
```

The "invalid UTF-8 byte sequence" here is our "Å" (C5) character as it's not present in UTF-8. Fortunately there are a few ways to solve this problem.

#### Solution 1 - Provide a source encoding
If you know the encoding in which the file was originally written then all you have to do is to provide the encoding name when reading the input file. Ruby will automatically handle the character conversion for you:
```ruby
irb(main):098:0> s = open("file.txt", "r:ISO-8859-1:UTF-8") { |io| io.read.split }
=> ["vandflyver", "Århus"]
irb(main):099:0> s[1][0].unpack("H*")
=> ["c385"]
```

In the last line I've used [String.unpack](https://ruby-doc.org/core-2.7.0/String.html#method-i-unpack) method to print the converted character's codepoint. As you can see it got correctly converted from `C5` to `C385` 🎉

#### Solution 2 - `String.encode` method
In many cases you won't be that lucky to know the original encoding of the file. In this case [String.encode](https://ruby-doc.org/core-2.7.0/String.html#method-i-encode) method comes in handy. You can use it to skip invalid UTF-8 characters or replace them with a string of your choice.

Check out the following examples:
```ruby
irb(main):102:0> open("file.txt", "r") { |io| io.read.encode("UTF-8", invalid: :replace) }
=> "vandflyver �rhus"
irb(main):103:0> open("file.txt", "r") { |io| io.read.encode("UTF-8", invalid: :replace, replace: "") }
=> "vandflyver rhus"
```

May not be beautiful, but it's still better than crashing the app, right?

#### Solution 3 - Detect source encoding
In case you don't know the source encoding and don't want to skip the invalid characters you can use a character encoding detection gem called [charlock_holmes](https://github.com/brianmario/charlock_holmes).
It'll analyze the string and provide you with the most probable source encoding and guess confidence (also a language code as a bonus :P).

Check it out in action:
```ruby
irb(main):015:0> s = open("file.txt", "r") { |io| io.read }
=> "vandflyver \xC5rhus"
irb(main):016:0> d = CharlockHolmes::EncodingDetector.detect(s)
=> {:type=>:text, :encoding=>"ISO-8859-1", :ruby_encoding=>"ISO-8859-1", :confidence=>70, :language=>"nl"}
irb(main):017:0> s.encode("UTF-8", d[:encoding], invalid: :replace, replace: "")
=> "vandflyver Århus"
```

#### Summary
First of all I hope that this post helped you to solve the Ruby issue you had. On the other hand I'm sure that also you've learned something useful. String encodings can sometimes be really f***ed up, so it's really worth knowing what's going on under the hood.
