---
layout: post
title: "Implementing a worker pool"
date: 2017-11-13
tags: [ruby, concurrency]
comments: true
---

Often in time when you need to execute multiple tasks it pays off running them in parallel, especially when they involve blocking I/O operations and CPU computations.
It may not always be possible to spawn as many threads as the number of tasks, think about the maximum number of connections allowed to a database server or threads that could use large chunk of memory. 

In order to improve performance of an application without sacrificing tonnes of resources the Worker Pool is a simple and efficient solution that powers many concurrency patterns.

There are a lot of opensource projects that implement this pattern, [Celluloid][celluloid] and [Concurrent Ruby][concurrent_ruby] are the most popular and better maintained ones. But I thought it would be fun to implement one from scratch in order to better understand the dynamics of this pattern.

## The unit: the worker

A worker is simply a component that pulls work to do from a queue and fulfills it.

The work to be fulfilled could be anything, a `Job` class with a `perform` method - like the popular Delayed Job library - a Ruby block or any arbitrary object. For simplicity here we implement a worker that expects jobs to be any __callable__ object. For callable object we intend any object that implements the `call` method. In our case we can enqueue lambdas which are a perfectly legit callable objects.

We start, using TDD, to define how we would like the worker to behave. When creating multiple workers it could be a good idea to define a `name` or `ID` attribute. It will easy debugging.

Then the obvious feature: as we queue jobs we expect them to be performed by the worker. To ensure that the jobs have been executed we can collect the results in an array to be inspected during the assertions. Also, as we saw in the [previous post][threads_and_queues], we need to tell the worker when to stop waiting for jobs. We will enqueue the symbol `:done` and we will add an exit condition.

{% highlight ruby %}
require 'minitest/autorun'

describe Worker do
  it 'has a name' do
    worker = Worker.new('worker_1', Queue.new)

    assert_equal 'worker_1', worker.name
  end

  it 'listens to jobs in the queue and performs them' do
    queue = Queue.new
    worker = Worker.new('worker_1', queue)
    results = []

    Thread.new do
      queue << -> { results.push('received job 1') }
      queue << -> { results.push('received job 2') }
      queue << :done
    end

    assert_equal [], results
    worker.join
    assert_equal ['received job 1', 'received job 2'], results
  end
end
{% endhighlight %}

To satisfy the tests we create a `Worker` class that accepts a name and a queue to listen for jobs. Also, as we initialize the object we run it straight away so it starts to listen to jobs. This part is important when we have multiple workers sharing the same queue as if we start all the workers at once while the queue has already been populated we don't guarantee randomization for the jobs scheduling. This is one of the first caveats of building a worker pool. Make sure the workers are running before queuing the jobs.

{% highlight ruby %}
class Worker
  attr_reader :name

  def initialize(name, queue)
    @name = name
    @queue = queue
    @pid = Thread.new { perform }
  end

  def join
    @pid.join
  end

  private

  def perform
    while (job = @queue.pop)
      break if job == :done
      job.call
      puts "#{name} got #{job}" # only for debugging
    end
  end
end
{% endhighlight %}

When we initialize the Worker it creates a new Thread that iterates and through the queue and executes each job until it meets the exit condition. Then, with the `join` method we wait for the Worker to complete.

## Managing workers: the worker pool

Now we can move up the abstraction ladder and assemble multiple workers together that share the same queue. The worker pool can be a very thin layer on top of an array of workers.

As usual we start by writing the tests first. We'll use the [Fibonacci sequence][fibonacci_wikipedia] so we can let the worker pool churning the calculations.

{% highlight ruby %}
describe WorkerPool do
  it 'allocates jobs to the workers and runs them in parallel' do
    expected_results = { 
      0=>0, 1=>1, 2=>1, 3=>2, 4=>3, 
      5=>5, 6=>8, 7=>13, 8=>21, 9=>34, 
      10=>55, 11=>89, 12=>144, 13=>233, 14=>377, 
      15=>610, 16=>987, 17=>1597, 18=>2584, 19=>4181, 
      20=>6765, 21=>10946, 22=>17711, 23=>28657, 24=>46368, 
      25=>75025, 26=>121393, 27=>196418, 28=>317811, 29=>514229
    }
    results = {}
    pool = WorkerPool.new(10)

    Thread.new do
      30.times do |n|
        pool << -> { results[n] = fib(n) }
      end
      pool << :done
    end
    pool.wait

    assert_equal expected_results, results
  end
