---
layout: post
title: "Worker pool from scratch"
date: 2017-11-06
tags: [ruby, concurrency]
comments: true
---

We start with defining one of the smallest unit: the Worker

For simplicity we could have a worker that expects jobs to be any __callable__ object. For callable object we intend any object that implements the `call` method. In our case we can enqueue lambdas which is a perfectly legit callable object.

To ensure that the jobs have been executed we can collect the results in an array to be inspected during the assertions. Also, as we saw in the [previous post][threads_and_queues], we need to tell the worker when to stop waiting for jobs. We will enqueue the symbol `:done` and we will add an exit condition.

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

Then with the following implementation we satisfy the tests. 

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
      puts "#{name} got #{job}"
    end
  end
end
{% endhighlight %}

When we initialize the Worker it creates a new Thread that iterates and through the queue and executes each job until it meets the exit condition. Then, with the `join` method we wait for the Worker to complete.

We have successfully queued jobs to the worker but lets write another test to describe how to handle jobs from the queue.

Now we can move up the abstraction layers and assemble multiple workers together. The Worker Pool will be a thin layer over an array of workers.

As usual we start by writing the tests. We will calculate the Fibonacci sequence (link to wikipedia) by delegating the calculations to a Worker Pool.

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

Where we define `fib` as:

{% highlight ruby %}
def fib(n)
  n < 2 ? n : fib(n-1) + fib(n-2)
end
{% endhighlight %}

Describe the test code here...

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

When running the tests we can see that they are all passing but especially that the system schedules the threads without any particular order. So, it is not guaranteed that `worker_0` will pick the first job.

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

## Improvements

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

{% highlight ruby %}
class WorkerPool
  def initialize(num_workers)
    @workers = Array.new(num_workers) { |n| Worker.new("worker_#{n}") }
  end

  def <<(job)
    if job == :done
      @workers.map { |w| w << :done }
    else
      # scheduling algorithm
      worker = @workers.sort_by(&:jobs_count).first
      worker << job
    end
  end

  # rest of the code...
end
{% endhighlight %}


[threads_and_queues]: /2017/ruby-threads-and-queues
