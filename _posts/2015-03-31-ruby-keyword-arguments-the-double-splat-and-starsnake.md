---
layout: post
title: Ruby keyword arguments, the double splat, and **_
categories: [ruby]
tags: [keyword arguments, ruby 2, double splat]
published: True

---

Ruby 2.0 introduced support for keyword arguments.

If you're already familiar with keyword arguments, feel free to skip to the next bit!

Keyword arguments
-----------------

Keyword arguments give you a convenient shorthand for using hashes as an argument to a method. So something like the following:

{% highlight ruby %}
def foo(args)
  raise ArgumentError, "missing keyword: key" unless args[:key]
  if (unknowns = args.keys.select { |key| key != :key }).any?
    raise ArgumentError, "unknown keywords: #{unknowns.join(", ")}"
  end
  key = args[:key]
  key
end

foo(key: "value") # => "value"
foo({}) # ArgumentError: missing keyword: key
foo(other_key: "other_value") # ArgumentError: missing keyword: key
foo(key: "value", other_key: "other_value") # ArgumentError: unknown keywords: other_key

def bar(args = {})
  value = args.fetch(:key, "default_value")
  if (unknowns = args.keys.select { |key| key != :key }).any?
    raise ArgumentError, "unknown keywords: #{unknowns.join(", ")}"
  end
  value
end

bar(key: "value") # => "value"
bar({}) # => "default_value"
bar(other_key: "other_value") # ArgumentError: unknown keywords: other_key
bar(key: "value", other_key: "other_value") # ArgumentError: unknown keywords: other_key
{% endhighlight %}

Can now be written like this:

{% highlight ruby %}
def foo(key:)
  key
end

foo(key: "value") # => "value"
foo({}) # ArgumentError: missing keyword: key
foo(other_key: "other_value") # ArgumentError: missing keyword: key
foo(key: "value", other_key: "other_value") # ArgumentError: unknown keyword: other_key

def bar(key: "default_value")
  key
end

bar(key: "value") # => "value"
bar({}) # => "default_value"
bar(other_key: "other_value") # ArgumentError: unknown keyword: other_key
bar(key: "value", other_key: "other_value") # ArgumentError: unknown keyword: other_key
{% endhighlight %}

Double splat!
-------------

There is a new operator, the double splat, that is sort of the [splat][splat] for keyword arguments. This is most easily explained in yet another "before and after" example.

Ruby 1.9:

{% highlight ruby %}
def foo(args)
  raise ArgumentError, "missing keyword: key" unless args[:key]
  key = args[:key]
  everything_else = args.delete_if { |k, _| k == :key}
  [key, everything_else]
end

foo(key: "value", other_key: "other_value") # => ["value", {:other_key=>"other_value"}]
{% endhighlight %}

Ruby 2.0:

{% highlight ruby %}
def foo(key:, **everything_else)
  [key, everything_else]
end

foo(key: "value", other_key: "other_value") # => ["value", {:other_key=>"other_value"}]
{% endhighlight %}

Pretty cute!

**_, the starsnake
------------------

But what if I want to use the new shorthand syntax without raising an error on any extra unused keys? In code:

{% highlight ruby %}
def foo(args)
  raise ArgumentError, "missing keyword: key" unless args[:key]
  # if (unknowns = args.keys.select { |key| key != :key }).any?
  #   raise ArgumentError, "unknown keywords: #{unknowns.join(", ")}"
  # end
  key = args[:key]
  key
end

foo(key: "value") # => "value"
foo(key: "value", other_key: "other_value") # => "value"
{% endhighlight %}

Introducing, the starsnake **_ (sorry, couldn't come up with a good name).

{% highlight ruby %}
def foo(key:, **_)
  key
end

foo(key: "value") # => "value"
foo(key: "value", other_key: "other_value") # => "value"
{% endhighlight %}

If you don't know about Ruby's magic underscore (a syntax borrowed from Perl) you can read about it [here][magic-underscore]. Basically, `_` is Ruby's variable name for storing values you don't need.

There may be some debate as to whether having a triple symbol expression or allowing arbitrary keys into your method is actually a good idea, but that's outside the scope of this post. ;)

[splat]: http://jacopretorius.net/2012/01/splat-operator-in-ruby.html
[magic-underscore]: http://po-ru.com/diary/rubys-magic-underscore/

