Threads
线程
=======

Wait a minute? Why are we on threads? Aren't event loops supposed to be **the
way** to do *web-scale programming*? Well no. Threads are still the medium in
which the processor does its job, and threads are mighty useful sometimes, even
though you might have to wade through synchronization primitives.
[吃惊]等一下？为什么讨论线程？难道事件循环不能成为一种 *web-scale programming* 的方式？
当然不是。线程仍然是处理器执行工作的载体，某些时候线程是有用武之地的，尽管你不得不费力
地处理同步原语。

Threads are used internally to fake the asynchronous nature of all the system
calls. libuv also uses threads to allow you, the application, to perform a task
asynchronously that is actually blocking, by spawning a thread and collecting
the result when it is done.
通过内部使用线程来使所有系统调用具有异步特性。libuv也利用线程来允许你的应用程序，
通过Spawning出一个新的线程，执行某些实际上是会阻塞的任务，当任务完成时，收集运行结果。
从而产生一种异步执行的效果。

Today there are two predominant thread libraries. The Windows threads
implementation and `pthreads`_. libuv's thread API is analogous to
the pthread API and often has similar semantics.
当前存在两个主流的线程库。一个是Windows线程实现，另一个是 `pthread`_.
libuv的线程API模拟了Pthread的API, 并具有相似的语义。

A notable aspect of libuv's thread facilities is that it is a self contained
section within libuv. Whereas other features intimately depend on the event
loop and callback principles, threads are complete agnostic, they block as
required, signal errors directly via return values and, as shown in the
:ref:`first example <thread-create-example>`, don't even require a running
event loop.
libuv线程设施的一个显著方面是它被自包含在libuv之中。尽管它的其它feature是紧密的依赖
于事件循环和回调原则，但线程是完全不可感知的，在必要时执行阻塞操作，也可以直接利用返
回值来告知错误发生，甚至可以像例子(:ref:`first example <thread-create-example>`)中那
样，不需要运行事件循环。

libuv's thread API is also very limited since the semantics and syntax of
threads are different on all platforms, with different levels of completeness.
libuv的线程API使用时受到了许多的限制，因为在不同的平台上实现进度的差异，
导致了线程的语义和用法并不一致。

This chapter makes the following assumption: **There is only one event loop,
running in one thread (the main thread)**. No other thread interacts
with the event loop (except using ``uv_async_send``). :doc:`multiple` covers
running event loops in different threads and managing them.
本章的使用将作这样一个假设： **在主线程中只运行一个事件循环** 。除了使用 ``uv_async_send`` 之外，
没有其它的线程和事件循环进行交互。另见:doc:`multiple` 讨论了在不同的线程中运行多个事件循环及其
交织过程。


Core thread operations
核心线程操作
----------------------

There isn't much here, you just start a thread using ``uv_thread_create()`` and
wait for it to close using ``uv_thread_join()``.
此例相当简单，你只要使用 ``uv_thread_create()`` 启动一个线程，然后使用 ``uv_thread_join()`` 
等待它关闭。

.. _thread-create-example:

.. rubric:: thread-create/main.c
.. literalinclude:: ../code/thread-create/main.c
    :linenos:
    :lines: 26-37
    :emphasize-lines: 3-7

.. tip::

    ``uv_thread_t`` is just an alias for ``pthread_t`` on Unix, but this is an
    implementation detail, avoid depending on it to always be true.
    在Unix上，``uv_thread_t`` 只是 ``pthread_t`` 的一个别名。但这是实现的细节，
    必须信赖这一前提，或许某天就不是这样了。

The second parameter is the function which will serve as the entry point for
the thread, the last parameter is a ``void *`` argument which can be used to pass
custom parameters to the thread. The function ``hare`` will now run in a separate
thread, scheduled pre-emptively by the operating system:
``uv_thread_create()`` 的第二个参数是作为线程入口点的函数，最后的参数是一个 ``void *``
类型的参数，可以用来向线程传递参数。函数 ``hare`` 将在单独的线程中执行，并由操作系统
进行抢占式调度。

