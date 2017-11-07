---
layout: post
title: "Ruby threads and queues"
date: 2017-11-05
tags: [ruby, concurrency]
comments: true
---

Lately I've been working on scaling up the algorithm of a Master-Slave architecture and given the amount of patterns I'm using I figured I'd write a series of blog posts on Ruby concurrency and its patterns.

![Ruby threads and queues](/images/ruby-threads-and-queues.jpg){:class="img-responsive"}

Ruby has concurrency primitives built in its standard library. Today we start with the fundamental blocks and we will move to more complex patterns later on. The Ruby `thread` standard library contains the `Thread` and `Queue` objects as well as many others that we will see in other posts.

Today we start with a simple design where a producer queues jobs to be processed by a consumer. The producer could be a web server queuing email notifications to be sent out, a main thread in a crawler that queues urls to downloader components... the list could be long.

The first tool we are going to need is a queue. A queue is thread-safe which means we can build a lot of concurrency patterns with just that.
{% highlight ruby %}
require 'thread'

work = Queue.new
{% endhighlight %}

We will use this queue to communicate between the producer and the consumer. 

Then we can flesh out a basic producer which will run in a loop, will produce some jobs and will push them on to the queue we just created. 

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

We implement the producer as a thread so that we can move the control of the program to the next block.

`Queue` has plenty of aliases for inserting and removing items: here we just used `<<` for insert but we could have used `enq` or `push`. Similarly, for removing items we can use `shift`, `deq` or `pop`. As of writing this post I personally prefer the `<<` operator for inserting messages as it's graphically expressive, then either `pop` or `deq` for pulling from the queue. In my opinion `enq` and `deq` are very similar in appearance to be used together (I generally prefer to use different looking terms to amplify the constrast) and `push` and `pop` are a bit misleading as highly used for stack behavior Last-In-First-Out.

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

Producer and consumer are not depending on each other. We could entirely redesign the consumer without impacting the producer and vice-versa. This because we have enstablished a communication protocol between the two components.

If we were to run our program right now we would see it exiting straight away despite having defined the two threads. This is because we havent told the main thread to wait for the other threads to finish. 
This has a similar effect to running a Unix command with `&` at the end, which will run the command in the background. We'll fix it by running the `Thread#join` method on each thread created.

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

If the produer creates constantly jobs at such pace the consumer would never be able to catch up. On a production applicaton this would not be acceptable. But we do care about performance and so we decide to deploy another consumer that pulls jobs from the same queue so that both consumers can keep up the pace with the producer.

We modify a little the consumer block to be an array of threads. A nice side effect is that we also have a variable that we can control to scale up/down the number of consumers.

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

## Exit strategy

For now our program will run forever and that's not a reasonable thing to test. If we were to write tests for this program we would probably have the producer queuing a finite number of jobs and then we would assert that the consumer handles them.

Lets modify the producer to only queue a finite number of jobs.

{% highlight ruby %}
producer = Thread.new do
  5.times do |n|
    puts "queuing job #{n}"
    work << "job #{n}"
    sleep 1
  end
end
{% endhighlight %}

If we run the program we see the consumers pulling jobs but then an unusual error occurs:

{% highlight bash %}
queuing job 0
consumer 0: job 0
queuing job 1
consumer 1: job 1
queuing job 2
consumer 0: job 2
queuing job 3
consumer 1: job 3
queuing job 4
consumer 0: job 4
finite_producer_consumer.rb:25:in `join': No live threads left. Deadlock? (fatal)`
{% endhighlight %}

### What is this error about?

The Ruby runtime realized that there are 2 other threads ready to pull from the queue (and in waiting state) but no other thread will enqueue new jobs because the consumer thread exited after queuing 5 jobs.

What we need is a mechanism to tell the consumers that no further jobs will be enqueued and that they can safely exit. 

We can enqueue a `:done` symbol as a signal for the consumer that will read it. As we have 2 consumers we do it twice.

{% highlight ruby %}
producer = Thread.new do
  5.times do |n|
    puts "queuing job #{n}"
    work << "job #{n}"
    sleep 1
  end

  # line added
  num_consumers.times { work << :done }
end
{% endhighlight %}

On the consumer side we have to check whether the job is the special case `:done` and interrupt the loop if so.

{% highlight ruby %}
consumers = Array.new(num_consumers) do |n|
  Thread.new do
    loop do
      job = work.pop
      break if job == :done  # line added
      puts "consumer #{n}: #{job}"
      sleep 2
    end
  end
end
{% endhighlight %}

With this small change our script is now able to exit graceully because there are no more threads hanging around.

Regardless whether we have a producer that enqueues a finite number of messages or that runs in a loops and produces messages forever, having an exit strategy for the consumers is always a good practice. For example we could catch the `Ctrl-C` signal and instead of exit immediately we could notify the consumers that we are closing the queue and no more work will be added, then do any teardown or cleanup to exit gracefully.
