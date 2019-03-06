---
title: '[translation] High-Performance Server Architecture'
date: 2019-01-28 23:31:59
tags: [High-Performance]
categories: translation
description: A translation access to techs article(unfinished).
keywords: [High-Performance,Server Architecture]
toc: true
---

## High-Performance Server Architecture-Jeff Darcy

The purpose of this document is to share some ideas that I've  developed over the years about how to develop a certain kind of  application for which the term "server" is only a weak approximation.   More accurately, I'll be writing about a broad class of programs that  are designed to handle very large numbers of discrete messages or  requests per second. Network servers most commonly fit this definition,  but not all programs that do are really servers in any sense of the  word.  For the sake of simplicity, though, and because "High-Performance  Request-Handling Programs" is a really lousy title, we'll just say  "server" and be done with it.

> 这篇文档的目的在于分享一些思路，关于我多年来如何开发某些特定的应用——可以近似地称为‘’服务器‘’。更准确地说，我即将写就的是关于广义上旨在解决吞吐量（每秒处理数量巨大的非连续消息或者请求）的一类程序。网络服务器最符合这个定义，但并非所有程序所完成的功能是真正意义上的服务器程序。简而言之，因为“高性能请求处理问题”真的是一个非常蹩脚称谓，所以我们称这类程序叫做“服务器”，且一直以来就是这么叫的。
>

<!--more-->

I will not be writing about "mildly parallel" applications, even  though multitasking within a single program is now commonplace.  The  browser you're using to read this probably does some things in parallel,  but such low levels of parallelism really don't introduce many  interesting challenges.  The interesting challenges occur when the  request-handling infrastructure itself is the limiting factor on overall  performance, so that improving the infrastructure actually improves  performance.  That's not often the case for a browser running on a  gigahertz processor with a gigabyte of memory doing six simultaneous  downloads over a DSL line.  The focus here is not on applications that  sip through a straw but on those that drink from a firehose, <u>on the very  edge of hardware capabilities where how you do it really does matter</u>.



> 我不去写关于“轻并行”(原文为mildly parallel)的应用，尽管现在单个程序中处理多个任务是家常便饭。你正在使用读取这篇文章的浏览器极有可能就是并行处理的，但是这些低层次的并行机制并不会引起多少有意思的挑战。有趣的挑战是发生在请求处理机制从整体性能意义上自身成为了限制因素，所以改进（请求处理）机制实质上是提高性能。一个浏览器运行在配置GB级别内存和GHz级别处理器的机器上，通过一根DSL（）线路同时处理6个下载任务，那不是经常发生的情况。这篇文章的焦点不是吸管式yunxi般的应用程序，而是消防管道式狂喷的应用程序，专注于硬件处理能力的各个边界， 
>



Some people will inevitably take issue with some of my comments and  suggestions, or think they have an even better way.  Fine.  I'm not  trying to be the Voice of God here; these are just methods that I've  found to work for me, not only in terms of their effects on performance  but also in terms of their effects on the difficulty of debugging or  extending code later.  Your mileage may vary.  If something else works  better for you that's great, but be warned that almost everything I  suggest here exists as an alternative to something else that I tried  once only to be disgusted or horrified by the results.  <u>Your pet idea  might very well feature prominently in one of these stories</u>, and  innocent readers might be bored to death if you encourage me to start  telling them.  You wouldn't want to hurt them, would you?



> 有些人不可避免地不同意我的某些评论和建议，或者认为他们有更好的方法。也行，在这里我并打算去当什么神的代言人；这些仅仅是我已发现于我行之有效的一些方法而已，(它们)不仅仅关于性能上的影响，还关于调试的复杂度上以及后期的扩展上的影响。你的开发可能会变化。如果有些（建议）更好地解决了你的问题自然很好，但是要小心的是，几乎我这里的每一项建议在处理其他状况都还有另一个选择，（针对这些情况）我仅试过一次，其结果是厌恶而又惊悚的。,,, 如果你鼓励我去告诉那些傲慢的读者，那他们将会无聊致死的。你也不想伤害他们，是吧？
>



The rest of this article is going to be centered around what I'll call the Four Horsemen of Poor Performance:

1. Data copies
2. Context switches
3. Memory allocation
4. Lock contention



> 文章接下类的部分将会把重心放在我所谓的差性能的四个消防员问题。
>
> 1. 数据拷贝
> 2. 上下文切换
> 3. 内存分配
> 4. 锁争用
>



There will also be a catch-all section at the end, but these are the  biggest performance-killers.  If you can handle most requests without  copying data, without a context switch, without going through the memory  allocator and without contending for locks, you'll have a server that  performs well even if it gets some of the minor parts wrong.

> 文章将会覆盖所有(四个)部分，但这些并不是最大的性能杀手。如果你可以处理大多数请求，而没有发生数据拷贝、没有发生上下文切换、没有发生内存分配、没有发生锁争用等问题，你所实现的是一个性能还不错的服务器，尽管它会发生一些微小的错误。
>



