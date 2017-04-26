---
title: You've been flagged.
description: Taking a look at functions that use flags to determine behavior
header: You've been flagged.
---
Let's play a game. I want you to take a look at the following function call and guess what it does:

{% highlight javascript %}
deleteAllUsers(true);
{% endhighlight %}

Do you have it? If you answer was 'uh, it does nothing', then you're correct! See, deleting all the users isn't so scary. So how does it work its magic?

{% highlight javascript linenos %}
function deleteAllUsers(inProductionMode) {
  if (inProductionMode) {
    return false;
  } else {
    fetch("/users", { method: "DELETE" }).then(function() {
      alert("You have no more users! Time to retire.");
    });
  }
}
{% endhighlight %}

A-ha, we have ourselves a 'magic flag'!

<div class="image-wrapper">
  <img src="/assets/images/itsmagic.jpg" width="500px" alt="It's magic!"/>
</div>

A magic flag is an argument whose value changes the behavior of the function being called. These usually show up as booleans, but they can also be numbers, objects, special secret strings...  you name it. Things get especially interesting when you combine several different types together. Let's sprinkle in some defaults too, cuz why not?

{% highlight ruby linenos %}
def display_contacts(contacts, user_is_mobile, mode = "condensed")
  # fun stuff
end
{% endhighlight %}

What does that thing do? What inputs are being passed to it, and why? And what will be returned? Your guess is as good as mine. When you're reading a method's code, magic flags make it hard to tell what's actually going on.

Adding a magic flag usually means that you're splitting each code path in two, which in the best case means there are now 2<sup>n</sup> paths through your function. Have three flags? There are now eight different possible outcomes you need to think about. Trying to work with code like that is bad news bears.

It's also difficult to reason about why the different code paths exist, and under what circumstances they'd be used. Odds are, you'll probably have to search through the entire codebase to examine every usage for context. And so will everyone else who reads the code. Who doesn't love a good word search?

<div class="image-wrapper">
  <img src="/assets/images/dog-search.jpg" width="500px" alt="Find the Dog"/>
</div>

Now imagine you're deep in the wilds of the codebase, and you come across a call to the above method.

{% highlight ruby %}
contacts_list = display_contacts(user.contacts, true, "full")
{% endhighlight %}

That line of code is quite puzzling. Why is there a random boolean argument? What does `"full"` mean? Welp, guess we have to go look it up and find out. And I guarantee you'll have to keep looking it up every time you come across it. By using magic flags, the author of the function is forcing the person calling it to know about the internal implementation details. Note that this is different than knowing what a function _does_. We use functions every day without having to read the source code, because we know what they _do_. That's how well-written code works - you shouldn't have to worry about the implementation details in most cases. But when you start adding in flags, you almost always need to peek behind the curtain to see what voodoo magic is going on. It's a bit personal, don't you think?

<div class="image-wrapper">
  <img src="/assets/images/wizard-of-oz.gif" width="500px" alt="Behind the curtain..."/>
</div>

How does code like this come about? And what can be done about it? In most cases, I've found that magic flags creep in after the original function has been written. Often someone will need it to do a slightly different thing, and rather than writing a separate function, they add in a flag to modify the behavior.

Luckily, this is usually a pretty straight-forward refactor, and can often be solved using function composition. Passing in a flag to a function means that there are now two different code paths to travel down, usually a base case and some optional tacked-on behavior. The base case can be separated out into its own function, and the optional behavior can be a separate function that calls the base function. To illustrate, let's take a look at an example similar to something I worked on recently.

{% highlight ruby linenos %}
def item_description(item, is_full_length = true)
  description = is_full_length ? item.description : item.short_description

  if item.available?
    description || ""
  else
    Item::DEFAULT_DESCRIPTION[item.status]
  end
end
{% endhighlight %}

This method existed originally as just a simple way to show either an item's description or some default text. As the codebase grew, item descriptions were needed in other places as well, but not always the 'full' version. So a magic flag was added to change the behavior. UI code changes every so often, so what happens when someone needs a slightly different version in a few months? Will this grow into

{% highlight ruby %}
def item_description(item, is_full_length, mode, device = "tablet")
{% endhighlight %}

Hell no! Not on my watch. Let's start by teasing out the base conditional:

{% highlight ruby linenos %}
def item_description_with_default(item, description)
  if item.available?
    description || ""
  else
    Item::DEFAULT_DESCRIPTION[item.status]
  end
end
{% endhighlight %}

Now we have a foundation to build on. Instead of using a flag to alter behavior, let's compose two new methods that call the `item_description_with_default` method.

{% highlight ruby linenos %}
def item_description(item)
  item_description_with_default(item, item.description)
end

def short_item_description(item)
  item_description_with_default(item, item.short_description)
end
{% endhighlight %}

Yah, naming is hard. But now adding new variations is easy! No magic flags needed. The base behavior is still shared, and we can easily compose new methods to get the output we need. The arguments make sense to the reader, and they don't need to bother with implementation details. Problem solved!

Granted, things aren't usually this simple, and searching through a codebase for 30 slightly different calls to the same function will make you reconsider your life choices. But over time, removing magic flags like this will make things a whole lot easier.
