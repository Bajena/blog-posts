---
title: Ruby TIL: Bubble babble encoding algorithm
published: true
tags: ruby, algorithms
cover_image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/g8ffzicm49l6637kipov.jpeg
---

Today I'd like to share an interesting finding from Ruby's standard library. The method is called `bubblebabble` and lives in the [Digest module](https://ruby-doc.org/stdlib-2.6.2/libdoc/digest/rdoc/Digest.html#method-c-bubblebabble).

The main point of this method is to create digests that look and sound similarly to human words, so that they can be recognized and visually compared more easily than hexadecimal digests.

You can either use the method directly on `Digest` module (it'll run the encoding algorithm directly on the provided string) or on a specific digest class, like `Digest::SHA1` (it'll then run the algorithm on a SHA1 digest of the input string):
```ruby
irb(main):001:0> require 'digest/bubblebabble'
=> true
irb(main):002:0> Digest.bubblebabble('a')
=> "ximex"
irb(main):003:0> Digest::SHA1.bubblebabble('a')
=> "xociz-lynaf-livip-huniz-samah-tolat-sivov-pipiv-petel-kynyr-mexix"
```

### Some history

The bubble babble encoding was invented in 2000 by Atti Huima. You can find the original document with algorithm's description at http://web.mit.edu/kenta/www/one/bubblebabble/spec/jrtrjwzi/draft-huima-01.txt.

According to the author, the name combines the name of a video game classic [Bubble Bobble](https://en.wikipedia.org/wiki/Bubble_Bobble) and the fact that the generated strings can be pronounced but sound like babbling.

### Is this used anywhere for real?
Yes, this funny encoding method is used by the SSH2 suite to display easy-to-remember key fingerprints. The key is converted into a textual form, digested using SHA1, and run through bubble babble to create the key fingerprint.

You can test it out yourself by running  `ssh-keygen` with `-B` option (http://man.openbsd.org/ssh-keygen.1#B).
