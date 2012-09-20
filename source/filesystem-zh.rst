Filesystem
文件系统
==========

Simple filesystem read/write is achieved using the ``uv_fs_*`` functions and the
``uv_fs_t`` struct.
通过uv进行简单的文件系统读写操作需要利用 ``uv_fs_*`` 族函数和 ``uv_fs_t`` 结构。

.. note::

    The libuv filesystem operations are different from :doc:`socket operations
    <networking>`. Socket operations use the non-blocking operations provided
    by the operating system. Filesystem operations use blocking functions
    internally, but invoke these functions in a thread pool and notify watchers
    registered with the event loop when application interaction is required.
    libuv的文件系统操作并不同于Socket操作（:doc:`socket operations <networking>`). 
    因为Socket操作使用的是由底层操作系统提供的非阻塞操作机制。而文件系统操作实现内
    部调用的操作系统API仍然还是阻塞函数，只是这些函数是在线程池中调用的，并在应用交
    互需要时，事件循环会通知已经注册的Wathers.

.. note::

    The fs operations are actually a part of `libeio
    <http://software.schmorp.de/pkg/libeio.html>`_ on Unix systems. ``libeio`` is
    a separate library written by the author of ``libev``.
    在Unix系统上，文件操作实际上包含在 `libeio<http://software.schmorp.de/pkg/libeio.html>`_ 中。
    ``libeio`` 是由 ``libev`` 的作者编写的独立的库。

All filesystem functions have two forms - *synchronous* and *asynchronous*.
所有文件系统函数都有两种调用方式—— *同步的* 和 *异步* 的。

The *synchronous* forms automatically get called (and **block**) if no callback
is specified. The return value of functions is the equivalent Unix return value
(usually 0 on success, -1 on error).
如果未指定回调函数的话，调用会自动的使用*同步* 方式并 *阻塞* 调用方。此时函数的返回值
与相应的Unix函数相同，通常0表示成功，-1表示失败。

The *asynchronous* form is called when a callback is passed and the return
value is 0.
当指定了回调函数，调用将以 *异步* 方式进行，返回值为0.

Reading/Writing files
文件读写
---------------------

A file descriptor is obtained using

.. code-block:: c

    int uv_fs_open(uv_loop_t* loop, uv_fs_t* req, const char* path, int flags, int mode, uv_fs_cb cb)

``flags`` and ``mode`` are standard
`Unix flags <http://man7.org/linux/man-pages/man2/open.2.html>`_.
libuv takes care of converting to the appropriate Windows flags.
文件描述符通过

.. code-block:: c

    int uv_fs_open(uv_loop_t* loop, uv_fs_t* req, const char* path, int flags, int mode, uv_fs_cb cb)

来获得。其中的 ``flags`` 和 ``mode`` 参数取值参考标准的 
`Unix flags <http://man7.org/linux/man-pages/man2/open.2.html>`_.
liuv来处理如何将这些参数转换成正确的Windows flags.

File descriptors are closed using

