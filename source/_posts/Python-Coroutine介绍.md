---
title: Python Coroutine介绍
date: 2017-12-11 14:58:15
tags: Python

---


## Concept
通过对generator的了解，我们知道caller通过next()方法调用函数里的yield item来生成一个值。

在coroutine里

1. yield item通常放在=的右边，比如datum = yield item，item可以为None.
2. 对于coroutine的调用，caller通过.send(datum)而不是next()，意味着caller把值传递给coroutine。
3. yield 有挂起的作用，而正是这一点可以使的coroutine可以控制程序流。
4. 增加了.throw()和.close()方法来扩展caller对coroutine的控制。


## The behavior and states of a generator operating as a coroutine.

我们先看一个基础的coroutine demo。

```
def simple_generator():
    print(' coroutine start.')
    x = yield
    print(' coroutine received ', x)

my_coro = simple_generator()
print(my_coro)
<generator object simple_generator at 0x1093572b0>
next(my_coro)
my_coro.send(42)

注意这里传递值时使用了.send()
```
看完代码，思考一下为什么要先next()，为什么要用.send()而不是继续使用next呢？接下来理解一些coroutine的概念，来帮助我们找出问题的答案。

coroutine只有四种状态：

1. 'GEN_CREATED' 此时等待开始操作。
2. 'GEN_RUINNING' 当前正在被解释器执行。
3. 'GEN_SUSPENDED' 当前被一条yield语句挂起。
4. 'GEN_CLOSED' coroutine已经全部执行完毕。

通过状态的定义，上面问题的答案是： 当coroutine处于挂起状态时，通过.send()来给挂起coroutine的那条yield语句传递值，而这个操作在'GEN_CREATED'状态的coroutine里是无法操作的。
`TypeError: can't send non-None value to a just-started generator`, 此时需要先通过next()来唤醒。

接下来我们打印出coroutine的状态来验证答案是否如我们解释的一样。

```
from inspect import getgeneratorstate
def simple_coro2(a):
    print('-> started: a =', a)
    b = yield a
    print('-> received: b =', b)
    c = yield a + b
    print('-> received: c = ', c)

if __name__ == '__main__':
    my_coro2 = simple_coro2(10)
    print(getgeneratorstate(my_coro2))
    next(my_coro2)
    print(getgeneratorstate(my_coro2))
    my_coro2.send(20)
    print(getgeneratorstate(my_coro2))
    my_coro2.send(30)
    
GEN_CREATED
-> started: a = 10
GEN_SUSPENDED
-> received: b = 20
GEN_SUSPENDED
-> received: c =  30
Traceback (most recent call last):
StopIteration

通过打印，看到代码运行的流程跟我们的答案符合。
```

顺便提起一个python赋值的概念，=右边的语句执行是在赋值操作之前的，
所以’b = yield a‘中b的只有caller唤醒后才会赋值。


接下来，我们来看一下如何终止一个coroutine.

```
def averager():
    total = count = 0
    average = None
    while True:  # 注意看这里，这意味着这个循环会一直进行下去。
        term = yield average
        total += term
        count += 1
        average = total / count
coro_avg = averager()
next(coro_avg)
print(coro_avg.send(10))
print(coro_avg.send(11))
print(coro_avg.send(12))
coro_avg.close()
10.0
10.5
11.0
调用.close()后，将其终止。
print(coro_avg.send(14))
Traceback (most recent call last):
    print(coro_avg.send(14))
StopIteration
调用已经终止的coroutine会抛出Stopiteration的异常。
```

## Priming a coroutine automaticall with a decorator.
通过上面的学习，我们知道了，如果我们不启动coroutine，后续的操作是无法进行的。这样的mechanism对于'慵懒'的我们，是拒绝的！！！ok,那就使用decorator来解决这个烦恼吧！

```
from functools import wraps
from inspect import getgeneratorstate

def coroutine(func):
    @wraps(func)
    def primer(*args, **kwargs):
        gen = func(*args, **kwargs)
        next(gen)
        return gen
    return primer

@coroutine
def simple_generator():
    print(' coroutine start.')
    x = yield
    print(' coroutine received ', x)

if __name__ == '__main__':
    my_coro = simple_generator()
    print(getgeneratorstate(my_coro))
    print(my_coro.send(10))
coroutine start.
GEN_SUSPENDED # 已经是挂起状态了!
coroutine received  10
```
我们通过decorator实现了一个小小的trick，不过很遗憾，这个trick与接下来学习的yield from是冲突的！