This could be a very short section, for one very simple reason: most  people have learned this lesson already.  Everybody knows data copies  are bad; it's obvious, right?  Well, actually, it probably only seems  obvious because you learned it very early in your computing career, and  that only happened because somebody started putting out the word decades  ago.  I know that's true for me, but I digress.  Nowadays it's covered  in every school curriculum and in every informal how-to.  <u>Even the  marketing types have figured out that "zero copy" is a good buzzword.</u>

> 本部分将会特别简短，一个简单的原因是：大多数人早已了解了这些节内容。每个人都知道数据拷贝是糟糕的；非常明显，是吧？因为在你的计算机学习阶段早就知道这个事实（数据拷贝不好），你之所以会学习到这样的观点是因为几十年前就有人提出了。我自己就是这样，但我并准备在这里讨论（数据拷贝是糟糕的）。目前来说，数据拷贝是糟糕这一说法，覆盖在每一所大学课程中，以及非正式的教程(原文为how-to)中。尽管工业界早已指明“零拷贝”是不错的流行说法。
>



Despite the after-the-fact obviousness of copies being bad, though,  there still seem to be nuances that people miss.  The most important of  these is that data copies are often hidden and disguised.  Do you really  know whether any code you call in drivers or libraries does data  copies?  It's probably more than you think.  Guess what "Programmed I/O"  on a PC refers to.  An example of a copy that's disguised rather than  hidden is a hash function, which has all the memory-access cost of a  copy and also involves more computation.  <u>Once it's pointed out that  hashing is effectively "copying plus" it seems obvious that it should be  avoided</u>, but I know at least one group of brilliant people who had to  figure it out the hard way.  If you really want to get rid of data  copies, either because they really are hurting performance or because  you want to put "zero-copy operation" on your hacker-conference slides,  you'll need to track down a lot of things that really are data copies  but don't advertise themselves as such.

> 尽管数据拷贝比较糟糕已为人所众知，还是有一些细微的差别会被忽视。主要是因为数据拷贝常常很隐蔽不易察觉。你真的知道调用驱动程序或者调用第三方库时是否发生了数据拷贝么？这个过程极有可能比你预想的发生了更多的拷贝。试想一下，PC端IO编程更青睐的处理方式。举个hash函数的迷惑人眼的数据拷贝例子，hash函数对于每一次拷贝都是访问内存的代价，并且伴随着更多的CPU计算。一旦指出了hash会发生内存级拷贝，那么避免它是毫无疑问的，然而据我所知，至少有一群专家级人物使用了非常艰难的方式。如果你真的想完全避免数据拷贝，要么因为其影响了性能，要么因为你想在你的分享会讲稿上展示你的“零拷贝操作”，那么你将需要深挖许多领域的真正发生数据拷贝之处，但不要对如此这些做法进行布道。
>



The tried and true method for avoiding data copies is to use  indirection, and pass buffer descriptors (or chains of buffer  descriptors) around instead of mere buffer pointers.  Each descriptor  typically consists of the following:

- A pointer and length for the whole buffer.
- A pointer and length, or offset and length, for the part of the buffer that's actually filled.
- Forward and back pointers to other buffer descriptors in a list.
- A reference count.

> 经验表明正确避免数据拷贝的方式是使用间接方式，通过传递缓冲去描述符（或者多个缓冲区描述符），而非传递缓冲区指针。典型做法是每个描述符都包含以下内容：
>
> - 一个指针以及该缓冲区的长度
> - 一个指针以及长度，可以是偏移量或者长度，用以指明缓冲区实际有效的数据部分
> - 缓冲区描述符双向链表中的前向指针与后向指针
> - 一个引用计数器
>



Now, instead of copying a piece of data to make sure it stays in  memory, code can simply increment a reference count on the appropriate  buffer descriptor.  This can work extremely well under some conditions,  including the way that a typical network protocol stack operates, but it  can also become a really big headache.  Generally speaking, it's easy  to add buffers at the beginning or end of a chain, to add references to  whole buffers, and to deallocate a whole chain at once.  Adding in the  middle, deallocating piece by piece, or referring to partial buffers  will each make life increasingly difficult.  Trying to split or combine  buffers will simply drive you insane.

> 目前，针对拷贝小段数据以保证其常驻内存，取而代之的做法是，在代码中增加缓冲区描述符的相应引用计数器。通常这样的做法会表现的极为高效，典型的应用有如特定网络协议栈操作，但是，这样也会让人头大。通常数来，在链表头部和尾部增加操作，对整个缓冲区描述符进行引用计数器加操作，以及一次释放所有缓冲区，这些都相对容易。在（链表）中间操作，对部分缓冲区描述符进行引用计数器减操作，或者挨个释放缓冲区(注：此处调整了原文顺序)，都将使得难度陡增。试着切分或者合并缓冲区都会让人陷入疯狂。
>



I don't actually recommend using this approach for everything, though.  Why not?  Because it gets to be a *huge*  pain when you have to walk through descriptor chains every time you  want to look at a header field.  There really are worse things than data  copies.  I find that the best thing to do is to identify the large  objects in a program, such as data blocks, make sure *those* get allocated separately as described above so that they don't need to be copied, and not sweat too much about the other stuff.

