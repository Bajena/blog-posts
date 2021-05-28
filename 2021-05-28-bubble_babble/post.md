---
title: Ruby Hash trick for creating an in-memory cache
published: false
description:
tags:
cover_image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/51g3bt9o9pqj25wsdkyq.jpeg
---

In this short post I'd like to show you a smart trick for memoizing results of computations in memory. Sometimes you need to store results of multiple operations in memory for reuse and typically it's done using a `Hash` instance.

E.g. let's say that we have a method called for checking whether a Star Wars character is from the dark side:
```ruby
def dark_side?(character_name)
  StarWars::Character.find_by(name: character_name).dark_side?
end
```

The method is heavy, because it needs to run a DB query in order to get a result for a given input, so if we know that there might be a need for calling it many times with the same `character_name` it might make sense to store the result for future use.

Here's one possibility of how the results can be memoized:
```ruby
def dark_side?(character_name)
  @dark_side_cache ||= {}

  @dark_side_cache[character_name] = StarWars::Character.find_by(name: character_name).dark_side? unless @dark_side_cache.key?(character_name)

  @dark_side_cache[character_name]
end
```

However, Ruby's `Hash` has a nice [constructor](https://ruby-doc.org/core-3.0.1/Hash.html#method-c-new) variant that allows passing a block.

Check this solution out, isn't it a pure Ruby beauty? 💅
```ruby
def dark_side?(character_name)
  @dark_side_cache ||= Hash.new do |hash, char_name|
    hash[char_name] = StarWars::Character.find_by(name: char_name).dark_side?
  end
  @dark_side_cache[character_name]
end
```
