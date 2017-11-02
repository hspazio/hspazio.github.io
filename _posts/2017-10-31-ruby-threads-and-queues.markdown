---
layout: post
title: "Ruby threads and queues"
date: 2017-10-28 
tags: [ruby, concurrency]
comments: true
---

Lately I've been working on scaling up the algorithm of a Master-Slave architecture and given the amount of patterns I'm using I figured I'd write a series of blog posts on Ruby concurrency and its patterns.

Ruby has concurrency primitives built in its standard library. Today we start with the fundamental blocks and we will move to more complex patterns later on.

To access to these building blocks we need to first require the `thread` library. This will let us use the `Thread` and `Queue` objects as well as many others that we will see later on.

First we start with a simple design where a producer queues jobs for a consumer. The producer could be a web server queuing email notifications to be sent out, a main thread in a crawler that queues urls to downloader components (consumer)... the list could be long.

The first tool we are going to need is a queue. 
{% highlight ruby %}
require 'thread'

work = Queue.new
{% endhighlight %}

We will use this queue to communicate between the producer and the consumer. Then we can flesh out a basic consumer which will run in a loop, will produce some jobs and will push them on to the queue we just created. 

{% highlight ruby %}
producer = Thread.new do
  count = 0
  loop do
    sleep 1 # some work done by the producer
    count += 1
    puts "queuing job #{count}"
    work << "job #{count}"
  end
end
{% endhighlight %}

We implement the producer as a thread so that we can move the control to the next block.

`Queue` has plenty of aliases for inserting and removing items: here we just used `<<` for insert but we could have used `enq` or `push`. Similarly, for removing items we can use `shift`, `deq` or `pop`. As of writing this post I personally prefer the `<<` operator for inserting as it's graphically expressive, and `deq` for dequeuing. I personally consider `enq` and `deq` very similar in appearance to be used together and `push` and `pop` a bit misleading as highly used for stack behavior.

Let's continue our journey towards the consumer side.
Here we now have to implent the consumer which is going to pull the next job from the queue and do some work with it.

{% highlight ruby %}
consumer = Thread.new do
  loop do
    job = work.pop
    puts "worker: #{job}"

    # some more long running job
    sleep 2
  end
end
{% endhighlight %}

When running our program we'll see that it's all working correctly except the fact that the process never ends - we will tackle this later on.

{% highlight ruby %}
# with a sleep of 2 secs the consumer will never catch up with the workload
# we can scale it up by adding another consumer
# new_consumer = Thread.new do
#   loop do
#     message = work.pop
#     break if message == :done
#     puts "worker: #{message}"
#     sleep 2 # simulate some work to do
#   end
# end
# new_consumer.join

producer.join
consumer.join
{% endhighlight %}

Producer and consumer are not depending on each other. We could entirely redesign the consumer without impacting the producer and vice-versa. This because we have enstablished a communication protocol between the two components
