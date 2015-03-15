---
layout: post
title: Debugging with PaperTrail
subtitle: Auditing and Versioning, with fancy Debugging too
comments: true
---
Yikes, it's been a while since I posted. I've been flat out busy, but I'll try harder, as writing is therapeutic and who knows, someone may find this useful.

[PaperTrail](https://github.com/airblade/paper_trail) is a great Ruby gem, and I use it on quite a few of the key models in the Rails project I'm currently working on. According to the ReadMe, "It's good for auditing or versioning", and it is. My prime motivation for using it was for auditing, to know that we can see who did what to a model over it's lifetime. Today (beware the ides of March) it also served as a powerful debugging tool too.

To explain, let's look at an `Order` model. In this model we need to know what currency the order will be made in, how much the order is for, and how much of a discount we might offer (this is a simplification to illustrate the point). Using another excellent gem, [Money-Rails](https://github.com/RubyMoney/money-rails), I set up the `amount_cents`, `discount_cents` and `currency` attributes:

{% highlight ruby %}
  class Order < ActiveRecord::Base
    has_paper_trail

    monetize :amount_cents, with_model_currency: :currency
    monetize :discount_amount_cents, allow_nil: true, with_model_currency: :currency

    #...More stuff here, but not much, it's a model!...
  end
{% endhighlight %}

Now, there's a trap for young players in this, but more later, given that it's the cause of the bug and I don't like spoilers.

The Rails app is live and for the most part working really well. The trigger for this exercise was when an order could not be finalised. The user clicked 'Finalise', but received a [422](http://www.restpatterns.org/HTTP_Status_Codes/422_-_Unprocessable_Entity) error. There are various checks and balances for an order to complete, and all the processing is in a transaction, so in this case the data was safe, but something unexpected has happened.

I got the [RollBar](https://rollbar.com) notification, and started looking into it. As soon as I saw at what stage of the process the error occurred, I looked at the relevant record in the database. To my surprise, the currency of the model was set to "USD". Now, there is nowhere in the user interface that a user can set the currency, it is automatically set when the order is created, based on the country the business is in, Australia in this case. Therefore, the currency should start as "AUD" and stay that way forever. And ever. Sanity checking the code (looking everywhere I knew used the money values, and full search for `.currency =`,  `.update(:currency)`, etc.) turned up nothing, as I expected. So how is the currency changing???

PaperTrail to the rescue. Firing up the Rails console, I stepped back through the versions like so:

```
order = Order.find(xx)
order = order.previous_version
order = order.previous_version
order = order.previous_version
...
```
Each time we set `order` the object prints to the screen, so I kept an eye on the `currency` attribute to see at which point it changed. The first few results (ie the most recent versions) were "USD", then all of a sudden there's a `NULL` value, and then the previous version to that was "AUD". PaperTrail has told us exactly where the change occurred, so now we can look at the other attributes that changed.

In this case, the user had set a discount amount, and then unset it. This set the `currency` to `NULL`. They then set the discount amount again, and as there was no currency, the Money-Rails default of "USD" was used.

Now this is probably the expected behaviour of the Money-Rails gem (and I'm looking into it), but it wasn't what I was expecting. Normally, in every other case I interact directly with the `xxxxx_cents` field for money values, and mostly use the attribute without the `_cents` (noncents??) for display and some currency conversion. However, in this case the form where the user entered the discount amount was using the `discount_amount` attribute, and not `discount_amount_cents`. When it was set to `NULL`, someone decided the `currency` should be set to `NULL` too. This seems a reasonable behaviour if you only have one Money field in the model, but if several fields are sharing the same `currency` value, then you're screwed.

A quick, hacky fix in the `OrderController` let's me get working code up and live, and then I can look into a better way to handle the situation, maybe a Form object to either change the user submitted value to `cents` or just watching out for `NULL` submissions and handling the `currency` differently.

Whichever way I go, PaperTrail was instrumental in finding the cause of a bug which would have been insanely hard for me to find any other way.
