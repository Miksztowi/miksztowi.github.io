---
title: Python内置模块bisect介绍
date: 2017-11-20 14:33:07
tags: Python
---



在Leetcode Weekly Contest 59上有一道题，需要在插入时保持序列顺序，从而我去了解了Python内置的**bisect**模块.	

## 介绍

对于bisect，官网的解释是：

This module provides support for maintaining a list in sorted order without having to sort the list after each insertion. For long lists of items with expensive comparison operations, this can be an improvement over the more common approach. The module is called bisect because it uses a basic bisection algorithm to do its work. The source code may be most useful as a working example of the algorithm (the boundary conditions are already right!).

大意是说：模块提供了对列表顺序的维护，无需每次插入后对列表排序。使用了基础的二分算法，对操作大数量的列表，可以显著的减少比较操作的次数。

## 模块内提供的函数

#### 查找：

`bisect.bisect_left(a, x, lo=0, hi=len(a))`

a是需要操作的对象，x是需要查找的目标，lo和hi会切割数组，使`all(val < x for val in a[lo:i])` 在左侧， `all(val >= x for val in a[i:hi])` 在右侧。返回值为可插入的第一个位置(有相同的item).

`bisect.bisect_right(a, x, lo=0, hi=len(a))`

`bisect.bisect(a, x, lo=0, hi=len(a))`

这两个函数相同，且与bisect.bisect_left操作类似。区别在于，返回值为可插入的最后一个位置。即`all(val <= x for val in a[lo:i])` 在左侧 `all(val > x for val in a[i:hi])` 在右侧。

#### 插入：
`bisect.insort_left(a, x, lo=0, hi=len(a))`

函数相当于a.insert(bisect.bisect_left(a, x, lo=0, hi=len(a)), x). 其中list.insert()的时间复杂度为O(n),查找的时间复杂度为O(logn).

`bisect.insort_right(a, x, lo=0, hi=len(a))`

`bisect.insort(a, x, lo=0, hi=len(a))`


## 事例讲解
用一个排序后的lists查找案例来练习一下吧。

```
def index(a, x):
    'Locate the leftmost value exactly equal to x'
    i = bisect.bisect_left(a, x)
    if i != len(a) and a[i] == x:
        return i
    raise ValueError


def find_lt(a, x):
    'Find rightmost value less than x'
    i = bisect.bisect_left(a, x)
    if i:
        return a[i - 1]
    raise ValueError


def find_le(a, x):
    'Find rightmost value less than or equal to x'
    i = bisect.bisect_right(a, x)
    if i:
        return a[i - 1]
    raise ValueError


def find_gt(a, x):
    'Find leftmost value greater than x'
    i = bisect.bisect_right(a, x)
    if i != len(a):
        return a[i]
    raise ValueError


def find_ge(a, x):
    'Find leftmost item greater than or equal to x'
    i = bisect.bisect_left(a, x)
    if i != len(a):
        return a[i]
    raise ValueError
```


## 进入正题

最后leetcode的题目是：

Implement a MyCalendar class to store your events.
A new event can be added if adding the event will not cause a double booking.
Your class will have the method, book(int start, int end).
Formally, this represents a booking on the half open interval [start, end),
the range of real numbers x such that start <= x < end.
A double booking happens when two events have some non-empty intersection
(ie., there is some time that is common to both events.)
For each call to the method MyCalendar.book,
return true if the event can be added to the calendar successfully without causing a double booking. Otherwise,
return false and do not add the event to the calendar.
Your class will be called like this: MyCalendar cal = new MyCalendar(); MyCalendar.book(start, end)


代码：

```
import bisect


class MyCalendar(object):
    def __init__(self):
        self.calendar = []
        self._start_sorted = []

    def book(self, start, end):
        """
        :type start: int
        :type end: int
        :rtype: bool
        """
        if not self.calendar:
            self.calendar += (start, end),
            self._start_sorted += start,
            return True
        floor_index = bisect.bisect_left(self._start_sorted, start)
        if floor_index and self.calendar[floor_index - 1][1] > start:
                return False

        ceiling_index = bisect.bisect_right(self._start_sorted, start)
        if self.calendar[ceiling_index - 1][0] == start and self.calendar[ceiling_index - 1][0] < end:
                return False
        elif ceiling_index != len(self.calendar) and self.calendar[ceiling_index][0] < end:
                return False

        self.calendar.insert(floor_index, (start, end))
        self._start_sorted.insert(floor_index, start)
        return True
```

## 最后
详细的代码和用例：
[https://github.com/Miksztowi/CorePython/blob/master/bisect_demo.py](https://github.com/Miksztowi/CorePython/blob/master/bisect_demo.py)


