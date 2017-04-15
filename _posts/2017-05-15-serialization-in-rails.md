---
title: Serialization in Rails
description: A look at different techniques and strategies for JSON serialization in Rails.
header: Serialization in Rails
---

So the time has finally come. You knew it would happen some day, but you'd be prepared. Your app would be **organized** and **fast** and **bulletproof**, ready for anything! Then out of nowhere, right in the middle of all your best laid plans, the front-end team shows up on your lawn with a keg of High Life and some turntables and demands **JAVASCRIPT NOW**. You tell them 'Be cool, young bloods. We can do JSON.' And that's how it begins.

<div class="image-wrapper">
  <img src="/assets/images/party-hard.png" width="500" alt="Serialization in Rails"/>
  <div class="caption">Our front-end team at the office</div>
</div>

Luckily, generating JSON in Rails is easy! All you need to do is add one line:

{% highlight ruby %}
render json: my_object
{% endhighlight %}

That's it. Now the Javascript folks can play around with their widgets, and you can go back to grumbling about 'service objects'.

A few more weeks go by, and the mobile app team comes knocking at your door. You think to yourself "when did we get a mobile app team?", but let them in anyway because they seem nice. They start talking about building an API, and you mumble something about how we've only just met and should take things slow. But man, they sure are convincing. You say 'yes'. You have a couple beers with them and talk it over.....  And you quickly realize that you're gonna need a bigger boat. It's time to start paying attention to how data is being serialized.

<div class="image-wrapper">
  <img src="/assets/images/serialization.jpg" width="500" alt="Serialization in Rails"/>
</div>

When you want more control over what your JSON looks like, you have a few different options in Rails land. You can use a templating system like <a href="https://github.com/nesquena/rabl" target="_blank">RABL</a> or <a href="https://github.com/rails/jbuilder" target="_blank">jBuilder</a>, create an object and manually transmogrify it using `to_json`, or use `ActiveModelSerializers` (`to_json`, but fancy).

**Templating**

Using a templating system has a very Rails-y MVC feel to it, so it may seem like the natural first choice for JSON-making. You create a 'view' file that is named after the corresponding controller action and plop it in a folder along with all the other view files for that controller. The JSON view has access to instance variables and helper methods, like any other Rails view. But unlike web pages that usually have unique markup, JSON responses are often built up from objects that get re-used in multiple places. Which means your nice name-spaced JSON view files often end up just rendering shared partials from some other folder. This creates a whole buncha extra files and folders, which gets annoying pretty quickly.

**Hashing it out**

Creating a Ruby object and converting it to JSON feels like a more, uh, literal approach. This is usually done by creating a Ruby hash and calling `.to_json` on it:

{% highlight bash %}
> { name: "Fred", address: { city: "Boston", state: "MA" } }.to_json
=> "{\"name\":\"Fred\",\"address\":{\"city\":\"Boston\",\"state\":\"MA\"}}"
{% endhighlight %}

This is nice in a way, because you get a visual representation in Ruby of what the JSON object will look like. But after a while, you end up with a bunch of hash cow-pies everywhere that are a total pain to clean up. I've been down this road on projects before, and even if you're super-duper organized, it just ain't worth it. So Ruby hashes are out.

**ActiveModelSerializers**

A more civilized approach is to use <a href="https://github.com/rails-api/active_model_serializers" target="_blank">ActiveModelSerializers</a>. Right out of the box, you get a nice, easy to understand syntax for object associations, via `has_many`, `has_one`, and `belongs_to`. Odds are that you're already using `ActiveRecord` to interact with your data layer, so this language should feel pretty natural. The conventions make sense, but can be easily customized. Adding custom key names, snake/camel casing things, and swapping out adapters is easy. Each serializer is a Ruby class, so all those methods that exist only to massage data for serialization finally have a place to live. A pretty great solution overall. And we haven't even gotten to the kicker...

**The speed issue**

Code that is organized and easy to understand is a great thing to have. But if that code is slow, no one will care about the amazing class architecture. So how do these different serialization options perform? Let's take two identical soon-to-be-JSON objects, one using jBuilder, and one using ActiveModelSerializers:

{% highlight ruby %}
# jBuilder - app/views/api/_address.json.jbuilder
json.(address, :id, :name, :street, :city,
  :state, :zip, :short_address, :full_address,
  :search_address, :delivery_instructions, :on_site_contact,
  :latitude, :longitude, :phone, :residence)

# AMS
module Api
  class AddressSerializer < ActiveModel::Serializer
    attributes :id, :name, :street, :city,
      :state, :zip, :short_address, :full_address,
      :search_address, :delivery_instructions, :on_site_contact,
      :latitude, :longitude, :phone, :residence
  end
end
{% endhighlight %}

And generate them 1000 times each:

{% highlight ruby %}
view_context = ApplicationController.view_context_class.new(
  "#{Rails.root}/app/views"
)

JbuilderTemplate.encode(view_context) do |json|
  1000.times do |i|
    json.partial! "api/address", address: address
  end
end

1000.times do |i|
  Api::AddressSerializer.new(address).to_json
end
{% endhighlight %}

The results?

{% highlight bash %}
jBuilder: 4523.67 ms
AMS: 178.94 ms
{% endhighlight %}

Whoa, not even close. <a href="https://medium.com/@lgmspb/how-we-increased-the-speed-of-json-generation-by-3000-times-ca9395ab7337" target="_blank">Other folks</a> have reported this as well. Plain ol' `to_json` is a bit faster, but the benefits of using ActiveModelSerializers more than make up for it. I think we have a winner!

There are <a href="http://vaidehijoshi.github.io/blog/2015/06/23/to-serialize-or-not-to-serialize-activemodel-serializers/" target="_blank">some</a> <a href="https://blog.engineyard.com/2015/active-model-serializers" target="_blank">great</a> <a href="https://robots.thoughtbot.com/better-serialization-less-as-json" target="_blank">posts</a> on how to get up and running with AMS, so I won't go into detail here. Usage is pretty straight-forward overall, but there are still some pot holes that you may run over at some point. Here are a couple that I've come across.

<div class="image-wrapper">
  <img src="/assets/images/fishing.jpg" width="500" alt="Pot hole fishing"/>
</div>

**How to Stay Single**

Bad dating advice, but a great way to keep your code syntax clean! Picture a controller action that returns JSON for an `address` object, serialized using AMS.

{% highlight ruby %}
def show
  address = Address.find(params[:id])

  render json: address, serializer: ::Api::AddressSerializer
end
{% endhighlight %}

The syntax is nice and simple. But what if we need to provide multiple top-level JSON objects in the same response? Then things start to look not-so-nice.

{% highlight ruby %}
def show
  address = Address.find(params[:id])
  business = Business.find(params[:business_id])
  user = find_user

  render json: {
    address: Api::AddressSerializer.new(address).as_json,
    business: Api::BusinessSerializer.new(business).as_json,
    user: Api::UserSerializer.new(user).as_json
  }
end
{% endhighlight %}

Ugh, we're back to using hash literals again. No bueno.

AMS likes single objects, so why not use one? Let's treat the JSON response as a serializable object, and create a separate serializer and object wrapper to handle everything. We can even put them all in one file!

{% highlight ruby %}
module Api
  class AddressResponseSerializer < ActiveModel::Serializer
    Inputs = Struct.new(:address, :business, :user) do
      alias :read_attribute_for_serialization :send
    end

    has_one :address, serializer: ::Api::AddressSerializer
    has_one :business, serializer: ::Api::BusinessSerializer
    has_one :user, serializer: ::Api::UserSerializer
  end
end
{% endhighlight %}

The three objects we need to serialize get wrapped in a simple `Struct`, and we can go back to using AMS's simple syntax:

{% highlight ruby %}
def show
  address = Address.find(params[:id])
  business = Business.find(params[:business_id])
  user = find_user

  response_object = Api::AddressResponseSerializer::Inputs.new(
    address,
    business,
    user
  )

  render json: response_object, serializer: Api::AddressResponseSerializer
end
{% endhighlight %}

If the combo of top-level objects becomes a popular one, it may make sense to ditch the `Struct` and move to using a proper class. But for simple responses, this pattern has worked well.

**When you gotta merge**

AMS currently has no nice syntax for merging together the output of multiple serializers. To make this work, you'll have to do a bit of fiddling under the hood.

Say we're building a serializer for a `Business` object, and we want to include some address data. There's already an `AddressSerializer` that we've been using for this, but in the new `Business` object, we want to flatten the address object and merge it with the business object. In order to make this work, we need to tap into a method in `ActiveModel::Serialization` that gets called by AMS, `def serializable_hash`.

`serializable_hash` is the method that returns the serialized Ruby hash of an object's attributes. In our `Business` serializer, we can override that method call, use `super` to access the original hash, and then merge in our additional address data.

{% highlight ruby %}
def serializable_hash
  super.merge(
    Api::AddressSerializer.new(object.address).as_json
  )
end
{% endhighlight %}

One thing to look out for - using `merge` will clobber existing keys with the same name as those being merged in. You may want to use `reverse_merge` here instead, depending on your use case. This 'hack' isn't the prettiest thing ever, but I've ended up needing it a fair amount to get the required data structure while still keeping things nice and DRY.

There are many methods of serializing data in Rails, and no single one is a silver bullet. But hopefully this has given a bit of insight into how to tackle JSON-making in a decent sized Rails app. Good luck, and happy serializing!