## How the caller can control a coroutine through .close() and .throw() methods of the generator object.
对于每一种机制的学习，缺少了终止／异常处理都是不完整的。当然coroutine也不会例外，如果上面的avg内部发生了异常会怎么样呢？

```
coro_avg = averager()
next(coro_avg)
print(coro_avg.send(10))
print(coro_avg.send(11))
print(coro_avg.send('a'))
10.0
10.5
TypeError: unsupported operand type(s) for +=: 'int' and 'str'
coro_avg.send(12)
StopIteration    ## 后续的调用会抛出Stopiteration的异常。
```
当avg出错后，任何尝试重新激活它的操作都会抛出StopIteration。
所以，当我们想终止一个coroutine时，我们可以.send()一些bad数据，或者直接.send(Stopiteration)

当然，generator对象有两种方法可以发送异常。

1. generator.throw(exc_type[, exc_value[, traceback]])如果传递的异常被内部处理了，那么仍然允许后续的调用。没有处理的话，异常会传递给caller。
2. generator.close()当接受到GeneratorExit的异常时，generator不能yield值，并且抛出RuntimeError的异常。但是如果generator抛出了其他的异常，那么会传递给caller。

介绍完这个两个概念后，我们用一个例子来更详细的解释一下这些概念。

```
from inspect import getgeneratorstate
class DemoException(Exception):
    """"An exception type for the  demonstration."""

def demo_exc_handling():
    print('-> coroutine started')
    while True:
        try:
            x = yield
        except DemoException:
            print('*** DemoException handled. Continuing...')
        else:
            print('-> coroutine received: {!r}'.format(x))

    raise RuntimeError('This line should never run.')
exc_coro = demo_exc_handling()
next(exc_coro)

	# Demo 1
	print(exc_coro.send(11))
	print(exc_coro.send(22))
	exc_coro.close()
	print(getgeneratorstate(exc_coro))

-> coroutine started
-> coroutine received: 11
-> coroutine received: 22
GEN_CLOSED

    # Demo 2
    exc_coro.send(11)
    exc_coro.send(22)
    exc_coro.throw(DemoException)
    print(getgeneratorstate(exc_coro))
    
-> coroutine started
-> coroutine received: 11
-> coroutine received: 22
*** DemoException handled. Continuing...
GEN_SUSPENDED

    # Demo 3
    exc_coro.send(11)
    exc_coro.send(22)
    exc_coro.throw(ZeroDivisionError)
    print(getgeneratorstate(exc_coro))

-> coroutine started
-> coroutine received: 11
-> coroutine received: 22
'GEN_CLOSED'
```

注意看上面三种异常的打印，以及异常发生后的generator的状态。

如果我们有：不论generator的结束状态如何，环境的清除代码都必须执行的话。那么不妨尝试用try/finally来解决。

```

def demo_exc_handling():
    print('-> coroutine started')
    try:
        while True:
            try:
                x = yield
            except DemoException:
                print('*** DemoException handled. Continuing...')
            else:
                print('-> coroutine received: {!r}'.format(x))
    finally:
        print('Coroutine ending...')
    raise RuntimeError('This line should never run.')
    
    exc_coro.send(11)
    exc_coro.send(22)
    exc_coro.throw(DemoException)
    print(getgeneratorstate(exc_coro))
    
-> coroutine started
-> coroutine received: 11
-> coroutine received: 22
*** DemoException handled. Continuing...
GEN_SUSPENDED
Coroutine ending...
```

OK，虽然try/finally可以解决我们的需求，但是python的yield from可以更好的解决这个问题。
## How coroutines can return value upon termination.

当coroutine终止时，是如何返回结果的呢？我们修改一下上面的averager的例子。

