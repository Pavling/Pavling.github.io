---
layout: post
title: Limiting ActiveRecord results
description: Or, how to trick Rails into thinking there are fewer records in your DB than there really are.
<!-- modified: 2015-02-23 -->
tags: [rails, activerecord, benchmarking]
image:
  feature: limiting-activerecord-results.jpg
  credit:
  creditlink: 
---

### TL; DR

No, adding `.limit(some_number)` won't cut it for this problem. Quick, [go here](#v4).


## What's the scenario?

In my current role, I'm working with fairly large data-sets (not really "big data", but no small shakes either). I'm writing reporting functionality on a PostgreSQL-backed Rails app -- migrating existing functionality from a C# .Net app (I know... the thought-process must've gone something like this: "Let's manipulate millions of records to generate totals and statistics. What framework is really not designed for that..." :-)

The main data table has around 2-million records, with a handful of associations on the base model (let's call it `Result`). Users will filter based on attributes of the associations and results themselves, and then show grouped totals and subtotals for combinations of the associations (yes, I'm being deliberately vague), all split out by month and year.

## Skip to the end...

So anyway, it was fairly clear to most of the team, that the vast majority of this needs to be done in the DB, with the most minimal amount of Rails model instanciation. But it soon became apparent I had two broad problems:

Firstly, while I was coding and trying stuff out; the general day-to-day tweaks and twiddles, I didn't want to have to wait for 2-million records to be operated on.

Secondly, one team member would not grok the absolute insanity of thinking that with the size of datasets we had, iteration of enumerable collections of it was acceptable.

## Solutions, solutions

The second problem was going be solved with facts; cold, hard evidence in the form of benchmarking results. All accompanied with a nice chart showing how the time increased disproportionately the larger the sets you were iterating became (big O, baby).

The first problem was going to be solved as a side effect of solving the second - I was going to end up with a nice little scope I could configure globally, which limited the records any ActiveRecord query would operate on. I would then use this scope in development to only perform calculations of a few ten thousand records (enough to see numbers coming out), but I could switch back to the full set at the drop of a hat to see the effect with the whole data set.

Ideal.

## Benchmarking setup

There's a programming mantra of [measure, don't guess][1] -- whenever I get on my high-horse and start exclaiming what approach a code-base should take, I always have a nagging little voice in my head asking "how would I back this up if someone challenged me".

So my first thought after pointing out the error of the nested `map` calls was to scuttle off and measure the operation compared to grouping in SQL.

