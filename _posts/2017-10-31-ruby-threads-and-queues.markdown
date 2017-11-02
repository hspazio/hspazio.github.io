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
    job = work.deq
    puts "worker: #{job}"

    # some more long running job
    sleep 2
  end
end
{% endhighlight %}

Producer and consumer are not depending on each other. We could entirely redesign the consumer without impacting the producer and vice-versa. This because we have enstablished a communication protocol between the two components

If we were to run our program right now we would see it exiting straight away despite having defined the two threads. This is because we havent told Ruby to start the threads. We do it by running the `Thread#join` method on each thread created.

{% highlight ruby %}
producer.join
consumer.join
{% endhighlight %}

When running our program we'll see that it's all working correctly except the fact that the process never ends - we will tackle this later on.

{% highlight bash %}
queuing job 1
consumer: job 1
queuing job 2
consumer: job 2
queuing job 3
queuing job 4
consumer: job 3
queuing job 5
queuing job 6
consumer: job 4
queuing job 7
queuing job 8
consumer: job 5
queuing job 9
queuing job 10
consumer: job 6
{% endhighlight %}

If you had notice from before, I've intentionally added `sleep 1` on the producer to simulate some work on its side and `sleep 2` on the consumer to simulate a long running job. In short I've made the consumer slower than the produer. Observing the output we can see that by the time we consumer processes job #6 the producer has already enqueued 10 jobs.

On a production applicaton this would not be acceptable performance. But we do care about performance and we decide to deploy another consumer that pulls jobs from the same queue so that both consumers can keep up the pace with the producer.

A quick fix is to duplicate the consumer block and assign it to a new variable. This code is not clean but we'll improve it later. For now 'done' is better than 'good'.
We modify a little the consumer block to be an array of threads and now we also have a variable that we can control to scale up/down the number of consumers.

{% highlight ruby %}
num_consumers = 2

consumers = Array.new(num_consumers) do |n|
  Thread.new do
    loop do
      job = work.deq
      puts "consumer #{n}: #{job}"
      sleep 2  # simulate some work to do
    end
  end
end

consumers.map(&:join)
{% endhighlight %}

Running the script now we notice that produer and consumers are running all at the same pace because one consumer is picking up a new job while the other one is still processing the previous job.

{% highlight bash %}
queuing job 1
consumer 1: job 1
queuing job 2
consumer 0: job 2
queuing job 3
consumer 1: job 3
queuing job 4
consumer 0: job 4
queuing job 5
consumer 1: job 5
queuing job 6
consumer 0: job 6
queuing job 7
consumer 1: job 7
{% endhighlight %}
