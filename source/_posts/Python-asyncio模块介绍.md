---
title: Python asyncio模块介绍
date: 2017-12-19 23:00:51
tags: Python

---


## Preface

本文介绍asyncio模块，这个模块使用事件轮询来驱动的coroutine对并发任务进行操作。

注意：本文的思路来自于Fluent Python，这也是我对书中asyncio章节的总结。

## A comparison between a simple threaded program and the asyncio equivalent

我们接下来做一个在线程中的循环打印出类似gif动画效果的demo,并且用一个time.sleep(3)的函数来模拟阻塞。

Thread demo:

```
class Signal:
    go = True


def spin(msg, signal):
    write, flush = sys.stdout.write, sys.stdout.flush
    for char in itertools.cycle('|/-\\'):
        status = char + ' ' + msg
        write(status)
        flush()
        write('\x08' * len(status))
        time.sleep(.1)
        if not signal.go:
            break
    write(' ' * len(status) + '\x08' * len(status))


def slow_function():
    time.sleep(3)
    return 42


def supervisor():
    signal = Signal()
    spinner = threading.Thread(target=spin,
                               args=('thinkg!', signal))
    print('spinner object:', spinner)
    spinner.start()
    result = slow_function()
    signal.go = False
    spinner.join()
    return result

def main():
    result = supervisor()
    print('Answer:', result)
因为用sys.stdout.write和sys.stdout.flush，所以输出结果类似于gif动画。为了观察demo的运行结果，最好尝试在电脑上运行一下这个demo。
```
注意itertools.cycle是无限的循环迭代，所以我们用一个signal来控制循环的结束。


### 关于asyncio版本的代码做几点解释：
1.  Coroutines要被asyncio模块使用的话，应该要使用asyncio.coroutine装饰器。这不是强制的，但是为了突出coroutine不同与一般函数，并且在被垃圾回收时，如果没有调用yield from那么在debug时会被当作bug抛出来，所以推荐使用asyncio.coroutine装饰器.
2. 使用yield from asyncio.sleep(.1) 替换了 time.sleep(.1)。即使后者可以在I/O操作时释放GIL，但是Coroutine的操作时在单线程中进行的。所以必须要使用前者，否则当前线程会被阻塞。
3. 当yield from asyncio.sleep()挂起当前操作时，程序的控制流会返回到主循环内，直到当前休眠时间结束，才会重新被唤醒。
4. aysncio.async(...)安排spin coroutine去运行，将其封装成Task对象后立即返回。
5. Task对象可以被结束，asyncio.CancelledError异常会在程序被挂起的地方抛出，但是异常被coroutine捕获后可能会被延迟结束或者拒绝结束。
6. run_ until_ complete()会驱动coroutine运行，返回的结果是coroutine的返回值。

```
@asyncio.coroutine
def spin(msg):
    write, flush = sys.stdout.write, sys.stdout.flush
    for char in itertools.cycle('|/-\\'):
        status = char + ' ' + msg
        write(status)
        flush()
        write('\x08' * len(status))
        try:
            yield from asyncio.sleep(.1)
        except asyncio.CancelledError:
            break
    write(' ' * len(status) + '\x08' * len(status))

@asyncio.coroutine
def slow_function():
    yield from asyncio.sleep(3)
    return 42

@asyncio.coroutine
def supervisor():
    spinner = asyncio.ensure_future(spin('thinking!'))
    print('spinner object:', spinner)
    result = yield from slow_function()
    spinner.cancel()
    return result


def main():
    loop = asyncio.get_event_loop()
    result = loop.run_until_complete(supervisor())
    loop.close()
    print('Answer:', result)
```


对比上面两个版本的demo,解释一下它们之间的区别：

