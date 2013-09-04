---
layout: post
title: "Getting started with GCD in MacRuby &amp; Rubymotion"
date: 2012-05-31 18:03
comments: true
categories: 
- MacRuby
- Rubymotion
- Ruby
- Grand Central Dispatch
- GCD
author: Mateus Armando
---
# Grand Central Dispatch
Grand Central Dispatch is the apple way to perform concurrent programming, by using it you're able to divide your program into peaces *tasks* that can be executed by a queue concurrently or serially. Since GCD is a low level C API, you can't communicate directly with it, but MacRuby has a wrapper for that. 
### What are Queues?
You can think of *__Dispatch::Queue__s* as workers waiting to execute undefined tasks, the can either execute tasks concurrently or serially.
A Serial Queue executes a single task at once, a concurrent queue is capable to execute as many tasks simultaneously as your system allow it execute.

## Creating and Managing Queues:

GCD comes with three different types of queues:<br />
1) *__The Main queue:__* the main queue is the application aka. main thread, you can get this queue by using:
```ruby get the man queue
	main_queue = Dispatch::Queue.main
```
2) *__Global / Concurrents queue:__* on pre Lion / iOS5 version of GCD there were only one concurrent queue that could have three defined priorities, but now this changed. You can create as much concurrent queues as you want that execute multiple blocks at the same time[¹](#one). 
```ruby Get / Create Concurrent queues
	# get the global concurrent queue on 10.6 OSX [priority can be :high, :low or :default]
	queue = Dispatch::Queue.concurrent(priority=:default)
	# On Lion or iOS5 
	queue = Dispatch::Queue.concurrent("com.company.application.tasks")
```
3) *__Customized queues:__* are lightweight list of blocks which can be executed one at a time in FIFO order, they can be compared with Ruby mutex or traditional ruby thread. They are perfectly suited for synchronization mechanism without have to deal with **lock** and **unlock**.[²](#two)
If you want to ensure that tasks execute in a predictable order, you should use the customized queues.
```ruby Create a custom Queue
	queue = Dispatch::Queue.new("com.company.application.task")
```	

## Submitting blocks to queues:
There are two ways to submit a block to a queue, the first is the asynchronous execution, which submits a block to a queue and returns immediately. e.g:
```ruby
	queue = Dispatch::Queue.concurrent('com.company.app.task')
	queue.async { puts :hallo }
```
The second method is the **Dispatch::Queue#sync** which submits a block on a dispatch queue and waits until that block completes. Unlike the **Dispatch::Queue#async** method, the block are synchronously executed. e.g:
```ruby
	queue = Dispatch::Queue.concurrent('com.company.app.task')
	queue.sync { puts :hallo }
```

#### Submitting blocks later:
**Dispatch::Queue#after** submits a block asynchronously to the given queue after the given delay (in seconds) is passed.
```ruby
	queue.after(0.5) { puts 'waiting for the world to change' }
```
#### Concurrently executing one block many times
**Dispatch::Queue#apply** submits a block to a dispatch queue for multiple execution, if the execution queue is a concurrent, the block will be executed concurrently.
``` ruby
	queue = Dispatch::Queue.concurrent('com.company.app.task')
	@result = []
	queue.apply(5) {|idx| @result[idx] = idx*idx }
	p @result  #=> [0, 1, 4, 9, 16]
```
### Managing Dispatch Objects
Dispatch Objects allow to manage the blocks execution by canceling, suspend and resume it. 
#### Suspending and resuming execution:
```ruby suspending and resuming execution
	queue = Dispatch::Queue.new('com.company.app.task')
	queue.async { sleep 1; puts :hallo }
	queue.suspend!
	queue.suspended?
	queue.resume!
```
#### Getting the internal Queue Object:
Sometimes when dealing with Cocoa / CocoaTouch API's you will need to have access to the Queue object, for this purpose Macruby's GDC-Wrapper delievers a method to get the queue object: *__Dispatch::Queue#dispatch_object__*

### Dispatch Constants
*__Dispatch::TIME_FOREVER__*: means infinity, queue or semaphore will wait till blocks are done <br />
*__ Dispatch::TIME_NOW__*: means zero, queue or semaphore will not wait for blocks at all<br />

# Coming next
 
### Dispatch Barrier
### Dispatch Group
### Dispatch Semaphore
### Dispatch Source

## *Recommendation:*
 <em>- [An Introduction to GCD with MacRuby by Patrick Thomson](http://macruby.macosforge.org/documentation/gcd.html)</em><br/>
 <em>-[Intro to Grand Central Dispatch, Part I: Basics and Dispatch Queues by Mike Ash](http://www.mikeash.com/pyblog/friday-qa-2009-08-28-intro-to-grand-central-dispatch-part-i-basics-and-dispatch-queues.html)</em>

 <div class="post-it">
 <em id="one">¹ Concurrently executed blocks may complete out of order</em><br/>
 <em id="two">² the queue names/labels are meant to help you debugging your application, and it should follow the reverse-DNS style convention</em>
 </div>
