---
layout: post
title: Rails UJS and Callbacks
description: Rails give us very nice unobtrusive JS features, but what's *the* way to handle responses?
<!-- modified: 2015-02-23 -->
tags: [rails, ujs, javascript, callbacks, jquery]
image:
  feature: rails-ujs-and-callbacks.jpg
  credit:
  creditlink: 
---

I was recently teaching a beginner's course in web development with Rails, and when we got to the "JavaScript week" everyone loved the ability to fire AJAX requests with jQuery and to handle the responses with callback functions. Then, when we looked at integrating with Rails, the "Rails Way" seemed at odds with the "pure JS" way.

In summary, [Rails' process][1] is to add `remote: true` to form- and link-helpers, then add `format.js` as a responder in the controller action, and set up `action.js.erb` view files to generate JavaScript that will get sent in the response to be executed in the browser.

However, there are [arguments against this approach][2], and people who would argue for the preference to keep JavaScript in the asset pipeline rather than scatter it around view files -- and such responded-with JS could be limited in its access to the rest of the app's JS if that was written inside a closure.

## So what are the polar choices

  1. Do things the Rails Way (and your JS will be scattered).
  2. Write all your own JS handlers for AJAX form and link submission, and attach callback handlers for success/error responses -- all the JS can be in the asset pipeline (but none of the Rails magic is going to be there to save you time) and effort.
  3. Find some middle-ground that uses some of the Rails Way, and some of your own JS to keep things a bit "neater", but make user of the helpers.

## A 'Middle' way

So this post is about a middle way -- not _the_ middle way, as you could draw the line yourself in different places. But the motivation is to give an end-to-end solution that "works".

If you search for "[ajax callbacks with Rails remote forms](https://www.google.co.uk/search?q=ajax+callbacks+with+Rails+remote+forms)", you get lots of resources, but none of them *quite* work 100% (unless you know what they're missing, and what other pieces need to be in place -- and if you know that, you don't need to Google for them).

We're going to imagine an app that models an index of products, and the index view has a search form on it to filter products by name -- pretty vanilla Rails CRUD (with a little feature we can wire our AJAX into).

> This code was written with and for Rails 4.1.9

{% highlight ruby %}
# products_controller.rb
def index
  if params[:search]
    @products = Product.where("name like ?", "%#{params[:search]}%")
  else
    @products = Product.all
  end
end
{% endhighlight %}

{% highlight erb %}
<!-- views/products/index.html.erb -->
<%= form_tag products_path, method: :get, id: :search_form do %>
  <p>
    <%= text_field_tag :search, params[:search] %>
    <%= submit_tag "Search", name: nil %>
  </p>
<% end %>

<table>
  <thead>
    <tr>
      <th>Name</th>
      <th>Price</th>
      <th colspan="3"></th>
    </tr>
  </thead>
  <tbody id='products_list'>
    <%= render @products %>
  </tbody>
</table>
{% endhighlight %}

{% highlight erb %}
<!-- views/products/_product.html.erb -->
<tr>
  <td><%= product.name %></td>
  <td><%= product.price %></td>
  <td><%= link_to 'Show', product %></td>
  <td><%= link_to 'Edit', edit_product_path(product) %></td>
  <td><%= link_to 'Destroy', product, method: :delete, data: { confirm: 'Are you sure?' } %></td>
</tr>
{% endhighlight %}

### A UJS form

First, we'll add the `remote: true` parameter to the form:

{% highlight erb %}
<!-- views/products/index.html.erb -->
<%= form_tag products_path, method: :get, id: :search_form, remote: true do %>
{% endhighlight %}

> This `:remote` parameter can also be used with other helper methods such as `link_to`, `button_to` and `form_for`.

If we reload the page and look at the source we can see how Rails builds the form now.

{% highlight html %}
<!-- view source of products index -->
<form action="/products" id="search_form" data-remote="true" method="get">
  <p>
    <input id="search" name="search" type="text" />
    <input type="submit" value="Search" />
  </p>
</form>
{% endhighlight %}

The form element is almost exactly the same as it was before we added the remote parameter: there is just one extra attribute for `data-remote="true"`.

There's no inline JavaScript, the new attribute is enough to make the form submit via AJAX -- this is handled by Rails' UJS functionality.

If we submit the form, nothing happens... that we see -- but we can use our browser development tools to see the background request, and the response.

### JS response handlers

We are going to add some JS to handle the response from the form being submitted. The JS attaches two event observers to the `#search_form` element -- one on success of an AJAX request, and one for if there was any AJAX error response.

Both functions are set initially to just console log the data that comes back to them (I always like to take things step-by-step, and logging to the console allows me to make sure everything up to this point is working).

{% highlight js %}
// application.js
$(function() {
  $('#search_form').
    on('ajax:success',function(evt, data, status, xhr){
      console.log('success:', data);
    }).
    on('ajax:error',function(xhr, status, error){
      console.log('failed:', error);
    }); 
});
{% endhighlight %}

### Controller action response

The search form is submitted to the ProductController's index action (it was before, and the `:remote` parameter doesn't change that) and we need to add a handler for an "XML HTTP request" -- to do something different from a normal HTML request.