1. 一个Task驱动一个coroutine,一个线程调用一个callable.
2. 我们不能实例一个Task对象，只能通过传递coroutine给asyncio.async(...)或者loop.create_ task(...)来获得Task对象。
3. 当你得到一个Task对象时，说明当前任务已经被安排运行了；一个线程实例的运行，必须要显示的调用.start()方法。
4. 在Thread demo中，slow_function是一个普通的函数，直接被当前线程调用；在asyncio中，slow_funciton是一个coroutine，被yield from驱动。
5. 没有从线程外部终结线程的API,因为强制的终结会让该线程系统处于一个不清晰的状态。对于Task来说，Task.cancle()的实例方法，可以在coroutine中抛出cancelledError，这个异常会在coroutine的yield语句处被捕捉。
6. supervisor必须在main函数中通过loop.run_ until_ complete()调用。
7. 在threads项目时，由于一个线程可能会在任一时刻被杀死，从而导致数据／系统状态变的不清晰。所以必须记得加锁去保护程序中重要的操作，避免在多个操作步骤中突然被终止的弊端；在coroutine中，避免意外的终止带来的弊端，所有的操作都被保护了。由于一个coroutine只有在被yield语句挂起时终止，所以我们可以在处理CancelledError异常时清理掉「不干净」的操作。

## How the asyncio.Future class differs from concurrent.futures.Future
如果你还不清楚future的概念，你可以参考我写的另一篇文章[Python concurrency with futures](http://www.ganbinwen.com/2017/12/13/Python-concurrency-with-futures/)，里面有相关介绍，或者可以自行参考官方文档/wiki。

请务必确认已经清楚了future的概念后再阅读下面内容 :)

asyncio中的future：

* asyncio中Task是Future的子类,故Task也是Future。
* 对future用yield from时，不会阻塞事件循环，因为在asyncio中yield from会将控制流返回到事件循环内。
* 对future进行yield from就相当于add _done _callback()函数，当延迟操作结束后，事件循环会设置future的结果，然后yield from表达式会在被挂起的coroutine内部产生一个返回值，并且重新唤起它。
* yield from my_ future 相当于 my_ future.add_ done_ callback()函数， result = yield from my_future,相当于my _future.result()函数，可见yield from在asyncio中是非常重要的。


asyncio中我们可以通过yield from得到asyncio.Future中的结果，这意味着res = yield from foo()中foo是一个coroutine函数的话可以返回一个coroutine，如果foo只是一个普通函数，那么返回一个Future或者Task实例。
为了执行操作，一个coroutine必须要被安排，然后将其封装成aysncio.Task对象，有两种方式可以得到一个Task对象：

1. aysncio.async(coro_or_future, *, loop=None)

	*  这个函数的第一个参数可以是coroutines或者futures。如果是Future/Task那么不做修改的返回。如果是一个coroutine，aysnc通过loop.create_ task()来创建一个Task对象返回。一个可选的关键字参数是loop,如果忽略这个参数，async会通过asyncio.get_ event_ loop()来的到一个loop对象。

	
2. BaseEventLoop.create_task(coro)

	* 这个方法安排coroutine操作，并且返回一个asyncio.Task对象。
	
几个不同的asyncio函数接受coroutines并将其封装为asyncio.Task对象。在内部使用asyncio.async自动完成封装,如接下来的demo中run_ until_complete()函数

```
import asyncio


def run_sync(coro_or_future):
    loop = asyncio.get_event_loop()
    return loop.run_until_complete(coro_or_future)


@asyncio.coroutine
def coro():
    yield from asyncio.sleep(1)


if __name__ == '__main__':
    a = run_sync(coro())
```
 	 

## Asynchronous programming manages high concurrency in network applications, without using threads or processes

我们使用aiohttp这个第三方库来完善我们的Http Client。

