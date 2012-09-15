Networking
网络功能
==========

Networking in libuv is not much different from directly using the BSD socket
interface, some things are easier, all are non-blocking but the concepts stay
the same. In addition libuv offers utility functions to abstract the annoying,
repetitive and low-level tasks like setting up sockets using the BSD socket
structures, DNS lookup, and tweaking various socket parameters.
使用libuv的网络功能，较之直接使用BSD套接字接口差异不大，但某些方面会比较容易，
并且所有的操作都是非阻塞的，但基本概念是完全一样的。另外，libuv提供了相关的
工具函数，抽象了那些烦人的、重复的、低级别的任务，比如使用BSD套接字结构启动
socket，DNS查询，调节不同的socket参数等。

The ``uv_tcp_t`` and ``uv_udp_t`` structures are used for network I/O.
网络IO中使用了 ``uv_tcp_t`` 和 ``uv_udp_t`` 数据结构。

TCP
---

TCP is a connection oriented, stream protocol and is therefore based on the
libuv streams infrastructure.

TCP是面向连接的、流传输协议，因此是基于libuv流基础设施。

Server
++++++

Server sockets proceed by:

1. ``uv_tcp_init`` the TCP watcher.
2. ``uv_tcp_bind`` it.
3. Call ``uv_listen`` on the watcher to have a callback invoked whenever a new
   connection is established by a client.
4. Use ``uv_accept`` to accept the connection.
5. Use :ref:`stream operations <buffers-and-streams>` to communicate with the
   client.
服务端Socket使用过程如下：
1. ``uv_tcp_init`` 初始化一个TCP Watcher.
2. ``uv_tcp_bind`` 进行绑定监听地址和端口。
3. ``uv_listen`` 在Watcher上绑定一个回调函数，当客户建立一个连接时，该回调函数将被调用。
4. 使用 ``uv_accept`` 来接受一个连接。
5. 使用 :ref:`流的操作 <buffers-and-streams>` 来与客户端进行通讯。

Here is a simple echo server

.. rubric:: tcp-echo-server/main.c
.. literalinclude:: ../code/tcp-echo-server/main.c
    :linenos:
    :lines: 50-
    :emphasize-lines: 4-5,7-9

下面是一个简单的Echo服务器实例。

.. rubric:: tcp-echo-server/main.c
.. literalinclude:: ../code/tcp-echo-server/main.c
    :linenos:
    :lines: 50-
    :emphasize-lines: 4-5,7-9

You can see the utility function ``uv_ip4_addr`` being used to convert from
a human readable IP address, port pair to the sockaddr_in structure required by
the BSD socket APIs. The reverse can be obtained using ``uv_ip4_name``.
实例代码中的利用辅助函数 ``uv_ip4_addr`` 将人类易读的IP地址，端口对转换为BSD套接字API
要求的sockaddr_in结构体。反过来可以使用 ``uv_ip4_name`` 从结构体中获取IP地址和端口号。

.. NOTE::

    In case it wasn't obvious there are ``uv_ip6_*`` analogues for the ip4
    functions.
    还有一系列的 ``uv_i6_*`` 函数族，它们提供了与ip4函数相同的功能。此例未展示。

Most of the setup functions are normal functions since its all CPU-bound.
``uv_listen`` is where we return to libuv's callback style. The second
arguments is the backlog queue -- the maximum length of queued connections.
大多数的设置函数是普通函数因为它们都是受限于CPU的。 ``uv_listen`` 调用之后我们又
开始使用libuv回调风格了。它的第二个参数是指积压的接收队列——表示最大的列队连接数。

When a connection is initiated by clients, the callback is required to set up
a watcher for the client socket and associate the watcher using ``uv_accept``.
In this case we also establish interest in reading from this stream.
当客户端发起了一个连接时，需要回调一个函数来为客户端Socket设置一个watcher，这可以
通过 ``uv_accept`` 来完成。既然这样，我们也要可以在这个流上面设置读操作。

.. rubric:: tcp-echo-server/main.c
.. literalinclude:: ../code/tcp-echo-server/main.c
    :linenos:
    :lines: 34-48
    :emphasize-lines: 9-10

The remaining set of functions is very similar to the streams example and can
be found in the code. Just remember to call ``uv_close`` when the socket isn't
required. This can be done even in the ``uv_listen`` callback if you are not
interested in accepting the connection.
实例代码中的其它函数的使用和流操作实例中的非常相似。不过请记住，当Sket不再需要时，
要调用 ``uv_close`` . 这个过程甚至可以在 ``uv_listen`` 的回调函数中完成，如果你
对所接受的连接不感兴趣的话。

Client
++++++

