---
layout: post
title: Ruby Timeout, how does it even work?
categories: [ruby]
tags: [timeout, thread]
published: True

---

[Timeout][timeout-rubydocs] is a useful tool in the Ruby standard library that allows a block of code to auto-terminate if execution time exceeds the specified timeout interval.

This is useful for when you have a block of code that could potentially run a lot longer than desired. In practice, I've found this useful for API calls.

{% highlight ruby %}
Timeout::timeout(TIMEOUT_SECONDS) do
  slow_api.call
end
{% endhighlight %}

The above will throw a `Timeout::Error` if the slow API call takes longer than the specified timeout to complete. Very simple and useful.

One day, I was debugging some funny errors being thrown in production, originating from inside a Timeout block. My immediate suspicion was that I didn't quite understand how Timeout worked (boy, was I right). So I opened up irb and tried a few simple examples. Then a few more. What? How does Timeout even work?

Whenever I see something really weird like this, I am apt to undertake a quest (sometimes called "witch hunt" when I'm feeling particularly frustrated by the problem). I drop everything and work until it's finished. So far, my quests have concluded in three ways:

1. I find out I was being an idiot. Also known as PEBKAC.
2. I find a bug that was already fixed in a later version.
3. I learn something new.

I dug deep inside my soul for strength, prepared to <s>waste some more time</s> slay another dragon.

Thus began a Timeout spirit quest/witch hunt in which I dug into the Ruby docs and source code and learned a thing or two about Timeout and threads.

Wait...what?
------------

So I start the tale of my quest with something every great quest-telling starts with...

Pop quiz! What is the output from each of the following?

{% highlight ruby %}
Timeout::timeout(10) { puts '10' }
Timeout::timeout(0) { puts '0' }
Timeout::timeout(-1) { puts '-1' }
{% endhighlight %}

Here were my answers:

{% highlight ruby %}
Timeout::timeout(10) { puts '10' } # 10
Timeout::timeout(0) { puts '0' } # throws Timeout::Error
Timeout::timeout(-1) { puts '-1' } # throws ArgumentError
{% endhighlight %}

Fire up irb and try it out! You might be surprised.

Chances are, you actually got each thing to successfully print its output. This is what I got from irb:

{% highlight ruby %}
>> require 'timeout'
=> true
>> Timeout::timeout(10) { puts '10' }
10
=> nil
>> Timeout::timeout(0) { puts '0' }
0
=> nil
>> Timeout::timeout(-1) { puts '-1' }
-1
=> nil
{% endhighlight %}

'10' printed out as expected, but so did '0' and '-1'. Weird.

The quest begins
----------------

If you were debugging a problem related to this in production, you might do what I did and take a look at the [docs for the timeout method][timeout-timeout-rubydocs] to find some answers.

Unfortunately, we only get one answer out of the two puzzlers:

> A value of 0 or nil will execute the block without any timeout.

That means a Timeout block with a `0` or `nil` given will execute as if it were not in a Timeout block at all. Okay, that's certainly strange behavior, but at least it's documented as a bit of a gotcha in the docs. But that doesn't quite explain why our `Timeout::timeout(-1)` case. Perhaps rubydocs misdocumented this, and in fact a value of `0` _or less_ will execute the block without timeout. An easy way to check is to run a long-running block in Timeout.

{% highlight ruby %}
>> Timeout::timeout(-1) { sleep(10); puts 'complete!' }
ArgumentError: time interval must be positive
{% endhighlight %}

That didn't work. Looks like we still get a timeout with negative numbers. But wait--we got an `ArgumentError`, which is actually what I initially guessed would happen in the `puts` example above. But in the `puts` example, the block completed successfully. What's going on here?

The information we got from the docs don't tell us anything about this behavior, so there's not much left but to dig into the ultimate truth: [the source][timeout-ruby-github].

Here be dragons
------------

I encourage you to read the source yourself and make some guesses on why we see such strange behavior from Timeout (beyond a cop-out answer like "yo, threads be crazy"). If you're not familiar with Ruby threads (or threads in general), it might be helpful to read [a quick tutorial][ruby-threads-article].

I've also copied the Timeout source code here for convenience.

{% highlight ruby %}
  def timeout(sec, klass = nil)   #:yield: +sec+
    return yield(sec) if sec == nil or sec.zero?
    message = "execution expired"
    e = Error
    bl = proc do |exception|
      begin
        x = Thread.current
        y = Thread.start {
          begin
            sleep sec
          rescue => e
            x.raise e
          else
            x.raise exception, message
          end
        }
        return yield(sec)
      ensure
        if y
          y.kill
          y.join # make sure y is dead.
        end
      end
    end
    if klass
      begin
        bl.call(klass)
      rescue klass => e
        bt = e.backtrace
      end
    else
      bt = ExitException.catch(message, &bl)
    end
    rej = /\A#{Regexp.quote(__FILE__)}:#{__LINE__-4}\z/o
    bt.reject! {|m| rej =~ m}
    level = -caller(CALLER_OFFSET).size
    while THIS_FILE =~ bt[level]
      bt.delete_at(level)
    end
    raise(e, message, bt)
  end
{% endhighlight %}

The dragon, slain
----------------

If you examine the source, you'll see that there is no checking for negative numbers at the start of the method call. Instead, it only checks the argument for `nil` and `0` and charges onward, executing the block (and another thread that throws the timeout error if necessary). So where does `ArgumentError: time interval must be positive` come from?

Doing a little more digging, we find that `ArgumentError` is actually not explicitly thrown by the timeout method but is thrown by [sleep][sleep-rubydocs].

When we give a block that is *long running* with a negative timeout, we expect that `y` will call `sleep`, which will throw an error that bubbles up to the main thread `x`. This is exactly what happens, as we saw in the example `Timeout::timeout(-1) { sleep(10); puts 'complete!' }`.

Recall our negative-timeout example that somehow successfully completed:

{% highlight ruby %}
Timeout::timeout(-1) { puts '-1' } # prints '-1'
{% endhighlight %}

What's happening in code is the thread `y` gets started and the current thread `x` *keeps going forward*, executing the given block. Since the job for `x` is a really fast operation `puts`, it completes and the method exits before thread `y` has time to call `sleep` and throw the resultant error into thread `x`.

Lessons learned
---------------

Looks like this was one of those "learning" quests. I'm still waiting for the day I find an active bug and submit and get a pull request merged into Ruby or Rails or something else big. Someday. For now, I continue learning lessons:

1. Okay, first off, the fact that `Timeout::timeout(0)` or `Timeout::timeout(nil)` is equivalent to no timeout is totally bizarre. But now I know.
2. The way threads are used in Timeout is about as simple and practical as you can get. Yet, I was still surprised by its behavior in certain cases. This was a great reminder that threads must be used and regarded with utmost care.
3. I didn't really touch on this in my post, but because of the way Timeout is implemented, you can get some really funny problems whenever your block execution is interrupted. This was actually the cause of the problem I was originally debugging (before embarking on the quest). [This article][timeout-dangerous-article] gives a really short and sweet explanation of why you should be careful using Ruby Timeout.

[timeout-rubydocs]: http://ruby-doc.org/stdlib-2.1.5/libdoc/timeout/rdoc/Timeout.html
[timeout-timeout-rubydocs]: http://ruby-doc.org/stdlib-2.1.5/libdoc/timeout/rdoc/Timeout.html#method-c-timeout
[timeout-ruby-github]: https://github.com/ruby/ruby/blob/0d74082eced0254a30b8f09a4d65fff357fdc6cd/lib/timeout.rb#L75-L115
[ruby-threads-article]: https://blog.engineyard.com/2011/a-modern-guide-to-threads
[sleep-rubydocs]: http://ruby-doc.org//core-2.1.5/Kernel.html#method-i-sleep
[timeout-dangerous-article]: https://coderwall.com/p/1novga/ruby-timeouts-are-dangerous
