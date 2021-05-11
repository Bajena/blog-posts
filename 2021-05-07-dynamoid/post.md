---
title: How to setup Dynamoid in Ruby projects to simplify DynamoDB interactions
published: false
description: Working with DynamoDB can be painful sometimes. This article will ease your pain by introducing the Dynamoid gem to your project.
tags: ruby, rails, dynamodb
//cover_image: https://direct_url_to_image.jpg
---

Working with DynamoDB in Ruby can be painful compared to how easy it is to configure and use SQL databases via ActiveRecord. Fortunately there's a way to speed things up thanks to [Dynamoid](https://github.com/Dynamoid/dynamoid) gem which introduces ActiveRecord-like abstractions for interacting with DynamoDB.

In this article I'd like to show you how you can fully setup your project to work with Dynamoid.

# Local DynamoDB setup
Before we start with Dynamoid itself I'd recommend you to setup DynamoDB locally.
First, start [dynamodb-local](https://hub.docker.com/r/amazon/dynamodb-local/) Docker image.

Next you can play around with your local DDB instance by using the [NoSQL Workbench](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/workbench.html) from Amazon:
![image](https://user-images.githubusercontent.com/5732023/116545482-c6736200-a8f0-11eb-840a-c39b26d34982.png)

# Dynamoid installation and configuration
After adding `gem "dynamoid"` to your Gemfile you'll need to configure the gem. If you're using Rails I suggest doing it in an initializer (e.g. in `config/initializers/dynamoid.rb` file).

You can find the full list of options in Dynamoid's README but here's the config I'm using
```ruby
require "dynamoid"

Dynamoid.configure do |config|
  # Local DDB endpoint:
  config.endpoint = "http://localhost:8000"

  # Fake AWS credentials for local development purposes:
  config.access_key = "abc"
  config.secret_key = "xyz"
  config.region = "localhost"

  # Do not add prefixes to table names. By default dynamoid uses `dynamoid_#{application_name}_#{environment}` prefix:
  config.namespace = nil

  # Tells Dynamoid to use exponential backoff for batch operations (BatchGetItem, BatchPutItem)
  config.backoff = { exponential: { base_backoff: 0.2.seconds, ceiling: 10 } }

  # Do not add timestamps (created_at, updated_at) fields by default
  config.timestamps = false

  # Store datetimes as ISO-8601 strings by default. Otherwise UNIX timestamps will be used.
  config.store_datetime_as_string = true
end
```

# Defining models
Let's say we're building an application for storing lists of flashcards, similar to [Flashcard Genius](http://flashcard-genius.com/).
In such app there could be a model called `FlashcardList` with:
- `user_id` partition key
- `list_id` sort key
- `name` attribute
- `created_at` attribute

in order to reflect that in Dynamoid model you can define a following class:

```ruby
class FlashcardList
  include Dynamoid::Document

  table(
    name: "flashcard-lists",
    # Defines getters/setters for `user_id` field as well
    key: :user_id
  )

  # Defines getters/setters for `list_id` field as well
  range :list_id, :string

  field :name, :string
  field :created_at, :datetime
end

```

That's it! Now you can read/update your data in a manner similar to ActiveRecord:
```ruby
[6] pry(main) FlashcardList.create_table(sync: true)
[7] pry(main)> FlashcardList.create(name: "My list", user_id: 1, list_id: "123", created_at: Time.zone.now)
=> #<FlashcardList:0x00007fb2c1a4ff08
 @_touch_record=nil,
 @associations={},
 @attributes={:user_id=>"1", :list_id=>"123", :name=>"My list", :created_at=>Thu, 29 Apr 2021 14:33:15 +0000},
 @attributes_before_type_cast={:user_id=>1, :list_id=>"123", :name=>"My list", :created_at=>Thu, 29 Apr 2021 14:33:15.977495000 UTC +00:00},
 @changed_attributes={},
 @errors=#<ActiveModel::Errors:0x00007fb2c1a4f210 @base=#<FlashcardList:0x00007fb2c1a4ff08 ...>, @errors=[]>,
 @new_record=false,
 @previously_changed={"user_id"=>[nil, "1"], "list_id"=>[nil, "123"], "name"=>[nil, "My list"], "created_at"=>[nil, Thu, 29 Apr 2021 14:33:15 +0000]},
 @validation_context=nil>
[8] pry(main)> FlashcardList.create(name: "My list 2", user_id: 2, list_id: "234", created_at: Time.zone.now)
=> #<FlashcardList:0x00007fb2c1bb5dc0
 @_touch_record=nil,
 @associations={},
 @attributes={:user_id=>"2", :list_id=>"234", :name=>"My list 2", :created_at=>Thu, 29 Apr 2021 14:33:25 +0000},
 @attributes_before_type_cast={:user_id=>2, :list_id=>"234", :name=>"My list 2", :created_at=>Thu, 29 Apr 2021 14:33:25.832570000 UTC +00:00},
 @changed_attributes={},
 @errors=#<ActiveModel::Errors:0x00007fb2c1bb50c8 @base=#<FlashcardList:0x00007fb2c1bb5dc0 ...>, @errors=[]>,
 @new_record=false,
 @previously_changed={"user_id"=>[nil, "2"], "list_id"=>[nil, "234"], "name"=>[nil, "My list 2"], "created_at"=>[nil, Thu, 29 Apr 2021 14:33:25 +0000]},
 @validation_context=nil>
[9] pry(main)> FlashcardList.import([{ name: "My list 3", user_id: 3, list_id: "456", created_at: Time.zone.now}, { name: "My list 4", user_id: 3, list_id: "567", created_at: Time.zone.now }])
=> [#<FlashcardList:0x00007fb2c134e6f0
  @associations={},
  @attributes={:user_id=>"3", :list_id=>"456", :name=>"My list 3", :created_at=>Thu, 29 Apr 2021 14:34:09 +0000},
  @attributes_before_type_cast={:user_id=>3, :list_id=>"456", :name=>"My list 3", :created_at=>Thu, 29 Apr 2021 14:34:09.230243000 UTC +00:00},
  @changed_attributes={"user_id"=>nil, "list_id"=>nil, "name"=>nil, "created_at"=>nil},
  @new_record=false>,
 #<FlashcardList:0x00007fb2c134da70
  @associations={},
  @attributes={:user_id=>"3", :list_id=>"567", :name=>"My list 4", :created_at=>Thu, 29 Apr 2021 14:34:09 +0000},
  @attributes_before_type_cast={:user_id=>3, :list_id=>"567", :name=>"My list 4", :created_at=>Thu, 29 Apr 2021 14:34:09.230263000 UTC +00:00},
  @changed_attributes={"user_id"=>nil, "list_id"=>nil, "name"=>nil, "created_at"=>nil},
  @new_record=false>]
[10] pry(main)> FlashcardList.all
=> #<Enumerator::Lazy: ...>
[11] pry(main)> FlashcardList.all.to_a
=> [#<FlashcardList:0x00007fb2a58ff458 @associations={}, @attributes={:user_id=>"1", :list_id=>"123", :name=>"My list", :created_at=>Thu, 29 Apr 2021 14:33:15 +0000}, @attributes_before_type_cast={:user_id=>"1", :list_id=>"123", :name=>"My list", :created_at=>Thu, 29 Apr 2021 14:33:15 +0000}, @changed_attributes={}, @new_record=false, @previously_changed={}>,
 #<FlashcardList:0x00007fb2a50c53c8
  @associations={},
  @attributes={:user_id=>"3", :list_id=>"456", :name=>"My list 3", :created_at=>Thu, 29 Apr 2021 14:34:09 +0000},
  @attributes_before_type_cast={:user_id=>"3", :list_id=>"456", :name=>"My list 3", :created_at=>Thu, 29 Apr 2021 14:34:09 +0000},
  @changed_attributes={},
  @new_record=false,
  @previously_changed={}>,
 #<FlashcardList:0x00007fb2a780f230
  @associations={},
  @attributes={:user_id=>"3", :list_id=>"567", :name=>"My list 4", :created_at=>Thu, 29 Apr 2021 14:34:09 +0000},
  @attributes_before_type_cast={:user_id=>"3", :list_id=>"567", :name=>"My list 4", :created_at=>Thu, 29 Apr 2021 14:34:09 +0000},
  @changed_attributes={},
  @new_record=false,
  @previously_changed={}>,
 #<FlashcardList:0x00007fb2a50bc8e0
  @associations={},
  @attributes={:user_id=>"2", :list_id=>"234", :name=>"My list 2", :created_at=>Thu, 29 Apr 2021 14:33:25 +0000},
  @attributes_before_type_cast={:user_id=>"2", :list_id=>"234", :name=>"My list 2", :created_at=>Thu, 29 Apr 2021 14:33:25 +0000},
  @changed_attributes={},
  @new_record=false,
  @previously_changed={}>]
  [12] pry(main)> FlashcardList.delete_table
=> "flashcard-lists"
  ```

Here's a corresponding log of DDB queries produced by the commands above. As you can see Dynamoid freed us from writing these huge ass queries which tidies up the code a lot.

```
D, [2021-04-29T16:32:23.509266 #43231] DEBUG: [Aws::DynamoDB::Client 200 0.065601 0 retries] create_table(table_name:"flashcard-lists",key_schema:[{attribute_name:"user_id",key_type:"HASH"},{attribute_name:"list_id",key_type:"RANGE"}],attribute_definitions:[{attribute_name:"user_id",attribute_type:"S"},{attribute_name:"list_id",attribute_type:"S"}],billing_mode:"PROVISIONED",provisioned_throughput:{read_capacity_units:100,write_capacity_units:20})

D, [2021-04-29T16:32:23.510232 #43231] DEBUG: (68.0 ms) CREATE TABLE
D, [2021-04-29T16:33:16.025411 #43231] DEBUG -- [Aws::DynamoDB::Client 200 0.046285 0 retries] put_item(table_name:"flashcard-lists",item:{"user_id"=>{s:"1"},"list_id"=>{s:"123"},"name"=>{s:"My list"},"created_at"=>{s:"2021-04-29T14:33:15+00:00"}},expected:{"user_id"=>{exists:false},"list_id"=>{exists:false}})

D, [2021-04-29T16:33:16.026200 #43231] DEBUG -- (47.79 ms) PUT ITEM - ["flashcard-lists", {:user_id=>"1", :list_id=>"123", :name=>"My list", :created_at=>"2021-04-29T14:33:15+00:00"}, {}]
D, [2021-04-29T16:33:25.881593 #43231] DEBUG -- [Aws::DynamoDB::Client 200 0.047884 0 retries] put_item(table_name:"flashcard-lists",item:{"user_id"=>{s:"2"},"list_id"=>{s:"234"},"name"=>{s:"My list 2"},"created_at"=>{s:"2021-04-29T14:33:25+00:00"}},expected:{"user_id"=>{exists:false},"list_id"=>{exists:false}})

D, [2021-04-29T16:33:25.882293 #43231] DEBUG -- (49.18 ms) PUT ITEM - ["flashcard-lists", {:user_id=>"2", :list_id=>"234", :name=>"My list 2", :created_at=>"2021-04-29T14:33:25+00:00"}, {}]
D, [2021-04-29T16:34:09.341679 #43231] DEBUG -- [Aws::DynamoDB::Client 200 0.109989 0 retries] batch_write_item(request_items:{"flashcard-lists"=>[{put_request:{item:{"user_id"=>{s:"3"},"list_id"=>{s:"456"},"name"=>{s:"My list 3"},"created_at"=>{s:"2021-04-29T14:34:09+00:00"}}}},{put_request:{item:{"user_id"=>{s:"3"},"list_id"=>{s:"567"},"name"=>{s:"My list 4"},"created_at"=>{s:"2021-04-29T14:34:09+00:00"}}}}]},return_consumed_capacity:"TOTAL",return_item_collection_metrics:"SIZE")

D, [2021-04-29T16:34:09.342284 #43231] DEBUG -- (111.47 ms) BATCH WRITE ITEM - ["flashcard-lists", [{:user_id=>"3", :list_id=>"456", :name=>"My list 3", :created_at=>"2021-04-29T14:34:09+00:00"}, {:user_id=>"3", :list_id=>"567", :name=>"My list 4", :created_at=>"2021-04-29T14:34:09+00:00"}]]
D, [2021-04-29T16:35:00.258852 #43231] DEBUG -- (0.04 ms) SCAN - ["flashcard-lists", {}]
D, [2021-04-29T16:35:00.283511 #43231] DEBUG -- [Aws::DynamoDB::Client 200 0.023691 0 retries] describe_table(table_name:"flashcard-lists")

D, [2021-04-29T16:35:00.305328 #43231] DEBUG -- [Aws::DynamoDB::Client 200 0.020683 0 retries] scan(table_name:"flashcard-lists",scan_filter:{},attributes_to_get:nil)

D, [2021-04-29T16:36:50.309800 #43231] DEBUG -- [Aws::DynamoDB::Client 200 0.057094 0 retries] delete_table(table_name:"flashcard-lists")

D, [2021-04-29T16:36:50.310466 #43231] DEBUG -- (58.15 ms) DELETE TABLE
```


# Dev/test environment setup

## Dev setup
First create a following file:
```ruby
class Dev::DynamoidSetup
  # List all your Dynamoid models here:
  DYNAMOID_MODELS = [FlashcardList].freeze

  class << self
    def create_tables
      check_env!

      DYNAMOID_MODELS.each do |m|
        puts "Creating table: #{m.table_name}..."
        m.create_table(sync: true)
      end
    end

    def delete_tables
      check_env!

      DYNAMOID_MODELS.each do |m|
        m.delete_table
      end
    end

    def check_env!
      return if Rails.env.development? || Rails.env.test?

      raise "Do not run on production envs!"
    end
  end
end
```

Now you can use `Dev::DynamoidSetup.create_tables` to automatically setup all required tables in dev environment.

You could optionally wrap it in a rake task and call it like `bundle exec rake setup_dynamodb`:
```ruby
desc "Sets up DDB tables for local development"
task setup_dynamodb: :environment do
  Dev::DynamoidSetup.create_tables
end
  ```

## RSpec setup
Once you have your development environment ready it's time to configure the tests.

First in `spec_helper.rb` add a following hook:
```ruby
config.before(:suite) do
  Dev::DynamoidSetup.delete_tables
end
  ```

This'll make sure there are no leftover tables from previous suite runs. In CI you could optionally skip this line as DynamoDB container should always be fresh there.

Then in order to manipulate the data in tables during the specs - I suggest having shared contexts for each table and including them only in the specs that really need access to a given table.

Check out e.g. this one:
```ruby
RSpec.shared_context "uses_flashcard_lists_table" do
    around do |example|
      FlashcardList.create_table(sync: true)
      example.run
      FlashcardList.delete_table
    end
end

RSpec.describe FlashcardList do
  include_context "uses_flashcard_lists_table"

  it "stores data in the DDB table" do
    expect { FlashcardList.create(name: "My list 3", user_id: 3, list_id: "456") }.to change(FlashcardList, :count).by(1)
  end
end
```

