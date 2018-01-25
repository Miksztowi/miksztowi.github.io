---
title: Python generator 介绍
date: 2017-12-02 17:12:38
tags: Python

---
## Preface
Generator是Python里相当有趣的trick.不过在阅读本文前，希望你对iterable和iterator的概念已经有了认识。如果没有的话建议先阅读本人的另一篇文章：[Python Iterators and Iterables介绍](http://miks.top/2017/12/02/Python-Iterators-and-Iterables%E4%BB%8B%E7%BB%8D/)



## How a generator function works in detail, with line by line descriptions.
介绍相关概念前，我们先看一个Pythonic的Iterator实现。

```
class Sentence(object):
    def __init__(self, text):
        self.text = text
        self.words = RE_WORD.findall(self.text)

    def __iter__(self):
        for word in self.words:
            yield word

    def __repr__(self):
        return 'Sentence(%s)' % reprlib.repr(self.text)  # reprlib.repr limits the generated string to 30 characters.

if __name__ == '__main__':
    s = Sentence('"The time has come," the Walrus said,')
    print(s)
    for word in s:
        print(word)
    print(list(s))
Sentence('"The time ha... Walrus said,')
The
time
has
come
the
Walrus
said
['The', 'time', 'has', 'come', 'the', 'Walrus', 'said']

对比之前实现的例子，可以看到已经简洁很多了！！ 
tip: 最简洁的是 return iter(self.words)
```

在Python中，如果函数里有yield关键字，那么代表这个函数就是一个生成器函数，生成器函数会返回一个generator。也就是说生成器函数是一个生成器工厂。

```
def gen_123():
    yield 1
    yield 2
    yield 3

print(gen_123)
print(gen_123())
for i in gen_123():
    print(i)
g = gen_123()
print(next(g))
print(next(g))
print(next(g))
print(next(g))

<function gen_123 at 0x10a6661e0>
<generator object gen_123 at 0x10a5d8258>
1
2
3
1
2
3
Traceback (most recent call last):
  File "/Users/wukong/Python/Git/CorePython/Fluent_Python_demo/Iterators_and_generators_demo/generator_demo.py", line 45, in <module>
    print(next(g))
StopIteration
当函数调用结束后，会抛出StopIteration.这是实现了Iterabtor的协议。不过for操作会捕获这个异常。
```
调用生成器函数会返回一个生成器，生成器通过关键字yield来产生值。

明白了生成器的概念后，我们再来了解一个跟生成器有关的Trick---lazy Implementation.A lazy implementation延迟了值的生成，直到使用时才会产生。这样的操作可以节约内存和避免不需要的生成。

我们用re模块自带的re.finditer来改造上面的例子，使其具有lazy属性。

```
import re
import reprlib

RE_WORD = re.compile(r'\w+')


class Sentence(object):
    def __init__(self, text):
        self.text = text
        self.words = RE_WORD.findall(self.text)

    def __iter__(self):
        for match in RE_WORD.finditer(self.text):
            yield match.group()

    def __repr__(self):
        return 'Sentence(%s)' % reprlib.repr(self.text)  # reprlib.repr limits the generated string to 30 characters.

```



## How the classic Iterator can be repalced by a generator function or generator expression.
尽管用generator function以及比iterator简洁很多了，但是generator expression可以更简洁！

generator expression会产生一个惰性list，只有当需要生成值时才会生成。同时generator expression也是generators的工厂。

```
def gen_AB():
    print('start')
    yield 'A'
    print('continue')
    yield 'B'
    print('end.')

if __name__ == '__main__':
    res1 = [x*3 for x in gen_AB()]
    for i in res1:
        print(i)
    res2 = (x*3 for x in gen_AB())
    for i in res2:
        print(i)
start
continue
end.
AAA
BBB


start
AAA
continue
BBB
end.

通过打印结果，可以验证generator expression是lazy的。
```


generator expression是python的语法糖，可以被generator function替换，但是有时使用它会让代码更简洁，清晰。

```
import re
import reprlib

RE_WORD = re.compile(r'\w+')


class Sentence(object):
    def __init__(self, text):
        self.text = text

    def __iter__(self):
        return (match.group() for match in RE_WORD.finditer(self.text))

    def __repr__(self):
        return 'Sentence(%s)' % reprlib.repr(self.text)  # reprlib.repr limits the generated string to 30 characters.
尽管没有使用yield，但是generator expression是generator工厂，所以返回了一个generator对象。
```
尽管generator expression有诸多优点，但是generator function更加的灵活，可以表达逻辑复杂的代码。所以在选择上，通常当逻辑代码超过两行时，推荐使用generator function.


最后，如果你不太明白lazy实现的优点，可以用一个案例来帮助你理解下。假设我们要实现对没有边界的集合的迭代，那么意味着不论内存多大，放入这个集合后都会导致OOM。如果避免OOM，并且实现这个目标呢？

```

class ArithmeticProgression(object):
    def __init__(self, begin, step, end=None):
        self.begin = begin
        self.step = step
        self.end = end

    def __iter__(self):
        result = type(self.begin + self.step)(self.begin)
        forever = self.end is None
        index = 0
        while forever or result < self.end:
            yield result
            index += 1
            result = self.begin + self.step * index
ap1 = ArithmeticProgression(1, 0.5, 2)
print(list(ap1))
ap2 = ArithmeticProgression(1, 0.5)
for i in ap2:
    print(i)

ap2是没有边界的，除非强制终止，否则for循环会一直打印。但是调用这样一个对象，并不会触发OOM。  
```


## Using the new yield from statement to combine generators

当我们的generator需要从另一个generator得到值时，要如何操作呢？

```
def chain(*iterables):
    for it in iterables:
        for i in it:
            yield i
s = 'ABC'
t = tuple(range(3))
print(list(chain(s, t)))

['A', 'B', 'C', 0, 1, 2]

要注意unpack也是generator哦，虽然实现了这样的功能。但是Python的语法糖可以帮你省去内置的for循环。
def chain(*iterables):
    for it in iterables:
        yield from it 
```
yield from 在内部生成器和外部生成器之间创建了一个通道，这个通道在coroutines中是非常重要的。


## Summary
最后，感谢阅读。generator的Trick就介绍到这里了，虽然它同python中的coroutines有一些共同点，但是它们也是不同的概念哦，深入的探索可以去了解coroutines的原理。

本章相关代码：
[https://github.com/Miksztowi/CorePython/tree/master/Fluent_Python_demo/Iterators_and_generators_demo](https://github.com/Miksztowi/CorePython/tree/master/Fluent_Python_demo/Iterators_and_generators_demo)