But --- I wanted to do this progressively; one of the arguments *for* the map was that "it's fine for the moment" (on the developer's dev machine with 10,000 records), and I wanted to show plainly how performance would degrade as the records accumulated.

Now you might say, "That's fine, just limit the records so:"

{% highlight ruby %}
Result.limit(10_000)
{% endhighlight %}

But unfortunately no -- because by doing a lot of grouping and summing in SQL, any limit condition would apply to the amount of records in the end relation, rather than the amount of records being used for the grouping.

{% highlight ruby %}
> Result.limit(2).group(:foo_id).count
=> {5=>23, 6=>19} # two items in the hash, not two records being grouped
> Result.limit(4).group(:foo_id).count
=> {5=>23, 6=>19, 12=>10, 13=>1}  # four items in the hash, see my problem?
{% endhighlight %}

I **wanted** to limit to counting just two records grouped by their foo_id, but I actually counted 42 records (judging by the totals of the count). And the second query brings even more in -- by limiting the size of the hash -- the output of the relation (instead of the records that go into generating that output).

## Stop, hammer time

Second solution:

Blow away the DB, and then populate it with the first 10,000, measure, add another 10,000, measure again, repeat...

Except this is a bit labour-intensive (since I needed to step up -- in increasing step sizes -- to 2-million-ish), and also hard to repeat when someone queries my numbers, or offers an improvement in code and wants to measure again. And it certainly won't offer me any chance to save time in my day-to-day work on the app.

## Time to get down and dirty

### v1

I had an little inkling in my head... there must be a way to add an elegant little scope to my ActiveRecord objects which would limit the records. The simplest way, I suppose, is to choose records by ID:

{% highlight ruby %}
scope :limited_records, ->(amount) {
  where("#{table_name}.id < ?", amount.to_i)
}
{% endhighlight %}

Ta da!

Now my console checks look something like this:

{% highlight ruby %}
> Result.limited_records(2).group(:foo_id).count
=> {5=>1, 6=>1} # two records grouped and counted
> Result.limited_records(4).group(:foo_id).count
=> {5=>1, 6=>2, 12=>1} # four records grouped and counted
> Result.limited_records(8).group(:foo_id).count
=> {5=>3, 6=>2, 12=>1, 13=>2} # eight records grouped and counted
{% endhighlight %}

Okay... what's wrong with that? It seemed to work at first pass. But the assumption is that there won't be any holes in the sequence of results, and that's a **bad** assumption.

If there's "holes" in my dataset, I could end up with something like this:

{% highlight ruby %}
> Result.count
=> 1000000 # one million exactly -- convenient
> Result.limited_records(10).count
=> 10 # all looks good here
> Result.limited_records(20).count
=> 18 # uh oh! in the first 20 records, two must have been deleted at some point
> Result.limited_records(30).count
=> 18 # OMG! it's more than just two that have been deleted :-(
{% endhighlight %}

### v2

{% highlight ruby %}
scope :limited_records, ->(amount) {
  id = order(:id).offset(amount.to_i).limit(1).pluck(:id).first
  where("#{table_name}.id < ?", id)
}
{% endhighlight %}

So this is a bit more like it -- beasting the AR methods!

I'm plucking the ID of the record after the position I want to go up to, then using that as a filter to include only records below it.

{% highlight ruby %}
> Result.limited_records(10).count
=> 10 # all looks good here
> Result.limited_records(20).count
=> 20 # still good
> Result.limited_records(999_999).count
=> 999999 # cool! well, that was easy enough
{% endhighlight %}

Although, as nice as the abstraction of those ActiveRecord methods are, it would be best for the purposes of benchmarking (remember why we were doing this!) if our code has as small an overhead as possible. So if you can do that in SQL it would be better.

### v3

{% highlight ruby %}
scope :limited_records, ->(amount) {
  where("#{table_name}.id < (select id from #{table_name} order by id limit 1 offset ?)", amount.to_i)
}
{% endhighlight %}

So *now* what's wrong? 

What would you expect to happen if you asked to limit the records to more records than were in the DB? The unsurprising behaviour might be to just return everything. Unfortunately, that's not what our scope does:

{% highlight ruby %}
> Result.limited_records(999_999).count
=> 999999 # cool
> Result.limited_records(1_000_000).count
=> 1_000_000 # cool, cool
> Result.limited_records(1_000_001).count
=> 0 # oh!
{% endhighlight %}

Since there is no record at that offset, there's no ID to be lower than -- so no records to return. The quick fix for this is a slightly uglier scope:

### v4

{% highlight ruby %}
scope :limited_records, ->(amount) {
  if count > amount.to_i
    where("#{table_name}.id < (select id from #{table_name} order by id limit 1 offset ?)", amount.to_i)
  else
    scoped # done for a Rails 3 app... Rails 4 should use `all`
  end
}
{% endhighlight %}

{% highlight ruby %}
> Result.count
=> 1000000 # cool
> Result.limited_records(999_999).count
=> 999999 # cool
> Result.limited_records(1_000_000).count
=> 1000000 # cool
> Result.limited_records(1_000_001).count
=> 1000000 # cool, cool, cool!
{% endhighlight %}

One hiccup though, is that your model might have a default scope applied to it -- and this might be filtering out records. Maybe not an issue, but be careful.


#### References

* [http://programmer.97things.oreilly.com/wiki/index.php/Measure_Don%27t_Guess][1]

[1]: http://programmer.97things.oreilly.com/wiki/index.php/Measure_Don%27t_Guess
