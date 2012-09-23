Processes
进程
=========

libuv offers considerable child process management, abstracting the platform
differences and allowing communication with the child process using streams or
named pipes.
libuv 提供了的子进程管理机制，抽象了不同平台的差异性，子进程之间可以使用流和
命名管道的来进行通讯。

Spawning child processes
Spawn子进程
------------------------

The simplest case is when you simply want to launch a process and know when it
exits. This is achieved using ``uv_spawn``.
举个最简单的例子——发起一个进程，然后在进程退出时获得通知。这个任务使用 ``uv_spawn`` 
就可以完成。

.. rubric:: spawn/main.c
.. literalinclude:: ../code/spawn/main.c
    :linenos:
    :lines: 5-7,13-
    :emphasize-lines: 11,13-17

The ``uv_process_t`` struct only acts as the watcher, all options are set via
``uv_process_options_t``. To simply launch a process, you need to set only the
``file`` and ``args`` fields. ``file`` is the program to execute. Since
``uv_spawn`` uses ``execvp`` internally, there is no need to supply the full
path. Finally as per underlying conventions, the arguments array *has* to be
one larger than the number of arguments, with the last element being ``NULL``.
``uv_process_t`` 结构体仅仅做为watcher，而所有的选项设置需要借助
``uv_process_options_t`` 来完成. 如果仅仅只是起动一个进程，你只要设置 ``file`` 和 
``args`` 两个字段就足够了。 ``file`` 指定需要执行的程序。因为 ``uv_spawn`` 内部使
用了 ``execvp`` ，它是不需要提供完整路径的。另外，根据传统，参数数组 ``args`` 的
长度要比实际传入的参数个数多一个，最后那个位置用来放置一个 ``NULL`` 哨兵。

After the call to ``uv_spawn``, ``uv_process_t.pid`` will contain the process
ID of the child process.
``uv_spawn`` 返回时， ``uv_process_t.pid`` 包含了新辟的子进程的进程ID.

The exit callback will be invoked with the *exit status* and the type of *signal*
which caused the exit.
exit回调函数在调用时会传入退出码(*exit status*)和导致进程退出的信号(*signal*)类型。

.. rubric:: spawn/main.c
.. literalinclude:: ../code/spawn/main.c
    :linenos:
    :lines: 9-12
    :emphasize-lines: 3

It is **required** to close the process watcher after the process exits.
老惯例，进程退出之后，我们也必须关闭这个进程的 watcher，释放相关资源。

Changing the process parameters
改变进程的参数集
-------------------------------

Before the child process is launched you can control the execution environment
using fields in ``uv_process_options_t``.
在子进程启动之前，你可以通过设置 ``uv_process_options_t`` 中的字段来控制子进程
的执行环境(execution environment)，比如工作目录、环境变量、搜索路径等。


改变工作目录 Change execution directory
++++++++++++++++++++++++++

Set ``uv_process_options_t.cwd`` to the corresponding directory.
修改 ``uv_process_options_t.cwd`` 来设置工作目录。


修改环境变量 Set environment variables
+++++++++++++++++++++++++

``uv_process_options_t.env`` is an array of strings, each of the form
``VAR=VALUE`` used to set up the environment variables for the process. Set
this to ``NULL`` to inherit the environment from the parent (this) process.
``uv_process_options_t.env`` 是一个字符串数组，每个元素形如 ``VAR=VALUE`` ，
这个字段用来设置进程的环境变量。如果将这个字段设置为 ``NULL`` ，则表示子进程
将沿用父进程（也就是指当前进程）的环境变量。


选项标志位 Option flags
++++++++++++

Setting ``uv_process_options_t.flags`` to a bitwise OR of the following flags,
modifies the child process behaviour:
``uv_process_options_t.flags`` 是一个由比特标记，可以通过以下标记的按位或
（bitwise OR）运算来改变子进程的行为：

* ``UV_PROCESS_SETUID`` - sets the child's execution user ID to ``uv_process_options_t.uid``.
                          将子进程执行时的用户ID设置为 ``uv_process_options_t.uid``.
