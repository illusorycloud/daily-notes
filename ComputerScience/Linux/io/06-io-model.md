---
title: "Linux下几种常见IO模型"
description: "Linux下几种常见IO模型"
date: 2020-10-24 22:00:00
draft: false
categories: ["Linux"]
tags: ["Linux"]
---

本文主要简单分析了Linux下的几种常见的IO模型。包括阻塞式I/O、非阻塞式I/O、I/O多路复用等。

<!-- more-->

## 1. 概述

IO 是主存和外部设备 ( 硬盘、终端和网络等 ) 拷贝数据的过程。 IO 是操作系统的底层功能实现，底层通过 I/O 指令进行完成。在本教程中，我们所说的 IO 指的都是网络 IO。

> 技术的出现一定是为了解决当前技术的某些痛点，IO 模型演化也是如此。最初 BIO 到 NIO 、select、epoll 都是为了解决某些不得不解决的问题。

五种 I/O 模型

* 1）阻塞式I/O：blocking IO
* 2）非阻塞式I/O： nonblocking IO
* 3）I/O复用（select，poll，epoll...）：IO multiplexing
* 4）信号驱动式I/O（SIGIO）：signal driven IO
* 5）异步I/O（POSIX的aio_系列函数）：asynchronous IO



在这里，我们以一个网络IO来举例，对于一个 network IO (以read举例)，它会涉及到两个系统对象：一个是调用这个 IO 的进程，另一个就是系统内核(kernel)。当一个 read 操作发生时，它会经历两个阶段：

* **阶段1**：等待数据准备 (Waiting for the data to be ready)

* **阶段2**： 将数据从内核拷贝到进程中 (Copying the data from the kernel to the process)

