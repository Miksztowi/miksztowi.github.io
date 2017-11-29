---
title: Python Function Decorators and Closures介绍
date: 2017-11-29 16:17:13
tags: Python

---


## Preface
本文尽量用最简单，清晰的文字来介绍python的闭包与装饰器。希望你可以静下心来跟我一同思考这个tricky mechanism. **注意**本文只讲解了函数装饰器，如果你想了解类装饰器，请不要继续阅读。

## How Python evaluates decorator syntax?
Python对装饰器的使用了语法糖，如：

```
@decorate
def target():
    print('Running target()')

def target():
    print('Running target()')
target = decorate(target)
```
二者的结果是相等的，但被装饰器注册后可以不必再引用原始的target对象，所以返回对象是由decorate(target)来决定的。

## When Python executes decorators?
首先明确import time和run time这两个概念，前者是模块被导入的时刻，后者是指模块内程序运行的时刻。
装饰器是在import time被执行的，我们来看一个例子：

```
registry = []
def register(func):
    print('running register(%s)' % func)
    registry.append(func)
    return func

@register
def f1():
    print('Running f1()')

@register
def f2():
    print('Running f2()')

def main():
    print('running main')
    print('registry ->', registry)
    f1()
    f2()
if __name__ == '__main__':
    main()
------------------------------------------------------------------------    
running register(<function f1 at 0x100c70158>)
running register(<function f2 at 0x100c701e0>)
running main
registry -> [<function f1 at 0x100c70158>, <function f2 at 0x100c701e0>]
Running f1()
Running f2()
```
通过打印我们可以看到，在main()调用之前，装饰器已经完成了注册。
## How Python decides whether a variable is local?
**Variable scope rules**这个概念非常重要，介绍这个概念前，我们先看两个非常有代表性的例子。

```
def f1(a):
    print(a)
    print(b)

f1(3)
3
NameError: name 'b' is not defined
b = 6
f1(3)
3
6

接下来的例子可能会让你很困惑，不过没关系，我会在后面解释清楚。
def f2(a):
    print(a)
    print(b)
    b = 9
f2(3)
3
UnboundLocalError: local variable 'b' referenced before assignment
希望你可以先思考一下为什么没有打印出6。
事实上Python在编译函数体时，因为有赋值操作，所以将b定义为local变量，而不是global变量，
print()时发现b还未被赋值，所以抛出了错误。
我们再通过字节码来确认一下是不是这样。

print(dis.dis(f1))
  5           0 LOAD_GLOBAL              0 (print)
              3 LOAD_FAST                0 (a)
              6 CALL_FUNCTION            1 (1 positional, 0 keyword pair)
              9 POP_TOP

  6          10 LOAD_GLOBAL              0 (print)
             13 LOAD_GLOBAL              1 (b)
             16 CALL_FUNCTION            1 (1 positional, 0 keyword pair)
             19 POP_TOP
             20 LOAD_CONST               0 (None)
             23 RETURN_VALUE

print(dis.dis(f2))
  9           0 LOAD_GLOBAL              0 (print)
              3 LOAD_FAST                0 (a)
              6 CALL_FUNCTION            1 (1 positional, 0 keyword pair)
              9 POP_TOP

 10          10 LOAD_GLOBAL              0 (print)
             13 LOAD_FAST                1 (b)
             16 CALL_FUNCTION            1 (1 positional, 0 keyword pair)
             19 POP_TOP

 11          20 LOAD_CONST               1 (9)
             23 STORE_FAST               1 (b)
             26 LOAD_CONST               0 (None)
对比一下。我们可以看到在f2里是LOAD_FAST，这也证明了我们的观点：b被当作了local变量。
```
如果你明白了作用域的规则，那么恭喜你，我们可以开始学习闭包的概念了；如果没有明白，我希望你能回头再理解一下，因为明白作用域的规则是非常重要的。



## Why closures exist and how they work?

**闭包**，老样子，我们还是先思考一个例子。
假设你现在需要一个函数去计算，某股票未来三天的均价，每天都会插入一个新的价格，同时我们也要保存之前的价格以便后续的计算。

