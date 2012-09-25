libuv的基本概念 Basics of libuv
===============

libuv enforces an **asynchronous**, **event-driven** style of programming.  Its
core job is to provide an event loop and callback based notifications of I/O
and other activities.  libuv offers core utilities like timers, non-blocking
networking support, asynchronous file system access, child processes and more.

libuv强调 **异步** ， **事件驱动** 的编程风格。它的内核提供了一个事件循环，以及基
于回调的IO和其它活动的通知机制。除此之外，还有一系列的核心工具集，比如计时器、无
阻塞的网络支持、异步的文件系统访问，子进程等等。


事件循环 Event loops
-----------

In event-driven programming, an application express interest in certain events
and respond to them when they occur. The responsibility of gathering events
from the operating system or monitoring other sources of events is handled by
libuv, and the user can register callbacks to be invoked when an event occurs.
The event-loop usually keeps running *forever*. In pseudocode:

在基于事件的编程中，应用程序将对某些事件的发生表示关注，并在事件发生时做出响应。
libuv负责从操作系统或者其它被监视的事件源收集事件，而用户则负责注册待事件发生时的
回调处理函数。事件循环通常是持续运行的（死循环）。用伪码表示如下：

.. code-block:: python

    while there are still events to process:      # 只要仍有事件需要处理就：
        e = get the next event                    # 获取下一个事件
        if there is a callback associated with e: # 有与这个事件相关的回调函数？
            call the callback                     # 调用这些回调函数

Some examples of events are:

比如以下这些事件：

* File is ready for writing　文件写就绪事件
* A socket has data ready to be read　网络套接字上有数据到达，读就绪事件
* A timer has timed out　计时器超时事件

This event loop is encapsulated by ``uv_run()`` -- the end-all function when using
libuv.

这个事件循环被封装在 ``uv_run()`` ——使用libuv时最后调用的函数。

The most common activity of systems programs is to deal with input and output,
rather than a lot of number-crunching. The problem with using conventional
input/output functions (``read``, ``fprintf``, etc.) is that they are
**blocking**. The actual write to a hard disk or reading from a network, takes
a disproportionately long time compared to the speed of the processor. The
functions don't return until the task is done, so that your program is doing
nothing. For programs which require high performance this is a major roadblock
as other activities and other I/O operations are kept waiting.

在进行系统编程时，最常见的活动就是处理输入输出，而不是做数值相关的计算。而使用
常规的输入输出函数( ``read``, ``fprintf`` 等)的问题在于它们都是 **阻塞** 的。在往
硬盘写数据或者从网络中读取数据过程所花费的时间，较之于处理器的速度是很不匹配的。
而这些输入输出函数在未完成实际的写入或者读出任务之前是不会返回的，这将导致你
的程序此时什么都不能做。对于追逐高性能的程序，它阻塞了其它活动的进行，
并迫使其它IO操作被动等待。

One of the standard solutions is to use threads. Each blocking I/O operation is
started in a separate thread (or in a thread pool). When the blocking function
gets invoked in the thread, the processor can schedule another thread to run,
which actually needs the CPU.

一种针对这种问题的标准解决方案是使用线程。每一个阻塞的IO操作在单独的线程中启动
(或者丢入线程池中处理). 当线程调用了具有阻塞性质的函数，如上面的 ``read`` 和 ``fprintf``,
处理器可以调入其它需要使用CPU的线程来运行。