```
from collections import namedtuple

Result = namedtuple('result', 'count average')
def averager():
    total = count = 0
    average = None
    while True:
        term = yield
        if term is None:
            break
        total += term
        count += 1
        average = total / count
    return Result(count, average)
    
    coro_avg = averager()
    next(coro_avg)
    coro_avg.send(10)
    coro_avg.send(11)
    print(coro_avg.send(None))
    
Traceback (most recent call last):
StopIteration: result(count=2, average=10.5)
注意这里的StopIteration.它将Result也一起打印了出来。

    try:
        coro_avg.send(None)
    except StopIteration as exec:
        print(exec.value)
result(count=2, average=10.5)
借助try/except我们可以的到正确的result.
```

后续的操作就是yield from和for循环在内部的处理，这样的mechanism使得StopIteration的值是value而不是它本身。

## Usage and semantics of the new yield from syntax.

前面铺垫了那么久的yield from，现在终于轮到正式介绍它了。

**yield from**是python里全新的constrcut，在generator里调用yield from subgen()时，subgen会控制程序流，将值直接返回给caller而不是调用它的generator.这就意味着，此时generator是被阻塞的，只有当subgen结束时，才会恢复阻塞。


**yield from x**这个表达式，是x先调用它的iter(x)函数去得到iterator。这意味着x可以是任何iterable对象。但是只通过这个角度去理解，是不足以说明yield from的神奇之处的。它最主要的功能是“委托给一个subgenerator”，它在最外层的caller和最内部的subgenerator之间开启了一个通道，所以值可以直接在通道之间进行传递，同理异常代码的处理也可以直接进行，而无需大量的try/except来完成。

**delegating generator**是指函数里包含了yield from <iterable>表达式。

**subgenerator**指yield from <iterable>表达式里<iterable>得到的generator。

**caller**指调用delagating generator的函数。

```
from collections import namedtuple

result = namedtuple('result', 'count average')


# subgenerator
def averager():
    total = count = 0
    average = None
    while True:
        term = yield
        if term is None:
            break
        total += term
        count += 1
        average = total / count
    return result(count, average)


# delegating generator
def grouper(results, key):
    while True:
        results[key] = yield from averager()

# caller
def main(data):
    results = {}
    for key, values in data.items():
        group = grouper(results, key)
        next(group)
        for value in values:
            group.send(value)
        group.send(None)  ## important

    report(results)


def report(results):
    for key, values in sorted(results.items()):
        group, unit = key.split(';')
        print('{:2} {:5} averaging {:.2f}'.format(
            values.count, group, values.average, unit
        ))
        
data = {
    'girls;kg':
        [40.9, 38.5, 44.3, 42.2, 45.2, 41.7, 44.5, 38.0, 40.6, 44.5],
    'girls;m':
        [1.6, 1.51, 1.4, 1.3, 1.41, 1.39, 1.33, 1.46, 1.45, 1.43],
    'boys;kg':
        [39.0, 40.8, 43.2, 40.8, 43.1, 38.6, 41.4, 40.6, 36.3],
    'boys;m':
        [1.38, 1.5, 1.32, 1.25, 1.37, 1.48, 1.25, 1.49, 1.46],
}
main(data)

 9 boys  averaging 40.42
 9 boys  averaging 1.39
10 girls averaging 42.04
10 girls averaging 1.43
```
理解一下上面的代码：
* 每一次迭代，调用grouper()创建的实例就是delegating generator.
* 调用next(group)启动delegating generator，之后进入循环中，在yield from处挂起。
* group.send(value)是直接调用subgenerator averager的，值是直接传递过去的，因此此时的results[key]还没有被赋值。
* 当赋值循环结束后，如果没有group.send(None),subgenerator永远不会结束，delegating generator就不再会被激活，那么results[key]的赋值也不会成功。
* 当执行一次内循环后，重新创建一个group实例，之前的实例也被垃圾回收机制给回收了，就算内部的subgenerator没有执行完，也同样会被回收。

上面代码的**核心**概念是：如果subgenerator没有结束，那么delegating generator就会永远在yield from处挂起。

如果还没有完全理解yield from,我们再从**PEP 380**中给出的介绍来理解：