* ``UV_PROCESS_SETGID`` - sets the child's execution group ID to ``uv_process_options_t.gid``.
                          将子进程执行时的组ID设置为 ``uv_process_options_t.gid`` 。

Changing the UID/GID is only supported on Unix, ``uv_spawn`` will fail on
Windows with ``UV_ENOTSUP``.
只有Unix系统才支持改变子进程的 UID/GID，在Windows平台上 ``uv_spawn`` 会以
``UV_ENOTSUP`` 错误宣告失败。

* ``UV_PROCESS_WINDOWS_VERBATIM_ARGUMENTS`` - No quoting or escaping of
  ``uv_process_options_t.args`` is done on Windows. Ignored on Unix.
  表明 ``uv_process_options_t.args`` 忽略引号和转义字符。该标记只在Windows有用，Unix上
  会忽略该标记。
* ``UV_PROCESS_DETACHED`` - Starts the child process in a new session, which
  will keep running after the parent process exits. See example below.
  在新的会话中启动子进行，当父进程退出后子进程仍继续运行。后续实例中展示了这个用法。

分离子进程 Detaching processes
-------------------

Passing the flag ``UV_PROCESS_DETACHED`` can be used to launch daemons, or
child processes which are independent of the parent so that the parent exiting
does not affect it.
利用 ``UV_PROCESS_DETACHED`` 标志，可以启动一个守护进程，或者让子进程独立于父进程，
不受父进程的退出影响。如果不分离的话，子进程是不活不过父进程的。

.. rubric:: detach/main.c
.. literalinclude:: ../code/detach/main.c
    :linenos:
    :lines: 12-27
    :emphasize-lines: 9,16

Just remember that the watcher is still monitoring the child, so your program
won't exit. Use ``uv_unref()`` if you want to be more *fire-and-forget*.
记得此时wather仍然还在监视子进程，所以你的程序不会退出。如果你实施 *fire-and-forget*
模型的话，要调用 ``uv_unref()`` .


信号和终结 Signals and termination
-----------------------

libuv wraps the standard ``kill(2)`` system call on Unix and implements one
with similar semantics on Windows, with *one caveat*. ``uv_kill`` on Windows
only supports ``SIGTERM``, ``SIGINT`` and ``SIGKILL``, all of which lead to
termination of the process. The signature of ``uv_kill`` is:: 

    uv_err_t uv_kill(int pid, int signum);

For processes started using libuv, you may use ``uv_process_kill`` instead,
which accepts the ``uv_process_t`` watcher as the first argument, rather than
the pid. In this case, **remember to call** ``uv_close`` on the watcher.
在unix平台上，libuv 封装了标准的 ``kill(2)`` 系统调用，而在windows上也实施了
相似的语义，with *on caveat*. ``uv_kill`` 在windows平台上只支持 ``SIGTERM``, 
``SIGINT`` 和 ``SIGKILL``, 这些信号都是将结束进程。 ``uv_kill`` 的签名为:: 

    uv_err_t uv_kill(int pid, int signum);

如果是由libuv启动的进程，你也可以使用 ``uv_process_kill`` ，这个函数接受的第
一个参数是一个``uv_process_t`` watcher，而不是pid, 这时你别忘记对 watcher 调
用 ``uv_close`` .


子进程的IO Child Process I/O
-----------------

A normal, newly spawned process has its own set of file descriptors, with 0,
1 and 2 being ``stdin``, ``stdout`` and ``stderr`` respectively. Sometimes you
may want to share file descriptors with the child. For example, perhaps your
applications launches a sub-command and you want any errors to go in the log
file, but ignore ``stdout``. For this you'd like to have ``stderr`` of the
child to be displayed. In this case, libuv supports *inheriting* file
descriptors. In this sample, we invoke the test program, which is:
每一个正常的、新辟出的进程都拥有三个标准的文件描述符，分别是0, 1, 2对应的，
标准输入(``stdin``)、标准输出(``stdout``)和标准错误输出(``stderr``)。有时你可能
希望和子进程共用这些文件描述符。比如，你的应用启动了一个子命令，并且希望将所有
的错误都输出到日志文件，而忽略标准输出 ``stdout`` 。这时，你可能希望子进程有一
个可以用于显示的 ``stderr``. 这种的场景下，libuv支持 *继承* 文件描述符。在下例中
我们通过了进程的方式调用test程序：