Where you do bind/listen/accept, on the client side its simply a matter of
calling ``uv_tcp_connect``. The same ``uv_connect_cb`` style callback of
``uv_listen`` is used by ``uv_tcp_connect``. Try::

    uv_tcp_t socket;
    uv_tcp_init(loop, &socket);

    uv_connect_t connect;

    struct sockaddr_in dest = uv_ip4_addr("127.0.0.1", 80);

    uv_tcp_connect(&connect, &socket, dest, on_connect);

where ``on_connect`` will be called after the connection is established.
相对于服务端的 bind/listen/accept操作，在客户端你只需要简单的调用 ``uv_tcp_connect`` ，
``uv_tcp_connect`` 使用与 ``uv_listen`` 相同的 ``uv_connect_cb`` 的风格的回调函数。
小试一下::

    uv_tcp_t socket;
    uv_tcp_init(loop, &socket);

    uv_connect_t connect;

    struct sockaddr_in dest = uv_ip4_addr("127.0.0.1", 80);

    uv_tcp_connect(&connect, &socket, dest, on_connect);

此例中 ``on_connect`` 将在连接建立之后被调用。


UDP
---

The `User Datagram Protocol`_ offers connectionless, unreliable network
communication. Hence, unlike TCP, it doesn't offer a stream abstraction since
each packet is independent. libuv provides non-blocking UDP support via the
`uv_udp_t` (for receiving) and `uv_udp_send_t` (for sending) structures and
related functions. That said, the actual API for reading/writing is very
similar to normal stream reads. To look at how UDP can be used, the example
shows the first stage of obtaining an IP address from a `DHCP`_ server -- DHCP
Discover.

用户数据报文协议（UDP, `User Datagram Protocol`_ ）提供无连接的、不可靠的网络通讯机制。因此，不像TCP，它不提供
流的抽象，因为每个数据包都是彼此独立的。 libuv通过 `uv_udp_t` （接收）和 `uv_udp_send_t` （发送）
两种数据结构及相关的函数，提供了非阻塞的UDP支持。据说，实际用于读写操作的API和普通的流读取
操作非常的相似。为展示如何使用UDP，接下来的实例演示了从 `DHCP`_ 服务器获取IP地址
的第一阶段——DHCP发现。

