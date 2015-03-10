---
layout: post
title: Rails 'Except' scope
description: What do you do if you want to return a set of records 'except' for certain records?
<!-- modified: 2015-02-23 -->
tags: [rails, scope, except, excluding]
image:
  feature: rails-except-scope.jpg
  credit:
  creditlink: 
---

If you want to be able to exclude certain records from an ActiveRecord query, this scope does the trick.

{% highlight ruby %}
  scope :excluding, -> (*values) { 
  where(
    "#{table_name}.id NOT IN (?)",
      (
        values.compact.flatten.map { |e|
          if e.is_a?(Integer) 
            e
          else
            e.is_a?(self) ? e.id : raise("Element not the same type as #{self}.")
          end
        } << 0
      )
    )
  }
{% endhighlight %}

I've been putting it into pretty much every app I've written in the last couple of years (which is a bit of a clue I should have made a gem out of it). If you want to exclude a subset of records, you can pass it an array of IDs, or objects (of the type you're querying), and the arguments passed to it will be excluded from the list.

For example, given the scope being included in a `Post` model, you could use the scope thus:

{% highlight ruby %}
  @posts = Post.excluding(current_user.posts)
{% endhighlight %}

Or you could populate a list of people to invite to an event - but ommitting those people that have already been invited (assuming the scope is in your `User` model):

{% highlight ruby %}
  @people = Person.excluding(@event.invitees)
{% endhighlight %}

And it can be chained with other scopes (as you should expect):

{% highlight ruby %}
  @winner = Person.non_winners.excluding(Person.staff).sample
{% endhighlight %}


## But what's it doing?

Okay, I don't blame you if you don't just copy it, paste it, and use it in blind faith.

Let's step through it.

{% highlight ruby %}
scope :excluding, -> (*values) { . . . }
{% endhighlight %}

We define a scope, called 'excluding', and it takes some arguments. The arguments are all 'splatted' into an array, and later we can refer to them as the variable `values`.

{% highlight ruby %}
where("#{table_name}.id NOT IN (?)", . . . )
{% endhighlight %}

The scope might be chained to other scopes, so any references to SQL field names is best 'disambiguated' by adding the table-name of the class to it.

This simple `where` condition is saying "get me all records except those that have an ID of ..." -- and the collection of IDs to exclude are passed in in the next argument.

{% highlight ruby %}
values.compact.flatten.map { |e| . . . } << 0
{% endhighlight %}

Let's take those values that were passed into the scope, compact out any nil values, and flatten any arrays that were passed in. Now we can iterate over it, and aim to get an array of integers to pass to the `where` condition.

But hang on... if no values are passed in, the `map` will return an empty array, and our `id NOT IN (?)` query will get no values replacing the question-mark placeholder -- That would cause a SQL error. Oh noes!

To avoid this, we're going to shovel a zero into the result of the map; so whatever happens, the array will have at least one value, and no records will (should...) ever have the ID '0', so it will not exclude anything it shouldn't (unless you have some very bad DBAs who start their sequences at zero... but in that case, while you work your notice period, you could shovel `-1` in instead).

{% highlight ruby %}
if e.is_a?(Integer) 
  e
else
  e.is_a?(self) ? e.id : raise("Element not the same type as #{self}.")
end
{% endhighlight %}

As we loop over the values, we'll keep what we were given if it was an integer, otherwise we'll ask the object for its ID. Which means these two give exactly the same result:

{% highlight ruby %}
Person.excluding(1,2,3,4)
Person.excluding(Person.find_by(id: [1,2,3,4]))
{% endhighlight %}

The idea being that if you have a relation, you can use it, but if you have a collection of IDs, that'll work too.

In the event that an object of the 'wrong' type is passed into the values, an exception should be raised -- all sorts of pain and suffering would ensue if the code worked happily for something like this: `@cats = Cat.excluding(Dog.all)`

## Aliases

The name "excluding" makes sense to me, but if you feel that you'd rather call it "except" instead -- if that trips off the tounge nicer for you; or you like to keep the interface of your ActiveRecord objects similar to Hash's method [`except`](http://apidock.com/rails/v4.1.8/Hash/except), then feel free to just name the scope something else (or alias it!).

