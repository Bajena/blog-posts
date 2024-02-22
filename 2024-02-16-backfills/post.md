
# Mastering Large Backfill Migrations in Rails

Migrating large datasets within a Rails application can be a daunting task. We've recently learned that the hard way at Productboard when trying to backfill data from a 230GB PostgreSQL table into a new one. It took us a few days, 2 database incidents, and a lot of nerves to get the task done. Was it perfect? Definitely not, but the upside of this process is that we've learned a lot, and now we can share it with you, dear reader :)

In this post, I'll explore effective strategies and best safety practices for tackling big data migrations using Rails and Sidekiq. Hopefully, it'll help you avoid repeating the same mistakes we made during our adventure.

## Why Use Rails at All?

Some could argue whether using Rails is necessary at all if the only thing we need to do is to extract the data from one PostgreSQL table into another. The thing is, this migration involved a lot of business logic, which is very well encapsulated in our application and would be very time-consuming and error-prone to duplicate into SQL, so we quickly abandoned this idea in favor of loading the records in the application, doing necessary transformations, and storing the data in the new format.

One additional argument for using Rails was the ability to use Sidekiq to parallelize the workload and process each of our DB tenants (Workspaces in our case) in a separate job.

## Rails Tips

ActiveRecord is a great pattern which abstracts a lot of DB complexity from the programmer. However, when manipulating large data sets without a deep understanding of what's hapenning under the hood, it's easy to shoot yourself in the foot.

Below, I present a few tips that can help you avoid some headscratches :)

### Use DB Replica

In the first iteration we've run our migration on the primary database that's used for most important user flows - the ones involving writes. There's nothing worse than e.g. writing a long comment just to see that saving failed when pressing ENTER.

That's why in the next iteration instead of the primary database, we utilise a read replica for fetching the data.

ActiveRecord provides an easy way of querying data from read replicas. Just wrap your selects in  `connected_to` method as in the example below:
```ruby
scope = ActiveRecord::Base.connected_to(role:  :reading, prevent_writes:  true) { SourceModel.all }

# Process the record set as always...
```

### Use `find_each` / `in_batches` for fetching the data
This tip is quite basic for advanced Rails developers, but still worth keeping in mind: when processing large amounts of records always make sure to query your tables in batches. You really don't want to run a separate DB query for each of your 10 million records in the table ;)

Rails provides two methods for this purpose:
- `in_batches` - loads records from table in batches (by default 1000 records per batch) and yields each batch in a block
- `find_each` - loads records from table in batches and yields each record separate in a block

```ruby
SourceModel.all.in_batches do |records|
  # Process batch of records at once
end

SourceModel.all.find_each do |record|
  # Process a single record
end
```

**Warning:** One commonly overlooked thing about these batch methods is that YOU CAN'T SORT the result set. Rails will always use the primary key for iterating. I've been hit by this problem quite a few times already 🙈

### Use Rails' `insert_all` methods to speed up writes

Similarly to the read methods listed above, Rails supports batch inserts and upserts (from version 6.0.0).

`insert_all` allows you to bulk insert records, significantly speeding up the process compared to individual `INSERT` statements. This method is ideal for quickly populating the new table with transformed data.

One great thing about this method is that in case if e.g. you're restarting the migration and you already have some records inserted in the table with unique index applied `insert_all` will automatically skip duplicate inserts. If you have multiple unique constraints on the same table, you can specify the desired one by providing `unique_by` option.

```ruby
models_data = [
  { first_name: "Foo", last_name: "Bar" },
  { first_name: "Eloquent Ruby", last_name: "Russ" }
]

DestinationModel.insert_all(models_data)
```

**Warning:** There are two caveats to using this method:
1. Keep in mind that the array (`models_data` in the example above) you're using for `insert_all` needs to have at least 1 element. Otherwise the call will crash.
2. All the hashes in the array have to have the same set of attributes, so even if e.g. one of your columns has some default value defined, you'll still have to provide that attribute in the hash.

### Prevent N+1 queries when loading associated records

Often it happens that in your migrations you need access to multiple related records. For example when migrating `Comment`s you might also need their `author` information.

Naive implementation could look like this:
```ruby
Comment.all.find_each do |comment|
  next if comment.author.inactive?

  # Other migration logic
end
```

Can you spot the problem already? Calling `comment.author` will fire a separate SQL query for each comment.

This can be solved by calling `includes(:author)` on your ActiveRecord query:
```ruby
Comment.all.find_each do |comment|
  next if comment.author.inactive?

  # Other migration logic
end
```

Sometimes  calling `includes` might not be possible, for example when comments and authors exist in different databases.

In such cases, you can simply fire a separate, explicit query to load authors and use `index_by(&:id)` to speed up accessing them by IDs in the memory:

```ruby
Comment.all.in_batches do |comments_batch|
  authors = User.where(id: comments_batch.map(&:author_id).index_by(&:id)

  comments_batch.each do |comment|
    author = authors[comment.author_id]
    next if author.inactive?

	# Other migration logic
  end
end
```

### Use `select` to Load Only Necessary Columns

Rails by default loads all the columns when querying the table. It can be unnecessarily heavy e.g. when the table has `text` fields with potentially long contents.

When you only need certain fields from the database, use `select` to load just those columns. This reduces memory usage and speeds up the data loading process.

```ruby
SourceModel.all.select(:id, :name).find_each do |model|
  # migration logic
end
```

## Sidekiq Tips

When processing large data sets in Ruby, Sidekiq can be extremely helpful for splitting the work into smaller, parallel chunks. Below you can find a few tips that can help you making your migrations bulletproof.

### Use `push_bulk` to schedule effectively

Sometimes you'll need to push lots of jobs to Sidekiq in order to process your migration. Scheduling each job using `perform_async` method is ineffective as it causes a lot of unnecessary round-trips to Redis.

Fortunately Sidekiq has a way to push jobs in batches. You can do it by using `push_bulk` method as in the example below:

```ruby
class MigrationJob
  include Sidekiq::Job

  def perform(record_id)
  end
end

class LoaderWorker
  include Sidekiq::Job

  SIZE = 1000

  def perform(idx)
    # assume we want to create a job for each of 200,000 database records
    # query for our set of 1000 records

    SourceModel.all.in_batches(of: 1000) do |batch|
      array_of_args = batch.map { |record| [record.id] }

	  # push 1000 jobs in one network call to Redis, saves 999 round trips
	  Sidekiq::Client.push_bulk('class' => MigrationJob, 'args' => array_of_args)
	end
  end
end
```

It's also worth mentioning that in version 6.3.0 Sidekiq  [introduced  `perform_bulk`](https://github.com/sidekiq/sidekiq/pull/5042)  method.

The code below is equivalent to the one above. It looks much cleaner

```ruby
class LoaderWorker
  include Sidekiq::Job

  def perform
    SourceModel.all.in_batches(of: 1000) do |batch|
      array_of_args = batch.map { |record| [record.id] }

      MigrationJob.perform_bulk(array_of_args)
	end
  end
end
```


### Use Sidekiq Batches to Monitor the Progress

Sidekiq Batches allow you to group jobs and monitor their overall progress. This is especially useful for large migrations, as it provides visibility into the process and helps identify issues early.


### Leverage Sidekiq::Iteration Gem

For processing large datasets in batches, the [sidekiq-iteration](https://github.com/fatkodima/sidekiq-iteration) gem is invaluable. It allows you to iterate over a collection and process it in manageable chunks.

What's great is that the gem takes care of handling job interruptions caused e.g. by new version deploys or DB connectivity issues. When a situation like this takes place, `sidekiq-iteration` saves the iteration cursor position and reschedules the job.

What you need to do is to include `SidekiqIteration::Iteration` module, define the collection to iterate over (`build_enumerator` method) and define your logic for processing a batch of records (`each_iteration` method):

```ruby
 class BackfillSourceRecordsJob
   include  Sidekiq::Worker
   include SidekiqIteration::Iteration

   # After 5 minutes the job will be rescheduled with current cursor position
   self.max_job_runtime = 5.minutes

   def build_enumerator(space_id, cursor: nil)
     scope = SourceRecord.where(space_id:)

     active_record_batches_enumerator(scope, cursor:, batch_size: 1000)
   end

   def each_iteration(records, *)
     # Process a batch of records
   end
end
```

Then simply schedule the job as you'd schedule any sidekiq job:
```ruby
BackfillSourceRecordsJob.perform_async(123)
```

### Control the number of concurrently processed jobs



#### Option 1: Use `perform_in`

P.S. `push_bulk` supports `at` option

#### Option 2:  Separate Kubernetes pods

#### Option 3: Use `Sidekiq::Limiter`

### Use Sidekiq::Limiter and `perform_in` to Limit the Amount of Concurrent Workers

Control your system's load by using `Sidekiq::Limiter` to limit the concurrency of specific jobs and `perform_in` to schedule jobs in the future, spreading out the workload over time.

### Job Idempotency

Ensure your jobs are idempotent so that retrying or duplicating jobs does not affect the integrity of your data. This is crucial for maintaining data consistency across retries and failures.


## General Tips

### Implement a Kill-Switch

Always have a kill-switch mechanism in place to immediately halt the migration if something goes wrong. This could be a feature toggle or a simple conditional check that stops the process based on external input.

### Logging and Monitoring

Implement thorough logging and set up monitoring for the migration process. Tools like DataDog or Splunk can provide real-time insights into the migration's performance and help quickly identify any issues.

### Use DB Indexes

Ensure your database tables, especially the new ones, are properly indexed based on the queries you'll run. Proper indexing is critical for maintaining performance during and after the migration.