1. 任何subgenerator中yield的值都会直接传递给caller.
2. 任何通过delegating generator的send()方法发送的值都会直接传递给subgenerator。如果传递的值是None, subgenerator的__next__()方法会被调用，如果抛出了StopIteration，那么delegating generator就会恢复。除此之外，抛出的其他类型异常都会被传递给delegating generator.
3. return expr在generator里会抛出StopIteration，并且退出这个generator.
4. yield from表达式返回的值，是subgeneration终止时抛出的StopIteration中的第一个参数。
5. 通过subgenerator的throw()方法传递异常，如果异常是StopIteration则delegating generator会恢复，否则其他异常会传递给delegating generator.
6. 如果GeneratorExit异常被扔进了delegating generator里，或者是delegating generator的close()方法被调用。那么当subgenerator也存在close()方法时也会被调用，调用产生的异常会传递给delegating generator.否则GeneratorExit会在delegating generator中被抛出。

为了更好的理解这些抽象的概念，我们用两个伪代码来看一下yield from内部的处理机制

```
RESULT = yield from EXPR

_i = iter(EXPR)
try:
    _y = next(_i)
except StopIteration as _e:
    _r = _e.value
else:
    while 1:
        _s = yield _y
        try:
            _y = _i.send(_s)
        except StopIteration as _e:
            _r = _e.value
            break

RESULT = _r

_i是subgenerator.
_y是subgenerator里yield出来的值.
_r是最终结果.
_s是caller传送给delegating generator的值，跳转给subgenerator.
_e是一个异常.

如果你已经理解了上面的代码，那么我们再来看yield from对于异常的处理。
import sys

RESULT = yield from EXPR
_i = iter(EXPR)
try:
    _y = next(_i)
except StopIteration as _e:
    _r = _e.value
else:
    while 1:
        try:
            _s = yield _y
        except GeneratorExit as _e:
            try:
                _m = _i.close
            except AttributeError:
                pass
            else:
                _m()
            raise _e
        except BaseException as _e:
            _x = sys.exc_info()
            try:
                _m = _i.throw
            except AttributeError:
                raise _e
            else:
                try:
                    _y = _m(*_x)
                except Exception as _e:
                    _r = _e.value
                    break
    else:
        try:
            if _s is None:
                _y = next(_i)
            else:
                _y = _i.send(_s)
        except StopIteration as _e:
            _r = _e.value
            break
RESULT = _r

```

## A use case: coroutines for managing concurrent activities in a simulation.

最后我们通过一个写并发概念的简单demo来梳理一下概念