> If you have any other responder blocks of code in your controller action, make sure that this comes before them.

{% highlight ruby %}
# products_controller.rb
render @products, layout: false if request.xhr?
{% endhighlight %}

To use that syntax of `render`, we need to have a partial that will be rendered for each object in the collection -- they're all `Product` objects in our case, so we need a `_product.html.erb`. So ensure your view code in the index calls the same `render @products` and you're not iterating the products in the index itself.

Now the console is showing the right response contents, we can do something with it -- like replace the current content of the product list.

Edit the success callback we wrote earlier to update the body of the table with the data that comes back from the server.

{% highlight js %}
on('ajax:success',function(evt, data, status, xhr){
  $('#products_list').html(data);
}).
{% endhighlight %}

### Further JS fun

We now have a search form which submits with Rails UJS to the products controller, and the controller sends us back some HTML that we insert into our app. If you wanted, you *could* respond with JSON, and build the HTML for the view client side (using [Underscore templates](http://underscorejs.org/#template), or something similar, maybe).

But that _would_ be duplicating view logic, so I'm going to draw the line (my totally arbitrary line) at rendering the HTML with Rails, and sending it to the JS callbacks for them to insert it in the view.

We could extend the functionality of our form though -- and get rid of the need to submit it manually.

{% highlight js %}
// application.js
$(function() {
  $('#search').on('keyup', function() {
    $('#search_form').submit();
  });
});
{% endhighlight %}

...but then we can carry on doing loads of great stuff to make our app more functional, so we'll stop there for now.

## A Caveat

One big issue that will arise if doing things this way is that we've used lots of jQuery shorthand for `$(document).on("ready", function { ... });`, and Rails now comes with Turbolinks built in.

Turbolinks borks the "on ready" bindings -- refreshing the products' index page will work fine, but following an internal link to it, and all our JavaScript stops running. So use the [jquery.turbolinks][3] gem to make it work again (or [change your JS][4] to bind to `page:change` instead)

#### References

* [http://homakov.blogspot.co.uk/2013/05/do-not-use-rjs-like-techniques.html][2]
* [jQuery UJS](http://github.com/rails/jquery-ujs/wiki/ajax
)
* [Rails guide for working with JavaScript
][1]
* [jQuery Turbolinks][3]
* [Rails' page change events][4]

[1]: http://guides.rubyonrails.org/working_with_javascript_in_rails.html
[2]: http://homakov.blogspot.co.uk/2013/05/do-not-use-rjs-like-techniques.html
[3]: http://github.com/kossnocorp/jquery.turbolinks
[4]: http://guides.rubyonrails.org/working_with_javascript_in_rails.html#page-change-events