end
{% endhighlight %}

In the test above we defined a `WorkerPool` that initializes 10 workers and we queued 30 lambas to it. Once each lamba is executed by a worker it pushes the result of `fib` to a Hash which is asserted at the end. Then we tell the pool that no more work will be queued and we wait for the workers to fully process the queue.

And for the sake of completeness we define `fib` as below:

{% highlight ruby %}
def fib(n)
  n < 2 ? n : fib(n-1) + fib(n-2)
end
{% endhighlight %}

Now we can satisfy the test. Notice the method `<<`, as we signal that no more jobs are queued, the WorkerPool in turn will signal each worker. Otherwise, a job is put into the queue.

{% highlight ruby %}
class WorkerPool
  def initialize(num_workers)
    @queue = Queue.new
    @workers = Array.new(num_workers) { |n| 
      Worker.new("worker_#{n}", @queue) 
    }
  end

  def <<(job)
    if job == :done
      @workers.size.times { @queue << :done }
    else
      @queue << job
    end
  end

  def wait
    @workers.map(&:join)
  end
end
{% endhighlight %}

When running the tests we can see that they are all passing and the system schedules the threads without any particular order. So, it is not guaranteed that `worker_0` will take the first job.

{% highlight ruby %}
worker_1 got #<Proc:0x007fc35a132d18@worker_pool_2.rb:40 (lambda)>
worker_0 got #<Proc:0x007fc35a130a40@worker_pool_2.rb:89 (lambda)>
worker_3 got #<Proc:0x007fc35a1309a0@worker_pool_2.rb:89 (lambda)>
worker_5 got #<Proc:0x007fc35a130950@worker_pool_2.rb:89 (lambda)>
worker_7 got #<Proc:0x007fc35a1308b0@worker_pool_2.rb:89 (lambda)>
worker_9 got #<Proc:0x007fc35a130810@worker_pool_2.rb:89 (lambda)>
worker_5 got #<Proc:0x007fc35a1305b8@worker_pool_2.rb:89 (lambda)>
# reduced output lines...
worker_4 got #<Proc:0x007fc35a130428@worker_pool_2.rb:89 (lambda)>
worker_6 got #<Proc:0x007fc35a130900@worker_pool_2.rb:89 (lambda)>
worker_2 got #<Proc:0x007fc35a130478@worker_pool_2.rb:89 (lambda)>
worker_1 got #<Proc:0x007fc35a1307c0@worker_pool_2.rb:89 (lambda)>
worker_8 got #<Proc:0x007fc35a130018@worker_pool_2.rb:89 (lambda)>
worker_4 got #<Proc:0x007fc35a1304f0@worker_pool_2.rb:89 (lambda)>
worker_0 got #<Proc:0x007fc35a1306f8@worker_pool_2.rb:89 (lambda)>
{% endhighlight %}

That's really it! This is a simple and at the same time flexible implementation that can help you boost parts of the code base that perform large I/O operations.

## Further improvements

The implementation of the WorkerPool above has a slight bug. If we have a producer that will continuously create jobs and running the jobs is a much slower process that creating them we would have made the Workers be the bottleneck with the risk that they would not be able to catch up. On production more and more jobs would be pushed to the queue until the interpreter runs out of memory.

We could fix this by limiting the number of jobs that the queue can hold. Enter the __SizedQueue__.

The SizedQueue is initialized with an Integer that represents the max number of items that it can contain. When the max is reached, the statement that pushes jobs to the queue is paused until an item is pulled from the queue. At this point the queue wakes up the previously paused statement.

We can now edit the `WorkerPool` initializer by limiting the number of items in the queue to the number of workers available. We could limit it to any arbitrary number. This way we ensure that producer and consmer maintain the same pace. The WorkerPool, in fact, will act as a traffic light: if there are too many jobs it will temporary block any threads that push onto the queue until there is space available again.

