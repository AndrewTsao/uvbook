引言 Introduction
============

This 'book' is a small set of tutorials about using libuv_ as
a high performance evented I/O library which offers the same API on Windows and Unix.

这本书由一系列的教程组成，教你如何使用 libuv_, 这是一个在Windows和Unix系统上提供相同的应用编程接口(API)的，性能优良的基于事件的IO库。


It is meant to cover the main areas of libuv, but is not a comprehensive
reference discussing every function and data structure. The `official libuv
documentation`_ is included directly in the libuv header file.

本书试图包含libuv中的主要内容，但不打算成为一本全面讨论每一个函数和数据结构的参考
手册。 libuv的官方文档(`official libuv documentation`_)已经附在libuv的头文件里。


.. _`official libuv documentation`: https://github.com/joyent/libuv/blob/master/include/uv.h

This book is still a work in progress, so sections may be incomplete, but
I hope you will enjoy it as it grows.

本书尚未付梓，许多章节有待完善，我希望你一边看它成长，一边欣赏。

为谁而书 Who this book is for
--------------------

If you are reading this book, you are either:

如果你正在阅读此书，你或许是下列两种人：

1) a systems programmer, creating low-level programs such as daemons or network
   services and clients. You have found that the event loop approach is well
   suited for your application and decided to use libuv.
   
   系统程序员，创建底层守护进程、网络服务或客户商端的应用程序。你已经发现基于事件循环的处理方法非常适合你的应用，并决定采用libuv.

2) a node.js module writer, who wants to wrap platform APIs
   written in C or C++ with a set of (a)synchronous APIs that are exposed to
   JavaScript. You will use libuv purely in the context of node.js. For
   this you will require some other resources as the book does not cover parts
   specific to v8/node.js.
   
   node.js模块的开发人员，并想要将一些使用C/C++编写的平台API以同步或者异步的方式暴露到JavaScript脚本运行时环境中。你将在node.js上下文里单纯的使用libuv。为此你可能需要必备一些其它的图书资源，介绍不在此书讨论范围内的（比如与v8或者node.js相关的）知识。

This book assumes that you are comfortable with the C programming language.

本书假设读者熟练掌握了C编程语言。


背景介绍 Background
----------

The node.js_ project began in 2009 as a JavaScript environment decoupled
from the browser. Using Google's V8_ and Marc Lehmann's libev_, node.js
combined a model of I/O -- evented -- with a language that was well suited to
the style of programming; due to the way it had been shaped by browsers. As
node.js grew in popularity, it was important to make it work on Windows, but
libev ran only on Unix. The Windows equivalent of kernel event notification
mechanisms like kqueue or (e)poll is IOCP. libuv is an abstraction around libev
or IOCP depending on the platform, providing users an API based on libev.

node.js_ 项目开始于2009年，以是一个从浏览器分离的JavaScript运行环境。它使用Google开发的V8_和Marc Lehmann开发的libev_,
node.js将基于事件的IO模型和一门非常适合于这种模型进行编程的语言融合在了一起，这种基于事件的编程模型是在浏览器中形成的。
随着node.js的日益盛行，让这一神器适应Windows成了重要工作，但libev只能在Unix下运行。在Windows平台下，和kqueue或(e)poll对应的内核事件通知机制是由IOCP提供的。libuv根据不同的平台，对libuv和IOCP进行了抽象封装，以给用户提供一套基于libev的API.


代码 Code
----

All the code from this book is included as part of the source of the book on
Github. `Clone`_/`Download`_ the book and run ``make`` in the ``code/``
folder to compile all the examples. This book and the code is based on libuv
version `node-v0.8.0` and a version is included in the ``libuv/`` folder
which will be compiled automatically.

本书相关的所有代码均包含在本书在Github上的源码库中。 克隆(`Clone`_) 或者
下载(`Download`_)此书源码库，并在目录 ``code/`` 运行 ``make`` 以编译书中所有实
例。本书和代码都是基于 `node-v0.8.0` 中的libuv的，并在源码库中的 ``libuv/`` 中包
含一份可以自动编译的拷贝版本。

.. _Clone: https://github.com/nikhilm/uvbook
.. _Download: https://github.com/nikhilm/uvbook/downloads
.. _node-v0.8.0: https://github.com/joyent/libuv/tags
.. _V8: http://code.google.com/p/v8/
.. _libev: http://software.schmorp.de/pkg/libev.html
.. _libuv: https://github.com/joyent/libuv
.. _node.js: http://www.nodejs.org
