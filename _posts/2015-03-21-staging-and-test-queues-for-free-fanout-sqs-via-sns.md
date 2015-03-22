---
layout: post
title: Staging and test queues for free? Fanout SQS via SNS
categories: [ruby]
tags: [aws, sqs, sns]
published: True

---

This is a short post about SQS fanout queues, and how to start using them using the [AWS SDK for Ruby][aws-sdk-ruby].

Overview
--------

In 2012, [SQS][sqs] got [a new feature that linked SQS and SNS][sqs-sns-announcement], allowing for something called "fanout". You can now send a message to an SNS topic, and any SQS queues subscribed to that topic will receive that message.

I had a hard time figuring out how to do this via the [AWS SDK docs][aws-sdk-docs], so hopefully this post will help someone else trying to do the same thing.

How to do it
------------

Let's cut right to the chase.

{% highlight ruby %}
topic = AWS::SNS.new.topics.create('foo_topic')
queue = AWS::SQS.new.queues.create('foo_queue')
subscription = topic.subscribe(queue)
{% endhighlight %}

If you don't want the default JSON document format (described [here][aws-sns-sqs-docs]), you can get your raw message by setting a flag:

{% highlight ruby %}
subscription.raw_message_delivery = true
{% endhighlight %}

Now you can send messages to your queue via SNS!

{% highlight ruby %}
topic.publish('hello world')
{% endhighlight %}

So what?
--------

What is the use of fanout? __Test queues__.

Let's say you have a process that pulls from SQS. This process can tough to do integrations testing on, because you have to either create test input by hand or copy real stuff from production. Both options are pretty inconvenient.

Enter, test queues! Start sending SQS messages via SNS, and at any time you can choose to also subscribe a test queue to the SNS topic to get real messages in your testing environment without affecting production.

If you have a staging environment set up, you can even run your process in parallel with production. This could be useful for company-critical processes, especially for those with high throughput. Spin up a staging box with your new feature branch and let it run for a day, reading from a staging queue subscribed to your SNS topic, and keeping an eye on the results.

Here's a quick example of what this could look like in code:

{% highlight ruby %}
# $env = Rails.env

class ProcessThatSendsToSNS
  def run
    topic = AWS::SNS.new.topics.named('foo_topic')
    topic.publish('hello world')
  end
end

class ProcessWeWantToIntegrationsTest
  def run
    # let's say this queue is already created and subscribed to "foo_topic"
    queue = AWS::SQS.new.queues.named("#{$env}_foo_queue")
    message = queue.receive_message
    do_crazy_stuff(message)
  end
end
{% endhighlight %}

When you want to test, just set your `$env` to staging or development or whatever is appropriate, and boom! You're playing with real data in a safe environment.

What's the catch?
-----------------

So you might be wondering, why send to SQS at all? Let's send all of our messages via SNS!

# Ladies, one at a time, please

Unfortunately, [unlike SQS][sqs-batch-send-docs], there is no batch send for SNS (yet), only [one-at-a-time publish][sns-publish-docs]. Depending on the throughput of your application, this may become a performance bottleneck.

One trick you could try using is packing a single message with multiple items, such as via a JSON collection. In 2013, [Amazon announced that a single message can now have up to 256KB][aws-256kb-announcement]. That's quite a lot! So instead of sending payloads like `{"foo":"bar"}`, you can accumulate a big batch of them like `[{"foo":"bar"},{"baz":"qux"}]`, taking care to stay under the 256KB maximum.

Another trick is to thread your API calls to reduce time waiting on IO.

# $$$!!! Wait, nope, that's fine.

Let's say you're sending to production, staging, and development queues at all times using the test queues pattern described above. That means there are three times more SQS messages being sent, which means our SQS send costs triple, right?

Turns out, no, your costs don't triple. [Sending to SQS via SNS is no extra charge][aws-sns-pricing], so you can rest easy.

Wrap it up
----------

Fanouts are a pretty neat way to get staging environments set up for free for your SQS-munching apps. There are some negative performance implications in publishing to SNS versus sending to SQS. But the value of having real test data handy whenever you need it potentially outweights any bit of extra work to get SNS fanouts set up.

[aws-sdk-ruby]: https://github.com/aws/aws-sdk-ruby/tree/aws-sdk-v1
[sqs]: http://aws.amazon.com/sqs/
[sqs-sns-announcement]: https://aws.amazon.com/blogs/aws/queues-and-notifications-now-best-friends/
[aws-sdk-docs]: http://docs.aws.amazon.com/AWSRubySDK/latest/index.html
[aws-sns-sqs-docs]: http://docs.aws.amazon.com/sns/latest/dg/SendMessageToSQS.html
[sqs-batch-send-docs]: http://docs.aws.amazon.com/AWSRubySDK/latest/AWS/SQS/Queue.html#batch_send-instance_method
[sns-publish-docs]: http://docs.aws.amazon.com/AWSRubySDK/latest/AWS/SNS/Topic.html#publish-instance_method
[aws-256kb-announcement]: http://aws.amazon.com/about-aws/whats-new/2013/06/18/amazon-sqs-announces-256KB-large-payloads/
[messagepack]: http://msgpack.org/
[aws-sns-pricing]: http://aws.amazon.com/sns/pricing/
