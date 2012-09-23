其它工具 Utilities
=========

This chapter catalogues tools and techniques which are useful for common tasks.
The `libev man page`_ already covers some patterns which can be adopted to
libuv through simple API changes.  It also covers parts of the libuv API that
don't require entire chapters dedicated to them.

这章内容将编目一些对常见任务非常有用的工具及技术。 libev 手册页(`libev man page`_) 已经涵盖的
一些使用模式，在libuv中只需简单地修改相应的API。还包含了一些....

计时器 Timers
------

Timers invoke the callback after a certain time has elapsed since the timer was
started. libuv timers can also be set to invoke at regular intervals instead of
just once.

计时器启动之后，会在一段时间之后调用回调函数。libuv计时器还可以设置为每间隔一段时间（周期性地）
调用一次回调函数。

Simple use is to init a watcher and start it with a ``timeout``, and optional ``repeat``.
Timers can be stopped at any time.

简单的用法是初始化watcher，在启动时传入一个 ``timeout`` 和可选的 ``repeat`` (是否重复).
计时器允许在任务时候停止。

.. code-block:: c

    uv_timer_t timer_req;

    uv_timer_init(loop, &timer_req);
    uv_timer_start(&timer_req, callback, 5000, 2000);

will start a repeating timer, which first starts 5 seconds (the ``timeout``) after the execution
of ``uv_timer_start``, then repeats every 2 seconds (the ``repeat``). Use:

此例的代码将启动一个周期性计时器，首次调用是在执行 ``uv_timer_start`` 之后的5秒(由 ``timeout`` 指定).
然后每间隔2秒(由 ``repeat`` 参数指定)。通过调用: 

.. code-block:: c

    uv_timer_stop(&timer_req);

to stop the timer. This can be used safely from within the callback as well.

来停止计时器。这个函数在 ``callback`` 函数内部调用是安全的。

The repeat interval can be modified at any time with::

    uv_timer_set_repeat(uv_timer_t *timer, int64_t repeat);

which will take effect **when possible**. If this function is called from
a timer callback, it means:

* If the timer was non-repeating, the timer has already been stopped. Use
  ``uv_timer_start`` again.
* If the timer is repeating, the next timeout has already been scheduled, so
  the old repeat interval will be used once more before the timer switches to
  the new interval.

你可以在任何时候对计时器的重复周期进行修改，调用::

    uv_timer_set_repeat(uv_timer_t *timer, int64_t repeat);

在 **恰当的时刻** (when possible) 生效。如果这个函数是在某个计时器回调函数内部调用的，那么:

* 如果计时器是非周期性触发的，则计时器已经停止。需要再次调用 ``uv_timer_start`` 来启动。
* 如果计时器是周期性触发的，则下一次的超时时间已经被安排过了，下一次触发的时间仍然是修改前
  的周期决定的。

The utility function::

    int uv_timer_again(uv_timer_t *)

applies **only to repeating timers** and is equivalent to stopping the timer
and then starting it with both initial ``timeout`` and ``repeat`` set to the
old ``repeat`` value. If the timer hasn't been started it fails (error code
``UV_EINVAL``) and returns -1.

辅助函数::

     int uv_timer_again(uv_timer_t *)

只适合周期性触发的计时器，它的作用相当于停止计时器，然后再将 ``timeout`` 和 ``repeat`` 
都设置为原来的 ``repeat`` 的值，并启动之。如果启动时还没有启动，那么它会以 ``UV_EINVAL`` 错误码
宣告失败并返回 -1.

An actual timer example is in the :ref:`reference count section
<reference-count>`.

一个真实的计时器实例参见 :ref:`reference count section<reference-count>`.

检查和准备watcher Check & Prepare watchers
------------------------

TODO

External I/O with polling
-------------------------

TODO

加载动态链接库 Loading libraries
-----------------

libuv provides a cross platform API to dynamically load `shared libraries`_.
This can be used to implement your own plugin/extension/module system and is
used by node.js to implement ``require()`` support for bindings. The usage is
quite simple as long as your library exports the right symbols. Be careful with
sanity and security checks when loading third party code, otherwise your
program will behave unpredicatably. This example implements a very simple
plugin system which does nothing except print the name of the plugin.

libuv提供了一套跨平台的动态加载共享库 `shared libraries`_ 的API。
可以用于实现你自己的插件、扩展和模块系统。在node.js中，用于支持接口绑定的 ``requre()`` 函数
的实现就用到这些API。只要你的动态库导出了正确的符号表，API使用起来相当简单。
当你加载第三方代码时，要做好安全检查工作，否则你的程序可能表现一些无法预料的行为。
实例中实现了一非常简单的插件系统，它除了会打印插件的名称之外，神马都没有做。

Let us first look at the interface provided to plugin authors.

首先我们看一个提供给插件作者的接口吧。

.. rubric:: plugin/plugin.h
.. literalinclude:: ../code/plugin/plugin.h
    :linenos:

.. rubric:: plugin/plugin.c
.. literalinclude:: ../code/plugin/plugin.c
    :linenos:

