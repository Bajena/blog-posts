---
title: What to do once you reach max integer value for an auto incremented column in Rails
published: true
description: If you use auto-incremented integers as primary keys in your database you may have wondered how a Ruby on Rails application behaves once the number of records in a table reaches the maximum value (`2147483647`). The chances that you haven't ever thought about it are probably quite high, same as it was for us. This oversight hit us in the best possible moment - on Friday afternoon...
tags: ruby, rails, sql
//cover_image: https://direct_url_to_image.jpg
---

If you use auto-incremented integers as primary keys in your database you may have wondered how a Ruby on Rails application behaves once the number of records in a table reaches the maximum value (`2147483647`). The chances that you haven't ever thought about it are probably quite high, same as it was for us. This oversight hit us in the best possible moment - on Friday afternoon...

If you've landed here simply out of curiosity then great - you'll learn how to avoid such mistakes in the future. If you're here because your application just started throwing errors then keep reading - you'll learn how to fix the problem easily.

# What happened?
On Friday afternoon we started seeing the following errors when trying to create records in a mysql table (let's call it `records` along this article):
```ruby
irb(main):007:0> Record.create(name: 'test-abc')
Traceback (most recent call last):
        2: from (irb):7
        1: from (irb):7:in `rescue in irb_binding'
ActiveRecord::RecordNotUnique (Mysql2::Error: Duplicate entry '2147483647' for key 'PRIMARY': INSERT INTO `records` (`name`, `created_at`, `updated_at`) VALUES ('test-abc', '2021-01-29 16:00:22', '2021-01-29 16:00:22'))
```

As you can see the error happens because MySQL tries to insert another record with the same ID as the already existing one. You can also verify that the auto increment value is "stuck" at max integer value by running:
```ruby
ActiveRecord::Base.connection.execute("SELECT `AUTO_INCREMENT` FROM INFORMATION_SCHEMA.TABLES WHERE TABLE_NAME = 'records'").to_a
=> [[2147483647]]
```

# How do I fix it?
Fortunately the fix is really simple - all you need to do is run a Rails migration to change the `id` column type from a basic 4 byte `:integer` to a bigger, 8 byte `:bigint` type:

```ruby
class ChangeRecordsIdToBigint < ActiveRecord::Migration[5.2]
  def change
    change_column :records, :id, :bigint, null: false, unique: true, auto_increment: true
  end
end
```
