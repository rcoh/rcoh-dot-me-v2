---
title: "Why you can have millions of Goroutines but only thousands of Java Threads"
date: 2018-04-12T14:28:46-07:00
draft: true
tags: ["language-internals", "golang", "scala", "java"]
---
Many seasoned engineers working in JVM based languages have seen errors like this:
```
[error] (run-main-0) java.lang.OutOfMemoryError: unable to create native thread: possibly out of memory or process/resource limits reached
[error] java.lang.OutOfMemoryError: unable to create native thread: possibly out of memory or process/resource limits reached
[error] 	at java.base/java.lang.Thread.start0(Native Method)
[error] 	at java.base/java.lang.Thread.start(Thread.java:813)
...
[error] 	at java.base/java.lang.Thread.run(Thread.java:844)
```
`OutOfMemory`...err...out of threads. On my laptop running Linux, this happens after a paltry 11500 threads. 

If you try the same thing in Go by starting Goroutines that sleep indefinitely, you get a very different result. On my laptop, I got up to 70 million goroutines before I got bored. So why can you have so many more Goroutines than threads? The answer is a fun journey down the the operating system and back up again. And this isn't just an academic issue -- it has real world implications for how you design software. I've run into JVM thread limits in production literally dozens of times, either because some bad code was leaking threads, or because an engineer simply wasn't aware of the JVM's thread limitations. The same story plays out in Apache vs. Nginx, actors vs. threads, Node vs. Python, Gunicorn vs. UWSGI, and many, many more.

### What's a thread anyway?
The term "thread" can mean a lot of different things. In this post, I'm going to use it to refer to a logical thread. That is: a series of operations with are run in a linear order; a logical path of execution. CPUs can only execute about one logical thread per-core truly concurrently.[^1] An inherent side effect: If you have more threads than cores, threads must be paused to allow other threads to do work, then later resumed when it's their turn again. To support being paused and resumed, a thread minimally needs two things:ack 

1. An instruction pointer of some kind. Aka: What line of code as I executing when I was paused?
2. A stack. Aka: What is my current state? The stack contains local variables as well as pointers to heap allocated variables. All threads within the same process share the same heap shared between multiple threads.[^4]

Given these two things, the system scheduling threads onto the CPU has enough information to pause a thread, allow other threads to run, then later resume the original thread where it left off. This operation is usually completely transparent to the threads. From the perspective of a thread, it is running continuously. The only way a thread could observe being descheduled is by measuring the time between subsequent operations.[^5]

Getting back to our original question, why can you have so many more Goroutines?

### Answer 1: Because the JVM uses operating system threads
Although, it's not required by the spec, all modern, general purpose JVMs that I'm aware of delegate threading to operating system threads on all platforms where this is possible. Threads implemented by the operating system have a two properties that drastically limit how many of them can exist. Going forward, I'll use the phrase "user space threads" to refer to threads that are scheduled by the language instead of by the kernel/OS.

#### Context Switching Delay
Because the JVM uses operating system threads, it relies on the operating system kernel to schedule them. The operating system has a list of all the running processes and threads, and attempts to give them each a "fair" share of time running on the CPU.[^nice] When the kernel switches from one thread to another, it has a significant amount of work do. The new thread or process running must be started with a view of world that abstracts away the fact that other threads are running on the same CPU. I won't the get into the nitty gritty here, but you can [read more](https://en.wikipedia.org/wiki/Context_switch) if you're curious. The critical takeaway here is that switching contexts will take on the order of 1-100µ seconds. This may not seem like much, but at a fairly realistic 10µ seconds per switch, if you want to schedule each thread at least once per second, you'll only be able to run about 100k threads on 1 core. And this doesn't actually give the threads time to do any useful work.

**Using operating system threads caps you in the double digit thousands, simply from context switching delay**.

#### Fixed Stack Size
The second major problem with operating system threads comes because each thread has its own stack. Operating system threads must have a fixed size stack -- by default, it's quite large! In a 64-bit environment, the JVM defaults to a 1MB stack per thread. You can make the default stack size smaller, but you tradeoff memory usage with increased risk of stack overflow if your code uses significant amounts of recursion. If you leave it at the default, 1000 threads will use almost 1GB of RAM! RAM is cheap these days, but almost no one has the terabyte of ram you'd need to run a million threads with this machinery.

**Using operating system threads incurs a constant, large, memory cost per thread**

### How Go Does it Differently

Obviously, no solution that maps language threads 1:1 with operating system threads will allow massive concurrency. Golang deals with both of these issues in pretty cool ways.

