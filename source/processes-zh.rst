Processes
进程
=========

libuv offers considerable child process management, abstracting the platform
differences and allowing communication with the child process using streams or
named pipes.
libuv 提供了的子进程管理机制，抽象了不同平台的差异性，需要子进程之间利用流和
命名管道的进行通讯。

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
``uv_process_t`` 结构体仅仅做为watcher，而所有的选项设置需要使用 ``uv_process_options_t``.
如果仅仅是起动一个进程，你只需要设置 ``file`` 和 ``args`` 两个字段。 ``file`` 指定
需要执行的程序。因为 ``uv_spawn`` 内部使用了 ``execvp`` ，它是不需要提供完整路径的。
另外，根据传统，参数数组 ``args`` 的长度要比实际传入的参数个数多一个，最后那个位置
用来放置一个 ``NULL`` 哨兵。

After the call to ``uv_spawn``, ``uv_process_t.pid`` will contain the process
ID of the child process.
``uv_spawn`` 返回时， ``uv_process_t.pid`` 包含了新辟的子进程的进程ID.

The exit callback will be invoked with the *exit status* and the type of *signal*
which caused the exit.
exit回调函数被调用时会传入退出码(*exit status*)和导致进程退出的信号(*signal*)类型。

.. rubric:: spawn/main.c
.. literalinclude:: ../code/spawn/main.c
    :linenos:
    :lines: 9-12
    :emphasize-lines: 3

It is **required** to close the process watcher after the process exits.
老惯例，进程退出之后，我们也必须关闭 这个进程的 watcher，释放相关资源。

Changing the process parameters
改变进程的参数集
-------------------------------

Before the child process is launched you can control the execution environment
using fields in ``uv_process_options_t``.
在子进程中启动之前，你可以通过配置 ``uv_process_options_t`` 中的字段来控制子进程
的执行环境(execution environment)，比如工作目录、环境变量、搜索路径等。

Change execution directory
改变工作目录
++++++++++++++++++++++++++

Set ``uv_process_options_t.cwd`` to the corresponding directory.
修改 ``uv_process_options_t.cwd`` 来设置工作目录。

Set environment variables
修改环境变量
+++++++++++++++++++++++++

``uv_process_options_t.env`` is an array of strings, each of the form
``VAR=VALUE`` used to set up the environment variables for the process. Set
this to ``NULL`` to inherit the environment from the parent (this) process.
``uv_process_options_t.env`` 是一个字符串数组，每个元素形如 ``VAR=VALUE`` ，
这个字段用来设置进程的环境变量。如果将这个字段设置为 ``NULL`` ，则表示子进程
将沿用父进程（也就是指当前进程）的环境变量。

Option flags
选项标志位
++++++++++++

Setting ``uv_process_options_t.flags`` to a bitwise OR of the following flags,
modifies the child process behaviour:
``uv_process_options_t.flags`` 是一个由比特标记，可以通过以下标记的按位或（bitwise OR）
运算来改变子进程的行为：

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

Detaching processes
分离子进程
-------------------

Passing the flag ``UV_PROCESS_DETACHED`` can be used to launch daemons, or
child processes which are independent of the parent so that the parent exiting
does not affect it.
利用 ``UV_PROCESS_DETACHED`` 标志，可以启动一个守护进程，或者让子进程独立于父进程，
不受父进程的退出影响。如果不分离的话，父进程退出时，子进程会被kill掉的。

.. rubric:: detach/main.c
.. literalinclude:: ../code/detach/main.c
    :linenos:
    :lines: 12-27
    :emphasize-lines: 9,16

Just remember that the watcher is still monitoring the child, so your program
won't exit. Use ``uv_unref()`` if you want to be more *fire-and-forget*.
记得此时wather仍然还在监视子进程，所以你的程序不会退出。如果你实施 *fire-and-forget*
模型的话，要调用 ``uv_unref()`` .

Signals and termination
信号和终结
-----------------------

libuv wraps the standard ``kill(2)`` system call on Unix and implements one
with similar semantics on Windows, with *one caveat*. ``uv_kill`` on Windows
only supports ``SIGTERM``, ``SIGINT`` and ``SIGKILL``, all of which lead to
termination of the process. The signature of ``uv_kill`` is:: 

    uv_err_t uv_kill(int pid, int signum);

For processes started using libuv, you may use ``uv_process_kill`` instead,
which accepts the ``uv_process_t`` watcher as the first argument, rather than
the pid. In this case, **remember to call** ``uv_close`` on the watcher.
在linux平台上，libuv 封装了标准的 ``kill(2)`` 系统调用，而在windows上也实施了
相似的语义，with *on caveat*. ``uv_kill`` 在windows平台上只支持 ``SIGTERM``, ``SIGINT``
和 ``SIGKILL``, 这些信号都是将结束进程。 ``uv_kill`` 的签名为:: 

    uv_err_t uv_kill(int pid, int signum);

如果是由libuv启动的进程，你也可以使用 ``uv_process_kill`` ，这个函数接受的第
一个参数是一个``uv_process_t`` watcher，而不是pid, 这时你别忘记对 watcher 调
用 ``uv_close`` .

Child Process I/O
子进程的IO
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
我们调用test程序：

.. rubric:: proc-streams/test.c
.. literalinclude:: ../code/proc-streams/test.c