```

import asyncio
import aiohttp
import time
import os
import sys

POP20_CC = ('CN IN US ID BR PK NG BD RU JP'
            'MX PH VN ET EG DE IR TR CD FR').split()
BASE_URL = 'http://flupy.org/data/flags'

DEST_DIR = 'downloads/'


def save_flags(img, filename):
    path = os.path.join(DEST_DIR, filename)
    with open(path, 'wb') as fp:
        fp.write(img)


def show(text):
    print(text, end=' ')
    sys.stdout.flush()


@asyncio.coroutine
def get_flag(cc):
    url = '{}/{cc}/{cc}.gif'.format(BASE_URL, cc=cc.lower())
    session = aiohttp.ClientSession()
    res = yield from session.get(url)
    # res = yield from aiohttp.request('GET', url) will occur error with 'Unclosed client session'.
    image = yield from res.read()
    session.close()
    return image


@asyncio.coroutine
def download_one(cc):
    image = yield from get_flag(cc)
    show(cc)
    save_flags(image, cc.lower() + '.gif')
    return cc


# @asyncio.coroutine
def download_many(cc_list):
    loop = asyncio.get_event_loop()
    to_do = [download_one(cc) for cc in sorted(cc_list)]
    wait_coro = asyncio.wait(to_do)
    res, _ = loop.run_until_complete(wait_coro)
    loop.close()

    return len(res)


def main(download_many):
    t0 = time.time()
    count = download_many(POP20_CC)
    elapsed = time.time() - t0
    msg = '\n{} flags download in {:.2f}s'
    print(msg.format(count, elapsed))
CN EG FR BR JPMX NG RU BD VN IN ID PK TR ET IR US CD PH DE 
19 flags download in 3.85s
```
asyncio.wait()不是一个阻塞函数，它会等到所有传递给它的coroutines操作结束。它接受coroutines/futures的iterable，将每个coroutine封装成Task实例
当事件循环运行时，loop.run_ until_ complete(wait_ coro)会阻塞。这个函数会返回2-tuple，tuple[0]是完成的集合，tuple[1]是未完成的集合。未完成的值可以通过在.wait()函数里传timeout关键字参数或者return_when关键字参数得到。

这个demo里包含了很多新的概念，但是简单的驱动流程就是，在get_ flag里用yield from驱动aiohttp所以get_ flag是一个coroutine，同样在download_ one里用yield from驱动get_ flag所以它也是一个coroutine，最后在download_ many通过loop.run_ until_ complete来驱动被asyncio.wait()方法实例过的Tasks。

可能明白了上述的驱动流程，但是对何时使用yield from还有困惑，那么我们再总结一下，yield from的使用：

1. 每一个用yield from排列好的coroutine链，最后调用的caller自身一定不能是coroutine。caller需要用next()/.send()来调用最外层的delegating generator.（隐式的for循环也可以）.
2. 链中最内层的subgenerator一定要是一个简单的generator，它只能使用yield或者它是一个iterable对象。

当在asyncio中使用yield from时不仅要注意以上两点，还有一些需要注意：
驱动最外层的delegating generator我们一般使用asyncio的API来完成，比如loop.run_ until_ complete(...) ；最里面的subgenerator通常yield from一些asyncio coroutine函数或者coroutine函数，总之就是最内部的subgenerator通常会是一些第三方的库函数来进行I/O操作。


## How coroutines are a major improvement over callbacks for asynchronous programming

通过上面的demo及介绍，细心的你有没有发现所有的操作都是在同一个线程中完成的？如果都在同一个线程里完成操作，那么如何提高5x的运行速度？

对于I/O操作有两种方法可以避免阻塞的方法去停止程序的整个进程：
1. 让阻塞操作在不同的线程中运行。
2. 让用非阻塞的异步操作来调用阻塞操作。

我们知道多线程的方式是有效的，但是每个线程都要消耗系统内存，当需要处理的线程数量级非常大时，我们承担不起每个线程连接需要耗费的资源。

使用coroutines时，generator提供了可选的方式去进行异步编程，用事件循环来回调或者使用.send()操作挂起的coroutine即可。每个挂起的coroutine都需要消耗内存资源，但开销远小于线程的开销。

```
```


