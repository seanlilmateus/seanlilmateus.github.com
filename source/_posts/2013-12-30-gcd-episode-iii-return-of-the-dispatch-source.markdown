---
layout: post
title: "GCD Episode III: Return of the Dispatch Source"
date: 2013-12-30 23:03
comments: true
categories: 
---

Today we will divy into [dispatch source][ds], first of all let's specify what [Dipatch::Source][ds] stands for. A [dispatch source][ds] is a [Grand Central Dispatch (GCD)][gcd] data structure that you create to process system-related events. 

#### In Other Words:
[Dispatch::Source][ds] are used to monitor a variety of system objects and events including file *descriptors, mach ports, processes, virtual filesystem nodes, signal delivery and timers*.

When a state change occurs, the [dispatch source][ds] will be submit its event handler block to its target queue.

#### Dispatch::Source.new
##### Methods in Dispatch::Source class:
All the `Dispatch::Source` types have the same initialization scheme:

	- new(type, handle, mask, queue) { ... } -> Source
		- [PARAM] type:
			- The type of the dispatch source.
		- [PARAM] handle:
			- The handle to monitor. If passeed `DATA_ADD` or `DATA_OR` 
			in type, specify the `0` in this argument.
		- [PARAM] mask:
			- The mask of flags.
		- [PARAM] queue:
			- The dispatch queue to which the event handler block is submitted.
		- [RETURN]
    		- Returns a Dispatch::Source instance.

Well there is no rule without exception, in this case the exception is `Dispatch::Source.timer`

	- timer(delay, interval, leeway, queue) { ... } -> Source
		- [PARAM] delay:
			- The start time of the timer.
		- [PARAM] interval:
			- The second interval for the timer.
		- [PARAM] leeway:
			- The amount of time, in seconds, that the system can defer the timer.
		- [PARAM] queue:
			- The dispatch queue to which the event handler block is submitted.
		- [RETURN]
    		- Returns a Dispatch::Source instance.

Just like `Dispatch::Queue`, `Dispatch::Source` also can be cancelled and tested if they are cancelled:
	
	- cancel!
		- Asynchronously cancels the dispatch source, preventing any 
		further of invocation of its event passed block
	
	- cancelled? -> bool
		- [RETURN]
    		- Returns a `true` if cancelled, otherwise `false`.


Other implemented Methods are:

	- handle
		- [RETURN]
    		- Returns the underlying Ruby handle for the dispatch source.
    - mask
    	- [RETURN]
    		- Returns returns the set of flags that were specified at source 
    		creation time via the mask argument.
	- data
		- [RETURN]
			- Returns the currently pending data for the dispatch source.


##DISPATCH SOURCE TYPES
[Grand Central Dispatch (GCD)][gcd] comes with 11 different type of Dispatch::Sources and at the moment only 8 are accessible to [MacRury][macrury] and [RubyMotion][rm].

- ##### IMPLEMENTED #####
	* `Dispatch::Source::DATA_ADD`
	* `Dispatch::Source::DATA_OR`
	* `Dispatch::Source::Timer`
	* `Dispatch::Source::PROC`
	* `Dispatch::Source::READ`
	* `Dispatch::Source::WRITE`
	* `Dispatch::Source::SIGNAL`
	* `Dispatch::Source::VNODE`
	
-------------------------
- ##### NOT IMPLEMENTED #####
	* `Dispatch::Source::MACH_SEND`
	* `Dispatch::Source::MACH_RECV`
	* `Dispatch::Source::MEMORYPRESSURE`

__1. Dispatch::Source::DATA_ADD and Dispatch::Source::DATA_OR__

Both Sources allow applications to manually trigger the source's event action vai a call of `Dispatch::Source#<<`. The Data will be merged to with the source's pending data via an atomic add or logic OR depending on the source type. the operation will happen within the target queue.

```ruby Dispatch::Source::DATA_ADD

progress = Progress.new
queue = Dispatch::Queue.new('source add example')
source = Dispatch::Source.new(Dispatch::Source::DATA_ADD, 0, 0, queue) do |src|
	progress.increment src.data
end
Dispatch::Queue.concurrent.apply(1000) { |idx| sleep 0.1; source << 1 }

```

_explanation:_ the above example increment our `progress` object, eventhought the numbers are generated concurrently the `source` will make it synchrone, so that are no ThreadSafe `progress` doesn't blows our intentions.


__2. Dispatch::Source::Timer__

```ruby  Dispatch::Source::Timer

queue = Dispatch::Queue.new 'example.timer'
timer = Dispatch::Source.timer(0.5, 1, 0.5, queue) do |s|
  puts "Wake up!"
end

```

_explanation:_ The Above source is a timer which get called every 1 second, but it has a leeway of 0.5 second, which means it may sometimes be called with a max interval of 1.5 seconds. The leeway is the torance.


__3. Dispatch::Source::PROC__

This type of sources monitpr procesdses state changes, the handle is the process identifier of the monitored process and te mask may be one or more of the following flags:

Flag                              |     | Meaning
--------------------------------- |-----| --------------------------------------------------------
Dispatch::Source::PROC_EXIT       |     | The process has exited and is available to wait
Dispatch::Source::PROC_FORK       |     | The process has created one or more child processes.
Dispatch::Source::PROC_EXEC       |     | The process has become another executable image.
Dispatch::Source::PROC_SIGNAL     |     | A Unix signal was delivered to the process.

How to use it?

```ruby Dispatch::Source::PROC
# Monitoring the death of a process
queue = Dispatch::Queue.new('Proc')
itunes = NSRunningApplication.runningApplicationsWithBundleIdentifier 'com.apple.iTunes'
return if itunes.empty?
pid = itunes.first.processIdentifier
opts = Dispatch::Source::PROC_EXIT
 # will observe safari, whenever it quits the action block will be called
Dispatch::Source.new(Dispatch::Source::PROC, pid, opts, queue) do |src|
	NSLog("%@, iTunes quit", src.mask) # 2147483648, iTunes quit
end
```
_explanation:_ On our example, we observer a iTunes Process, the block is called whenever the application is quited.


__4. Dispatch::Source::READ__

```ruby Dispatch::Source::READ

filename = 'PATH/file2read.txt'
@result = ""
read_queue = Dispatch::Queue.new "read source queue"
@src = Dispatch::Source.new(Dispatch::Source::READ, file, 0, read_queue) do |src|
begin
    @result << @file.read(s.data) # ideally should read_nonblock
 rescue Exception => error
    puts error
  end
end

```
_explanation:_ The above Source read the content of our file `filename` asynchronously, the content of the file will be copy in our `@result` variable.


__5. Dispatch::Source::WRITE__

```ruby Dispatch::Source::WRITE
path = 'PATH/hello.txt'
filename = File.new(path, File::WRONLY|File::CREAT|File::TRUNC, 0644)
@msg = "#{$$}: #{Time.now} queue%s"
@pos = 0
writer = Dispatch::Source.new(Dispatch::Source::WRITE, fd, 0, queue) do |src|
  file = src.handle
  begin
    npos = @pos + src.data - 1
    msg = @msg % Dispatch::Queue.current
    file.write("#{msg[@pos..npos]}") # ideally should write_nonblock
    @pos = npos + 1
  rescue Exception => error
    puts error
  end  
end
```

_explanation:_ The above Source writes the content of the variable `@msg` asynchronously to the file `filename`.



__6. Dispatch::Source::VNODE__

Flag                              |    | Meaning
--------------------------------- |----| --------------------------------------------------------
Dispatch::Source::VNODE_WRITE     |    | The process has exited and is available to wait
Dispatch::Source::VNODE_DELETE    |    | The process has become another executable image.
Dispatch::Source::VNODE_EXTEND    |    | The process has created one or more child processes.
Dispatch::Source::VNODE_RENAME    |    | The process has become another executable image.
Dispatch::Source::VNODE_ATTRIB    |    | The process has become another executable image.
Dispatch::Source::VNODE_REVOKE    |    | The process has become another executable image.
Dispatch::Source::VNODE_LINK      |    | The process has become another executable image.


```ruby Dispatch::Source::VNODE
O_EVTONLY = 0x8000
queue = Dispatch::Queue.new 'example.vnode'
type  = Dispatch::Source::VNODE
opts  = Dispatch::Source::VNODE_WRITE | Dispatch::Source::VNODE_DELETE
dirPath = '/Desktion/dir'
fd = open(path, O_EVTONLY)
Dispatch::Source.new(type, dirPath, opts, queue) do |src|
  data = src.data
  if data == Dispatch::Source::VNODE_WRITE
    puts "changed directory data"
  elsif data == Dispatch::Source::VNODE_DELETE
    puts 'The directory has been deleted.'
  end
end
```
_explanation:_ The above Source observes the directory `dirPath`, whenever the content of the directory changes (write or deleted), the source triggers it block.


As you can see, GCD is a mighty tool and it dvantages is concurrency and Asynchronousity. Whenever you have to get critical tasks done, you should consider GCD.

This is the last part of the GCD Series. The coming posts will cover other topics. I hope you enjoyed it.


 <h2 id="source">Sources:</h2> 

- [Macruby WIKI - Dispatch::Source Class][mrw]

- [OS X Man Pages](https://developer.apple.com/library/mac/documentation/Darwin/Reference/ManPages/man3/dispatch_source_cancel.3.html)


[ds]:https://developer.apple.com/library/mac/documentation/Darwin/Reference/ManPages/man3/dispatch_source_create.3.html
[gcd]:https://developer.apple.com/library/mac/documentation/Performance/Reference/GCD_libdispatch_Ref/Reference/reference.html
[macruby]:http://macruby.org
[rm]:http://www.rubymotion.com
[mrw]:https://github.com/MacRuby/MacRuby/wiki/Dispatch::Source-Class