> 但事实上，我并不建议把它当万能膏药。因为，当你需要遍历描述符表时，每一次都需要访问表头，相较于数据拷贝而言，这无疑更糟糕，带来更大的性能负担。我认为，最为有效的方式，识别你程序中诸如数据块等的大(内存)对象，确保这些对象如上述方式那样单独分配内存以使它们不必每次都拷贝，而其他(小内存对象)则可以不必太过于关注(其数据拷贝引发的性能负担)。
>



This brings me to my last point about data copies: don't go overboard  avoiding them.  I've seen way too much code that avoids data copies by  doing something even worse, like forcing a context switch or breaking up  a large I/O request.  Data copies are expensive, and when you're  looking for places to avoid redundant operations they're one of the  first things you should look at, <u>but there is a point of diminishing  returns.  Combing through code and then making it twice as complicated  just to get rid of that last few data copies is usually a waste of time  that could be better spent in other ways.</u>

> 最后关于数据拷贝的一点是，不要尝试去避免它。我见过大量诸如通过强制使用上下文切换或者中断一个大数据IO请求等，试图避免数据拷贝的代码，反而使得事情更糟。在避免大量操作的情况下数据拷贝是首要需要关注的，因为数据拷贝的代价昂贵，但是，
>



Whereas everyone thinks it's obvious that data copies are bad, I'm  often surprised by how many people totally ignore the effect of context  switches on performance.  In my experience, context switches are  actually behind more total "meltdowns" at high load than data copies;  the system starts spending more time going from one thread to another  than it actually spends within any thread doing useful work.  The  amazing thing is that, at one level, it's totally obvious what causes  excessive context switching.  The #1 cause of context switches is having  more active threads than you have processors.  As the ratio of active  threads to processors increases, the number of context switches also  increases - linearly if you're lucky, but often exponentially.  This  very simple fact explains why multi-threaded designs that have one  thread per connection scale very poorly.  The only realistic alternative  for a scalable system is to limit the number of active threads so it's  (usually) less than or equal to the number of processors.  One popular  variant of this approach is to use only one thread, ever; while such an  approach does avoid context thrashing, and avoids the need for locking  as well, <u>it is also incapable of achieving more than one processor's  worth of total throughput and thus remains beneath contempt unless the  program will be non-CPU-bound (usually network-I/O-bound) anyway.</u>

> 每个人都会觉得数据拷贝是糟糕的做法，而我常常诧异于有如此多的人完全忽视了上下文切换对于性能的影响。从我的经验看来，上下文切换实际上背后的负担比数据拷贝更高；相较于线程本身所做的有效工作，系统花费了更多的时间用于线程之间切换。非常有趣的是，一方面，什么造成了上下文切换事比较明显的，第一，运行着的线程(原文为active)数量比处理器核心(原文为processor)数多的多，由于运行态线程数与处理器核心数的比值的增加，线程上下文切换的次数也会增加——如果比较幸运的话是线性增加，单常常是指数增加。这个简单事实解释了，为什么在大规模系统的多线程设计中，一个线程处理一个连接是非常低效的。在大规模系统下，唯一务实的选择是限制运行态的线程数量使其不大于处理器核心数。一个比较流行的实现是只用一个线程，在解决了线程切换造成的新能负担的同时，还解决了对（线程）锁的需求，... 
>