#### Run multiple Goroutines on a single OS thread
Golang effectively implements it's own scheduler that allows many Goroutines to run on the same OS thread. Even if Go ran the same context switching code, it would save a significant amount of time but avoiding the need to switch into ring-0 to run the kernel and back again. Go has more tricks up its sleeve which will get to in a little bit.


#### Dynamically Sized Stacks
Golang also sidesteps the problems of large (mostly unused) stacks running the system out of memory, with a very clever trick: Go's stacks are dynamically sized, growing and shrinking with the amount of data stored. This isn't a trivial thing to do, and the design has gone through a [couple](https://blog.cloudflare.com/how-stacks-are-handled-in-go/) [of](https://groups.google.com/forum/#!topic/golang-dev/i7vORoJ3XIw) [iterations](https://docs.google.com/document/d/1wAaf1rYoM4S4gtnPh0zOlGzWtrZFQ5suE8qr2sD8uWQ/pub).[^2] While I'm not going to get into the internal details here (they're more than enough for they're own posts and others have written about it at length), the most important takeaway is that a new goroutine will have a stack of only about 4KB. With 4KB per stack, you can put 2.5 million goroutines in a gigabyte of RAM -- a huge improvement over Java's 1MB per thread.

### Answer 2: Because Java doesn't know what threads to run 
Even if Java implemented these dynamically sized stacks and brought threads to user space, it still wouldn't be able to support millions of threads. Suppose for a minute that in your new system, switching between new threads takes only 100 nanoseconds. Even if all you did was context switch, you could only run about a million threads if you wanted to schedule each thread 10 times per second. More importantly, you'd be maxing out your CPU to do so. Supporting truly massive concurrency requries one more trick: Only schedule a thread when you know it can do useful work! If you're running that many threads, only handful can be be doing useful work anyway. Go facilitates this s by it's tight integration between channels, channel primitives, and the scheduler. If a goroutine is waiting on a empty channel, the scheduler can see that and it won't run the Goroutine. Go goes one step further and actually sticks the mostly-idle goroutines on their own operating system thead. This way the (hopefully much smaller) number of active goroutines can be round-robined by a thread while the potentially millions of mostly-sleeping goroutines can be tended to separately. This helps keep latency down.

Unless Java added language features that the scheduler could access, this sort of thing is impossible. However, you could build a runtime scheduler in "user space" Java that is aware of when a thread can do work.[^3] 

### Closing Thoughts
Transitioning from a model using operating system threads to a model using lightweight, user space threads has happened over and over again and will probably continue to happen. For use cases where a high degree of concurrency is required, it's simply the only option. However, it doesn't come without considerable complexity. If Go opted for OS threads instead of their own scheduler and growable-stack scheme, they would shave thousands of lines off the runtime. Using Python's `gevent` to achieve the similar behavior is a terrifying leap of faith that everything will both be monkeypatched correctly and is correct.

But, thousands use both in production every day. For many use cases, it's simply a better model. The complexity can be abstracted away by language and library writers, and software engineers can write massively concurrent programs.

***
{{% subscribe %}}

[^1]: [Hyperthreading](https://en.wikipedia.org/wiki/Hyper-threading) doubles the effective cores. Instruction pipelining will also increase the effective parallelism the CPU allows. At the end of the day, however, it will be `O(numCores)`.

[^2]: Golang first started with a segmented-stacks model where the stack would actually expand into a separate area of memory, using some clever bookkeeping to keep track. A later implementation improved performance in specific cases by instead using a contiguous stack where instead of splitting the stack, much like resizing a hashtable, a new large stack is allocated and threw some very tricky pointer manipulation, all the contents are carefully copied into the new, larger, stack.

[^3]: Actors serve the same purpose as Goroutines for Scala/Java by enabling massive concurrency. Just like Goroutines, the actor scheduler can see which actors have messages in their mailbox and only run actors that are ready to do useful work. You can actually have even more actors than you can have goroutines because actors don't need a stack. However, this means that if an actor does not quickly process a message, the scheduler will be blocked (since it can't pause in the middle of the message since the Actor doesn't have it's own stack). A blocked scheduler means no messages are processed and things will quickly grind to a halt. Trade offs!

[^4]: There's probably some esoteric case where this isn't true. I'm sure someone will tell me about it.

[^5]: This is actually an attack vector. Javascript can detect minute differences in timing caused by keyboard interrupts. This can be used by a malicious website to listen, not to your keystrokes, but to their timing. https://mlq.me/download/keystroke_js.pdf

[^nice]: Threads can mark their priority by calling `nice` (see `man nice`) for more info to control how frequently they're scheduled.