.. rubric:: thread-create/main.c
.. literalinclude:: ../code/thread-create/main.c
    :linenos:
    :lines: 6-14
    :emphasize-lines: 2

Unlike ``pthread_join()`` which allows the target thread to pass back a value to
the calling thread using a second parameter, ``uv_thread_join()`` does not. To
send values use :ref:`inter-thread-communication`.
不像 ``pthread_join()`` 允许目标线程利用第二个参数向调用线程回传一个值， ``uv_thread_join()`` 没有这样的能力。 如果需要传递值的参见:ref:`inter-thread-communication`.

Synchronization Primitives
同步原语
--------------------------

This section is purposely spartan. This book is not about threads, so I only
catalogue any surprises in the libuv APIs here. For the rest you can look at
the pthreads `man pages <pthreads>`_
这节内容是有意简单处理的。这本书并不是关于线程的，因此我这里仅将libuv API中的
不同的地方分类编目。其它的相关内容你可以查阅pthread的手册页(`man pages <pthreads>`_)

Mutexes
互斥
~~~~~~~

The mutex functions are a **direct** map to the pthread equivalents.
与互斥相关的函数直接对应于pthread中的相应函数。

.. rubric:: libuv mutex functions libuv里的互斥函数
.. literalinclude:: ../libuv/include/uv.h
    :lines: 1578-1582

The ``uv_mutex_init()`` and ``uv_mutex_trylock()`` functions will return 0 on
success, -1 on error instead of error codes.

``uv_mutex_init()`` 和 ``uv_mutex_trylock()`` 函数如果执行成功则返回0, 否则返回-1，
而不是错误码。

If `libuv` has been compiled with debugging enabled, ``uv_mutex_destroy()``,
``uv_mutex_lock()`` and ``uv_mutex_unlock()`` will ``abort()`` on error.
Similarly ``uv_mutex_trylock()`` will abort if the error is anything *other
than* ``EAGAIN``.
如果`libuv`是debug版本的话， ``uv_mutex_destroy()``, ``uv_mutex_lock()`` 和
``uv_mutext_unlock()`` 在出错时调用 ``abort()``. 相似的 ``uv_mutex_trylock()`` 
除了 ``EAGAIN`` 错误之外，其它错误也将执行 abort操作。

Recursive mutexes are supported by some platforms, but you should not rely on
them. The BSD mutex implementation will raise an error if a thread which has
locked a mutex attempts to lock it again. For example, a construct like::

    uv_mutex_lock(a_mutex);
    uv_thread_create(thread_id, entry, (void *)a_mutex);
    uv_mutex_lock(a_mutex);
    // more things here

can be used to wait until another thread initializes some stuff and then
unlocks ``a_mutex`` but will lead to your program crashing if in debug mode, or
otherwise behaving wrongly.
可重入互斥体只在一些平台上受到支持，所以你不应该太过于依赖它们。如果某个线程试图重复
锁住一个该线程已经锁定的互斥体，将会导致BSD的互斥体实现产生错误。比如，经常使用的这
类代码:: 

    uv_mutex_lock(a_mutex);
    uv_thread_create(thread_id, entry, (void *)a_mutex);
    uv_mutex_lock(a_mutex);
    // more things here

来让当前线程等待新辟的线程完成某些初始化工作，新辟线程通过完成工作之后释放 ``a_mutex``
来通知主线程。但如果在debug模式下，这会导致你的程序崩溃，如果是release模式下则引发意料
之外的行为。


.. tip::

    Mutexes on linux support attributes for a recursive mutex, but the API is
    not exposed via libuv.
    linux平台支持互斥体，但是libuv的API却没有暴露它们。

Locks
锁
~~~~~