The approach followed by libuv uses another style, which is the **asynchronous,
non-blocking** style. Most modern operating systems provide event notification
subsystems. For example, a normal ``read`` call on a socket would block until
the sender actually sent something. Instead, the application can request the
operating system to watch the socket and put an event notification in the
queue. The application can inspect the events at its convenience (perhaps doing
some number crunching before to use the processor to the maximum) and grab the
data. It is **asynchronous** because the application expressed interest at one
point, then used the data at another point (in time and space). It is
**non-blocking** because the application process was free to do other tasks.
This fits in well with libuv's event-loop approach, since the operating system
events can be treated as just another libuv event. The non-blocking ensures
that other events can continue to be handled as fast they come in [#]_.

libuv采用另一种风格——异步、非阻塞的风格来解决这个问题。大多数现代操作系统提供了
事件通知子系统。举个例子，当应用程序在Socket上执行一个普通的 ``read`` 调用时将被阻塞，
直到另一端的发送者发送一些数据过来。另一种替代方式是，应用程序可以请求操作系统去监视socket，
并将事件通知放入到队列当中。而应用程序则在方便的时候（也许当前CPU正忙于数值计算）去检查队列
里的事件并提取数据。这是异步的，是因为应用在某一个时刻表达对数据感兴趣，而在另外一个时刻使
用这些数据。这是非阻塞的，因为应用可以自由的处理是其它任务。这很好的适应了libuv的事件循环方法，
因为操作系统事件正可以作为一个libuv事件来对待。这种非阻塞方式确保其它更快的事件得以尽快的被处理掉.

.. NOTE::

    How the I/O is run in the background is not of our concern, but due to the
    way our computer hardware works, with the thread as the basic unit of the
    processor, libuv and OSes will usually run background/worker threads and/or
    polling to perform tasks in a non-blocking manner.
    
    我们并不在意I/O在后台是如何进行的。但由于我们的计算机硬件工作方式——处理器线程为基本单元，
    libuv和操作系统往往都会以后台线程、工作者线程或者轮询的等非阻塞的方式来执行它们。

你好，世界 Hello World
-----------

With the basics out of the way, lets write our first libuv program. It does
nothing, except start a loop which will exit immediately.

了解了这些基本概念以后，我们来编写第一个libuv程序。它在开启一个事件循环之后就立即退出了，并没做什么事情。

.. rubric:: helloworld/main.c
.. literalinclude:: ../code/helloworld/main.c
    :linenos:

This program quits immediately because it has no events to process. A libuv
event loop has to be told to watch out for events using the various API
functions.

这个程序立即退出是因为它没有任何事件需要处理。我们可以使用不同的API来通知libuv事件循环监视不同的事件。

默认的事件循环 Default loop
++++++++++++

A default loop is provided by libuv and can be accessed using
``uv_default_loop()``. You should use this loop if you only want a single
loop.

libuv提供了一个默认的事件循环，可以使用 ``uv_default_loop()`` 来访问。如果你们只想要一个事件循环，你就应该使用这个。

.. note::

    node.js uses the default loop as its main loop. If you are writing bindings
    you should be aware of this.
    
    node.js使用默认的事件循环作为它的主循环。如果你正在为它编写接口绑定函数的话，你应该是知道这个的。

Watchers
--------

Watchers are how users of libuv express interest in particular events. Watchers
are opaque structs named as ``uv_TYPE_t`` where type signifies what the watcher
is used for. A full list of watchers supported by libuv is:

libuv用户通过Watchers来表示关注特定事件的发生。Watchers是一些以 ``uv_TYPE_t`` 的方式来命名的不透明的数据结构，
其中的type表示Watcher所针对的事件类型。以下列表完整列举了libuv所支持的Watcher.

.. rubric:: libuv watchers
.. literalinclude:: ../libuv/include/uv.h
    :lines: 184-202

.. note::

    All watcher structs are subclasses of ``uv_handle_t`` and often referred to
    as **handles** in libuv and in this text.
    
    所有Watcher结构都是 ``uv_handle_t`` 的子类，在libuv和本文中经常称为 **handles** （句柄）。

Watchers are setup by a corresponding::

各种类型的Watchers以相应初始化函数进行初始配置：

    uv_TYPE_init(uv_TYPE_t*)

function.

.. note::

    Some watcher initialization functions require the loop as a first argument.
    
    还有一些类型的Watcher的初始化函数需要以loop作为第一个参数。

A watcher is set to actually listen for events by invoking::

让一个Watcher开始监听事件，可以通过调用：

    uv_TYPE_start(uv_TYPE_t*, callback)

and stopped by calling the corresponding::

相应的让wather停止监听事件通过调用：

    uv_TYPE_stop(uv_TYPE_t*)

Callbacks are functions which are called by libuv whenever an event the watcher
is interested in has taken place. Application specific logic will usually be
implemented in the callback. For example, an IO watcher's callback will receive
the data read from a file, a timer callback will be triggered on timeout and so
on.

Callbacks（回调函数）是指当某些被Watcher所关注的事件在任何时候发生时将由libuv所调用的函数。
应用程序特定的处理逻辑通常是以回调的方式来实现的。比如，一个IO Watcher的回调函数将接收从文件读取的数据，
而一个计时器回调函数将在计时器超时时被触发，诸如此类。

Idling　空闲状态
++++++

Here is an example of using a watcher. An idle watcher's callback is repeatedly
called. There are some deeper semantics, discussed in :doc:`utilities`, but
we'll ignore them for now. Let's just use an idle watcher to look at the
watcher life cycle and see how ``uv_run()`` will now block because a watcher is
present. The idle watcher is stopped when the count is reached and ``uv_run()``
exits since no event watchers are active.
下面给出一个使用Watcher的例子。一个空闲状态Watcher的回调函数将被反复调用。这里涉及了一些
在 :doc:`utilities` 中讨论的较深入的语义，但现在我们忽略它们。让我们仅仅使用Idle watcher来
审视一下watcher的生命周期，并且看看 ``uv_run()`` 是如何在watcher存在时被阻塞的。Idle watcher
将在计数器计数满时停止，然后 ``uv_run()`` 因没有活动的watcher存在而退出。

.. rubric:: idle-basic/main.c
.. literalinclude:: ../code/idle-basic/main.c
    :emphasize-lines: 6,10,14-17

void \*data pattern

note about not necessarily creating type structs on the stack

注意并没有必要将这些类型的数据结构创建在栈上，此处只是为了演示。

.. [#] Depending on the capacity of the hardware of course.