## How to avoid blocking the event loop by offloading blocking operations to a thread pool
尽管我们已经可以处理网络I/O的延迟了，但是对于文件I/O操作的延迟，要如何处理呢？
在asyncio中，有一个thread pool executor，我们可以将callable通过run _in _ executor()传递。

```
@asyncio.coroutine
def download_one(cc, base_url, semaphore, verbose):
    try:
        with (yield from semaphore):
            image = yield from get_flag(base_url, cc)
    except web.HTTPNotFound:
        status = HTTPStatus.NOT_FOUND
        msg = 'not found'
    except Exception as exc:
        raise FetchError(cc) from exc
    else:
        loop = asyncio.get_event_loop()
        loop.run_in_executor(None, save_flags, image, cc.lower() + '.gif') # Avoiding blocking.
        status = HTTPStatus.OK
        msg = 'OK'

    if verbose and msg:
        print(cc, msg)

    return Result(status, cc)
```


## Callback Hell
关于这个话题，我有自己的亲身体验。之前写FireStone的胎压信息的爬虫时，需要连续构造异步请求，才能从车的厂商信息获取到每台特定型号车辆的胎压信息。类似的逻辑如下所示：

```
def stage1(response1):
    request2 = step1(response1)
    api_call2(request2, stage2)

def stage2(response2):
    request3 = step2(response2)
    api_call3(request3, stage3)

def stage3(response3):
    step(response3)

api_call(request1, stage1)
```

分析一下这种风格的代码：
读起来麻烦，写起来更麻烦。每个函数都在执行整个任务的一部分，同时为下一次callback做配置及调用。这意味着下一次callback时，本地的上下文就已经丢失了。比如当stage2运行时，request2这个变量已经不存在了。如果需要保存这个变量，那必须要依赖闭包操作或者是用外部的数据结构来保存。
接下来我们再用coroutine来完成上述逻辑。

```
@asyncio.coroutine
def three_stages(request1):
    response1 = yield from api_call1(request1)

    request2 = step1(response1)
    response2 = yield from api_call2(request2)
    
    request3 = step2(response2)
    response3 = yield from api_call3(request3)
    
    step3(request3)
```
不需要callbacks，也可以异步的完成三个连续操作。普通的callbacks对于异常的处理，需要在每个函数里写好两种处理情况：操作正常的返回和操作不正常的返回。这样大大加大了函数的繁琐程度。而coroutine版本的只需要在同一个函数里用try/except来处理就可以了。

尽管coroutine对比普通的版本，避免了callback hell，但是一旦使用coroutine,意味着所有的函数都需要发生改动。要让整个调用流程必须符合coroutine chain的规则，如最内部的函数需要被安排进任务里，最外部的需要用loop.create_ task(three_ stages(request1))来调用。



## Writing asyncio servers, and how to rethink Web applications for high concurrency 
接下来我们一起创造一个TCP Server，这个server允许client用名称单词来查询Unicode自字符。

```
import sys
import asyncio

from Fluent_Python_demo.Asyncio_demo.charfinder import UnicodeNameIndex

CRLF = b'\r\n'
PROMPT = b'?>'
index = UnicodeNameIndex()

@asyncio.coroutine
def handle_queries(reader, writer):
    while True:
        writer.write(PROMPT)
        yield from writer.drain()
        data = yield from reader.readline()
        try:
            query = data.decode().strip()
        except UnicodeDecodeError:
            query = '\x00'
        client = writer.get_extra_info('peername')
        print('Received from {}: {!r}'.format(client, query))
        if query:
            if ord(query[:1]) < 32:
                break
            lines = list(index.find_description_strs(query))
            if lines:
                writer.writelines(line.encode() + CRLF for line in lines)
            writer.write(index.status(query, len(lines)).encode() + CRLF)

            yield from writer.drain()
            print('Sent {} results'.format(len(lines)))
    print('Close the client socket')
    writer.close()

def main(address='127.0.0.1', port=2323):
    port = int(port)
    loop = asyncio.get_event_loop()
    server_coro = asyncio.start_server(handle_queries, address, port, loop=loop)
    server = loop.run_until_complete(server_coro)

    host = server.sockets[0].getsockname()
    print('Serving on {} Hit CTRL-C to stop.'.format(host))
    try:
        loop.run_forever()
    except KeyboardInterrupt:
        pass

    print('Server shutting down.')
    server.close()
    loop.run_until_complete(server.wait_closed())
    loop.close()


if __name__ == '__main__':
    main(*sys.argv[1:])
```
希望你尝试运行上面的代码后再继续阅读，用loop.run_ forever()来阻塞代码，控制流在事件循环中，一直停留在这。直到需要处理handle_ queries coroutine时，将控制流跳到yield中等待client发送数据。当事件循环存在的时候，每个client连接到server时都会实例一个新的handle_ queries，这样简单的server也可以并发处理多个client了。