```
它的实现结果看起来应该是这样的
avg(10)
10.0
avg(11)
10.5
avg(12)
11.0

如果你已经思考过怎么实现这个函数了，那么继续往下看吧。

def make_averager():
    series = []

    def averager(new_value):
        series.append(new_value)
        total = sum(series)
        return total / len(series)

    return averager

avg = make_averager()
print(avg(10))
print(avg(11))
print(avg(12))

```
我们通过series这个变量来记录每一天的价格，并用它来计算均值。但是要注意series是
make_averager的本地变量，我们在创建avg = make _averager()已经初始化了series，
并且在后续的调用时make _averager的local作用域已经结束了。那么是python如何series的呢？

python里有个概念是**free variable**，而avarager里的series就是**free variable**.我们可以打印出来看看是不是这样。

```
    print(avg.__code__.co_varnames)
    print(avg.__code__.co_freevars)
    print(avg.__closure__[0].cell_contents)
('new_value', 'total')
('series',)
[10, 11, 12]

```
可以看到，确实存在**free variable**这个属性，并且值是series.

**总结一下**
闭包保存了函数在绑定时local作用域的对象，并将其保存为自由变量。所以尽管被绑定的函数local作用域不再可用，但是自由变量是仍然可以使用的。


## What problem is solved by nonlocal?
爱思考的你，想必已经发现了，之前我们实现的averager其实并不高效，因为我们要耗费空间去保存每一天的价格，并且每次都要调用sum()去计算总价。ok，既然我们已经发现了问题所在，那么一起来优化一下averager吧。

```
def make_averager():
    count = 0
    total = 0
    def averager(new_value):
        total += new_value
        count += 1
        return total / count
    return averager

if __name__ == '__main__':
    avg = make_averager()
    avg(10)    
>>> UnboundLocalError: local variable 'total' referenced before assignment

因为 total += new_value 相当于 total = total + new_value, 执行了赋值操作，
根据上面介绍的作用域规则，我们很快能想到，因为赋值操作导致了total变成了local变量
而不再是自由变量了。而前一个版本的实现，由于我们没有对series进行赋值操作，所以series
一直是avg的自由变量。
```
所有的不可变类型变量，对其操作都会将其作为local变量进行重新绑定。因此变量不能保存在闭包里，后续如果对其操作，就会导致错误的发生。而python3.x中增加了nonlocal操作，可以很好的解决这个问题。

```
def make_averager():
    count = 0
    total = 0
    def averager(new_value):
        nonlocal total, count
        total += new_value
        count += 1
        return total / count
    return averager
avg = make_averager()
print(avg(10))
print(avg(11))
print(avg(12))

10.0
10.5
11.0
```

## Implementing a simple decorator

通过上面概念的学习，我们可以开始实现一个简单的函数装饰器了。

```
import time

def clock(func):
    series = []
    def clocked(*args):
        t0 = time.perf_counter()
        series.append(func)
        _result = func(*args)
        elapsed = time.perf_counter() - t0
        name = func.__name__
        arg_str = ','.join(repr(arg) for arg in args)
        print('[%0.8fs] %s (%s) -> %r ' % (elapsed, name, arg_str, _result))
        return _result
    return clocked

@clock
def snooze(seconds):
    time.sleep(seconds)

print('*' * 40, 'Calling snooze(0.123)')
snooze(.123)

>>>**************************************** Calling snooze(0.123)
>>>[0.12566963s] snooze (0.123) -> None

print(snooze.__name__)
clocked

如果你想要消除对__name__和__doc__的影响，可以使用@functools.wraps()

def clock(func):
    series = []
    @functools.wraps(func)
    def clocked(*args):
        t0 = time.perf_counter()
        series.append(func)
        _result = func(*args)
        elapsed = time.perf_counter() - t0
        name = func.__name__
        arg_str = ','.join(repr(arg) for arg in args)
        print('[%0.8fs] %s (%s) -> %r ' % (elapsed, name, arg_str, _result))
        return _result

    return clocked
像上面这样的使用，就能消除装饰器的影响了。
```

## Summary
感谢阅读，希望你能从这篇文章中有所收获。
如果想要进一步的阅读，可以去探索内置的lru_cache和singledispatch装饰器的内部实现。

本文的代码链接：[https://github.com/Miksztowi/CorePython/tree/master/Fluent_Python_demo/Function_decorators](https://github.com/Miksztowi/CorePython/tree/master/Fluent_Python_demo/Function_decorators)
