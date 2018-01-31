---
title: I／O复用的模型对比及python实例讲解
date: 2018-01-31 22:38:54
tags: Python

---

## 什么是I/O复用？
关于这个问题的答案，请参考我的另一篇博客[系统级I/O及Linux-I/O模式解释](http://miks.top/2018/01/25/%E7%B3%BB%E7%BB%9F%E7%BA%A7I-O%E5%8F%8ALinux-I-O%E6%A8%A1%E5%BC%8F%E8%A7%A3%E9%87%8A/)。

I/O复用也是系统级并发的一种手段。

## kqueue(), epoll(), select()对比
首先它们能支持的系统并不相同。

比如在Python封装的接口中，epoll()支持Linux2.5+, kqueue()只支持大多数的BSD系统。对于Windows系统，它只支持socket。

select()相比于epoll()：

* select()的句柄受限，而epoll()没有，它的限制是最大的打开文件句柄数目。
* epoll()不会随着fd的数目增长而降低效率，在select()中需要用轮询来找到就绪的fd，所以会随着fd的增加而增加查找耗时。
* epoll()有两个工作模式：ET(Edge Triggered)仅当状态发生变化时才会通知，epoll_wait返回。换句话，就是对于一个事件,只通知一次，且只支持非阻塞的socket。 LT(Level Triggered)类似select/poll,只要还有没有处理的事情就会一直通知，以LT方式调用epoll接口的时候，它就相当于一个速度比较poll。支持阻塞和不阻塞的socket。


epoll()相比于kqueue():

* 性能方面，epoll设计有一个弱点，它不支持在单个系统中设置多个事件，当有100个文件描述符来注册事件时，需要进行一百次的epoll.register().相对于kqueue()来说，可以在单个kevent中注册多个事件。
* 非文件的支持，epoll()的使用范围有限。它旨在提高select()/poll()的性能，但是除此之外，epoll只能用于文件描述符。我们时常说“在Unix中，一切都是文件”，这是真实的，但并非总是如此，例如，定时器不是文件，信号不是文件，进程不是文件，网络设备不是文件，此时不能用select()/poll()来对这些事物进行I/O复用。典型的网络服务器除了套接字外，还管理各种类型的资源。你可能想用一个统一的界面监视它们，但是你不能。为了解决这个问题，Linux支持很多补充的系统调用，比如signalfd()，eventfd()和timerfd_ create(), 它们将非文件资源转换为文件描述符，以便将它们与epoll()复用。而在kqueue()中，通用的kevent支持各种非文件类型描述符。例如，你的应用程序可以在子进程退出时获取通知(filter = EVFILT_ PROC，ident = pid和fflags = NOTE_ EXIT）。即使当前内核版本不支持某些资源或事件类型，这些资源或事件类型也会在未来的内核版本中进行扩展，而不会对API进行任何更改。

## Python演示

  基本思路生成socket，绑定IP端口，listen后使用select或者kqueue管理链接是否可以读写操作。获得链接后，等待客户端输入，将从客户端收到的数据原样返回给客户端。实现基本的回显操作。  
  
#### select()函数：select.select(rlist, wlist, xlist[, timeout])

*  rlist: 等待准备好的可读fds, wlist: 等待准备好可写的fds, xlist: 异常的fds。
*  如果没有设置timeout参数，那么函数会一直阻塞到至少有一个准备就绪的fd。如果timeout设置且为0，则表明不会进行阻塞。
*  返回的值是一个三元列表，里面存放的是已经就绪的fd，但是如果当timeout超时后，会返回空列表。

```
def select_mode():
    server = socket_server.server_start()
    BUFFER_SIZE = 1024
    inputs = [server, sys.stdin]
    running = True
    while running:
        try:
            readable, writeable, exceptional = select.select(inputs, [], [])  # 等待至少一个准备就绪的fd.
        except select.error as e:
            break

        for each in readable:
            # 如果是server，则accept创建链接，并将链接的描述符放入到inputs进行select监听
            if each == server:
                conn, addr = server.accept()
                inputs.append(conn)
            # 如果是系统输入，则获取系统输入，停止程序
            elif each == sys.stdin:
                junk = sys.stdin.readline()
                running = False
            # 如果是其它的，则为socket链接，读取链接数据，并返回读取到的数据。
            else:
                try:
                    data = each.recv(BUFFER_SIZE)
                    if data:
                        each.send(data)
                        print(data)
                        if str(data).endswith('\r\n\r\n'):
                            inputs.remove(each)
                            each.close()
                    else:
                        inputs.remove(each)
                        each.close()
                except socket.error as e:
                    inputs.remove(each)
    server.close()
```



#### epoll()函数： select.epoll(sizehint=-1, flags=0) 

* epoll()支持用with来进行上下文的管理。
* 仅支持Linux2.5.44之后的版本。
* 返回轮询对象，供后续边缘触发模式(ET)或者水平触发模式(LT)使用。
* epoll.close() 关闭epoll控制的fd。
* epoll.closed 当epoll关闭后，为True。
* epoll.fileno() 返回控制的fds数量。
* epoll.fromfd(fd) 从给予的fd，创建一个epoll对象。
* epoll.register(fd[, eventmask]) 给epoll对象注册一个fd。
* epoll.modify(fd, eventmask) 修改epoll对象已经注册的一个fd.
* epoll.unregister(fd) 将epoll对象中已经注册的某个fd移除。
* epoll.poll(timeout=-1, maxevents=-1) 在timeout时间内等待事件。

```
def epoll_mode():
    server = socket_server.server_start()
    running = True
    BUFFER_SIZE = 1024
    epoll = select.epoll()
    epoll.register(server.fileno(), select.EPOLLIN)
    conn_list = {}

    while True:
        try:
            events = epoll.poll(1)
        except:
            break

        if events:
            for fileno, event in events:
                # 如果是新链接，则accept，并将新链接注册到epoll中
                if fileno == server.fileno():
                    conn, addr = server.accept()
                    conn.setblocking(False)
                    epoll.register(conn.fileno(), select.EPOLLIN)
                    conn_list[conn.fileno()] = conn
                # 如果是其他的可读链接，则获取到链接，recv数据，如果是\n， 则close，并从epoll中unregister掉这个文件描述符
                elif event == select.EPOLLIN:
                    data = conn_list[fileno].recv(BUFFER_SIZE)
                    if data.startwith('\n'):
                        conn_list[fileno].close()
                        epoll.unregister(fileno)
                        del conn_list[fileno]
                    else:
                        conn_list[fileno].send(data)

    server.close()
```

#### kqueue()函数： select.kqueue()

* 仅支持BSD系统。
* 返回kernel queue 对象。	 
* kqueue.close(), kqueue.fileno(), kqueue.fromfd(fd), kqueue.closed都跟epoll()的相同，这里不再赘述。
* kqueue.control(changelist, max_events[, timeout=None]) → eventlist
	1. changelist必须是由kevent组成的iterable对象或者是None。
	2. max_events必须为0，或者是一个正数值。
* select.kevent(ident, filter=KQ_FILTER_READ, flags=KQ_EV_ADD, fflags=0, data=0, udata=0) 返回kernel kevent对象。
	* kevent.ident 值用来区别事件
	* kevent.data 过滤特定的数据
	* kevent.udata 用户指定的数据
	
```
def kqueue_mode():
    running = True
    BUFFER_SIZE = 1024
    kq = select.kqueue()
    conn_list = {}
    index = 1
    server = socket_server.server_start()

    events = [
        select.kevent(server.fileno(), select.KQ_FILTER_READ, select.KQ_EV_ADD)
    ]

    while running:
        try:
            # 开始kqueue，如果有可执行kevent，则返回对应的kevent列表
            eventlist = kq.control(events, 1)
        except select.error as e:
            break

        if eventlist:
            for each in eventlist:
                # 如果是socket链接，则accept，将conn创建kevent放入到events进行监听，将链接放入到conn_list进行保存，key为index。
                if each.ident == server.fileno():
                    conn, addr = server.accept()
                    conn_list[index] = conn
                    events.append(
                        select.kevent(conn_list[index].fileno(), select.KQ_FILTER_READ, select.KQ_EV_ADD, udata=index)
                    )
                    index += 1

                else:
                    try:
                        if each.udata >= 1 and each.flags == select.KQ_EV_ADD and each.filter == select.KQ_FILTER_READ:

                            conn = conn_list[each.udata]
                            data = conn.recv(BUFFER_SIZE)
                            if str(data).startswith('\n'):
                                conn.close()
                                events.remove(select.kevent(conn_list[each.udata].fileno(), select.KQ_FILTER_READ,
                                                            select.KQ_EV_ADD, udata=each.udata))
                                del conn_list[each.udata]
                            else:
                                conn.send(data)
                                print(data)

                    except Exception as e:
                        print(e)

    server.close()
```
  


## I/O多路复用技术的优劣
正如我们所知道的，系统级并发的手段除了I/O复用以外，还有进程与线程。那么对I/O对比它们，又有什么优劣呢？

优点：

* 它比基于进程的设计给了程序员更多的对程序行为的控制。例如，我们可以设想编写一个事件驱动的并发服务器，为某些客户端提供它们需要的服务，而对于基于进程的并发服务器来说，是很困难的。
* 基于I/O多路复用的事件驱动服务器是运行在单一进程上下文中的，因此每个逻辑流都能访问该进程的全部地址空间。这使得在流之间共享数据变得很容易。
* 事件驱动设计常常比基于进程的设计要高效的多，因为它不需要进程上下文切换来调度新的流。

缺点：

* 编码复杂，并且随着并发粒度的减小，复杂性会继续上升。
* 不能充分利用多核处理器。


## 参考

[IO模型及select、poll、epoll和kqueue的区别](https://www.cnblogs.com/linganxiong/p/5583415.html) 

[Scalable Event Multiplexing: epoll vs. kqueue](http://people.eecs.berkeley.edu/~sangjin/2012/12/21/epoll-vs-kqueue.html)


## 最后

希望你能带着批判的眼光思考本文，如有纰漏，请不吝赐教。