Tips: 在上面的代码中，我们使用了高级的asyncio Streams API，它已经准备好应用在server中了，所以我们只需要去写好handler函数即可(普通的函数 或者 coroutine).


接下来再看一下http版本的：

```
@asyncio.coroutine
def init(loop, address, port):
    app = web.Application(loop=loop)
    app.router.add_route('GET', '/', home)
    handler = app.make_handler()
    server = yield from loop.create_server(handler, address, port)

    return server.sockets[0].getsockname()


def main(address='127.0.0.1', port=8888):
    port = int(port)
    loop = asyncio.get_event_loop()
    host = loop.run_until_complete(init(loop, address, port))
    print('Servering on {}. Hit CTRL-C to stop.'.format(host))
    try:
        loop.run_forever()
    except KeyboardInterrupt:
        pass
    print('Server shutting down.')
    loop.close()


def home(request):
    query = request.GET.get('query', '').strip()
    print('Query: {!r}'.format(query))
    if query:
        descriptions = list(index.find_descriptions(query))
        res = '\n'.join(ROW_TPL.format(**vars(descr))
                        for descr in descriptions)
        msg = index.status(query, len(descriptions))

    else:
        descriptions = []
        res = ''
        msg = 'Enter words describing characters.'

    html = template.format(query, res, msg)

    print('Sending {} results'.format(len(descriptions)))
    return web.Response(content_type=CONTENT_TYPE, text=html)
```

对于server的启动以及handler大致都与上面TCP版本的相同，我们这里从home()函数开始介绍。仔细观察的你，能发现home()是一个普通的函数，这意味着，从request的处理到可能会进行的database查询都是阻塞的，如果database查询数量很大的话，通常是秒级以上的，我们不能容忍这样的情况。不过有过web开发的小伙伴通常都会想到AJAX技术，这确实是一种好的处理方法。我们可以将查询限制在200rows,在前端界面用滚动条来控制这个间隔。


## Parallelism and Concurrency
给大家分享《Fluent Python》中的一则quote.

*Concurrency is about dealing with lots of things at once.*

*Parallelism is about doing lots of things at once.*

*Not the same, but related.*

*One is about structure, one is about execution.*

*Concurrency provides a way to structure a solution to solve a problem that may(but not necessarily)be parallelizable.*  

									-----Rob Pike
									
这段quote揭示了中文里的**并行**和**并发**的区别。对于真实的并行来说，你的CPU必须要有多核。笔记本大多是4核的，可实际上运行的进程数量远高于这个数量级，所以大多数的进程运行是并发而不是并行的。计算机即使同时处理100+进程，也可以保证每个进程都与机会去处理其中的任务。

对于这组概念，网络上有很多教材／wiki／视频，讲解的都远比我更详细和精准。如果你对它们很陌生或者心中仍存有困惑，请自行查阅资料:).

## Summary
最后，感谢你的阅读。如果你认为文章中有描述不清楚，或者错误的地方，请务必告诉我。 期待你的feedback :)

本文代码地址：[asycnio_demo](https://github.com/Miksztowi/CorePython/tree/master/Fluent_Python_demo/Asyncio_demo)

