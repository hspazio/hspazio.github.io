---
layout: post
title: "Worker pool from scratch"
date: 2017-11-06
tags: [ruby, concurrency]
comments: true
---

We start with defining one of the smallest unit: the Worker

{% highlight ruby %}
require 'minitest/autorun'

describe Worker do
  it 'has a name and an inbox queue that shows the count of jobs assigned' do
    worker = Worker.new('worker_1')

    assert_equal 'worker_1', worker.name
    assert_equal 0, worker.jobs_count
  end

  it 'accepts jobs to be queued' do
    worker = Worker.new('worker_1')
    worker << :job
    worker << :job

    assert_equal 2, worker.jobs_count
  end
end
{% endhighlight %}

Then with the following implementation we satisfy the tests. This class represent a very thin layer on top of the Queue object.

{% highlight ruby %}
class Worker
  attr_reader :name

  def initialize(name)
    @name = name
    @queue = Queue.new
  end

  def jobs_count
    @queue.size
  end

  def <<(job)
    @queue << job
  end
end
{% endhighlight %}

We have successfully queued jobs to the worker but lets write another test to describe how to handle jobs from the queue.

For simplicity we could have a worker that expects jobs to be any __callable__ object. For callable object we intend any object that implements the `call` method. In our case we can enqueue lambdas which is a perfectly legit callable object.

To ensure that the jobs have been executed we can collect the results in an array to be inspected during the assertions. Also, as we saw in the [previous post][threads_and_queues], we need to tell the worker when to stop waiting for jobs. We will enqueue the symbol `:done` and we will add an exit condition.

{% highlight ruby %}
describe Worker do
  # previous tests...

  it 'performs a callable job asynchronously' do
    worker = Worker.new('worker_1')
    results = []

    Thread.new do
      worker << -> { results << 'received job 1' }
      worker << -> { results << 'received job 2' }
      worker << :done
    end

    assert_equal [], results
    worker.run
    assert_equal ['received job 1', 'received job 2'], results
  end
end
{% endhighlight %}

The `run` method initializes a new Thread that iterates and through the queue and executes each job until it meets the exit condition.

{% highlight ruby %}
class Worker
  # previous implementation...

  def run
    Thread.new { perform_async }.join
  end

  private

  def perform_async
    while job = @queue.pop do
      break if job == :done
      job.call
    end
  end
end
{% endhighlight %}

[threads_and_queues]: /2017/ruby-threads-and-queues