{% highlight ruby %}
class WorkerPool
  def initialize(num_workers)
    @queue = SizedQueue.new(num_workers)
    @workers = Array.new(num_workers) { |n| 
      Worker.new("worker_#{n}", @queue) 
    }
  end

  # remaining code...
end
{% endhighlight %}

## Implement scheduling algorithms

An alternative approach could be having the WorkerPool playing a more active role by being responsible for the scheduling. This means that instead having all workers pulling from the same queue, eqch worker will have its own queue. This approach could be useful if you want to segregate the workers by responsibility - for example mailer workers, file download workers, database workers, etc.

In this section we are going to explore some variations to solve different problems.

### Least-Busy First

We modify the previous Worker implementatin by dropping the `queue` argument in the initializer and instead having the Worker defining a queue internally. This new dedicated queue will be the "inbox" of the Worker. 

Now we need to allow the Worker to receive jobs from the outside. We define a `<<` operator that delegates to the inbox queue and a `jobs_count` to allow the WorkerPool inspect the load of each worker and schedule jobs by consequence.

{% highlight ruby %}
class Worker
  attr_reader :name

  def initialize(name)
    @name = name
    @queue = Queue.new
    @pid = Thread.new { perform }
  end

  def <<(job)
    @queue << job
  end

  def jobs_count
    @queue.size
  end

  # rest of the code...
end
{% endhighlight %}

The WorkerPool code will then need to be changed by initializing workers without passing in the shared queue. Also the `<<` method needs to be rewritten because:
1. It does not push to the shared queue
2. It chooses which workers to assign the job to according to the scheduling algorithm
3. It closes the pool by pushing the `:done` signal to each workers

Given that we want to implement different scheduling algorithms we could isolate the scheduler's responsibility from the WorkerPool by creating a new Scheduler class that the WorkerPool will delegate to for dispatching the jobs. We'll use composition!

{% highlight ruby %}
class LeastBusyFirstScheduler
  def initialize(workers)
    @workers = workers
  end

  def schedule(job)
    worker = @workers.sort_by(&:jobs_count).first
    worker << job
  end
end
{% endhighlight %}

That's it for this scheduler. A small single responsibility!

{% highlight ruby %}
class WorkerPool
  def initialize(num_workers, scheduler_factory)
    @workers = Array.new(num_workers) { |n| Worker.new("worker_#{n}") }
    @scheduler = scheduler_factory.new(@workers)
  end

  def <<(job)
    if job == :done
      @workers.map { |w| w << :done }
    else
      @scheduler.schedule(job)
    end
  end

  # rest of the code...
end
{% endhighlight %}

There you have! A pool of workers that schedules jobs to the least busy worker.

## Round-Robin

Now that we have laid the ground work for the WorkerPool to be composed up we can define a new Scheduler that uses the Round-Robin algorithm.

{% highlight ruby %}
class RoundRobinScheduler
  def initialize(workers)
    @current_worker = workers.cycle
  end

  def schedule(job)
    @current_worker.next << job
  end
end
{% endhighlight %}

Notice that differetly than the `LestBusyFirstScheduler`, here we don't need to point to the whole array of workers. We can use the Enumerable#cycle method (with link to doc) which generates an Enumerator that, as the name says, cycles through the element of the array and restarts when it reaches the end.

## Topic Scheduler

Imagine like on a Pub-Sub system we have workers can only perform jobs with a specific tag. This concept is implemented in RabbitMQ Topic Exchange.

{% highlight ruby %}
class TopicScheduler
  TOPICS = [:service_1, :service_2, :service_3]

  def initialize(workers)
    @workers = {}
    workers_per_topic = workers / TOPICS.size
    workers.each_slice(workers_per_topic).each_with_index do |slice, index|
      topic = TOPICS[index]
      @workers[topic] = slice
    end
  end

  def schedule(job)
    worker = @workers[job.topic].sort_by(&:jobs_count).first
    worker << job
  end
end
{% endhighlight %}

[threads_and_queues]: /2017/ruby-threads-and-queues
[celluloid]: https://github.com/celluloid/celluloid
[concurrent_ruby]: https://github.com/ruby-concurrency/concurrent-ruby
[fibonacci_wikipedia]: https://en.wikipedia.org/wiki/Fibonacci_number