.. rubric:: proc-streams/test.c
.. literalinclude:: ../code/proc-streams/test.c

The actual program ``proc-streams`` runs this while inheriting only ``stderr``.
The file descriptors of the child process are set using the ``stdio`` field in
``uv_process_options_t``. First set the ``stdio_count`` field to the number of
file descriptors being set. ``uv_process_options_t.stdio`` is an array of
``uv_stdio_container_t``, which is:

``proc-streams`` 程序在运行上面这个测试时，只传承了 ``stderr`` 流。这是利用
``uv_process_options_t`` 的 ``stdio`` 字段，设置子进程的文件描述符来实现的。首先
要设置 ``stdio_count`` 为要设置的文件描述符的数目。 
``uv_process_options_t.stdio`` 是一个 ``uv_stdio_container_t`` 数组，
``uv_stdio_container_t`` 的定义如下：

.. literalinclude:: ../libuv/include/uv.h
    :lines: 1188-1195

where flags can have several values. Use ``UV_IGNORE`` if it isn't going to be
used. If the first three ``stdio`` fields are marked as ``UV_IGNORE`` they'll
redirect to ``/dev/null``.
其中的 ``flags`` 可以设置一些值。如果不希望被使用的话，设置为 ``UV_IGNORE`` .
如果前三个 ``stdio`` 被标记为 ``UV_IGNORE``, 它们则被重定向到 ``/dev/null`` 。

Since we want to pass on an existing descriptor, we'll use ``UV_INHERIT_FD``.
Then we set the ``fd`` to ``stderr``.
因为例中我们希望传递一个已存在的文件描述符，所以我们使用 ``UV_INHERIT_FD``, 然后
再将 ``fd`` 设置为 ``stderr`` .

.. rubric:: proc-streams/main.c
.. literalinclude:: ../code/proc-streams/main.c
    :linenos:
    :lines: 15-17,27-
    :emphasize-lines: 6,10,11,12

If you run ``proc-stream`` you'll see that only the line "This is stderr" will
be displayed. Try marking ``stdout`` as being inherited and see the output.
运行 ``proc-streams`` 控制台将只显示一行 "This is stderr"。若是将 ``stdout``
也继承给子进程的话，你就可以看到子进程的标准输出的内容。

It is dead simple to apply this redirection to streams.  By setting ``flags``
to ``UV_INHERIT_STREAM`` and setting ``data.stream`` to the stream in the
parent process, the child process can treat that stream as standard I/O. This
can be used to implement something like CGI_.
使用流重定向功能也非常简单。只要设置 ``flags`` 为 ``UV_INHERIT_STREAM`` 并将
``data.stream`` 设置为父进程的某个流，子进行就可以像处理标准输出输出流一样使用
这些流。比如一个简单的 CGI_ 功能中可能出现这种场景。

.. _CGI: http://en.wikipedia.org/wiki/Common_Gateway_Interface

A sample CGI script/executable is:
比如下面这个简单的CGI脚本（是个执行程序）: 

.. rubric:: cgi/tick.c
.. literalinclude:: ../code/cgi/tick.c

The CGI server combines the concepts from this chapter and :doc:`networking` so
that every client is sent ten ticks after which that connection is closed.
我们将结合了本章的概念和 :doc:`networking` 来实现一个CGI服务器，服务器会给每一个
连入的客户端发送10个tick，然后关闭连接。

.. rubric:: cgi/main.c
.. literalinclude:: ../code/cgi/main.c
    :linenos:
    :lines: 47,53-62
    :emphasize-lines: 5

