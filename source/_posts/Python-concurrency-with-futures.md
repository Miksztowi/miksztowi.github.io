---
title: Python concurrency with futures
date: 2017-12-13 11:42:48
tags: Python

---
## Preface

当你想要高效地处理网络I/O时，你就需要理解「并发」的概念。本章计划列举Fluent Python书中的两个Web下载实例来帮助你理解这个概念。


## Sequential 
```
import os
import time
import sys

import requests  # It's not the standard library, so separate it from top with a blank line.

POP20_CC = ('CN IN US ID BR PK NG BD RU JP'
            'MX PH VN ET EG DE IR TR CD FR').split()

BASE_URL = 'http://flupy.org/data/flags'

DEST_DIR = 'downloads/'


def save_flags(img, filename):
    path = os.path.join(DEST_DIR, filename)
    with open(path, 'wb') as fp:
        fp.write(img)


def get_flag(cc):
    url = '{}/{cc}/{cc}.gif'.format(BASE_URL, cc=cc.lower())
    resp = requests.get(url)
    return resp.content


def show(text):
    print(text, end=' ')
    sys.stdout.flush()  # flush stdout buffter.


def download_many(cc_list):
    for cc in sorted(cc_list):
        image = get_flag(cc)
        show(cc)
        save_flags(image, cc.lower() + '.gif')

    return len(cc_list)


def main(download_many):
    t0 = time.time()
    count = download_many(POP20_CC)
    elapsed = time.time() - t0
    msg = '\n{} flags download in {:.2f}s'
    print(msg.format(count, elapsed))


if __name__ == '__main__':
    main(download_many)

BD BR CD CN DE EG ET FR ID IN IR JPMX NG PH PK RU TR US VN 
19 flags download in 21.66s
```


## Concurrent.futures

主要的Concurrent.futures的包是ThreadPoolExecutor和ProcessPoolExecutor，它们实现的接口允许你使用多个线程或者进程，但是不用去关心底层的细节。

```
from concurrent import futures

from Fluent_Python_demo.Concurrency_demo.sequential_script_demo import save_flags, get_flag, show, main

MAX_WORKERS = 20

def download_one(cc):
    image = get_flag(cc)
    show(cc)
    save_flags(image, cc.lower() + '.gif')
    return cc

def download_many(cc_list):
    workers = min(MAX_WORKERS, len(cc_list))
    with futures.ThreadPoolExecutor(workers) as executor:
        res = executor.map(download_one, sorted(cc_list))

    return len(list(res)) # 如果任意一个线程的发生了异常，由于隐式调用了next()方法得到返回值，所以异常会在这里被抛出。

if __name__ == '__main__':
    main(download_many)
    
TR RU IN JPMX BR VN DE EG FR NG ID CN PK PH CD IR ET US BD 
19 flags download in 1.69s

```
1. 在上下文管理器中，executor的__exit__方法会调用executor.shutdown(wait=True),这个方法会阻塞直到所有的线程结束。
2. 注意download_one()这个函数与上面的代码做个对比，我们可以发现将sequential的代码重构为concurrent时，可以直接将for循环里面的操作拆分成一个单独的函数，来支持后续的并发处理。


看完了上面的代码，我们可能会产生一个困惑：future到底是啥？它背后的逻辑是怎么样的？

* future是本章以及后面介绍的asyncio的内部逻辑，但是作为用户是无法看到内部操作的。
* 它是Future class的实例，表示已经结束或者还未结束的计算的延时。
* Futures是挂起操作，所以它们可以被放进队列中。它们的状态是可以被查询的，并且当挂起操作结束后，它们的结果是可以被保存或者调用的。
* 虽然它看起来很有意义，但是在使用时，应该让并发的框架来完成实例，而不是我们来控制。一个Future就代表一个最终会执行的操作，而唯一能确定某个操作会被执行的方法是这个操作已经被安排到任务中了。因此，Future实例是安排执行操作的Concurrent.futures.Future类创建的。
* 只有并发的框架会在future结束后改变它的状态，但是我们不能控制future何时发生，所以我们写的代码不支持改变future的状态。Future的.done()方法是非阻塞的，并且它会返回一个boolean类型的值来表明当前的future是否结束了。.add_done_callback()，需要我们来设定一个callable参数，当future结束后会将其当作参数传给callbale，并且调用它。.result()方法会返回callable的结果，或者抛出一些设定的异常。但是.result()这个方法在future没有结束前，在concurrency.futures.Future中，调用f.result()的话，将会阻塞线程，直到结果返回。如果我们设定了timeout参数话，结果没有在timeout内返回就会抛出TimeoutError异常。这个方法在asyncio中是非阻塞的，同时也不支持timeout设置，结果是通过yield from来返回的，这点与concurrency.futures.Future是不同的。