You can similarly add more functions that plugin authors can use to do useful
things in your application [#]_. A sample plugin using this API is:

你可以在你的应用之中类似地加入更多的函数，以便于插件作者用来未完成更多有用的事情[#]_。
下面是一个使用了这个API的简单插件: 

.. rubric:: plugin/hello.c
.. literalinclude:: ../code/plugin/hello.c
    :linenos:

Our interface defines that all plugins should have an ``initialize`` function
which will be called by the application. This plugin is compiled as a shared
library and can be loaded by running our application::

我们的接口定义要求所有的插件都必须具备一个 ``initialize`` 函数，以便于应用程序调用。
这个插件要以动态链接库的形式进行编译，才可以被我们的应用加载运行。

    $ ./plugin libhello.dylib
    Loading libhello.dylib
    Registered plugin "Hello World!"

This is done by using ``uv_dlopen`` to first load the shared library
``libhello.dylib``. Then we get access to the ``initialize`` function using
``uv_dlsym`` and invoke it.

上述工作是这样实现的，首先使用 ``uv_dlopen`` 加载动态链接库 ``libhello.dylib``, 
然后利用 ``uv_dlsym`` 函数获取 ``initialize`` 函数的入口，再调用之。

.. rubric:: plugin/main.c
.. literalinclude:: ../code/plugin/main.c
    :linenos:
    :lines: 7-
    :emphasize-lines: 14, 20, 25

``uv_dlopen`` expects a path to the shared library and sets the opaque
``uv_lib_t`` pointer. It returns 0 on success, -1 on error. Use ``uv_dlerror``
to get the error message.

``uv_dlopen`` 期待一个共享库的路径，并会设置一个不透明的 ``uv_lib_t`` 结构体指针.
如果成功则返回0，错误则返回-1。利用 ``uv_dlerror`` 获取错误消息。

``uv_dlsym`` stores a pointer to the symbol in the second argument in the third
argument. ``init_plugin_function`` is a function pointer to the sort of
function we are looking for in the application's plugins.

``uv_dlsym`` 会将指向第二个参数要求的 symbol 的指针保存到第三个参数中。 
``init_plugin_function`` 是一个函数指针，其类型正是我们在应用程序的插件中查找的函数的类型。

.. _shared libraries: http://en.wikipedia.org/wiki/Shared_library#Shared_libraries

Idle Watcher模式 Idle watcher pattern
--------------------

The callbacks of idle watchers are only invoked when the event loop has no
other pending events. In such a situation they are invoked once every iteration
of the loop. The idle callback can be used to perform some very low priority
activity. For example, you could dispatch a summary of the daily application
performance to the developers for analysis during periods of idleness, or use
the application's CPU time to perform SETI calculations :) An idle watcher is
also useful in a GUI application. Say you are using an event loop for a file
download. If the TCP socket is still being established and no other events are
present your event loop will pause (**block**), which means your progress bar
will freeze and the user will think the application crashed. In such a case
queue up and idle watcher to keep the UI operational.

Idle（空闲） Watcher 的回调函数只有在事件循环中没有其它未处理事件时才会被调用。
在这种情形下，它们会在每次循环时被调用一次。空闲回调函数可以用于执行一些优先级
非常低的任务。比如，你可以在空闲回调中发送应用程序每日的性能摘要以供开发人员分
析，或者使用应用程序的CPU进行SETI计算:) Idle Watcher对于GUI也非常有用，比如你
利用事件循环来进行文件下载，如果TCP连接正在建立时，没有其它事件需要处理时，
事件循环会暂停(**block**)， 这就意味着你的进度条停滞不前，用户会以为程序挂了。
这种情况下，利用向队列丢入一个Idle watcher，使得UI是可操作的。

SETI@home 是一项利用全球联网的计算机共同搜寻地外文明的科学实验计划，

“SETI”是英文Search for Extraterrestrial Intelligence（搜寻外星智能）的缩写。

.. rubric:: idle-compute/main.c
.. literalinclude:: ../code/idle-compute/main.c
    :linenos:
    :lines: 5-9, 32-
    :emphasize-lines: 38

Here we initialize the idle watcher and queue it up along with the actual
events we are interested in. ``crunch_away`` will now be called repeatedly
until the user types something and presses Return. Then it will be interrupted
for a brief amount as the loop deals with the input data, after which it will
keep calling the idle callback again.

这里我们初始化一个idle watcher并在我们真正感兴趣的事件之后丢入队列中。 ``crunch_away`` 
会被周期性的调用，直到用户输入一些东西并回车。然后事件循环需要处理用户输入的数据，
它会被中断一小会，然后它又被Idle回调函数周期性的调用。

.. rubric:: idle-compute/main.c
.. literalinclude:: ../code/idle-compute/main.c
    :linenos:
    :lines: 10-19

.. _reference-count:

事件循环的引用计数 Event loop reference count
--------------------------

The event loop only runs as long as there are active watchers. This system
works by having every watcher increase the reference count of the event loop
when it is started and decreasing the reference count when stopped. It is also
possible to manually change the reference count of handles using::

    void uv_ref(uv_handle_t*);
    void uv_unref(uv_handle_t*);

事件循环只有在有活动的watcher时才会进行。每个watcher启动时会增加事件循环的引用计数，
并在停止的时候减小引用计数。也可以利用这两个函数来手动改变句柄的引用计数。

These functions can be used to allow a loop to exit even when a watcher is
active or to use custom objects to keep the loop alive.

这俩函数可以允许还有活动的watcher时让循环退出，或者在一些自定义的对象中阻止循环的退出。


The former can be used with interval timers. You might have a garbage collector
which runs every X seconds, or your network service might send a heartbeat to
others periodically, but you don't want to have to stop them along all clean
exit paths or error scenarios. Or you want the program to exit when all your
other watchers are done. In that case just unref the timer immediately after
creation so that if it is the only watcher running then ``uv_run`` will still
exit.

前一种用法可以和周期性计时器一起使用。你可能需要一个每隔几秒运行一次的垃圾收集器，
或者你的网络服务需要周期地向外发送心跳，但是你并不希望得在所有的安全地或者错误退
出的执行路径中停止它们才能让人你程序退出。或者你希望当你程序中其它的Watcher都
结束的时候，程序能够自动退出。这种些场景中，你只要在创建计时器之后unref它一下，
这样当只有这一个watcher还在运行的话， ``uv_run`` 就会退出。

The later is used in node.js where some libuv methods are being bubbled up to
the JS API. A ``uv_handle_t`` (the superclass of all watchers) is created per
JS object and can be ref/unrefed.

后一种用法在node.js中使用了，出现在那些从libuv中提升的JS API中。 ``uv_handle_t`` 
（是所有watcher的父类）可以被引用或者取消引用。

.. rubric:: ref-timer/main.c
.. literalinclude:: ../code/ref-timer/main.c
    :linenos:
    :lines: 5-8, 17-
    :emphasize-lines: 9

We initialize the garbage collector timer, then immediately ``unref`` it.
Observe how after 9 seconds, when the fake job is done, the program
automatically exits, even though the garbage collector is still running.

例中我们初始化了一个垃圾回收的计时器，然后立即对它执行了 ``unref`` 操作。
可以观察到9秒之后，当fake任务完成时，程序就自动结束了，虽然此刻垃圾收集器
仍在运行中。

.. _baton:

向工作者线程传递数据 Passing data to worker thread
-----------------------------

When using ``uv_queue_work`` you'll usually need to pass complex data through
to the worker thread. The solution is to use a ``struct`` and set
``uv_work_t.data`` to point to it. A slight variation is to have the
``uv_work_t`` itself as the first member of this struct (called a baton [#]_).
This allows cleaning up the work request and all the data in one free call.

在使用 ``uv_queue_work`` 时，你经常需要向工作者线程传递一些复杂的数据。
解决的方法是使用一个 ``struct`` 并将其地址设置到 ``uv_work_t.data``. 也可以做下小
的调整，将 ``uv_work_t`` 结构作为你自己结构体的每一个成员（称这为拉力棒 baton [#]_），
这样就可以在清理过程中使用一个free调用来释放work request和所有的数据了。

.. code-block:: c
    :linenos:
    :emphasize-lines: 2

    struct ftp_baton {
        uv_work_t req;
        char *host;
        int port;
        char *username;
        char *password;
    }

.. code-block:: c
    :linenos:
    :emphasize-lines: 2

    ftp_baton *baton = (ftp_baton*) malloc(sizeof(ftp_baton));
    baton->req.data = (void*) baton;
    baton->host = strdup("my.webhost.com");
    baton->port = 21;
    // ...

    uv_queue_work(loop, &baton->req, ftp_session, ftp_cleanup);

Here we create the baton and queue the task.

这里我们创建了一个baton并将之丢入工作池队列。

Now the task function can extract the data it needs:

此时任务函数可以拿到它想要的数据了:

.. code-block:: c
    :linenos:
    :emphasize-lines: 2, 12

    void ftp_session(uv_work_t *req) {
        ftp_baton *baton = (ftp_baton*) req->data;

        fprintf(stderr, "Connecting to %s\n", baton->host);
    }

    void ftp_cleanup(uv_work_t *req) {
        ftp_baton *baton = (ftp_baton*) req->data;

        free(baton->host);
        // ...
        free(baton);
    }

We then free the baton which also frees the watcher.

最后我们释放baton的时候，也同时将watcher一起释放了。

控制台 TTY
---

TODO

.. [#] mfp is My Fancy Plugin
.. [#] I was first introduced to the term baton in this context, in Konstantin
       Käfer's excellent slides on writing node.js bindings --
       http://kkaefer.github.com/node-cpp-modules/#baton

.. _libev man page: http://pod.tst.eu/http://cvs.schmorp.de/libev/ev.pod#COMMON_OR_USEFUL_IDIOMS_OR_BOTH