Here we simply accept the TCP connection and pass on the socket (*stream*) to
``invoke_cgi_script``.
这段代码简单的接受TCP连接，并将socket(*stream*) 传给 ``invoke_cgi_script``.

.. rubric:: cgi/main.c
.. literalinclude:: ../code/cgi/main.c
    :linenos:
    :lines: 16, 25-45
    :emphasize-lines: 8-9,17-18

The ``stdout`` of the CGI script is set to the socket so that whatever our tick
script prints gets sent to the client. By using processes, we can offload the
read/write buffering to the operating system, so in terms of convenience this
is great. Just be warned that creating processes is a costly task.
将CGI脚本的标准输出 ``stdout`` 定向到 socket，使得脚本打印出的所有tick都被
发送到客户端。通过利用进程，我们将缓冲区读写的责任移交给了操作系统，使用起来很方
便。但要记住创建进程本身是一项代价比较高的任务。

.. _pipes:


管道 Pipes
-----

libuv's ``uv_pipe_t`` structure is slightly confusing to Unix programmers,
because it immediately conjures up ``|`` and `pipe(7)`_. But ``uv_pipe_t`` is
not related to anonymous pipes, rather it has two uses:
libuv 的 ``uv_pipe_t`` 结构令Unix程序员感到困惑。因为它立刻让人想起 ``|`` 和 `pipe(7)`_.
但是 ``uv_pipe_t`` 和匿名管道没什么关系。它有两种用法：

#. Stream API - It acts as the concrete implementation of the ``uv_stream_t``
   API for providing a FIFO, streaming interface to local file I/O. This is
   performed using ``uv_pipe_open`` as covered in :ref:`buffers-and-streams`.
   You could also use it for TCP/UDP, but there are already convenience functions
   and structures for them.
   流API － 它实现了 ``uv_stream_t`` API，提供了一套针对本地文件的IO流接口。
   这需要使用 ``uv_pipe_open`` 函数，具体内容在 :ref:`buffers-and-streams` 中讨论了。
   你也可以将它用于 TCP/UDP，只不过它们已经拥有了一套方便的函数和结构了。

#. IPC mechanism - ``uv_pipe_t`` can be backed by a `Unix Domain Socket`_ or
   `Windows Named Pipe`_ to allow multiple processes to communicate. This is
   discussed below.
   IPC 机制 － ``uv_pipe_t`` 依靠 `Unix Domain Socket`_ 或者 `Windows Named Pipe`_
   从而允许多进程进行通讯。下面来讨论这些内容。

.. _pipe(7): http://www.kernel.org/doc/man-pages/online/pages/man7/pipe.7.html
.. _Unix Domain Socket: http://www.kernel.org/doc/man-pages/online/pages/man7/unix.7.html
.. _Windows Named Pipe: http://msdn.microsoft.com/en-us/library/windows/desktop/aa365590(v=vs.85).aspx

父子进程之间的 IPC Parent-child IPC
++++++++++++++++

A parent and child can have one or two way communication over a pipe created by
settings ``uv_stdio_container_t.flags`` to a bit-wise combination of
``UV_CREATE_PIPE`` and ``UV_READABLE_PIPE`` or ``UV_WRITABLE_PIPE``. The
read/write flag is from the perspective of the child process.

父子进程之间可以利用管道来建立单向或者双向通讯。在创建子进程时将
``uv_stdio_container_t.flags`` 设置为 ``UV_CREATE_PIPE`` 和 ``UV_READABLE_PIPE`` 
或者 ``UV_WRITABLE_PIPE`` 的按位组合。所谓的读写是以子进程的角度而言的。

任意进程间的IPC Arbitrary process IPC
+++++++++++++++++++++

