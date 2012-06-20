---
layout: post
title: "GCD for Everyone"
date: 2012-06-10 21:05
comments: true
categories: 
---


This is the third part of a serie of four blog articles on how to use Grand Central Dispatch with Macruby and rubymotion, today I want to Introduce you to tDispatch Semaphores.
##Understanding and using Dispatch Semaphores 
Dispatch semaphore is GCD’s implementations of traditional counting semaphore not to be confused whit Ruby’s Mutex which only implements a simple semaphore lock but also allows coordinated access to shared data from multiple Threads. Traditional semaphores always require calling down to the kernel to test the semaphore, but the dispatch semaphore tests semaphore in user space, and only traps into the kernel when the test fails and needs to block the thread, this makes Dispatch Semaphore efficient and lightweight.

A Dispatch Semaphore object on mainly responds to two methods: 
	- semaphore#signal
	- semaphore#wait()
*-When a semaphore is signaled, the counter is incremented. When the thread waits in a semaphore, it will block, if necessary until is greater than 0 and then decrement the count.
	
Let's look into some code for better understanding:
On This example I'll try to solve the ''__The Dining Philosophers Problem__'' which is an example problem often used in concurrent algorithm design to illustrate synchronization issues and techniques for resolving them.

##The Problem
The dining philosophers problem is invented by E. W. Dijkstra. Imagine that five philosophers who spend their lives just thinking and easting. In the middle of the dining room is a circular table with five chairs. The table has a big plate of spaghetti. However, there are only five chopsticks available, as shown in the following figure. Each philosopher thinks. When he gets hungry, he sits down and picks up the two chopsticks that are closest to him. If a philosopher can pick up both chopsticks, he eats for a while. when a philosopher finishes eating, he puts down the chopsticks and starts to think [more information](http://www.cs.mtu.edu/~shene/NSF-3/e-Book/SEMA/TM-example-philos-4chairs.html).

{% gist 2961258 dining_philosophers.rb %}

On this example the Chopsticks are our Infinite resource each philosopher has one, but to eat he needs two, whenever a philosopher picks a chopsticks he send a wait, and when he is done the send a signal to wake the others waiting for the chopsticks.

Let's take a look into how semaphores really work:
```ruby
	# we create an semaphore with the number of resources available: 1
	semaphore = Dispatch::Semaphore.new(1)
	
	# Increment the counting semaphore. If the previous value was less than zero, 
	# this function wakes a thread currently waiting in semaphore#wait
	semaphore.signal
	
	# Decrement the counting semaphore. If the resulting value is less than zero, 
	# this function waits in FIFO order for a signal to occur before returning.
	semaphore.wait(time) # time can be Dispatch::TIME_FOREVER or Dispatch::TIME_NOW 
```
Another example sleeping barber problem
{% gist 2961387 sleeping_barber.rb %}

As you can see, Dispatch Semaphore is great way to control limited resource, this are basically the fundamental usage of dispatch semaphore. On the last blog post of this serie I’ll show the principles of Dispatch Source.

# Coming next
 
### Dispatch Source


## *Recommendation:*
 <em>- [Apple's Concurrency Programming Guide](http://developer.apple.com/library/ios/#documentation/General/Conceptual/ConcurrencyProgrammingGuide/Introduction/Introduction.html#//apple_ref/doc/uid/TP40008091-CH1-SW1)</em><br/>