接下来我们用concurrency.futures.as_completed()修改上面写过的代码。

```
def download_many(cc_list):
    cc_list = cc_list[:5]
    with futures.ThreadPoolExecutor(max_workers=3) as executor:
        to_do = []
        for cc in sorted(cc_list):
            future = executor.submit(download_one, cc)
            to_do.append(future)
            msg = 'Scheduled for {}: {}'
            print(msg.format(cc, future))

        results = []
        for future in futures.as_completed(to_do):
            res = future.result()
            msg = '{} result: {!r}'
            print(msg.format(future, res))
            results.append(res)

    return len(results)

Scheduled for BR: <Future at 0x1020f2080 state=running>
Scheduled for CN: <Future at 0x1020f27f0 state=running>
Scheduled for ID: <Future at 0x1020f27b8 state=running>
Scheduled for IN: <Future at 0x1020fe320 state=pending>
Scheduled for US: <Future at 0x1020fe3c8 state=pending>
BR <Future at 0x1020f2080 state=finished returned str> result: 'BR'
ID <Future at 0x1020f27b8 state=finished returned str> result: 'ID'
IN <Future at 0x1020fe320 state=finished returned str> result: 'IN'
US <Future at 0x1020fe3c8 state=finished returned str> result: 'US'
CN <Future at 0x1020f27f0 state=finished returned str> result: 'CN'

5 flags download in 3.39s
```

executor.submit(download_one, cc)安排download _one被执行，它会返回future。
注意看只有三个任务是runnning状态，另外的都是pending，这是因为只有三个worker导致的。



##### Experimenting with Executor.map

```
from time import sleep, strftime
from concurrent import futures


def display(*args):
    print(strftime('[%H:%M:%S]'), end=' ')
    print(*args)


def loiter(n):
    msg = '{}loiter({}): doing nothing for {}s...'
    display(msg.format('\t' * n, n, n))
    sleep(n)
    msg = '{}loiter({}): done'
    display(msg.format('\t' * n, n))
    return n * 10


def main():
    display('Script starting.')
    executor = futures.ThreadPoolExecutor(max_workers=3)
    results = executor.map(loiter, range(5))
    display('results:', results)
    display('Wating for individual results:')
    for i, result in enumerate(results):
        display('result {}: {}'.format(i, result))
if __name__ == '__main__':
    main()
[10:08:03] Script starting.
[10:08:03] loiter(0): doing nothing for 0s...
[10:08:03] loiter(0): done
[10:08:03] 	loiter(1): doing nothing for 1s...
[10:08:03] 		loiter(2): doing nothing for 2s...
[10:08:03] results: <generator object Executor.map.<locals>.result_iterator at 0x10d6ef468>
[10:08:03] Wating for individual results:
[10:08:03] result 0: 0
[10:08:03] 			loiter(3): doing nothing for 3s...
[10:08:04] 	loiter(1): done
[10:08:04] 				loiter(4): doing nothing for 4s...
[10:08:04] result 1: 10
[10:08:05] 		loiter(2): done
[10:08:05] result 2: 20
[10:08:06] 			loiter(3): done
[10:08:06] result 3: 30
[10:08:08] 				loiter(4): done
[10:08:08] result 4: 40
注意看打印时间。__iter__方法会阻塞代码，直到future返回结果。
```

推荐你在你的电脑上运行一下这段代码，你很可能会发现尽管代码相同，但是打印顺序是不同的。由于YMMV(You Mileage May Vary):在多线程运行时，你永远不会准确的知道同一时刻内发生的事情。换台机器可能同一时刻内发生的情况就有所不同。

这段代码的阻塞情况与上面是不同的，它返回的结果是跟调用顺序完全相同的。比如你设定第一个任务需要等待10秒，其余任务执行时间为1秒，那么.map()会先阻塞10秒后再执行其余任务。所以如果你的任务需要按照顺序来执行，那么使用.map()是ok的，但是如果你的任务对调用顺序无所谓的话，Executor.submit,futures.as_completed()会更适合。


## Downloads with progress display and error handing

我们先介绍python中一个有趣的库**tqdm**，利用这个库我们可以打印出运行的进度条，和接下来需要的时间。