Since domain sockets [#]_ can have a well known name and a location in the
file-system they can be used for IPC between unrelated processes. The D-BUS
system used by open source desktop environments uses domain sockets for event
notification. Various applications can then react when a contact comes online
or new hardware is detected. The MySQL server also runs a domain socket on
which clients can interact with it.

因为domain sockets [#]_ 是可以有命名的，并且在文件系统中有一个location，利用
这些可以进行无关的进程之间的IPC. 开源桌面环境中使用的D-BUS系统就利用了
domain sockets进行事件通知。借此许多应用程序就可以对联系人上线或是侦测到新硬件
时做出响应。MySQL服务器也运行了一个domain socket，客户可以利用它与服务器交互。

When using domain sockets, a client-server pattern is usually followed with the
creator/owner of the socket acting as the server. After the initial setup,
messaging is no different from TCP, so we'll re-use the echo server example.
在使用domain socket的时候，一般会采取客户－服务器模式，socket的创建者（或所有者）
作为服务器角色。初始化设置完成之后，消息传递过程就与TCP没什么两样了，因此我们再
次使用echo 服务器的例子来完成阐述。

.. rubric:: pipe-echo-server/main.c
.. literalinclude:: ../code/pipe-echo-server/main.c
    :linenos:
    :lines: 56-
    :emphasize-lines: 5,9,13

We name the socket ``echo.sock`` which means it will be created in the local
directory. This socket now behaves no different from TCP sockets as far as
the stream API is concerned. You can test this server using `netcat`_::

    $ nc -U /path/to/echo.sock

我们这里将socket命名为 ``echo.sock`` ， 这样它将会在当前目录上创建。对于流API，
这个socket和TCP的socket是没有两样的。借助 `netcat`_ 可以测试这个服务器::
    $ nc -U /path/to/echo.sock

A client which wants to connect to a domain socket will use::

    void uv_pipe_connect(uv_connect_t *req, uv_pipe_t *handle, const char *name, uv_connect_cb cb);

where ``name`` will be ``echo.sock`` or similar.

客户端如果要连接domain socket，可以使用::
    void uv_pipe_connect(uv_connect_t *req, uv_pipe_t *handle, const char *name, uv_connect_cb cb);

这里将 ``name`` 设置为 ``echo.sock`` 就可以了。

.. _netcat: http://netcat.sf.net

通过管道发送文件描述符 Sending file descriptors over pipes
+++++++++++++++++++++++++++++++++++

The cool thing about domain sockets is that file descriptors can be exchanged
between processes by sending them over a domain socket. This allows processes
to hand off their I/O to other processes. Applications include load-balancing
servers, worker processes and other ways to make optimum use of CPU.

借助domain socket可以完成一件很酷的事情，那就是通过domain socket可以在进程之间
交换文件描述符。让进程把它们的IO传递给别的进程。应用场景包括使用负载均衡服务器、
工作者进程或其它的方式来优化CPU的使用。

.. warning::

    On Windows, only file descriptors representing TCP sockets can be passed
    around.
    在Windows上，只有当文件描述符是表示TCP socket时才可以这样传递。

To demonstrate, we will look at a echo server implementation that hands of
clients to worker processes in a round-robin fashion. This program is a bit
involved, and while only snippets are included in the book, it is recommended
to read the full code to really understand it.
作为演示，我们再来实现一个echo server，这次服务器将以round-robin(负载均衡)的
方式调度工作者进程来处理客户请求。程序有点难懂，本书中仅包含一些片段，所以推荐
阅读全部代码来理解它。

The worker process is quite simple, since the file-descriptor is handed over to
it by the master.

由于文件描述符是由master传递过来的，所以工作作进程的实现比较简单。

.. rubric:: multi-echo-server/worker.c
.. literalinclude:: ../code/multi-echo-server/worker.c
    :linenos:
    :lines: 7-9,53-
    :emphasize-lines: 7-9

``queue`` is the pipe connected to the master process on the other end, along
which new file descriptors get sent. We use the ``read2`` function to express
interest in file descriptors. It is important to set the ``ipc`` argument of
``uv_pipe_init`` to 1 to indicate this pipe will be used for inter-process
communication! Since the master will write the file handle to the standard
input of the worker, we connect the pipe to ``stdin`` using ``uv_pipe_open``.

这里 ``queue`` 是一个管道，用于连接另一端的master，新的文件描述符通过此管道发送
过来。我们利用 ``read2`` 函数表明对文件描述符感兴趣。需要强调的是，在调用 ``uv_pipe_init`` 时将参数 ``ipc`` 设置为1，以指明这个管道将用于进行进程间通讯。因为
master会将文件句柄写入到worker的标准输入，所以我们使用 ``uv_pipe_open`` 新将建立
的管道连接到标准输入流。

.. rubric:: multi-echo-server/worker.c
.. literalinclude:: ../code/multi-echo-server/worker.c
    :linenos:
    :lines: 36-52
    :emphasize-lines: 9

Although ``accept`` seems odd in this code, it actually makes sense. What
``accept`` traditionally does is get a file descriptor (the client) from
another file descriptor (The listening socket). Which is exactly what we do
here. Fetch the file descriptor (``client``) from ``queue``. From this point
the worker does standard echo server stuff.
虽然 ``accept`` 在这段代码里看起来别扭，但确实合理。传统的 ``accept`` 做的事情
就是从一个文件描述符（侦听socket）获取另一个文件描述符（客户的socket）。这里也
是这么干的。从 ``queue`` 取得一个文件描述符(``client``)，从这时开始，worker就
做着标准echo服务器的活了。

Turning now to the master, let's take a look at how the workers are launched to
allow load balancing.
接来再介绍master，让我们来看看它是如何启动worker并进行负载均衡的。

.. rubric:: multi-echo-server/main.c
.. literalinclude:: ../code/multi-echo-server/main.c
    :linenos:
    :lines: 6-13

The ``child_worker`` structure wraps the process, and the pipe between the
master and the individual process.

``child_worker`` 结构将进程和master和进程之间的管道打包以了一起。

.. rubric:: multi-echo-server/main.c
.. literalinclude:: ../code/multi-echo-server/main.c
    :linenos:
    :lines: 49,61-93
    :emphasize-lines: 15,18-19

In setting up the workers, we use the nifty libuv function ``uv_cpu_info`` to
get the number of CPUs so we can launch an equal number of workers. Again it is
important to initialize the pipe acting as the IPC channel with the third
argument as 1. We then indicate that the child process' ``stdin`` is to be
a readable pipe (from the point of view of the child). Everything is
straightforward till here. The workers are launched and waiting for file
descriptors to be written to their pipes.

在初始化工作者进程组的时候，我们利用了另一个很棒的libuv函数 —— ``uv_cpu_info`` 来
获取当前CPU的数目，然后发起等量的工作者进程。再强调一次，在创建作为IPC通道的管道
时，第三个参数需要设置为1. 然后将子进程的 ``stdin`` 标记为可读的管道（从子进程的
角度看）。到这时所有事情都很简单。工作者进程组被启动，并等待文件描述符被写入到它
们的管道中。

It is in ``on_new_connection`` (the TCP infrastructure is initialized in
``main()``), that we accept the client socket and pass it along to the next
worker in the round-robin.

在 ``on_new_connection`` 函数中，我们接受客户socket，并将之传递给一个工作者进程。具体传给哪一个工作者，就是采用了round-robin的方式，轮流的每个工作者处理一个客户
socket。

.. rubric:: multi-echo-server/main.c
.. literalinclude:: ../code/multi-echo-server/main.c
    :linenos:
    :lines: 29-47
    :emphasize-lines: 9,12-13

Again, the ``uv_write2`` call handles all the abstraction and it is simply
a matter of passing in the file descriptor as the right argument. With this our
multi-process echo server is operational.

这里的 ``uv_write2`` 的调用简化了文件描述符传递过程。利用这个我们的多进程echo
服务器就可以运行了。

TODO what do the write2/read2 functions do with the buffers?

----

.. [#] In this section domain sockets stands in for named pipes on Windows as
    well.