.. code-block:: c

    int uv_fs_close(uv_loop_t* loop, uv_fs_t* req, uv_file file,
文件描述符通过

.. code-block:: c

    int uv_fs_close(uv_loop_t* loop, uv_fs_t* req, uv_file file,
来关闭。

Filesystem operation callbacks have the signature:

.. code-block:: c

    void callback(uv_fs_t* req);

文件系统操作的回调函数签名是：

.. code-block:: c

    void callback(uv_fs_t* req);

Let's see a simple implementation of ``cat``. We start with registering
a callback for when the file is opened:
有了libuv提供的这些函数，现在让我们来实现一个简单的 ``cat``吧. 
首先，我们为文件已打开事件注册一个回调函数，实现如下。

.. rubric:: uvcat/main.c - opening a file 打开文件句柄
.. literalinclude:: ../code/uvcat/main.c
    :linenos:
    :lines: 39-48
    :emphasize-lines: 2

The ``result`` field of a ``uv_fs_t`` is the file descriptor in case of the
``uv_fs_open`` callback. If the file is successfully opened, we start reading it.
``uv_fs_open`` 回调时传入的 ``uv_fs_t`` 参数中的 ``result`` 字段是一个文件描述符。
如果文件已成功打开，我们可以在该描述符上进行读取操作了。

.. warning::

    The ``uv_fs_req_cleanup()`` function must be called to free internal memory
    allocations in libuv.
    为了释放libuv内部实现时分配的内存资源，我们必须调用``uv_fs_req_cleanup()`` 。

.. rubric:: uvcat/main.c - read callback 文件读回调函数
.. literalinclude:: ../code/uvcat/main.c
    :linenos:
    :lines: 24-37
    :emphasize-lines: 6,9,12

In the case of a read call, you should pass an *initialized* buffer which will
be filled with data before the read callback is triggered.
在进行文件读取操作时，你需要输入一块已初始化的缓冲区，在读回调操作被触发之前，libuv会
将读取到的数据保存到这块缓冲区里面。

In the read callback the ``result`` field is 0 for EOF, -1 for error and the
number of bytes read on success.
在读回调函数被调用时，传入的 ``result`` 字段如果是0则表示读取了文件结束符(EOF)(文件已完全读入)，
-1表示有错误发生了，其它的数值表示已读入的字节数。

Here you see a common pattern when writing asynchronous programs. The
``uv_fs_close()`` call is performed synchronously. *Usually tasks which are
one-off, or are done as part of the startup or shutdown stage are performed
synchronously, since we are interested in fast I/O when the program is going
about its primary task and dealing with multiple I/O sources*. For solo tasks
the performance difference usually is negligible and may lead to simpler code.
接下来你将看到异步读取操作编程的常见模式。 ``uv_fs_close()`` 以同步的方式进行调用。
通常，对于只会执行一次的任务，或者作为启动和结束时期执行的操作以同步的方式进行，
因为当我们的程序执行主要任务或者处理多IO事件源时，我们关注的快速的IO。对于单个任务
而言，性能上的差异是可以忽略的，这样我就可以写出更加简洁的代码来。

We can generalize the pattern that the actual return value of the original
system call is stored in ``uv_fs_t.result``.
我们可以总结出，底层操作系统调用的真实返回结果都是存储在 ``uv_fs_t.result`` 中的。

Filesystem writing is similarly simple using ``uv_fs_write()``.  *Your callback
will be triggered after the write is complete*.  In our case the callback
simply drives the next read. Thus read and write proceed in lockstep via
callbacks.
类似的，文件系统的写操作简单的使用 ``uv_fs_write()``. *你的回调函数在写操作完成之后被
触发。* 在我们的case里，回调函数就是简单的发起下一次读操作。这样，读和写的过程由许多次
回调函数一环套一环地进行着。


.. rubric:: uvcat/main.c - write callback 读回调
.. literalinclude:: ../code/uvcat/main.c
    :linenos:
    :lines: 14-22
    :emphasize-lines: 7

.. note::

    The error usually stored in `errno
    <http://man7.org/linux/man-pages/man3/errno.3.html>`_ can be accessed from
    ``uv_fs_t.errorno``, but converted to a standard ``UV_*`` error code.  There is
    currently no way to directly extract a string error message from the
    ``errorno`` field.
    错误通常是保存在 `errno
    <http://man7.org/linux/man-pages/man3/errno.3.html>`_, 可以使用 ``uv_fs_t.errorno``来
    访问，但是已经被转换成标准的 ``UV_*`` 错误代码了。目前没有办法直接从 ``errorno`` 字段
    获取字符串格式的错误消息。

.. warning::

    Due to the way filesystems and disk drives are configured for performance,
    a write that 'succeeds' may not be committed to disk yet. See
    ``uv_fs_fsync`` for stronger guarantees.
    由于文件系统和硬盘驱动器鉴于性能考虑而进行的配置方式，一个成功的写操作并不一定保证写入到
    了磁盘（还有可能在操作系统的缓冲区内）。如果需要更强的保证，可以参见 ``uv_fs_fsync`` 的
    相关内容。

We set the dominos rolling in ``main()``:
我们在 ``main()`` 函数中开始推倒多米诺骨牌：
.. rubric:: uvcat/main.c
.. literalinclude:: ../code/uvcat/main.c
    :linenos:
    :lines: 50-54
    :emphasize-lines: 2

Filesystem operations
文件系统操作
---------------------

All the standard filesystem operations like ``unlink``, ``rmdir``, ``stat`` are
supported asynchronously and have intuitive argument order. They follow the
same patterns as the read/write/open calls, returning the result in the
``uv_fs_t.result`` field. The full list:
所有的标准文件系统操作，比如 ``unlink``, ``rmdir``, ``stat`` 都是支持异步的调用方式，并
具有相当自然的参数顺序。它们和文件read／write／open函数的调用方法一样，返回的结果包含也
都保存在 ``uv_fs_t.result`` 字段中。下列全部的文件操作：

.. rubric:: Filesystem operations 文件系统操作
.. literalinclude:: ../libuv/include/uv.h
    :lines: 1390-1466

.. _buffers-and-streams:

Buffers and Streams
缓冲区和流
-------------------

The basic I/O tool in libuv is the stream (``uv_stream_t``). TCP sockets, UDP
sockets, and pipes for file I/O and IPC are all treated as stream subclasses.
libuv提供了一个基本的IO工具——流(Stream, ``uv_stream_t``). 不管是TCP和UDP套接字，还是
用于文件和IPC的管道，都被当作流的子类来处理。

Streams are initialized using custom functions for each subclass, then operated
upon using

.. code-block:: c

    int uv_read_start(uv_stream_t*, uv_alloc_cb alloc_cb, uv_read_cb read_cb);
    int uv_read_stop(uv_stream_t*);
    int uv_write(uv_write_t* req, uv_stream_t* handle,
                uv_buf_t bufs[], int bufcnt, uv_write_cb cb);
对于不同的子类，流使用特定的函数进行初始化，然后可以对其执行如下操作：

.. code-block:: c

    int uv_read_start(uv_stream_t*, uv_alloc_cb alloc_cb, uv_read_cb read_cb);
    int uv_read_stop(uv_stream_t*);
    int uv_write(uv_write_t* req, uv_stream_t* handle,
                uv_buf_t bufs[], int bufcnt, uv_write_cb cb);

The stream based functions are simpler to use than the filesystem ones and
libuv will automatically keep reading from a stream when ``uv_read_start()`` is
called once, until ``uv_read_stop()`` is called.
基于流的函数使用起来要比文件系统更加简单，一旦你调用了 ``uv_read_start()`` ，
libuv就会自动地持续读取流中的数据，直到 ``uv_read_stop()`` 被调用。

The discrete unit of data is the buffer -- ``uv_buf_t``. This is simply
a collection of a pointer to bytes (``uv_buf_t.base``) and the length
(``uv_buf_t.len``). The ``uv_buf_t`` is lightweight and passed around by value.
What does require management is the actual bytes, which have to be allocated
and freed by the application.
缓冲区—— ``uv_buf_t`` 是指数据的不连续单元。它仅包含一个指向字节数组的指针
(``uv_buf_t.base``)和一个表示字节数组容量的长度值(``uv_buf_t.len``). 这样轻量
的 ``uv_buf_t`` 是按值传递的。真正需要管理是的实际存放字节数组的内存空间，它们
必须由应用负责分配和释放。

To demonstrate streams we will need to use ``uv_pipe_t``. This allows streaming
local files [#]_. Here is a simple tee utility using libuv.  Doing all operations
asynchronously shows the power of evented I/O. The two writes won't block each
other, but we've to be careful to copy over the buffer data to ensure we don't
free a buffer until it has been written.
我们必须使用 ``uv_pipe_t`` 来演示流及其操作。这允许流化本地的文件 [#]_ . 接下来我们
利用libuv来实现一个简单的tee程序。程序将以异步的方式执行所有的操作，可以窥见强大的基
于事件的IO。两个写操作之间不会彼此阻塞，但我们必须小心的拷贝缓冲中的数据，确保在它们
被成功写入之前不会被释放掉。

The program is to be executed as::

    ./uvtee <output_file>

We start of opening pipes on the files we require. libuv pipes to a file are
opened as bidirectional by default.

输入::

    ./uvtee <output_file>

来执行我们的程序。首先我们必须在文件上建立管道。libuv从文件上建立的管道默认是双向的。

.. rubric:: uvtee/main.c - read on pipes 从管道上进行读取操作
.. literalinclude:: ../code/uvtee/main.c
    :linenos:
    :lines: 62-81
    :emphasize-lines: 4,5,15

The third argument of ``uv_pipe_init()`` should be set to 1 for IPC using named
pipes. This is covered in :doc:`processes`. The ``uv_pipe_open()`` call
associates the file descriptor with the file.
为了让IPC使用命名管道， ``uv_pipe_init()`` 的第三个参数应该设置成１. 具体内容请
参考 :doc:`processes`. ``uv_pipe_open()`` 的调用会将文件描述符与文件关联起来。

We start monitoring ``stdin``. The ``alloc_buffer`` callback is invoked as new
buffers are required to hold incoming data. ``read_stdin`` will be called with
these buffers.
我们开始监视 ``stdin``. 随着数据的输入，libuv要创建新的缓冲区来保存数据，此时，
它会调用``alloc_buffer`` 回调函数。然后这些缓冲区又会被输入到 ``read_stdin`` 中。

.. rubric:: uvtee/main.c - reading buffers 读取缓冲区
.. literalinclude:: ../code/uvtee/main.c
    :linenos:
    :lines: 19-22,44-60

The standard ``malloc`` is sufficient here, but you can use any memory allocation
scheme. For example, node.js uses its own slab allocator which associates
buffers with V8 objects.
在这里我们使用标准 ``malloc`` 已经足够了，但你可以使用任意一种内存分配方案。比如，
在node.js的实现中， 他们设计了自己的 slab 分配器，来将V8虚拟机对象和缓冲区关联起来。

The read callback ``nread`` parameter is -1 on any error. This error might be
EOF, in which case we close all the streams, using the generic close function
``uv_close()`` which deals with the handle based on its internal type.
Otherwise ``nread`` is a non-negative number and we can attempt to write that
many bytes to the output streams. Finally remember that buffer allocation and
deallocation is application responsibility, so we free the data.
当发生错误时，读回调函数传入的 ``nread`` 参数被置为-1. 这种错误可能是读到了文件结束符
(EOF)，在这种情况下，我们调用通用的关闭函数 ``uv_close()`` 来关闭所有的流。称之为通用
是因为 ``uv_close()`` 可以处理所有基于libuv内部类型的句柄。
如果``nread``是非负数，我们就可以将它写入到输出流中。记住，缓冲区的分配和释放是由应用
程序负责的，因此最后我们需要释放这些数据。

.. rubric:: uvtee/main.c - Write to pipe 向管道进行写操作
.. literalinclude:: ../code/uvtee/main.c
    :linenos:
    :lines: 9-13,23-42

``write_data()`` makes a copy of the buffer obtained from read. Again, this
buffer does not get passed through to the callback trigged on write completion.
To get around this we wrap a write request and a buffer in ``write_req_t`` and
unwrap it in the callbacks.
``write_data()`` 复制了读操作中获取的缓冲区。另外，这个缓冲区不会在写操作完成时
触发的回调函数传入，但我们需要在这个时刻来回收这个缓冲区的内存空间，为此，我们
将缓冲区包裹到写请求 ``write_req_t`` 的结构里面，然后再在写完成回调中拆出来。

.. WARNING::

    If your program is meant to be used with other programs it may knowingly or
    unknowingly be writing to a pipe. This makes it susceptible to `aborting on
    receiving a SIGPIPE`_. It is a good idea to insert::

        signal(SIGPIPE, SIG_IGN)

    in the initialization stages of your application.
    如果你的程序打算被其它的程序使用，它应该被写成一个管道。这使它容许在接到 
    `SIGPIPE信号时取消`_ 。好的作法是在你的程序初始化阶段插入这句::
      
        signal(SIGPIPE, SIG_IGN)



.. _SIGPIPE信号时取消: http://pod.tst.eu/http://cvs.schmorp.de/libev/ev.pod#The_special_problem_of_SIGPIPE

File change events
文件修改事件
------------------

All modern operating systems provide APIs to put watches on individual files or
directories and be informed when the files are modified. libuv wraps common
file change notification libraries [#fsnotify]_. This is one of the more
inconsistent parts of libuv. File change notification systems are themselves
extremely varied across platforms so getting everything working everywhere is
difficult. 
所有现代操作系统都提供了API用于监视单独的文件或目录，当文件遭修改时应用程序会被告知。
libuv 封装了常用的文件修改通知库 [#fsnotify]_. 这是libuv中较不一致的一部分内容。文件
修改通知系统在不同的平台上差异极大，因此要让所有东西在任何地方都可以工作是相当困难的。

To demonstrate, I'm going to build a simple utility which runs
a command whenever any of the watched files change::

    ./onchange <command> <file1> [file2] ...
作为演示，我打算构建一个简单的工具，它会监视某些文件，并在文件被修改时执行一条命令。
你可以这样启动它 ::

    ./onchange <command> <file1> [file2] ...

The file change notification is started using ``uv_fs_event_init()``:
通过调用 ``uv_fs_event_init()`` 启动文件修改通知。

.. rubric:: onchange/main.c
.. literalinclude:: ../code/onchange/main.c
    :linenos:
    :lines: 29-32
    :emphasize-lines: 3

The third argument is the actual file or directory to monitor. The last
argument, ``flags``, can be:

.. literalinclude:: ../libuv/include/uv.h
    :lines: 1501,1510

but both are currently unimplemented on all platforms.

第三个参数指定要监视的文件或者目录。最后一个参数 ``flags`` ，可以是以下这些已定义的值：

.. literalinclude:: ../libuv/include/uv.h
    :lines: 1578,1588

但是，目前所有的平台都没有实施这些标志。

.. warning::

    You will in fact raise an assertion error if you pass any flags. So stick
    to 0.
    如果你传入任何这些标志的话，你会触发一个断言。因此只能传入0.


The callback will receive the following arguments:

  #. ``uv_fs_event_t *handle`` - The watcher. The ``filename`` field of the watcher
     is the file on which the watch was set.
  #. ``const char *filename`` - If a directory is being monitored, this is the
     file which was changed. Only non-``null`` on Linux and Windows. May be ``null``
     even on those platforms.
  #. ``int flags`` - one of ``UV_RENAME`` or ``UV_CHANGE``.
  #. ``int status`` - Currently 0.

回调函数将会收到下列参数:
  #. ``uv_fs_event_t *handle`` - watcher. watcher的 ``filename`` 字段指示当前的watcher所监视的目标文件。
  #. ``const char *filename`` - 如果被监视的对象是一个目录，这个参数指示被修改的文件。只有在Linux和Windows
       上这些参数不为 ``null``. 在其它平台上，可能为 ``null``.
  #. ``int flags`` - 不是 ``UV_RENAME`` 就是 ``UV_CHANGE``.
  #. ``int status`` - 目前为0.

In our example we simply print the arguments and run the command using
``system()``.

在我们的例子中，我们只要简单的打印这些参建并使用 ``system()`` 来运行命令。

.. rubric:: onchange/main.c - file change notification callback 文件修改通知回调
.. literalinclude:: ../code/onchange/main.c
    :linenos:
    :lines: 9-18

.. rubric:: Footnotes
.. [#fsnotify] inotify on Linux, kqueue on BSDs, ReadDirectoryChangesW on
    Windows, event ports on Solaris, unsupported on Cygwin
.. [#] see :ref:`pipes`
