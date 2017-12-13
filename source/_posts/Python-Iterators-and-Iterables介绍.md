---
title: Python Iterators and Iterables介绍
date: 2017-12-02 15:02:24
tags: Python

---
## Concept
Iteration是对数据进行函数式的操作。假设当你需要从内存无法装下的数据集中读取数据时，你可能需要对数据集进行**惰性**读取，意味着每次操作只读取一条数据-----而这就是Iterator模式。


## Iterables
`iterable `  任何对象通过内置iter()方法都可以得到一个iterator。对象如果实现了__ iter__方法并且返回一个iterator，那么称对象是iterable.

在Python里所有的集合都是可迭代的，例如：

* for 循环
* 集合类型和其扩展
* list, dict, set类型的解析式
* tuple类型的unpack
* 对函数的*args形参进行unpack

诸如此类的操作，都是iterable.

通过例子我们再来探索一下

```
RE_WORD = re.compile(r'\w+')
class Sentence(object):
    def __init__(self, text):
        self.text = text
        self.words = RE_WORD.findall(self.text)

    def __len__(self):
        return len(self.words)

    def __getitem__(self, index):
        return self.words[index]

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

通过迭代，for对其进行了打印操作，并尝试让Sentence对象成为list对象。
```


## How the iter(...) built-in function is used to internally to hanlde iterable objects.

每一个Python编程人员，都知道序列对象是可以迭代的，如list, dict。可为什么序列对象都支持迭代呢？
这是因为当我们进行迭代时，解释器会自动调用对象的iter()方法。

iter()方法的使用规则：

1. 检查对象是否实现了__ iter__方法，如果有，则调用得到一个iterator.
2. 如果__ iter__ 方法没有被实现，但是对象实现了__ getitem__方法，那么Python会创建一个Iterator并按照顺序从0开始索引.
3. 如果没有实现以上方法，或者操作失败。Python会抛出**TypeError**， 如：TypeError: 'Sentence' object is not iterable.

而正是Python的序列对象都实现了 __ getitem__方法，所以它们都是可迭代的。（通常标准的sequences也会实现 __ iter__方法）


## How to implement the classic Iterator pattern in Python.
iterator都是由iterable对象得到的， 举一个日常代码中经常出现的例子:

```
s = 'ABC'
for char in s:
    print(char)
A
B
C
这应该是最常见的iterator实例了，iterator背后的使用逻辑如下。

s = 'ABC'
it = iter(s)
while True:
    try:
        print(next(it))
    except StopIteration:
        del it
        break
```
**StopIteration** 意味着iterator失效了。这个异常会被for循环或者其他迭代操作捕获到，先对iterator进行释放后，之后退出循环。

标准的iterator接口有两个方法：

1.  __ next__  返回下一个可使用的item， 当没有可用的items时会抛出Stopiteration的异常。
2.  __ iter__  返回自身；这样可用于for循环操作。

当需要判断iterator是否有其余的items时，只能通过next()来进行。同样，当一个iterator失效后，不可能进行重设操作。如果你需要重新操作一次iterator的话，那么只能通过iterable对象的iter()方法来得到一个新的iterator。要记住，即使iterator实现了__ iter__ 方法，调用它也只能获得它本身，所以不可能进行重设操作。

`iterator` 是指实现了__ next__ 和__ iter__方法的对象, __ next__ 方法会在对象没有剩余items时抛出Stopiteration的异常来终止操作。由于Python的iterator内部也实现了__ iter__方法，所以iterator同时也是iterable。

接下来通过一个标准的iterator实现来理解下iterable和iterator的概念及不同：

```
import re
import reprlib

RE_WORD = re.compile(r'\w+')

class Sentence(object):
    def __init__(self, text):
        self.text = text
        self.words = RE_WORD.findall(self.text)

    def __repr__(self):
        return 'Sentence(%s)' % reprlib.repr(self.text)  # reprlib.repr limits the generated string to 30 characters.

    def __iter__(self):
        return SentenceIterator(self.words)


class SentenceIterator(object):
    def __init__(self, words):
        self.words = words
        self.index = 0

    def __iter__(self):
        return self

    def __next__(self):
        try:
            word = self.words[self.index]
        except IndexError:
            raise StopIteration()
        self.index += 1
        return word


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
```
通过这个例子，可以看到： iterable实现了 __ iter__方法，每次调用会返回一个iterator实例。iterator对象实现了 __ next__ 方法，每次返回一个item，并在没有剩余items时抛出StopIteration(), __ iter__ 方法每次返回自身。
因此，我们可以说**iterator是iterable，但是iterable不是iterator.**


最后我们作出一个猜想： 我们可不可以在Sentence类里实现__ next__方法呢，使Sentence既是iterator也是iterable，岂不美哉？

答案很明确，这是一个非常糟糕的想法。但是如果你对答案产生困惑也没关系，我们再一起来了解一下，为什么不要在Python里尝试这样做。

**Iterator Pattern**里有一点：to support multiple traversals of aggregate objects.
如果要满足这点，意味着iterable必须产生多个独立的新iterator对象。每一个iterator都在内部保持自己的状态，不进行互相干扰。所以iterable的本质就是要对其进行iter()操作时可以获得独立的，新的iterator。而如果实现了__ next__方法，则不能产生新的iterator，会破坏这一设计原则，所以这是很糟糕的想法。


## Summary
感谢阅读，希望你读到这里时，对于iterable和iterator的概念，能有一个新的认识。

本文代码链接:[https://github.com/Miksztowi/CorePython/tree/master/Fluent_Python_demo/Iterators_and_generators_demo](https://github.com/Miksztowi/CorePython/tree/master/Fluent_Python_demo/Iterators_and_generators_demo)

如果文章中有错误，或者你还有不理解的地方，欢迎在下方的评论／邮件中提交你的想法／困惑。

值得一说的是，尽管我们对iterable和iterator的介绍已经结束了，但是在Python中还有一个非常重要的概念**generator**也与其有关。因为篇幅原因，我将其拆分为另一篇文章[a](a)，打铁要趁热！！！快来学习一下！！！