.. note::

    You will have to run `udp-dhcp` as **root** since it uses well known port
    numbers below 1024.
    你需要以 **root** 身份来运行 `udp-dhcp`` ，因为它使用了小于1024以下的知名端口。

.. rubric:: udp-dhcp/main.c
.. literalinclude:: ../code/udp-dhcp/main.c
    :linenos:
    :lines: 7-10,105-
    :emphasize-lines: 8,10-11,14,15,21

.. note::

    The IP address ``0.0.0.0`` is used to bind to all interfaces. The IP
    address ``255.255.255.255`` is a broadcast address meaning that packets
    will be sent to all interfaces on the subnet.  port ``0`` means that the OS
    randomly assigns a port.
    IP地址 ``0.0.0.0`` 表示绑定所有的网卡。IP地址 ``255.255.255.255`` 是广播地址，
    表示将数据发送到当前子网中的所有设备上。端口 ``0`` 意味着操作系统随机分配一个端口。

First we setup the receiving socket to bind on all interfaces on port 68 (DHCP
client) and start a read watcher on it. Then we setup a similar send socket and
use ``uv_udp_send`` to send a *broadcast message* on port 67 (DHCP server).
首先我们设置一个接口Socket并绑定所有的网卡的68端口(DHCP)端口并开始监视上面的读操作。
然后，我们设置一个相似的发送Socket，并使用 ``uv_udp_send`` 向67端口(DHCP服务器)发送
一个 *广播消息* 。

It is **necessary** to set the broadcast flag, otherwise you will get an
``EACCES`` error [#]_. The exact message being sent is irrelevant to this book
and you can study the code if you are interested. As usual the read and write
callbacks will receive a status code of -1 if something went wrong.
这里 **必需** 设置广播标志，否则你将得到一个 ``EACCES`` 的错误 [#]_. 至于消息是
如何发送的过程并非此书所关心的，如果你感兴趣的话，可以研习代码。和一般的读写回
调一样，如果过程中有错误的话，你将收到一个被标记为-1的状态码。

Since UDP sockets are not connected to a particular peer, the read callback
receives an extra parameter about the sender of the packet. The ``flags``
parameter may be ``UV_UDP_PARTIAL`` if the buffer provided by your allocator
was not large enough to hold the data. *In this case the OS will discard the
data that could not fit* (That's UDP for you!).
因为UDP套接字并不连接到某一个特定端，读回调函数会接受到一个关于数据包发送者
信息的参数。 如果你的分配器未能提供足够空间的缓冲区容纳接收到的数据的话， 
``flags`` 参数可能是 ``UV_UDP_PARTIAL`` 。这时候，操作系统将丢弃那些容不下的数据。
(这正是UDP所决定的!)。

.. rubric:: udp-dhcp/main.c
.. literalinclude:: ../code/udp-dhcp/main.c
    :linenos:
    :lines: 15-27,38-41
    :emphasize-lines: 1,16

UDP Options
UDP选项
+++++++++++

The TTL of packets sent on the socket can be changed using ``uv_udp_set_ttl``.
如果必要，可以使用 ``uv_udp_set_ttl`` 来修改发送到某个 socket 上的数据包的TTL.

IPv6 stack only
限用IPv6协议
~~~~~~~~~~~~~~~

IPv6 sockets can be used for both IPv4 and IPv6 communication. If you want to
restrict the socket to IPv6 only, pass the ``UV_UDP_IPV6ONLY`` flag to
``uv_udp_bind6`` [#]_.
IPv6 套接字能够同时用于IPv4和IPv6的通讯，如果你希望仅限于IPv6的话，你可以在调用
``uv_udp_bind6`` 时传入 ``UV_UDP_IPV6ONLY`` 标志。

Multicast
多播
~~~~~~~~~

A socket can (un)subscribe to a multicast group using:
一个套接字可以如下的方式订阅或者退订一个多播组：
.. literalinclude:: ../libuv/include/uv.h
    :lines: 738

where ``membership`` is ``UV_JOIN_GROUP`` or ``UV_LEAVE_GROUP``.
参数 ``membership`` 可以是 ``UV_JOIN_GROPU`` 和 ``UV_LEAVE_GROUP``.

Local loopback of multicast packets is enabled by default [#]_, use
``uv_udp_set_multicast_loop to switch it off``.
多插包的本地回环是被默认设置的 [#]_, 可以使用 ``uv_udp_set_multicast_loop`` 来关闭它。


The packet time-to-live for multicast packets can be changed using
``uv_udp_set_multicast_ttl``.
多播包的TTL可以利用 ``uv_udp_set_multicast_ttl`` 修改。

Querying DNS
DNS 查询
------------

libuv provides asynchronous DNS resolution. For this it provides its own
``getaddrinfo`` replacement [#]_. In the callback you can
perform normal socket operations on the retrieved addresses. Let's connect to
Freenode to see an example of DNS resolution.

libuv提供了异步的域名解析功能。为些它提供了它自己的 ``getaddrinfo`` 替代函数 [#]_.
在回调函数中你可以在查询到的地址上执行Socket操作。下例中通过连接Freenode来展示一下DNS解析过程。

.. rubric:: dns/main.c
.. literalinclude:: ../code/dns/main.c
    :linenos:
    :lines: 61-
    :emphasize-lines: 12

If ``uv_getaddrinfo`` returns non-zero, something went wrong in the setup and
your callback won't be invoked at all. All arguments can be freed immediately
after ``uv_getaddrinfo`` returns. The `hostname`, `servname` and `hints`
structures are documented in `the getaddrinfo man page <getaddrinfo>`_.
如果 ``uv_getaddrinfo`` 返回非零，表示你的设置哪里出了问题，此时不要指望你的回
调函数会被调用。所有的参数可以在 ``uv_getaddrinfo`` 返回之后立即释放掉。 `hostname`,
`servname` 和 `hints` 结构体可以参考 `the getaddrinfo man page <getaddrinfo>`_.

In the resolver callback, you can pick any IP from the linked list of ``struct
addrinfo(s)``. This also demonstrates ``uv_tcp_connect``. It is necessary to
call ``uv_freeaddrinfo`` in the callback.
在解析器回调函数中，你可以使用 ``struct addrinfo(s)`` 链表中的任何一个IP地址。
下例也演示了 ``uv_tcp_connect`` 的使用方法。在回调函数中，你必须调用 ``uv_freeaddrinfo`` 
释放相关资源。

.. rubric:: dns/main.c
.. literalinclude:: ../code/dns/main.c
    :linenos:
    :lines: 41-59
    :emphasize-lines: 8,16

Network interfaces
------------------

TODO

.. _c-ares: http://c-ares.haxx.se
.. _getaddrinfo: http://www.kernel.org/doc/man-pages/online/pages/man3/getaddrinfo.3.html

.. _User Datagram Protocol: http://en.wikipedia.org/wiki/User_Datagram_Protocol
.. _DHCP: http://tools.ietf.org/html/rfc2131

.. rubric:: Footnotes
.. [#] http://beej.us/guide/bgnet/output/html/multipage/advanced.html#broadcast
.. [#] on Windows only supported on Windows Vista and later.
.. [#] http://www.tldp.org/HOWTO/Multicast-HOWTO-6.html#ss6.1
.. [#] libuv use the system ``getaddrinfo`` in the libuv threadpool. libuv
    v0.8.0 and earlier also included c-ares_ as an alternative, but this has been
    removed in v0.9.0.
