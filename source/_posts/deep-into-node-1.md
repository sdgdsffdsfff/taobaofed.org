---
title: Node.js 探秘（一）- 初识单线程的 Node.js
categories: 
  - Node.js
tags:
  - Node
date: 2015-10-29 15:44:30
author: 凌恒
published: true
---

![Node.js 探秘（一）- 初识单线程的 Node.js](https://img.alicdn.com/tps/TB1v9UnKXXXXXc5XXXXXXXXXXXX-900-500.png)

## 前言

从Node.js进入人们的视野时，我们所知道的它就由这些关键字组成 **事件驱动、异步、单线程、非阻塞I/O**，它的官网也是这么说的。

> Node.js® is a JavaScript runtime built on Chrome's V8 JavaScript engine. Node.js uses an event-driven, non-blocking I/O model that makes it lightweight and efficient.

由此，我们可能会想到一些问题：

+ 如何与操作系统进行交互？
+ 如何实现异步I/O？
+ 如何实现非阻塞I/O的？
+ 如何实现事件驱动的？
+ 难道全是异步调用和非阻塞I/O，就不会有并发问题嘛？

在解决这些问题之前，我们先来陈述一些概念：

+ 事件驱动（event-driven）：
  
  这对前端来说应该不陌生，在我们工作中，会接触到很多事件的概念，比如DOM事件（GUI层面的应用），事件回调（异步的一种实现）等等，简单来说，事件一般表现为程序某些信息状态上的变化。事件驱动即是一种托付者式的被动等通知，行为取决于外来突发事件的编程模式。

+ 同步/异步（synchronization/asynchronization）：
  
  同步，就是在产生一个 **调用** 时，在没有得到结果之前，该 **调用** 就不会返回，一旦 **调用** 返回，就得到返回值了，是 **调用者** 主动等待这个 **调用** 的结果。
  
  异步，则是在 **调用** 发出后，这个 **调用** 就直接返回了，所以没有返回结果。之后， **被调用者** 通过状态、通知等来通知 **调用者** ，或者通过回调函数处理这个 **调用** 。
  
+ 阻塞/非阻塞（blocking/non-blocking）

  阻塞调用是指调用结果返回之前，当前线程会被挂起。调用线程只有在得到结果之后才会返回。
  非阻塞调用指在不能立刻得到结果之前，该调用不会阻塞当前线程。
  
虽然，看起来同步/异步和阻塞/非阻塞看起来有些相似，但是前者关注的是一种 **消息通知机制** ，后者关注的是 **程序在等待调用结果（消息、返回值）时的状态** ，所以两者没有必然联系。

基础概念介绍完了，下面我们开始解决问题，先看下 Node.js 的结构图。

![Node.js Architecture](http://img4.tbcdn.cn/L1/461/1/a9e67142615f49863438cc0086b594e48984d1c9)

从图中我们可以看出，Node.js 整体分为三层（从上至下）：

 + Javascript 封装，就是我们在写的时候调用的 Node.js API。
 + Node bindings，用来连接底层 C/C++ 模块和 Javascript 层的桥梁，它可以让 Javascript 和 C/C++ 相互调用，交换数据等。
 + 底层结构，这是比较复杂的一层，它包括 V8 引擎、libuv、C-ares、http_parser 等等，这些都是真正执行上层调用的模块。

总体来看，Node.js 实际上是为 Javascript 提供了在非浏览器环境中执行的环境，这样通过 V8 解释过的代码，才能运行。

这个结构里面有个重点，就是 **libuv** 它是 Node.js 极其重要的组成部分，我们解决上面的问题也会围绕它进行。

## libuv

简单介绍一下，libuv 具体干了什么？下面是官网的一张图。

![libuv_architecture](http://img1.tbcdn.cn/L1/461/1/cdd725280cce7929c5dd526bbf19e9de36591e09)

从图中可以看到，libuv 可以做很多事情，网络 I/O，文件 I/O，线程管理等等，几乎囊括了各种与操作系统交互的工作。

本来，在不同操作系统上，这些操作都有不同的实现，Node.js 作者为了做到“提供统一的调用接口，对用户隐藏底层实现”的目的，将 libev、libeio 以及 IOCP 等进行了封装，对外提供统一的 API， 这是 Node.js 能够跨平台很重要的一个原因。

## 如何与操作系统进行交互？

在简单了解了 libuv 之后，这个问题就很简单了，Node.js 正是通过调用 libuv 来实现与操作系统的交互。不过，还有更细节的内容，我们通过一个例子来说明。

````javascript
var fs = require('fs');
fs.open('./test.txt', "w", function(err, fd) {
	//..do something
});
````
这是代码描述了一个很简单的文件 I/O 操作，打开文件。这段代码执行的流程如下图：

![Node.js File操作](http://img2.tbcdn.cn/L1/461/1/fad8e5f6433ca965b3fcf282910ba5bc3bff65cf)

Node.js 的 Javascript 代码通过 bindings 调用了 C/C++ 代码中对应的函数，然后这一层会封装一个叫做 **请求对象** 的东西，交给 libuv，其中包含了要执行的功能，参数，以及完成的回调，由 libuv 执行指定操作并处理相应回调。

当然，这其中的平台判断不是在运行时执行的，libuv 在编译时就已经决定调用那种实现了。

## Node.js 如何实现异步 I/O

上文说到 Node.js 使用了 libuv 来实现了与操作系统的交互，其中说到 libuv 执行完 I/O 操作后，会调用回调，来返回 I/O 结果，那么问题来了。

### libuv 何时执行回调？

最容易想到的，就是在 I/O 执行完，就立即调用回调。但是，这是 **极不安全** 的做法。

因为 Javascript 的执行进程是单线程的，如果两个回调同时返回，会出现 **回调竞争** ，导致某个回调未执行或者丢失，这是个很严重的问题，极大的增加了系统的不稳定性。

所以，libuv 采用了一种事件循环（Event Loop）的机制，来接收并处理回调函数。同时，它还在其中维护了多个观察者，比如时间、网络等等，使用者可以在观察者中注册回调，当事件发生时，其状态会被变成 pending，libuv 就会去触发这种状态的事件。

Event loop 类似于一个`while true`的无限循环，在每次循环中，会不断的检查各个队列中是否有 pending 状态的事件，有就触发。一般来说，这里可能出现问题，如果队列中同时有两个或多个事件处于 pending 状态，一样会发生竞争。libuv 为了处理这个问题，在每次事件循环中，只会执行一个回调，避免竞争发生。


总结来说，一个异步 I/O 的大致流程如下：

+ 发起 I/O 调用

    1. 用户通过 Javascript 代码调用 Node 核心模块，将参数和回调函数传入到核心模块；
    2. Node 核心模块会将传入的参数和回调函数封装成一个请求对象；
    3. 将这个请求对象推入到 I/O 线程池等待执行；
    4. Javascript 发起的异步调用结束，Javascript 线程继续执行后续操作。

+ 执行回调
  
    1. I/O 操作完成后，会将结果储存到请求对象的 result 属性上，并发出操作完成的通知；
    2. 每次事件循环时会检查是否有完成的 I/O 操作，如果有就将请求对象加入到 I/O 观察者队列中，之后当做事件处理；
    3. 处理 I/O 观察者事件时，会取出之前封装在请求对象中的回调函数，执行这个回调函数，并将 result 当参数，以完成 Javascript 回调的目的。

![Node.js 异步](http://img2.tbcdn.cn/L1/461/1/6a9490a6529010804899437ad63645233356d6bb)

如图所示，Node.js 通过线程池来实现异步 I/O 操作。除了 Javascript 执行线程是单线程外，所有I/O都是多线程并行执行的。

## 如何实现非阻塞I/O的？

非阻塞 I/O 指的是在程序执行过程中，I/O 操作不会阻塞程序的执行，它可以继续执行其他代码。我们从上一个问题可以看出，Node.js 的 I/O 操作都是异步的，交由其他线程来执行，不会阻塞当前线程的执行，也就实现了非阻塞的 I/O。

## 如何实现事件驱动的？

正常来说，实现事件驱动，一个线程是做不到的。恰恰 Node.js 正是单线程的，其所依赖的 Javascript 环境 V8 引擎，设计上是不支持多线程的，因为 Javascript 在浏览器端本身就是单线程执行的。

那么 Node.js 是如何做的呢？

先来看张图，libuv 中 `uv_run` 函数的流程图：

![uv_run 流程图](http://img2.tbcdn.cn/L1/461/1/b1d06a5e6d9965267cbddc95219a325e85b3ad2b)

### I/O 操作的事件

上文，我们提到了文件 I/O 和网络 I/O 这两种情况，这两种 I/O 操作，都是通过 libuv 来实现异步操作的，而 Node.js 正是通过 libuv 提供的事件循环，事件队列和 Javascript 本身的回调机制，来实现事件驱动的能力。

### timer 事件

除了上面提到的两种情况外，还有一种情况，就是定时器，即 timer。它并不是通过单独开立线程实现的，而是直接在事件循环中就完成。

在事件循环中，会维护一个 time 的字段，在初始化时会被赋值为0，每次循环都会更新这个值。所有与时间相关的操作，都会和这个值进行比较，来决定是否执行。

在图中，与 timer 相关的过程如下：

1. 更新当前循环的 time 字段，即当前循环下的“现在”；
2. 检查循环中是否还有需要处理的任务（handlers/requests），如果没有就不必循环了，即是否 alive。
3. 检查注册过的 timer，如果某一个 timer 中指定的时间落后于当前时间了，说明该 timer 已到期，于是执行其对应的回调函数；
4. 执行一次 I/O polling（即阻塞住线程，等待 I/O 事件发生），如果在下一个 timer 到期时还没有任何 I/O 完成，则停止等待，执行下一个 timer 的回调。如果发生了 I/O 事件，则执行对应的回调；由于执行回调的时间里可能又有 timer 到期了，这里要再次检查 timer 并执行回调。

Node.js 会一直调用 `uv_run` 直到到循环不在 alive。

## 难道全是异步调用和非阻塞I/O，就不会有并发问题嘛？

上文已经提到，Node.js 的单线程实际是指 Javascript 运行线程是单线程的，实际的 I/O 操作都是由 libuv 通过线程池来实现的。既然存在了线程池，那么这个池子大小有多大？会不会可以提高大小来提高性能呢？

翻了下 libuv 的文档，[这里面](https://github.com/libuv/libuv/blob/bb2632b339d86f69f7dd9aa7168e4b46bf4126a5/docs/src/threadpool.rst)说到，这个线程池的默认大小为4，即同时能有4个线程来进行 I/O 操作，剩下的请求则会被挂起等待，直到线程池有空余，所以，Node.js 还是需要考虑 I/O 的并发问题。

当然，这个线程池的大小也是可以改变的，通过`UV_THREADPOOL_SIZE`这个环境变量来改变，或者`process.env.UV_THREADPOOL_SIZE`来设置。要注意的是，增加线程池大小，会增加内存的占用（128 线程大约占用 1MB），但是是可以在运行时提高性能的，典型的空间换时间。不过增加大小也是有限制的，最大128。

虽然 I/O 都是由其他线程来做，但是这些线程的维护者不一样。从上面 `libuv` 官网那张图，可以看到，文件 I/O、DNS 操作、用户代码，都是由 `libuv` 维护的线程池来完成，所以前文说到的 `process.env.UV_THREADPOOL_SIZE` 的参数只能影响这些操作。

而有关网络 I/O，TCP、UDP 等，都是采用的纯事件机制，`libuv` 实际使用的是操作系统底层方法，在不同的操作系统中选择了不同的解决方案。它将事件绑定在网络内核，内核操作完后，会告知 `libuv`，再进行下一步操作。

可以理解为：`Javascript 绑定事件` → `libuv 绑定事件` → `网络内核监听事件`，`内核事件触发 ` → `libuv` → `Javascript 这样一个过程`。

这样，其瓶颈就在与对应操作系统的网络内核中，这个内容，有机会再细说。

## 总结

+ Node.js 通过 `libuv` 来处理与操作系统的交互
+ `libuv` 是 Node.js 非阻塞、异步 I/O 和事件驱动能力的根源
+ Node.js 的单线程只得是 Javascript 执行线程，实际的 I/O 操作都是多线程执行的。