```
from collections import namedtuple

Event = namedtuple('Event', 'time proc action')


def taxi_process(ident, trips, start_time=0):
    """Yield to simular issuing event at each state change."""
    time = yield Event(start_time, ident, 'leave garge.')
    for i in range(trips):
        time = yield Event(time, ident, 'pick up passenger.')
        time = yield Event(time, ident, 'drop off passenger.')
    yield Event(time, ident, 'going home.')
    taxi = taxi_process(ident=1, trips=2)
    print(next(taxi))
    print(taxi.send(7))
    print(taxi.send(15))
    print(taxi.send(16))
    print(taxi.send(23))
    print(taxi.send(30))
Event(time=0, proc=1, action='leave garge.')
Event(time=7, proc=1, action='pick up passenger.')
Event(time=15, proc=1, action='drop off passenger.')
Event(time=16, proc=1, action='pick up passenger.')
Event(time=23, proc=1, action='drop off passenger.')
Event(time=30, proc=1, action='going home.')
这是一辆出租车可以执行的事件循环，接下来我们看看多辆出租车的执行。

import queue


class Simulator(object):
    def __init__(self, procs_map):
        self.events = queue.PriorityQueue()
        self.procs = dict(procs_map)

    def run(self, end_time):
        """Schedule and display events utill time is up."""
        for _, proc in sorted(self.procs.items()):
            first_event = next(proc)
            self.events.put(first_event)

        sim_time = 0
        while sim_time < end_time:
            if self.events.empty():
                print('*** end of events ***')
                break
            current_event = self.events.get()
            sim_time, proc_id, previous_action = current_event
            print('taxi:', proc_id, proc_id * '  ', current_event)
            active_proc = self.procs[proc_id]
            next_time = sim_time + compute_duration(previous_action)
            try:
                next_event = active_proc.send(next_time)
            except StopIteration:
                del self.procs[proc_id]
            else:
                self.events.put(next_event)
        else:
            msg = '*** end of simulation time: {} events pending ***'
            print(msg.format(self.events.qsize()))


def compute_duration(previous_action):
    duration = 1
    if previous_action == 'pick up passenger.':
        duration = 3
    elif previous_action == 'drop off passenger.':
        duration = 5
    return duration
if __name__ == '__main__':
    DEPARTURE_INTERVAL = 2
    num_taxis = 3
    taxis = {
        i: taxi_process(i, (i + 2) * 2, i * DEPARTURE_INTERVAL)
        for i in range(num_taxis)
    }
    sim = Simulator(taxis)
    print(sim.run(100))
taxi: 0  Event(time=0, proc=0, action='leave garge.')
taxi: 0  Event(time=1, proc=0, action='pick up passenger.')
taxi: 1    Event(time=2, proc=1, action='leave garge.')
taxi: 1    Event(time=3, proc=1, action='pick up passenger.')
taxi: 0  Event(time=4, proc=0, action='drop off passenger.')
taxi: 2      Event(time=4, proc=2, action='leave garge.')
taxi: 2      Event(time=5, proc=2, action='pick up passenger.')
taxi: 1    Event(time=6, proc=1, action='drop off passenger.')
taxi: 2      Event(time=8, proc=2, action='drop off passenger.')
taxi: 0  Event(time=9, proc=0, action='pick up passenger.')
taxi: 1    Event(time=11, proc=1, action='pick up passenger.')
taxi: 0  Event(time=12, proc=0, action='drop off passenger.')
taxi: 2      Event(time=13, proc=2, action='pick up passenger.')
taxi: 1    Event(time=14, proc=1, action='drop off passenger.')
taxi: 2      Event(time=16, proc=2, action='drop off passenger.')
taxi: 0  Event(time=17, proc=0, action='pick up passenger.')
taxi: 1    Event(time=19, proc=1, action='pick up passenger.')
taxi: 0  Event(time=20, proc=0, action='drop off passenger.')
taxi: 2      Event(time=21, proc=2, action='pick up passenger.')
taxi: 1    Event(time=22, proc=1, action='drop off passenger.')
taxi: 2      Event(time=24, proc=2, action='drop off passenger.')
taxi: 0  Event(time=25, proc=0, action='pick up passenger.')
taxi: 1    Event(time=27, proc=1, action='pick up passenger.')
taxi: 0  Event(time=28, proc=0, action='drop off passenger.')
taxi: 2      Event(time=29, proc=2, action='pick up passenger.')
taxi: 1    Event(time=30, proc=1, action='drop off passenger.')
taxi: 2      Event(time=32, proc=2, action='drop off passenger.')
taxi: 0  Event(time=33, proc=0, action='going home.')
taxi: 1    Event(time=35, proc=1, action='pick up passenger.')
taxi: 2      Event(time=37, proc=2, action='pick up passenger.')
taxi: 1    Event(time=38, proc=1, action='drop off passenger.')
taxi: 2      Event(time=40, proc=2, action='drop off passenger.')
taxi: 1    Event(time=43, proc=1, action='pick up passenger.')
taxi: 2      Event(time=45, proc=2, action='pick up passenger.')
taxi: 1    Event(time=46, proc=1, action='drop off passenger.')
taxi: 2      Event(time=48, proc=2, action='drop off passenger.')
taxi: 1    Event(time=51, proc=1, action='going home.')
taxi: 2      Event(time=53, proc=2, action='pick up passenger.')
taxi: 2      Event(time=56, proc=2, action='drop off passenger.')
taxi: 2      Event(time=61, proc=2, action='pick up passenger.')
taxi: 2      Event(time=64, proc=2, action='drop off passenger.')
taxi: 2      Event(time=69, proc=2, action='going home.')
*** end of events ***
```

请记住这个例子，之后在介绍**asyncio**时会进行详细的讲解。


## Summary



对于coroutine的介绍就到这里了，希望你已经理解了**yield from**的概念以及内部机制了。如果文章中有让你不理解或者介绍不清楚的地方，请务必让我知道！

感谢你的阅读，希望能收到你的feedback :)

本文代码地址: [https://github.com/Miksztowi/CorePython/tree/master/Fluent_Python_demo/Coroutines_demo](https://github.com/Miksztowi/CorePython/tree/master/Fluent_Python_demo/Coroutines_demo)