The first thing that a "thread-frugal" program has to do is figure  out how it's going to make one thread handle multiple connections at  once.  This usually implies a front end that uses select/poll,  asynchronous I/O, signals or completion ports, with an event-driven  structure behind that.  Many "religious wars" have been fought, and  continue to be fought, over which of the various front-end APIs is best.   Dan Kegel's [C10K paper](http://www.kegel.com/c10k.html) is  a good resource is this area.  Personally, I think all flavors of  select/poll and signals are ugly hacks, and therefore favor either AIO  or completion ports, but it actually doesn't matter that much.  They all  - except maybe select() - work reasonably well, and don't really do  much to address the matter of what happens past the very outermost layer  of your program's front end.

> “简约线程型”程序的首要一着是，必须解决一次保证多个连接问题。通常的做法是前端使用select/poll，异步IO，信号机制，或者[IOCP](https://en.wikipedia.org/wiki/Input/output_completion_port)(注:译文添加，原文无此链接)，后端使用事件驱动架构。许多关于哪个前端API是最好的“信仰战争”被引发，现在还在激战。 [C10K paper](http://www.kegel.com/c10k.html) ——Dan Kegel的这篇文章是这个领域的非常好的资料。个人而言，所有支持select/poll以及信号机制的，都是比较蹩脚的骇客，那些要么支持异步IO要么支持IOCP的人，也不见得高明多少。他们都期望`select()`函数逻辑表现很好，而并没有解决程序前端传输发生了什么这一个事实。
>



The simplest conceptual model of a multi-threaded event-driven server  has a queue at its center; requests are read by one or more "listener"  threads and put on queues, from which one or more "worker" threads will  remove and process them.  Conceptually, this is a good model, but all  too often people actually implement their code this way.  Why is this  wrong?  Because the #2 cause of context switches is transferring work  from one thread to another.  <u>Some people even compound the error by  requiring that the response to a request be sent by the original thread -  guaranteeing not one but two context switches per request</u>.  It's very  important to use a "symmetric" approach in which a given thread can go  from being a listener to a worker to a listener again without ever  changing context.  Whether this involves partitioning connections  between threads or having all threads take turns being listener for the  entire set of connections seems to matter a lot less.

> 最为简单的多线程事件驱动服务器概念模型是持有一个队列；请求被一个或者多个“监听者”判断就绪后，就加到这个队列中，而后，一个或者多个“工作者”线程从该队列中摘取请求来进行处理。从概念上讲，这是一个较好的模型。但是，很少有人按照这个模型去实现。为什么会这样南辕北辙呢？因为第二个步骤会引发工作线程之间的切换。<u>有些人在处理应答原始线程发过来的请求时——为了保证每个请求是一个线程处理而没有发生两个线程切换，把事情搞得更加糟糕</u>。使用“对称方法”，即使得给定一个线程能够从“监听者”到“工作者”，再到“监听者”，而没有进行线程切换，是非常重要的。这个过程中，是否包含了线程间局部连接 或者所有线程轮训作为监听者，对性能影响微乎其微。
>



Usually, it's not possible to know how many threads will be active  even one instant into the future.  After all, requests can come in on  any connection at any moment, or "background" threads dedicated to  various maintenance tasks could pick that moment to wake up.  If you  don't *know* how many threads are active, how can you *limit*  how many are active?  In my experience, one of the most effective  approaches is also one of the simplest: use an old-fashioned counting  semaphore which each thread must hold whenever it's doing "real work".   If the thread limit has already been reached then each listen-mode  thread might incur one extra context switch as it wakes up and then  blocks on the semaphore, but once all listen-mode threads have blocked  in this way they won't continue contending for resources until one of  the existing threads "retires" so the system effect is negligible.  More  importantly, this method handles maintenance threads - which sleep most  of the time and therefore dont' count against the active thread count -  more gracefully than most alternatives.

> 通常情况下，不可能知道未来有多少个运行态线程，在单进程（原文为instance）下也是如此。毕竟，请求是神出鬼没的，以及，“后端”线程(即多个就绪态任务)也在等着请求随时来触发。如果你不知道有多少个运行态的线程，那如何设置运行态线程的数量阀值。我的经验是，一个最为高效的做法同时也是最为简单的：使用一个老套的信号量计数器，每个线程只要在做“有效工作”就必须保持（这个计数器）。如果运行态的线程数量超过了设置的阀值，耳后每一个监听态的线程就需要一个额外的上下文切换，因为其首先进入运行态然后因为信号量而阻塞，但是，一旦所有的监听态的线程都因为超过阀值都进入了阻塞态，它们将不会再继续参与资源竞争，直到运行态下的某个线程被淘汰，因此对整个系统的效率影响比较小。更为重要的是，这种方法比其他方法在处理就绪态线程（大多数时候都在休眠因而不必与运行态的线程争夺引用计数）这种情况时显得更为轻量（原文为gracefully）。
>



Once the processing of requests has been broken up into two stages  (listener and worker) with multiple threads to service the stages, it's  natural to break up the processing even further into more than two  stages.  In its simplest form, processing a request thus becomes a  matter of invoking stages successively in one direction, and then in the  other (for replies).  However, things can get more complicated; a stage  might represent a "fork" between two processing paths which involve  different stages, or it might generate a reply (e.g. a cached value)  itself without invoking further stages.  Therefore, each stage needs to  be able to specify "what should happen next" for a request.  There are  three possibilities, represented by return values from the stage's  dispatch function:

- <u>The request needs to be passed on to another stage (an ID or pointer in the return value).</u>
- The request has been completed (a special "request done" return value)
- The request was blocked (a special "request blocked" return value).   This is equivalent to the previous case, except that the request is not  freed and will be continued later from another thread.

> 一旦请求处理被分成多线程下提供服务的两个状态（监听者和工作者），很自然地处理过程得处理超过两个状态。在这种简单模式下，一个请求处理即为触发从监听态到工作态的改变，以及响应(客户端)请求这么个过程。然而，事情却比这要复杂得多；一种可能是在监听态和工作态之间的切换态，一种(例如缓存值)可能是进行了回复却没有进行状态切换。因此，每一个状态都需要有能力指明对于每一个请求的下一个状态是什么。通过采用处理函数(注：原文为dispatch)返回值方式，有三种可能策略：
>
> - 请求需要被传递到其他状态(返回值的ID或者指针)
> - 请求已经完成(一个特定“请求处理完成”的返回值)
> - 请求被阻塞(一个特定“请求被阻塞”的返回值)，除了请求没有被释放还可以被其他线程激活外，这和前一种情况相同
>



Note that, in this model, queuing of requests is done *within*  stages, not between stages.  This avoids the common silliness of  constantly putting a request on a successor stage's queue, then  immediately invoking that successor stage and dequeuing the request  again; I call that lots of queue activity - and locking - for nothing.

> 需要注意的是，在这种模型下，请求是在某一个状态下处理，而非两个状态之间。这就避免了总是把请求放在队尾弊端，同时马上能够触发尾部状态的那个请求出队；我认为大量队列活动以及锁操作不值一提。
>



If this idea of separating a complex task into multiple smaller  communicating parts seems familiar, that's because it's actually very  old.  <u>My approach has its roots in the [Communicating Sequential Processes](http://www.afm.sbu.ac.uk/csp/)  concept elucidated by C.A.R. Hoare in 1978, based in turn on ideas from  Per Brinch Hansen and Matthew Conway going back to 1963 - before I was  born!</u>  However, when Hoare coined the term CSP he meant "process" in the  abstract mathematical sense, and a CSP process need bear no relation to  the operating-system entities of the same name.  In my opinion, the  common approach of implementing CSP via thread-like coroutines within a  single OS thread gives the user all of the headaches of concurrency with  none of the scalability.

> 如果这个可以把一个复杂任务分解为多个较小的互相通信的部分的这个观点看起来很类似，那是因为这个思路本来就历史久远。我的方式来自
>



A contemporary example of the staged-execution idea <u>evolved in a saner direction is Matt Welsh's [SEDA](http://www.cs.berkeley.edu/~mdw/proj/seda/)</u>.   In fact, SEDA is such a good example of "server architecture done  right" that it's worth commenting on some of its specific  characteristics (especially where those differ from what I've outlined  above). 

1. SEDA's "batching" tends to emphasize processing multiple requests  through a stage at once, while my approach tends to emphasize processing  a single request through multiple stages at once.
2. SEDA's one significant flaw, in my opinion, is that it allocates a  separate thread pool to each stage with only "background" reallocation  of threads between stages in response to load.  As a result, the #1 and  #2 causes of context switches noted above are still very much present.
3. In the context of an academic research project, implementing SEDA in  Java might make sense.  In the real world, though, I think the choice  can be characterized as unfortunate.

> 当下的状态执行观点例证是...，事实上，
>



Allocating and freeing memory is one of the most common operations in  many applications.  Accordingly, many clever tricks have been developed  to make general-purpose memory allocators more efficient.  However, <u>no  amount of cleverness can make up for the fact that the very generality  of such allocators inevitably makes them far less efficient than the  alternatives in many cases</u>.  I therefore have three suggestions for how  to avoid the system memory allocator altogether.

> 内存分配与释放是许多应用的公共操作之一。因此，许多聪明的技巧被开发出来用以使得内存分配器更加高效。但是，再多的聪明技巧也无法弥补这样一个事实：在许多情况下，这种(内存)分配器的通用性不可避免地使它们的效率远远低于其他选择。因此，对于如何完全避免系统内存分配器，我有三个建议。
>

Suggestion #1 is simple preallocation.  We all know that static  allocation is bad when it imposes artificial limits on program  functionality, but there are many other forms of preallocation that can  be quite beneficial.  Usually the reason comes down to the fact that one  trip through the system memory allocator is better than several, even  when some memory is "wasted" in the process.  Thus, if it's possible to  assert that no more than N items could ever be in use at once,  preallocation at program startup might be a valid choice.  Even when  that's not the case, preallocating everything that a request handler  might need right at the beginning might be preferable to allocating each  piece as it's needed; aside from the possibility of allocating multiple  items contiguously in one trip through the system allocator, this often  greatly simplifies error-recovery code.  If memory is very tight then  preallocation might not be an option, but in all but the most extreme  circumstances it generally turns out to be a net win.

> 第一个建议是简单的预分配。静态(内存)分配在手动限制程序功能是糟糕的，但是仍有许多其他形式的预分配是非常有益的。通常一个原因是一次系统内存分配比多次分配要好，尽管进程中会“浪费”一些内存。因此，如果可以断言一次使用的内存项不超过N，那么在程序启动时进行(内存)预分配可能是一个有效的选择。即使不是这样，在一开始就预先分配请求处理程序可能需要的所有(内存)也可能比在需要时才进行一点点地分配要好。除了在一次系统分配中持续分配多个内存项的可能性，预分配常会极大简化错误恢复码。如果系统内存非常紧张，那么预分配不会是个好选择，然而，在除了极其极端的情况外的所有情况，预分配通常是非常好的选择。
>



Suggestion #2 is to use lookaside lists for objects that are  allocated and freed frequently.  The basic idea is to put recently-freed  objects onto a list instead of actually freeing them, in the hope that  if they're needed again soon they need merely be taken off the list  instead of being allocated from system memory.  As an additional  benefit, transitions to/from a lookaside list can often be implemented  to skip complex object initialization/finalization.

> 第二个建议是，对哪些经常需要分配和释放的对象使用一个旁路查询表。最基本的观点是，把那些最近释放的对象放到一个链表中而非直接释放掉，旨在它们需要被立刻调用时能够从链表中取出而非重新从系统中申请内存。作为一个额外的好处是，从旁路查询表中添加或者移除一个对象都会(注:原文为often，这里译作always)跳过复杂得对象析构与构造(注:这里调整了原文顺序)操作。
>



It's generally undesirable to have lookaside lists grow without  bound, never actually freeing anything even when your program is idle.   Therefore, it's usually necessary to have some sort of periodic  "sweeper" task to free inactive objects, but it would also be  undesirable if the sweeper introduced undue locking complexity or  contention.  A good compromise is therefore a system in which a  lookaside list actually consists of separately locked "old" and "new"  lists.  Allocation is done preferentially from the new list, then from  the old list, and from the system only as a last resort; objects are  always freed onto the new list.  The sweeper thread operates as follows:

1. Lock both lists.
2. Save the head for the old list.
3. Make the (previously) new list into the old list by assigning list heads.
4. Unlock.
5. Free everything on the saved old list at leisure.

> 通常旁路查询表无限的增长并不是所理想的，即使当你的程序空闲时也没有释放任何(对象)。因此，周期性的释放不活跃的对象的“清理”任务，通常是有必要的。但是，如果清理任务引进了复杂的锁机制和(资源)竞争，仍然不是理想的。因此，一个比较好的折中做法是，在实现了旁路查询表的系统中分别使用一个“老的”的和一个“新的”查询表。分配操作优先从“新的”查询表中开始，其次才是“老的”查询表，最后才是从系统中申请。清理线程的操作如下：
>
> 1. 对两个查询表加锁
> 2. 保存老查询表的表头
> 3. 通过分配表头把(之前的)新表变为老表
> 4. 解锁
> 5. 在空闲时释放老表上的所有对象。
>



Objects in this sort of system are only actually freed when they have  not been needed for at least one full sweeper interval, but always less  than two.  Most importantly, the sweeper does most of its work without  holding any locks to contend with regular threads.  In theory, the same  approach can be generalized to more than two stages, but I have yet to  find that useful.

> 在此类系统下的对象仅在至少一个完整的清理周期内没有被再次使用才会被释放，但不会超过两个周期。最为重要的是，清理线程在大部分情况并没有和常规工作线程产生锁争用。理论上，同样的方法可以应用到不止两个状态的情况，但我并没有其实用之处。
>



One concern with using lookaside lists is that the list pointers  might increase object size.  In my experience, most of the objects that  I'd use lookaside lists for already contain list pointers anyway, so  it's kind of a moot point.  Even if the pointers were only needed for  the lookaside lists, though, the savings in terms of avoided trips  through the system memory allocator (and object initialization) would  more than make up for the extra memory.

> 使用旁路查询表的一个担忧是，链表的指针域可能会使对象大小膨胀。我的经验是，大多数我已经采用旁路查询表的对象早就包含指针域，所以这是个有争议的问题。但是，即使这些指针只用于旁路查询表，通过避免系统内存分配器(和对象初始化)的访问所节省的开销也足以弥补额外的内存。
>



Suggestion #3 actually has to do with locking, which we haven't  discussed yet, but I'll toss it in anyway.  Lock contention is often the  biggest cost in allocating memory, even when lookaside lists are in  use.  One solution is to maintain multiple private lookaside lists, such  that there's absolutely no possibility of contention for any one list.   For example, you could have a separate lookaside list for each thread.   One list per processor can be even better, due to cache-warmth  considerations, but only works if threads cannot be preempted.  The  private lookaside lists can even be combined with a shared list if  necessary, to create a system with extremely low allocation overhead.

> 第三个建议是采用还没有讨论的锁机制，但我还是把它提出来。即使是采用了旁路查询表，锁争用通常还是内存分配的最大消耗。一种解决方案是维护多个私有的旁路查询表，这样在任何一个查询表中就不存在竞争了。比如：你可以使用每个线程使用一个查询表。处于热缓存考量，一个核一个查询表将会更好，但只有线程不被抢占时才有效。私有旁路查询表甚至可以在需要时与共享表相结合使用，用以建立一个内存开销极低的系统。
>



Efficient locking schemes are notoriously hard to design, because of  what I call Scylla and Charybdis after the monsters in the Odyssey.   Scylla is locking that's too simplistic and/or coarse-grained,  serializing activities that can or should proceed in parallel and thus  sacrificing performance and scalability; Charybdis is overly complex or  fine-grained locking, with space for locks and time for lock operations  again sapping performance.  Near Scylla are shoals representing deadlock  and livelock conditions; near Charybdis are shoals representing race  conditions.  In between, there's a narrow channel that represents  locking which is both efficient and correct...or is there?  Since  locking tends to be deeply tied to program logic, it's often impossible  to design a good locking scheme without fundamentally changing how the  program works.  This is why people hate locking, and try to rationalize  their use of non-scalable single-threaded approaches.

> 总所周知，高效的锁机制模式设计起来非常困难，因为我所谓的继《奥德赛》中的怪物：希拉以及卡里布迪斯。希拉是简单的、粗粒度的锁，序列化了那些本可以也应该并行处理的操作，从而牺牲了性能和伸缩性；卡里布迪斯是复杂的、细粒度的锁，其操作在空间和时间上都降低性能。接近怪物希拉，代表着死锁与活锁情况；接近怪物卡里布迪斯，代表着资源争用情况。在这两者之间，存在着一条代表着通往既高效又正确的狭窄通道 ... 或者是在那两者之间吗？由于锁机制会深深绑定到程序逻辑中，因此常常不太可能设计一个比较好的、不会根本性的改变程序原来运作方式的锁模式。这是为什么人们不喜欢锁机制的原因，从而试图使用非伸缩的单线程方式是系统合理化。
>



Almost every locking scheme starts off as "one big lock around  everything" and a vague hope that performance won't suck.  When that  hope is dashed, and it almost always is, the big lock is broken up into  smaller ones and the prayer is repeated, and then the whole process is  repeated, presumably until performance is adequate.  Often, though, each  iteration increases complexity and locking overhead by 20-50% in return  for a 5-10% decrease in lock contention.  With luck, the net result is  still a modest increase in performance, but actual decreases are not  uncommon.  The designer is left scratching his head (I use "his" because  I'm a guy myself; get over it).  "I made the locks finer grained like  all the textbooks said I should," he thinks, "so why did performance get  worse?"

> 几乎每一中锁模式都是从所谓的“所有情况的一个大锁”开始的，寄希望于性能不会变的糟糕。当希望落空时，几乎总是将大锁分解成较小的锁在期望性能不会变糟，然后重复整个处理过程，直到性能满足需求。
>



In my opinion, things got worse because the  aforementioned approach is fundamentally misguided.  Imagine the  "solution space" as a mountain range, with high points representing good  solutions and low points representing bad ones.  The problem is that  the "one big lock" starting point is almost always separated from the  higher peaks by all manner of valleys, saddles, lesser peaks and dead  ends.  It's a classic hill-climbing problem; trying to get from such a  starting point to the higher peaks only by taking small steps and never  going downhill almost never works.  What's needed is a *fundamentally* different way of approaching the peaks.

> 我的观点是，事情变糟糕的原因是上述的方法从本质上说就带有误导性。想象一个大山范围的“解空间”，高点代表着良好的解决方案，低点代表糟糕的解决方案。问题是，大锁的起点几乎总是偏离（大山）的最高点。
>



The first thing you have to do is form a mental map of your program's locking.  This map has two axes:

- The vertical axis represents code.  If you're using a staged  architecture with non-branching stages, you probably already have a  diagram showing these divisions, like the ones everybody uses for  OSI-model network protocol stacks.
- The horizontal axis represents data.  In every stage, each request  should be assigned to a data set with its own resources separate from  any other set.

> 你首要需要处理的是形成一张程序的锁机制脑图，这张图包含两个坐标：
>
> - 纵轴代表着代码。如果你正使用无分支状态的体系结构，你可能早已有了一个刻画这些维度的示图，就像大家都是用的OSI模型网络协议栈一样。
> - 横轴代表着数据。在每个状态下，每个请求都应该被分配至一个数据集中，该数据集的资源与其他数据集资源分开。
>



You now have a grid, where each cell represents a particular data set  in a particular processing stage.  What's most important is the  following rule: two requests should not be in contention unless they are  in the same data set *and* the same processing stage.  If you can manage that, you've already won half the battle.

> 现在你有了一张表格，这个表格中的每一个单元格代表着一个特定状态下的相应数据集。最重要的是这些规则：两个请求不应该产生竞争，除非他们在同一个数据集，而且在同一个处理状态下。如果你可管理这些，在这场战斗中就已经赢了一半。
>



Once you've defined the grid, every type of locking your program does  can be plotted, and your next goal is to ensure that the resulting dots  are as evenly distributed along both axes as possible.  Unfortunately,  this part is very application-specific.  You have to think like a  diamond-cutter, using your knowledge of what the program does to find  the natural "cleavage lines" between stages and data sets.  Sometimes  they're obvious to start with.  Sometimes they're harder to find, but  seem more obvious in retrospect.  Dividing code into stages is a  complicated matter of program design, so there's not much I can offer  there, but here are some suggestions for how to define data sets:

- If you have some sort of a block number or hash or transaction ID  associated with requests, you can rarely do better than to divide that  value by the number of data sets.
- Sometimes, it's better to assign requests to data sets dynamically,  based on which data set has the most resources available rather than  some intrinsic property of the request.  Think of it like multiple  integer units in a modern CPU; those guys know a thing or two about  making discrete requests flow through a system.
- It's often helpful to make sure that the data-set assignment is  different for each stage, so that requests which would contend at one  stage are guaranteed not to do so at another stage.

> 一旦你定义了这个表格，就可以绘制程序中各个类型的锁了，下一个目标是确保每一个生成的点尽可能均匀分布在两个坐标上。不幸的是，这部分跟具体应用程序关系密切。你得像钻石切割师一样思考，调用你知道程序原理的相关知识去找到处于状态和数据集之间的自然“切割线”。有时像昨夜西风凋碧树独上高楼望尽天涯路般，有时需要衣带渐宽终不悔为伊消得人憔悴般，但更多时候是众里荨她千百度慕然回首那人却在灯火阑珊处般。将程序(注:原文为code)分解成若干个状态本是程序设计的复杂之处，而我这里也不能分享(注:原文为offer)很多，但是，我可以给出一些就如何定义数据的建议。
>
> - 如果你有许多请求相关的块号或者hash值又或者事务ID，你最好将他们除以数据集
> - 有时候，相较于请求的固有属性，基于哪些数据集有更加可用的资源，动态地为其数据集分配请求会更好。把它想象成线程CPU中的多个整数单元；它们对离散的请求在系统中流动知道一二。
> - 确保每个状态下数据集的不同，常常是有帮助的。因此，请求在一个状态中竞争将会被保证不会在另一个状态中也如此。
>



If you've divided your "locking space" both vertically and  horizontally, and made sure that lock activity is spread evenly across  the resulting cells, you can be pretty sure that your locking is in  pretty good shape.  There's one more step, though.  Do you remember the  "small steps" approach I derided a few paragraphs ago?  It still has its  place, because now you're at a good starting point instead of a  terrible one.  In metaphorical terms you're probably well up the slope  on one of the mountain range's highest peaks, but you're probably not at  the top of one.  Now is the time to collect contention statistics and  see what you need to do to improve, splitting stages and data sets in  different ways and then collecting more statistics until you're  satisfied.  If you do all that, you're sure to have a fine view from the  mountaintop.

> 如果你已将所空间延横、纵轴划分，并且确保了锁操作均匀分布在生成的单元格中，那么就可以认定你的锁机制还不错。尽管，还有一步。还记得此前段落中我嘲讽的“众多步骤”？它们仍有借鉴意义，因为你正处于一个好的起点而非糟糕的起点。用比喻的说法就是，你已经爬上这座大山的峰顶之一的斜坡上，还没有到达最高点。现在是收集争用统计数据的时候了，看看需要做哪些改进，以不同的方式切分状态和数据集，然后收集更多的统计数据，知道满足需求。如果你做了所有这些，你一定可以从峰顶看到美丽的风景。
>



As promised, I've covered the four biggest performance problems in  server design.  There are still some important issues that any  particular server will need to address, though.  Mostly, these come down  to knowing your platform/environment:

- How does your storage subsystem perform with larger vs. smaller  requests?  With sequential vs. random?  How well do read-ahead and  write-behind work?
- How efficient is the network protocol you're using?  Are there  parameters or flags you can set to make it perform better?  Are there  facilities like TCP_CORK, MSG_PUSH, or the Nagle-toggling trick that you  can use to avoid tiny messages?
- Does your system support scatter/gather I/O (e.g. readv/writev)?   Using these can improve performance and also take much of the pain out  of using buffer chains.
- What's your page size?  What's your cache-line size?  Is it worth it  to align stuff on these boundaries?  How expensive are system calls or  context switches, relative to other things?
- Are your reader/writer lock primitives subject to starvation?  Of  whom?  Do your events have "thundering herd" problems?  Does your  sleep/wakeup have the nasty (but very common) behavior that when X wakes  Y a context switch to Y happens immediately even if X still has things  to do?

> 如前所述，我已经介绍了服务器设计中的4个最大性能难题。尽管，对于任何一个特定服务器需要解决(设计问题)，仍有许多重要的议题。最重要的是，这些都取决你对于平台环境的了解：
>
> - 你的存储子系统在大请求以及小请求下的如何表现？是顺序请求还是随机请求？先读与后写如何工作得怎样？
> - 你的网络协议效率怎样？有设置参数或者标记来使得性能更好么？有类似TCP_CORK，MSG_PUSH或者Nagle算法之类的设施来比喵小消息么？
> - 你的系统是否支持类似readv/writev的分散/集中IO么？通过使用这些可以提高性能，而且可以从缓冲链中解脱出来。
> - 页表多大？cache行块多大？是否值得边界对其？系统调用或者上下文切换对于其他操作的代价是多少？
> - 读者/写者元语是否会导致饥饿？导致读饥饿还是写饥饿？事件驱动模型是否会有[惊蛰问题](https://en.wikipedia.org/wiki/Thundering_herd_problem)(注:超链接为译文所加)。当X唤醒Y时，是否是马上切换到Y，尽管X还有其他事情需要完成，休眠/唤醒元语是否会有脏数据行为（非常常见）。
>



I'm sure I could think of many more questions in this vein.  I'm sure  you could too.  In any particular situation it might not be worthwhile  to do anything about any one of these issues, but it's usually worth at  least thinking about them.  If you don't know the answers - many of  which you will not find in the system documentation - *find out*.   Write a test program or micro-benchmark to find the answers  empirically; writing such code is a useful skill in and of itself  anyway.  If you're writing code to run on multiple platforms, many of  these questions correlate with points where you should probably be  abstracting functionality into per-platform libraries so you can realize  a performance gain on that one platform that supports a particular  feature.

> 我相信我可以许多类似的问题，我认为你也可以。在任何特定的场景下，关于任何这些议题任意一个的处理可能都是不得当的，但是常常指的思考一下。如果你不知道答案，那么找到答案，因为从许多系统文档都找不到。写一个测试程序或者微基准测试程序，从经验上找答案；写这些代码本身就是非常有用的技能。如果你写多个平台下的代码，许多这些相关问题 应该被抽象到每一个系统库中，从而发现各个系统支持特定特性的性能提高。
>



The "know the answers" theory applies to your own code, too.  Figure  out what the important high-level operations in your code are, and time  them under different conditions.  This is not quite the same as  traditional profiling; it's about measuring *design* elements,  not actual implementations.  Low-level optimization is generally the  last resort of someone who screwed up the design.

> “知道答案”这一方法论，也可以应用到你自己的代码中。指出你代码中什么是重要高层操作，并在不同条件下计时。这不同于常规的性能分析；而是测试设计的技能要素，而非实际上的代码实现。低层次的优化通常是搞砸设计的那些人的最后一招。
>