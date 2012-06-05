---
layout: post
title: "Dancing with GCD Group and Barriers"
date: 2012-06-03 22:12
comments: true
categories: 
- MacRuby
- Rubymotion
- Ruby
- Grand Central Dispatch
- GCD
author: Mateus Armando
---
Today we will dinner with GCD Group and barriers features. This article assumes you have already read my previous article [Getting Started With GCD in MacRuby & Rubymotion](/blog/2012/05/31/getting-started-with-grand-central-dispatch-in-macruby-and-rubymotion/).

Now that we know how to use Grand Central Dispatch to make our application concurrent and parallel, this time I'll try to show you how GCD makes it easier for us to syncronize blocks and queue tasks.

A dispatch group is a way to monitor a set of block objects for completion. (You can monitor the blocks synchronously or asynchronously depending on your needs.) Groups provide a useful synchronization mechanism for code that depends on the completion of other tasks. Dispatch::Group acts more or less like Thread#join in plain ruby.

Let's look at some examples to figure out how how Dispatch::Group works:
{% gist 2869475 movie_night.rb %}

On this example, Derpina will wait till Derp is back from the kitchen to press the play button. You don't have to use two queues, both tasks could be executed on the same queue.

How useful are groups on GCD? well let's take a look into another example, do you know how difficult is to implement [Promises and Futures](http://en.wikipedia.org/wiki/Futures_and_promises) in plain Ruby? well this is how you can implement in MacRuby or Rubymotion:

```ruby
include Dispatch
class Future
  def initialize(&block)
    # Each thread gets its own FIFO queue upon which we will dispatch
    # the delayed computation passed in the &block variable.
    Thread.current[:futures] ||= Queue.new("org.macruby.futures-#{Thread.current.object_id}")
    @group = Group.new
    # Asynchronously dispatch the future to the thread-local queue.
    Thread.current[:futures].async(@group) { @value = block.call }
  end
  def value
    # Wait for the computation to finish. If it has already finished, then
    # just return the value in question.
    @group.wait
    @value
  end
end
```

Now it’s easy to schedule long-running tasks in the background:
```ruby
some_result = Future.new do
  p 'Engaging delayed computation!'
  sleep 2.5
  :done # Your result would go here.
end

p some_result.value
```
 <em>this example is brought to you by [patrickt (Patrick Thomson)](https://github.com/patrickt) and [benstiglitz (Benjamin Stiglitz)](https://github.com/benstiglitz)</em>
 

### Barriers
I think we have enough of groups, now let's take a look into something new, barrier were introduced with OS X Lion and iOS 5. Barriers are a specialized version of the Dispatch::Queue#async method. __When a block enqueued with barrier_async reaches the front of a private concurrent queue, it waits until all other enqueued blocks to finish executing, at which point the block is executed. No blocks submitted after a call to barrier_async will be executed until the enqueued block finishes. It returns immediately__ (from Macruby Source code)
If the provided queue is not a concurrent private queue, this function behaves identically to the #async function. 

let's look into some code:
```ruby Dispatch::Queue#barrier_async
queue = Dispatch::Queue.concurrent('com.company.application.task')
@i = ""
queue.async { @i += 'a' }
queue.async { @i += 'b' }
queue.barrier_async { @i += 'c' }
p @i #=> either prints out 'abc' or 'bac'
``` 
When should we use #barrier_async ? Well barrier_async are pretty useful for example to manipulate data structures which can be read but not written concurrently.

There is a second barrier method, which blocks until the provided task or block is executed.
e.g.:
```ruby Dispatch::Queue#barrier_sync
queue = Dispatch::Queue.concurrent('com.company.application.task')
@i = ""
queue.async { @i += 'a' }
queue.async { @i += 'b' }
queue.barrier_sync { @i += 'c' } # blocks
p @i #=> either prints out 'abc' or 'bac'
```
**more barrier example** inspired by [Mike Ash](http://www.mikeash.com/pyblog/friday-qa-2011-10-14-whats-new-in-gcd.html)
Let's imagine that we have a Hash that's being used as a cache. Hash is thread safe for reading, but doesn't allow any concurrent access while modifying its contents, not even if the other access is simple reading." 

```ruby 
framework 'Foundation'

class Cache
  def initialize
    @cache = Hash.new
    @queue = Dispatch::Queue.concurrent('com.company.application.cache')
  end
 
  def []=(key, value)
    @queue.barrier_async{ @cache[key] = value }  # GCD Version
    #@cache[key] = value							# Ruby Version
  end
 
  def [](key)
    @queue.sync { return @cache[key] }	 	 # GCD Version
    #@cache[key]									# Ruby Version
  end
 
  def inspect
    @cache
  end
end

cache = Cache.new
list = File.read("/usr/share/dict/words").split.select{ |word| word[0,1].downcase == "a" }

# cocoa's concurrent enumeration of NSArray
list.enumerateObjectsWithOptions(NSEnumerationConcurrent, usingBlock:-> word, idx, stop {
  cache[idx.to_s] = word
  p cache[idx.to_s]
})
```
To have a better understanding, you should try both versions out (GCD and Ruby Version) one of them will hang, unfortunately the actual MacRuby version is not compiled on Mac OS X Lion, so Dispatch::Queue#barrier_async is not availible, but you can compile it yourself... or ask me on twitter, if I can send you an installer package ;-)

###Final
Since one of biggest difference between Android and iOS application is the UI responsiveness, I hope this motivates you to use GCD to improving the user experience of your rubymotion application.

# Coming next
 
### Dispatch Semaphore
### Dispatch Source


## *Recommendation:*
 <em>- [Apple's Concurrency Programming Guide](http://developer.apple.com/library/ios/#documentation/General/Conceptual/ConcurrencyProgrammingGuide/Introduction/Introduction.html#//apple_ref/doc/uid/TP40008091-CH1-SW1)</em><br/>