```
import time
from tqdm import tqdm
    for i in tqdm(range(100)):
        time.sleep(0.1)
100%|██████████| 100/100 [00:10<00:00,  9.70it/s]
```

先看sequentially download

```

def get_flag(base_url, cc):
    url = '{}/{cc}/{cc}.gif'.format(base_url, cc=cc.lower())
    resp = requests.get(url)
    if resp.status_code != 200:
        resp.raise_for_status()
    return resp.content


def download_one(cc, base_url, verbose=False):
    try:
        image = get_flag(base_url, cc)
    except requests.exceptions.HTTPError as exc:
        res = exc.response
        if res.status_code == 404:
            status = HTTPStatus.NOT_FOUND
            msg = 'not found'
        else:
            raise

    else:
        save_flags(image, cc.lower() + '.gif')
        status = HTTPStatus.OK
        msg = 'OK'

    if verbose:
        print(cc, msg)

    return Result(status, cc)


def download_many(cc_list, base_url, verbose, max_req):
    counter = Counter()
    cc_iter = sorted(cc_list)
    if not verbose:
        cc_iter = tqdm(cc_iter)
    for cc in cc_iter:
        try:
            res = download_one(cc, base_url, verbose)
        except requests.exceptions.HTTPError as exc:
            error_msg = 'HTTP error {res.status_code} - {res.reason}'
            error_msg = error_msg.format(res=exc.response)
        except requests.exceptions.ConnectionError as exc:
            error_msg = 'Connection error'
        else:
            error_msg = ''
            status = res.status

    if error_msg:
        status = HTTPStatus.error

    counter[status] += 1
    if verbose and error_msg:
        print('*** Error for {}: {}'.format(cc, error_msg))

    return counter
```


我们用futures.as_completed来做thread error handling。


```
def download_many(cc_list, base_url, verbose, concur_req):
    counter = collections.Counter()
    with futures.ThreadPoolExecutor(max_workers=concur_req) as executor:
        to_do_map = {}
        for cc in sorted(cc_list):
            future = executor.submit(
                download_one,
                cc, base_url, verbose
            )
            to_do_map[future] = cc

        done_iter = futures.as_completed(to_do_map)

        if not verbose:
            done_iter = tqdm.tqdm(done_iter, total=len(cc_list))

        for future in done_iter:
            try:
                res = future.result()
            except requests.exceptions.HTTPError as exc:
                error_msg = 'HTTP error {res.status_code} - {res.reason}'
                error_msg = error_msg.format(res=exc.response)
            except requests.exceptions.ConnectionError as exc:
                error_msg = 'Connection error'
            else:
                error_msg = ''
                status = res.status

            if error_msg:
                status = HTTPStatus.error

            counter[status] += 1

            if verbose and error_msg:
                cc = to_do_map[future]
                print('*** Error for {}: {}'.format(cc, error_msg))

    return counter
```


## Blocking I/O and the GIL


尽管上面concurrent版本相比sequential来说运行速度快了近13倍，但严格来说，concurrent也并非是**并行的**。这是因为CPython解释器中的GIL模式导致的。

**为什么要有GIL呢？**因为CPython内部是线程不安全的，所以用Global Interpreter Lock来保证运行时只有一个线程在运行python的字节码，同时GIL也让python进程**通常**无法同时使用CPU的多核。

当我们写Python代码时，我们是无法控制GIL的，但是内置的函数或者由C扩展的函数可以在运行多任务时释放GIL，事实上Python里由C语言写的库是可以管理GIL的，启动它自己的系统线程并且控制所有可用的CPU核。不过这大大增加了库的复杂程度，所以大部分作者并不尝试这么做。

然而，所有的标准库在等待从OS返回阻塞的I/O结果时，**都可以释放GIL
！**这就意味着I/O限制的Python程序是可以在Python层面利用多线程的。当一个Python线程等待网络回应时，阻塞的I/O函数会释放GIL，这样另一个线程就可以运行了。*Tips: time.sleep()函数也会释放GIL.* 


ProcessPoolExecutor和ThreadPoolExecutor的接口大致相同。需要注意的是后者可以设定规模较大的Worker数量，而前者默认是由os.cpu_count()来设定的。


## 最后
感谢阅读。

本文代码地址：[https://github.com/Miksztowi/CorePython/tree/master/Fluent_Python_demo/Concurrency_demo](https://github.com/Miksztowi/CorePython/tree/master/Fluent_Python_demo/Concurrency_demo)

