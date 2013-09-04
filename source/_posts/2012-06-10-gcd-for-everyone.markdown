---
layout: post
title: "GCD for Everyone"
date: 2012-06-10 21:05
comments: true
categories: 
---

This is the third part in a series of four blog articles on how to use Grand
Central Dispatch (GCD) with MacRuby and RubyMotion. Today, I want to introduce
you to Dispatch Semaphores.

## Understanding and using Dispatch Semaphores

Dispatch Semaphore is GCD’s implementation of a [traditional counting
semaphore][1]. It should not be confused with Ruby’s Mutex which only
implements a simple semaphore lock and allows coordinated access to shared data
from multiple threads. Traditional semaphores always call down to the kernel
to test the semaphore, but the Dispatch Semaphore tests semaphores in user
space and only traps into the kernel when the test fails and needs to block the
thread. This makes Dispatch Semaphore efficient and lightweight.

A Dispatch Semaphore object mainly responds to two methods:

```
- Dispatch::Semaphore#signal
- Dispatch::Semaphore#wait(timeout)
```

When a semaphore is signaled, the counter is incremented. When a thread waits
in a semaphore, it will block, if necessary until it is greater than 0 and then
decrement the count.

Let's look into some code for better understanding:

In this example I'll try to solve the ''__The Dining Philosophers Problem__''
which is an example problem often used in concurrent algorithm design to
illustrate synchronization issues and techniques for resolving them.

## The Problem

The dining philosophers problem is invented by E. W. Dijkstra. Imagine five
philosophers who spend their lives just thinking and eating. In the middle of
the dining room is a circular table with five chairs. The table has a big plate
of spaghetti. There are only five chopsticks available, as shown in the
following figure. Each philosopher thinks. When he gets hungry, he sits down
and picks up the two chopsticks that are closest to him. If a philosopher can
pick up both chopsticks, he eats for a while. When a philosopher finishes
eating, he puts down the chopsticks and starts to think ([more information][2]).

{% gist 2961258 semaphore.rb %}

In this example, the chopsticks are our infinite resource. Each philosopher has
one, but each needs two to eat. Whenever a philosopher picks up a pair of
chopsticks, he send a __wait__, and when he is done, he sends a __signal__ to
the others waiting for the chopsticks.

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
Another example is the sleeping barber problem
{% gist 2961387 sleeping.rb %}

As you can see, Dispatch Semaphore is great way to control the allocation of
limited resources. In the last blog post in this series, I’ll discuss the
principles of Dispatch Source.

# Coming next:

### Dispatch Source

# Recommended Reading:
 - [Apple's Concurrency Programming Guide](http://developer.apple.com/library/ios/#documentation/General/Conceptual/ConcurrencyProgrammingGuide/Introduction/Introduction.html#//apple_ref/doc/uid/TP40008091-CH1-SW1)

[1]: https://en.wikipedia.org/wiki/Semaphore_(programming) 'Semaphore (programming)'
[2]: http://www.cs.mtu.edu/~shene/NSF-3/e-Book/SEMA/TM-example-philos-4chairs.html 'The Dining Philosophers Problem with Four Chairs'
