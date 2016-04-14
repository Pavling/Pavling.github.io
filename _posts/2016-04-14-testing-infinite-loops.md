---
layout: post
title: Testing Infinite Loops
description: How can you write a test to check that a method gets itself into an infinite loop (without your tests getting into an infinite loop)?
<!-- modified: 2015-11-08 -->
tags: [ruby rspec testing paradox]
image:
  feature: testing-infinite-loops.jpg
  credit:
  creditlink:
---

I had an interesting problem today -- well, interesting for me.

While working through adding tests to a code-base woefully bereft of any such safety nets, I happened upon a method that had an obvious risk of getting into an infinite loop (in an extreme edge case).

The method's job is to return three random letters from a 'secret word' that a user has previously stored, and the position they were taken from in the word. The goal being to augment a login process.

{% highlight ruby %}
  # general sense of the existing method
  def sample_of_secret_word_characters
    chosen_characters = {}

    until chosen_characters.length == 3 do
      index = rand(secret_word_characters.length)

      if !chosen_characters.has_key?(index)
        chosen_characters[index] = secret_word_characters[index]
      end
    end

    ## the rest of the code to sort and format the characters...
  end
{% endhighlight %}

Okay -- I'm not stressed about the current code for now (or rather, I'm trying not to be); as it's running in live, and has been for a couple of years. What I need to do is write tests to make sure it carries on doing what it's supposed to. I can't refactor until I have 'green' tests for all the method's behaviour.

The tests for the happy-path are easy enough, but you may have spotted the "what if"... what if the `chosen_characters` hash never gets to have three elements in it? In that situation, the code would loop forever.

In practice, other methods ensure that the secret word has a minimum length, but it's perfectly reasonable to imagine that at some point, the database could be updated directly, or even the validations could be (accidentally) removed -- and since there were previously no tests for them either, it could have happened. If one of these -- admittedly unlikely -- events would occur, there'd be a problem.

But *how* do I test the existing functionality that, given a fewer-than-three-character secret word, the code gets in a loop? I can't rightly just start hacking away at code that has tens of thousands of users (because, honestly, the risk of a loop wasn't the biggest issue, but it was the weirdest to test).

After a little Googling, and StackOverflowing, I happened on (what I thought, anyway) was a neat little solution: Run the offending method in a new thread, and assume that if the thread doesn't finish quickly, it must be looping.

{% highlight ruby %}
  # temporary test of existing faulty operation
  it 'enters an infinite loop if the secret_word is fewer than three characters' do
    begin
      thread = Thread.new do
        user.secret_word = 'ab'
        user.sample_of_secret_word_characters
      end
      sleep 0.01
      expect(thread.status).to be_truthy
    ensure
      thread.kill
    end
  end
{% endhighlight %}

So with that in place, I had tests for all the behaviour (even the undesirable infinite looping) of the method. So I could get on refactoring (, and also alter the test for what to do in the case of the secret_word being fewer than three characters (I set it to raise a custom exception -- it would be, after all, an exceptional situation).

{% highlight ruby %}
  # refactored method
  def sample_of_secret_word_characters
    raise SecretWordError if secret_word.size < 3

    chosen_characters = secret_word_characters.each_with_index.to_a.shuffle.first(3)

    ## the rest of the code to sort and format the characters (equally refactored :-) ...
  end
{% endhighlight %}