The actual program ``proc-streams`` runs this while inheriting only ``stderr``.
The file descriptors of the child process are set using the ``stdio`` field in
``uv_process_options_t``. First set the ``stdio_count`` field to the number of
file descriptors being set. ``uv_process_options_t.stdio`` is an array of
``uv_stdio_container_t``, which is:

``proc-streams`` 程序在运行上面这个测试时，只传承了 ``stderr`` 流。利用 ``uv_process_options_t``
的 ``stdio`` 字段，可以设置子进程的文件描述符。首先要设置 ``stdio_count`` 为要设置的
文件描述符的数目。 ``uv_process_options_t.stdio`` 是一个 ``uv_stdio_container_t`` 数组，
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
因为我们希望传递一个已存在的文件描述符，我们使用 ``UV_INHERIT_FD``, 然后再将
``fd`` 设置为 ``stderr`` .

.. rubric:: proc-streams/main.c
.. literalinclude:: ../code/proc-streams/main.c
    :linenos:
    :lines: 15-17,27-
    :emphasize-lines: 6,10,11,12

If you run ``proc-stream`` you'll see that only the line "This is stderr" will
be displayed. Try marking ``stdout`` as being inherited and see the output.
运行 ``proc-streams`` 控制台将只显示一行 "This is stderr"。若是将 ``stdout``
也继承给子进程，你就可以看到子进程的输出的内容。

It is dead simple to apply this redirection to streams.  By setting ``flags``
to ``UV_INHERIT_STREAM`` and setting ``data.stream`` to the stream in the
parent process, the child process can treat that stream as standard I/O. This
can be used to implement something like CGI_.
使用流重定向功能也非常简单。只要设置 ``flags`` 为 ``UV_INHERIT_STREAM`` 并将
``data.stream`` 设置为父进程的某个流，子进行就可以像处理标准输出输出流一样使用
这些流。比如一个简单的 CGI_ 功能中可能出现这种场景。

.. _CGI: http://en.wikipedia.org/wiki/Common_Gateway_Interface

A sample CGI script/executable is:
下边是一个简单的CGI脚本或执行程序: 

.. rubric:: cgi/tick.c
.. literalinclude:: ../code/cgi/tick.c

The CGI server combines the concepts from this chapter and :doc:`networking` so
that every client is sent ten ticks after which that connection is closed.
CGI 服务器结合了本章的概念和 :doc:`networking` ，服务器会给每一个连入的客户端
发送10个tick，然后关闭连接。

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
发送到了客户端。通过利用进程，我们将缓冲区读写的责任移交给了操作系统，使用起来很方便。
但要记住创建进程本身是一项代价比较高的任务。

.. _pipes:

Pipes
管道
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
   这需要使用 ``uv_pipe_open`` 函数，具体内容在 :ref:`buffers-and-streams` 中讨论。
   你也可以将它用于 TCP/UDP，只不过它们已经拥有了一套方便的函数和结构了。

#. IPC mechanism - ``uv_pipe_t`` can be backed by a `Unix Domain Socket`_ or
   `Windows Named Pipe`_ to allow multiple processes to communicate. This is
   discussed below.
   IPC 机制 － ``uv_pipe_t`` 停靠 `Unix Domain Socket`_ 或者 `Windows Named Pipe`_
   从而允许多进程进行通讯。下面来讨论这些内容。

.. _pipe(7): http://www.kernel.org/doc/man-pages/online/pages/man7/pipe.7.html
.. _Unix Domain Socket: http://www.kernel.org/doc/man-pages/online/pages/man7/unix.7.html
.. _Windows Named Pipe: http://msdn.microsoft.com/en-us/library/windows/desktop/aa365590(v=vs.85).aspx

Parent-child IPC
++++++++++++++++

TODO

Arbitrary process IPC
+++++++++++++++++++++

Since domain sockets [#]_ can have a well known name and a location in the
file-system they can be used for IPC between unrelated processes. The D-BUS
system used by open source desktop environments uses domain sockets for event
notification. Various applications can then react when a contact comes online
or new hardware is detected. The MySQL server also runs a domain socket on
which clients can interact with it.

When using domain sockets, a client-server pattern is usually followed with the
creator/owner of the socket acting as the server. After the initial setup,
messaging is no different from TCP, so we'll re-use the echo server example.

.. rubric:: pipe-echo-server/main.c
.. literalinclude:: ../code/pipe-echo-server/main.c
    :linenos:
    :lines: 56-
    :emphasize-lines: 5,9,13

We name the socket ``echo.sock`` which means it will be created in the local
directory. This socket now behaves no different from TCP sockets as far as
the stream API is concerned. You can test this server using `netcat`_::

    $ nc -U /path/to/echo.sock

A client which wants to connect to a domain socket will use::

    void uv_pipe_connect(uv_connect_t *req, uv_pipe_t *handle, const char *name, uv_connect_cb cb);

where ``name`` will be ``echo.sock`` or similar.

.. _netcat: http://netcat.sf.net

Sending file descriptors over pipes
+++++++++++++++++++++++++++++++++++

TODO

does ev limitation of ev_child only on default loop extend to libuv? yes it
does for now

.. [#] In this section domain sockets stands in for named pipes on Windows as
    well.