![io-process](https://github.com/lixd/blog/raw/master/images/linux/io/io-process.png)

当进程请求 I/O 操作的时候，它执行一个系统调用 syscall 将控制权移交给内核。当内核以这种方式被调用，它随即采取任何必要步骤，找到进程所需数据，并把数据传送到用户空间内的指定缓冲区。

内核试图对数据进行高速缓存或预读取，因此进程所需数据可能已经在内核空间里了。如果是这样，该数据只需简单地拷贝出来即可。如果数据不在内核空间，则进程被挂起，内核着手把数据读进内存。



## 2. 阻塞式I/O

在 linux 中，默认情况下所有的 socket 都是 blocking，一个典型的读操作流程大概是这样：

* **第一步**：通常涉及`等待数据从网络中到达`。当所有等待数据到达时，它被`复制到内核中的某个缓冲区`。

* **第二步**：把数据从`内核缓冲区`复制到`应用程序缓冲区`。

![](https://github.com/lixd/blog/raw/master/images/linux/io/bio.png)

当用户进程调用了 recvfrom 这个系统调用，kernel 就开始了 I/O 的**第一阶段**：准备数据。对于 network io 来说，很多时候数据在一开始还没有到达（比如，还没有收到一个完整的 UDP 包），这个时候 kernel 就要等待足够的数据到来。而在用户进程这边，整 个进程会被阻塞。**第二阶段**当 kernel 等到数据准备好了，它就会将数据从 kernel 中拷贝到用户内存，然后 kernel 返回结果，用户进程才解除 block 的状态，重新运行起来。



**所以，blocking IO 的特点就是在 IO 执行的两个阶段都被 block了。**

### 栗子

举个栗子，如下所示是一个简单的 socket server：

```go
func main() {
	// 建立socket，监听端口  第一步:绑定端口
	netListen, err := net.Listen("tcp", "localhost:9800")
	if err != nil {
		panic(err)
	}
	// defer 关闭资源，以免引起内存泄漏
	defer netListen.Close()
	Log("Waiting for clients")
	for {
		// 第二步:等待连接
		conn, err := netListen.Accept()
		if err != nil {
			continue
		}
		Log(conn.RemoteAddr().String(), " tcp connect success")
		// 使用goroutine来处理用户的请求
		go handleConnection(conn)
	}
}
```

**说明**

首先 listen 监听端口，然后在一个 for循环中 等待 accept（accept 过程是阻塞的），也就是说每次只能在处理某个请求或者阻塞在 accept。于是每次请求都开一个新的 goroutine 来处理。`go handleConnection(conn)`。

**具体的调用**

```shell
# 获取socket fd 文件描述符 假设是 fd5
socket = fd5
# 绑定端口
bind 9800
# 监听 这个 文件描述符
listen fd5
# 然后 accept 等待连接过来 比如这里 fd6 就是进来的连接
accept fd5 = fd6
# 然后从 fd6 读取数据
recvfrom fd6       
```

可以看到过程中会发生很多 syscall 系统调用。

### 问题

这样做存在很多问题，需要为每个连接开一个 goroutine ，如果有 1W 链接那就要开 1W goroutine 。

* 1）消耗资源，每个 goroutine  是需要一定内存的。
* 2）cpu 对这么多 goroutine 调度也会浪费资源
* 3）最大问题是 这个 accept 是阻塞的，所以才需要开这么多 goroutine 

> 当然了 goroutine 协程 相对于 线程 是非常轻量级的，调度也是有自己的 GPM 模型，goroutine 的切换也全是在用户态，相较之下已经比线程好很多了。

但是一旦涉及到系统调用 就会很慢，那么能不能让 accept 不阻塞呢？

> 就是因为 accept 是阻塞的，所以只能开多个 线程 or 协程 来勉强使用。



### 解决方案

kernel 中 提供了 sock_nonblock 方法，可以实现非阻塞，于是 NIO 出现了。



## 3. 非阻塞式I/O

linux 下，可以通过设置 socket 使其变为 non-blocking。当对一个 non-blocking socket 执行读操作时，流程是这个样子：

![nio](https://github.com/lixd/blog/raw/master/images/linux/io/nio.png)

从图中可以看出，当用户进程发出 read 操作时，如果 kernel 中的数据还没有准备好，那么它并不会 block 用户进程，而是立刻返回一个 error。 

从用户进程角度讲 ，它发起一个 read 操作后，并不需要等待，而是马上就得到了一个结果。用户进程判断结果是一个 error 时，它就知道数据还没有准备好，于是它可以再次发送 read 操作。一旦 kernel 中的数据准备好了，并且又再次收到了用户进程的 system call，那么它马上就将数据拷贝到了用户内存，然后返回。

**所以，用户进程第一个阶段不是阻塞的,需要不断的主动询问kernel数据好了没有；第二个阶段依然总是阻塞的。**



使用 sock_nonblock 使得该过程非阻塞

```shell
# 获取socket fd 文件描述符 假设是 fd5
socket = fd5
# 绑定端口
bind 9800
# 监听 这个 文件描述符
listen fd5
# 然后 accept 等待连接过来 比如这里 fd6 就是进来的连接
accept fd5 = fd6
# 然后从 fd6 读取数据
recvfrom fd6       
```

### 问题

虽然是非阻塞了，但是还是存在另外的问题，比如 C10K 问题。

假设有 1W 客户端，那么每次循环都需要对每个客户端进行一个 系统调用，假设 1W 客户端里其实就只有 1 个是有消息的，相当于另外 9999 次调用都是浪费。

每次循环时间复杂度为 O(n),但是实际上只需要处理有消息的连接，即O(m)，其中n 为连接数，m 为有消息的连接数

### 解决方案

同样的 kernel 通过 提供 select 方法来解决这个问题。

## 4. 多路复用I/O

IO 复用同非阻塞 IO 本质一样，不过利用了新的 select 系统调用，由内核来负责本来是请求进程该做的轮询操作。

看似比非阻塞 IO 还多了一个系统调用开销，不过因为可以支持多路 IO，才算提高了效率。



### 1. select

它的基本原理就是 select 这个 function 会不断的轮询所负责的所有 socket，当某个 socket 有数据到达了，就通知用户进程。它的流程如图：

![select](https://github.com/lixd/blog/raw/master/images/linux/io/select.png)

select 方法接收多个文件描述符，最后会返回需要处理的那几个文件描述符。这样就不需要遍历所有连接了，只需要处理有消息的那几个。

假设有 1W 连接，即 1W 个文件描述符。

以前是 循环中 为了处理那几个有消息的连接，调用 1W 次 syscall，现在就先调用 `select(fds)` ,把 1W 个 文件描述符发给 kernel，内核处理后，假设只有 2个 连接是有消息的，需要处理，就只会返回对应的 fd。然后程序拿到需要处理的 fd 列表后再挨个处理，这样就减少了 9998 次 syscall。

> **注意** 多路复用返回的是状态，具体读写操作还是由程序来控制。

#### 问题

那么这个模型有没有问题呢？

问题就是 每次 都要发 1W 个 fd 到 kernel ,然后内核中也要完成 1W 次遍历O(n)，但是以前是 1W 次系统调用，现在是 1次系统调用，里面遍历 1W 次，还是优化了不少的。不过还可以继续优化：

* 1） 每次要传 1W 个 fd 给内核
* 2） 内核每次要遍历 1W 个 fd,才能返回需要处理的 fds

#### 解决方案

如何优化呢？

在内核中开辟一块空间，每次有新的 fd 就直接存到这个空间里，不需要了就移除掉。这样问题1 就解决了。

问题2 的话就不好处理了，除非内核中提供某种机制，不需要自己去遍历 fds，等有消息来的时候 主动通知内核哪个 fd 来消息了，这样就很完美了。

于是 kernel 中就提供了 epoll 来解决这些问题。

### 2. epoll

epoll 与 select 相比就是 不需要自己去遍历 fds 了,由事件驱动的方式，有消息了就主动通知。

一共有 3 个方法

* 1）epoll_create()
* 2）epoll_ctl()
* 3）epoll_wait()

具体信息都可以通过`man name`进行查看，比如`man epoll_create()`

> 如果提示 未找到 man 手册的话 可以手动安装一下 以下是 ubuntu 的安装命令 centos 用 yum install 应该是一样的
>
> apt-get install manpages-de  manpages-de-dev  manpages-dev glibc-doc manpages-posix-dev manpages-posix

```shell
epoll_create()
epoll_create() returns a file descriptor referring to the new epoll in‐
       stance. 
# 返回一个文件描述符。
# 这个就是前面说的，开内核中开辟一块空间来存放 fds 返回的文件描述符就是描述这块空间的
```

```shell
epoll_ctl()
# 用于管理 前面创建的空间中的 fds
# 比如新增一个 fd，或者删除某个 fd
```

```shell
epoll_wait()
# 这个就是等 等着某个 fd需要处理了就返回 不用自己去遍历了
```



伪代码

```shell
# 获取socket fd 文件描述符 假设是 fd5
socket = fd5
# 绑定端口
bind 9800
# 监听 这个 文件描述符
listen fd5
# 开辟一块用于存储 fds 的空间，假设为 fd8
epoll_create fd8
# 然后第一件事就是把 前面的 server 端 socket fd5 存进去
apoll_ctl(fd8,add,fd5,accept)
# 然后就开始等待了
epoll_wait(fd8)

# 假设此时有连接进来了 假设为 fd6
accept fd5---> fd6
# 也是 第一件事就存进去
apoll_ctl(fd8,fd6)
...
# 最后 epoll 空间里肯定就存了很多 fd
```

那么问题来了，他是如何指定那个 fd 需要处理的?

就是靠的**事件驱动**，收到消息后，网卡向 CPU 发送一个 中断事件,CPU 根据这个中断号就能判断出具体是什么事件了，然后通过回调方法就去拿到数据。

> 具体是在写入fd时会给每个fd绑定一个回调函数，触发事件时就调用回调函数，将这个fd添加到一个链表中，这样epoll只需要检查链表中有没有fd就知道有没有数据准备好了。

epoll 因为采用 mmap的机制, 使得 内核 socket buffer 和用户空间的 buffer 共享, 从而省去了 socket data copy, 这也意味着, 当 epoll 回调上层的 callback 函数来处理 socket 数据时, 数据已经从内核层 "自动" 到了用户空间。

#### epoll和select区别

select 需要遍历所有注册的I/O事件，找出准备好的的I/O事件。
而 epoll 则是由内核主动通知哪些I/O事件需要处理，不需要用户线程主动去反复查询，因此大大提高了事件处理的效率。



## 5. 信号驱动式I/O

![signal-driven-io](https://github.com/lixd/blog/raw/master/images/linux/io/signal-driven-io.png)

## 6. 异步I/O

![async-io](https://github.com/lixd/blog/raw/master/images/linux/io/async-io.png)



用户进程发起read操作之后，立刻就可以开始去做其它的事。而另一方面，从kernel的角度，当它受到一个asynchronous read之后，首先它会立刻返回，所以不会对用户进程产生任何block。然后，kernel会等待数据准备完成，然后将数据拷贝到用户内存，当这一切都 完成之后，kernel会给用户进程发送一个signal，告诉它read操作完成了。 在这整个过程中，进程完全没有被block。

## 7. 小结

* 1）阻塞式I/O：全程阻塞，只能为每个连接起一个线程单独处理。
* 2）非阻塞I/O：数据未准备好立即返回错误，只能通过循环遍历方式一直检测数据是否准备好
* 3）多路复用I/O
  * select：将 fd 发送给内核，由内核遍历检测数据是否准备好，（由应用进行遍历转为内核进行遍历）将多次系统调用转为一次系统调用
  * epoll：将 fd 存在内存中，省去 fd 传输过程，同时 socket 数据准备好时，由硬件（网卡）发起中断，再次省去系统调用。
* 4）信号驱动式I/O：用得不多
* 5）异步I/O：发起调用后直接返回，调用处理完发送信号通知，异步操作，没有任何阻塞。

其实前四种 I/O 模型都是同步 I/O 操作，他们的区别在于第一阶段，而他们的第二阶段是一样的：在数据从内核复制到应用缓冲区期间（用户空间），进程阻塞于recvfrom 调用。 

可以看到比较优秀的 I/O 模型就是 epoll 了，redis、nginx等等中间件中也是广泛的在使用 epoll。

## 8. 参考

`《UNIX网络编程：卷一》`

`https://www.zhihu.com/question/19732473/answer/241673170`

`http://blog.chinaunix.net/uid-14874549-id-3487338.html`

`http://www.tianshouzhi.com/api/tutorials/netty/221`