Read-write locks are the other synchronization primitive supported. TODO some
DB read/write example
读写锁是已经被支持的另一种同步原语。(TODO: 比如举个数据库读写锁的例子)

Others
其它方面
~~~~~~

Semaphores and condition variables are not implemented yet. Their are a couple
of patches for condition variable support in libuv [#]_ [#]_, but since the
Windows condition variable system is available only from Windows Vista and
Windows Server 2008 onwards [#]_, its not in libuv yet.
信号量和条件变量尚未实施。但已有两个补丁程序为libuv提供了条件变量的支持[#]_ [#]_,
由于Windows条件变量系统只受Windows Vista和Windows Server 2008及之后的版本才提供支
持，因此目前并未包含到libuv。

libuv work queue
libuv的工作队列
----------------

``uv_queue_work()`` is a convenience function that allows an application to run
a task in a separate thread, and have a callback that is triggered when the
task is done. A seemingly simple function, what makes ``uv_queue_work()``
tempting is that it allows potentially any third-party libraries to be used
with the event-loop paradigm. When you use event loops, it is *imperative to
make sure that no function which runs periodically in the loop thread blocks
when performing I/O or is a serious CPU hog*, because this means the loop slows
down and events are not being dealt with at full capacity.
``uv_queue_work()`` 轻易的允许一个应用在单独的线程中运行一个任务，并在任务执行
结束时触发一个回调函数。看似简单的函数，其魅力在于本质上可以让任意一个第三方库
都能够与事件循环模式一起工作。因为使用事件循环模式时，它强制要求你确保没有一个
函数会在周期性执行的循环体内进行IO操作或者严重吃CPU的操作。因为这就意味着事件
循环过程性能降低，而不能全力以赴地处理事件。

事件循环所有线程做的唯一的事情就是尽可能快地处理事件。而IO或者计算任务都应该
在该线程之外进行。

But a lot of existing code out there features blocking functions (for example
a routine which performs I/O under the hood) to be used with threads if you
want responsiveness (the classic 'one thread per client' server model), and
getting them to play with an event loop library generally involves rolling your
own system of running the task in a separate thread.  libuv just provides
a convenient abstraction for this.
但大量现存代码都带有阻塞性质（因为内部调用了IO例程），要使用这些代码而你又想具备
高响应性，则必须和线程一起使用（经典的多线程服务器模型）。要让它们能够与事件循
环配合工作，就要求你设计的系统在单独的线程运行任务。libuv刚好为此工作提供了一个
合理可行的抽象。

Here is a simple example inspired by `node.js is cancer`_. We are going to
calculate fibonacci numbers, sleeping a bit along the way, but run it in
a separate thread so that the blocking and CPU bound task does not prevent the
event loop from performing other activities.
下例是受到了 `node.js is cancer`_ 的启发。我们准备计算一个斐波那契数列，并会休眠
一会，但是由于它运行在一个独立的线程之中，所以它的阻塞以及耗CPU的任务不会阻止事件
循环处理其它活动。

.. rubric:: queue-work/main.c
.. literalinclude:: ../code/queue-work/main.c
    :linenos:
    :lines: 17-29

The actual task function is simple, nothing to show that it is going to be
run in a separate thread. The ``uv_work_t`` structure is the clue. You can pass
arbitrary data through it using the ``void* data`` field and use it to
communicate to and from the thread. But be sure you are using proper locks if
you are changing things while both threads may be running.
实际的任务函数是简单的，没有要求要在独立的线程中运行。 ``uv_work_t`` 结构体允许你
通过它的 ``void *data`` 字段传递任意数据给工作线程或者从工作线程返回数据以达到通讯
的目的。但是如果你想在线程正在运行过程中去修改数据的话，请务必执行恰当的锁定操作。

The trigger is ``uv_queue_work``:

.. rubric:: queue-work/main.c
.. literalinclude:: ../code/queue-work/main.c
    :linenos:
    :lines: 31-44
    :emphasize-lines: 40

The thread function will be launched in a separate thread, passed the
``uv_work_t`` structure and once the function returns, the *after* function
will be called, again with the same structure.
线程函数会在独立的线程中启动，传入的 ``uv_work_t`` 结构，会被再次传入到函数返回
时回调的 ``after`` 函数。


For writing wrappers to blocking libraries, a common :ref:`pattern <baton>`
is to use a baton to exchange data.
对于那些希望包裹现有的阻塞性质的库的童鞋，可以参考:ref:`pattern <baton>` ，它使用
一个 `接力棒` 进行数据交换。

.. _inter-thread-communication:

Inter-thread communication
线程间通讯
--------------------------

Sometimes you want various threads to actually send each other messages *while*
they are running. For example you might be running some long duration task in
a separate thread (perhaps using ``uv_queue_work``) but want to notify progress
to the main thread. This is a simple example of having a download manager
informing the user of the status of running downloads.
有些时候你希望不同的线程之间能够在 *线程运行过程中* 相互发送消息。比如，你可能在
一个线程(可能使用的 ``uv_queue_work``)里运行一个周期比较长的任务，并希望将任务进
度通知给主线程。简单的例子就是下载管理器要让用户知晓当前正在下载的任务的进度如何。

.. rubric:: progress/main.c
.. literalinclude:: ../code/progress/main.c
    :linenos:
    :lines: 7-8,34-
    :emphasize-lines: 2,11

The async thread communication works *on loops* so although any thread can be
the message sender, only threads with libuv loops can be receivers (or rather
the loop is the receiver). libuv will invoke the callback (``print_progress``)
with the async watcher whenever it receives a message.
`async` 线程通过事件循环进行通讯，因此尽管任何线程都是可以是消息的发送者，但
只有libuv事件循环所在的线程可以是接收者（或者说只有事件循环才能接收消息）。
libuv会在它收到消息时调用async watcher上的回调函数（ ``print_progress`` ）。

.. warning::

    It is important to realize that the message send is *async*, the callback
    may be invoked immediately after ``uv_async_send`` is called in another
    thread, or it may be invoked after some time. libuv may also combine
    multiple calls to ``uv_async_send`` and invoke your callback only once. The
    only guarantee that libuv makes is -- The callback function is called *at
    least once* after the call to ``uv_async_send``. If you have no pending
    calls to ``uv_async_send``, the callback won't be called. If you make two
    or more calls, and libuv hasn't had a chance to run the callback yet, it
    *may* invoke your callback *only once* for the multiple invocations of
    ``uv_async_send``. Your callback will never be called twice for just one
    event.
    请务必知晓消息的发送是异步的，回调函数也许会在 ``uv_async_send`` 之后立即
    在另一个线程中被调用，但也许会滞后一小会。libuv也可能做合并操作——即发送者
    调用多次 ``uv_async_send``，但回调函数只会被调用一次。libuv所做出的承诺是
    回调函数在 ``uv_async_send`` 调用之后至少会被调用一次。如果你没有发出
    ``uv_async_send``，回调函数是不会被调用的。如果你发送的两次或多次的
    ``uv_async_send`` 调用，但libuv没有机会去调用回调函数的话，它就有可能只调
    用一次回调函数。永远不会发生的事情是一个事件触发多次回调函数调用。

.. rubric:: progress/main.c
.. literalinclude:: ../code/progress/main.c
    :linenos:
    :lines: 10-23
    :emphasize-lines: 7-8

In the download function we modify the progress indicator and queue the message
for delivery with ``uv_async_send``. Remember: ``uv_async_send`` is also
non-blocking and will return immediately.
在download函数中，我们利用 ``uv_async_send`` 向队列投递消息，以修改了进度指示器。
请记住： ``uv_async_send`` 也是非阻塞的，它在调用后立即返回。

.. rubric:: progress/main.c
.. literalinclude:: ../code/progress/main.c
    :linenos:
    :lines: 30-33

The callback is a standard libuv pattern, extracting the data from the watcher.
在回调函数中，从watcher中提取数据的过程是libuv的一个标准模式。？

Finally it is important to remember to clean up the watcher.
最后务必记住清理watcher，以释放资源。

.. rubric:: progress/main.c
.. literalinclude:: ../code/progress/main.c
    :linenos:
    :lines: 25-28
    :emphasize-lines: 3

After this example, which showed the abuse of the ``data`` field, bnoordhuis_
pointed out that using the ``data`` field is not thread safe, and
``uv_async_send()`` is actually only meant to wake up another thread. Use
a mutex or rwlock to ensure accesses are performed in the right order.
[??] 实例结束时，要指出的 ``data`` 字段被错误使用了。 bnoordhuis_ 指出使用
``data`` 字段不是线程安全的， ``uv_async_send()`` 只能用于唤醒其它线程。
如果要以正确的顺序访问数以的话，请使用mutex和rwlock。

.. warning::

    mutexes and rwlocks **DO NOT** work inside a signal handler, whereas
    ``uv_async_send`` does.
    mutex和rwlock 不能在singal handler中工作，但 ``uv_async_send`` 是可以的。

One use case where uv_async_send is required is when interoperating with
libraries that require thread affinity for their functionality. For example in
node.js, a v8 engine instance, contexts and its objects are bound to the thread
that the v8 instance was started in. Interacting with v8 data structures from
another thread can lead to undefined results. Now consider some node.js module
which binds a third party library. It may go something like this:
另一个必须使用 ``uv_async_send`` 的场景是，协调某些要求线程绑定（thread affinity）
的库一起工作来完成任务。比如在node.js当中，v8引擎实例、上下文及其对象都是被绑定到
v8实例启动时的线程上的。从别的线程操作v8的数据结构可能导致意料之外的后果。现在来思
考一下某些binding第三方库的node.js模块。可能需要像这样做：

1. In node, the third party library is set up with a JavaScript callback to be
   invoked for more information::

    var lib = require('lib');
    lib.on_progress(function() {
        console.log("Progress");
    });

    lib.do();

    // do other stuff
   在node中，第三方库通过设置JavaScript回调函数来获取更多信息，比如:: 
    var lib = require('lib');
    lib.on_progress(function() {
        console.log("Progress");
    });

    lib.do();

    // do other stuff

2. ``lib.do`` is supposed to be non-blocking but the third party lib is
   blocking, so the binding uses ``uv_queue_work``.
   ``lib.do`` 是被假设为非阻塞的，但第三方库是阻塞的，因此binding时使用 ``uv_queue_work``.

3. The actual work being done in a separate thread wants to invoke the progress
   callback, but cannot directly call into v8 to interact with JavaScript. So
   it uses ``uv_async_send``.
   实际的工作正在独立的线程中进行处理，并意欲调用progress回调函数，但是不能直接调到
   v8里面的JavaScript。因为它使用 ``uv_async_send``.

4. The async callback, invoked in the main loop thread, which is the v8 thread,
   then interacts with v8 to invoke the JavaScript callback.
   async回调在主线程中被调用，这正好是v8所在的线程，在这里可以访问v8虚拟机，并调用到
   JavaScript回调函数。

.. _pthreads: http://man7.org/linux/man-pages/man7/pthreads.7.html

.. rubric:: Footnotes
.. [#] https://github.com/nikhilm/libuv/compare/condvar
.. [#] https://github.com/bnoordhuis/libuv/compare/uv_cond
.. [#] http://msdn.microsoft.com/en-us/library/windows/desktop/ms683469(v=vs.85).aspx
.. _node.js is cancer: https://raw.github.com/teddziuba/teddziuba.github.com/master/_posts/2011-10-01-node-js-is-cancer.html
.. _bnoordhuis: https://github.com/bnoordhuis
