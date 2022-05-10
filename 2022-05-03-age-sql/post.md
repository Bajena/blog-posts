---
title: Think twice before using AGE in PotgreSQL
published: true
description: In this post you'll learn about a subtle difference between Postgres' time calculation functions that bit me in the ass recently.
tags: sql, time
cover_image: https://images.unsplash.com/photo-1621252179027-94459d278660?ixlib=rb-1.2.1&ixid=MnwxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8&auto=format&fit=crop&w=2940&q=80
---

Recently my friend at [Productboard](https://productboard.com/) noticed an interesting bug in one of our services. For some reason our code responsible for calculating how many days our customers' features spend in certain states (Idea, Discovery, Delivery, etc) in some cases would give us wrong results.

For example: even though you can tell by looking at the calendar that the days difference between `2021-02-28 00:00:00` and `2022-05-03 00:00:00` is `429 days`, our code would return `428.25 days` for that timespan.

After some research it turned out that the problem was caused by the fact that we were using PostgreSQL's `AGE` function for calculating days difference.

According to PostgreSQL's [docs](https://www.postgresql.org/docs/current/functions-datetime.html) `AGE` function calculates `“symbolic” result that uses years and months, rather than just days`.
It wasn't super clear to me what "symbolic" means, so I started digging a bit and I realised that the purpose of `AGE` is not to calculate a precise time difference, but rather calculate age the way we, humans  do it. Instead of subtracting UNIX timestamps like computers do, we subtract each component of the date and then adjust the negative values.

So, in the previous example (2021-02-28 - 2022-05-03):
- Years difference is `1`
- Months difference is `3`
- Days difference is `-25`, so we subtract 1 month, check how many days are left in February (0) and then add the days from May (3). Eventually we end up with `3` days.

We end up with `1 year, 2 months and 3 days`. Now, why does Postgres return `428.25` here?

It's because:
- The number of days returned by `AGE`  is `365.25` - it's an average number of days in a year when we take leap years into consideration.
- Postgres uses `30` as number of days in each month

So, now it all makes sense - `1 year, 2 months and 3 days` leave us with `365.25 + 2 * 30 + 3 = 428.25` days 🤓.

Fortunately the solution to our problem was extremely simple - we just had to **replace the `AGE` function with the subtraction operator**.
In order to show you the difference, here's a query that I ran in an online [postgres query tool](https://extendsclass.com/postgresql-online.html):

```sql
SELECT
	EXTRACT(epoch FROM ('2022-05-03 00:00:00'::timestamp - '2021-02-28 00:00:00'::timestamp)) / (3600 * 24) as subtraction_days,
	EXTRACT(epoch FROM AGE('2022-05-03 00:00:00'::timestamp, '2021-02-28 00:00:00'::timestamp)) / (3600 * 24) as age_days
FROM "current_schema"()
```

<img width="1209" alt="image" src="https://user-images.githubusercontent.com/5732023/167474336-a6d67989-46b6-487c-a0a9-343826e4dbac.png">

The moral of this story is that time calculations are an extremely sensitive matter that can be approached by computers and humans differently. Fortunately PostgreSQL has all of the possible approaches covered. We, as developers, just have to understand our use case, read the documentation and think twice in order to pick an appropriate one.
