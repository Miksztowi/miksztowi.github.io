---
title: '%_and_format()案例解释'
date: 2018-01-29 14:11:10
tags: Python

---
## 题记
因为对python2x和python3x都有使用经历，所以常常对**%**和**.format()**记忆混乱，索性今天在网上参考了几篇案例较多的tutorial,把这两种格式化风格的使用案例做一个归档。

为了后面叙述方便，我们将%定义为旧风格，.format()定义为新风格。


## 通用的格式化
简单的位置格式化应该是最常用的，通常在参数顺序不变的情况下使用。

```
print('%s %s' % (one, two)) 
print('{} {}'.format(one, two))
input>>one two
```

但是对于.format()，可以使用占位符来显式定义参数顺序。

```
# 这个操作对于%s的格式化操作来说是不可用的.
print('{1} {0}'.format(one, two))
input>>two one
```

## 值的转化

.format()函数，调用的是对象的__format__()方法，如果你需要输出str(...)或者repr(...)，你可以使用!s或者!r来转化标记。 

在旧风格中使用%s或者%r来转化。

```
class Data:
    def __str__(self):
        return 'str'

    def __repr__(self):
        return 'repr'


print('%s %r' % (Data(), Data()))
print('{!s} {!r}'.format(Data(), Data()))
input>>str repr
```


## 填充以及校准字符串

默认的格式化值是用给予的值来填充的，然而有时候也需要格式化值达到一个确定的长度。

关于这点，两种风格就显得截然不同了。

```
print('%10s' % one)
print('{:>10}'.format(one))

# 左校准
print('%-10s' % one)
print('{:10}'.format(one))

input>>       one
inupt>>one       
```

而新风格，还提供了自定义填充字符。

```
# 自定义填充字符，下列操作在旧风格中均不可行。
print('{:_<10}'.format(one))
print('{:^10}'.format(one))  # 居中
print('{:^6}'.format('zip'))  # 居中操作中，遇到奇数分割的时候，会将多出来的填充位放在右边。
input>>one_______
input>>   one    
input>> zip  
```

## 截断长字符串

相对于填充字符串达到固定长度来说，我们也可以截断字符串来达到固定长度。在格式化字符串中.后的数字是表明截断的长度。

```
str_ = 'hello world.'
print('%.5s' %(str_))
print('{:.5}'.format(str_))
input>>hello
```

## 填充与截断的结合使用

```
str_ = 'hello world.'
print('%10.5s' % str_)
print('{:>10.5}'.format(str_))
input>>     hello
```

## 数字

格式化数字也是常见的。

```
number_integer = 42
print('%d' % (number_integer))
print('{:d}'.format(number_integer))
input>>42

number_float = 3.141592653589793
print('%f' % number_float)
print('{:f}'.format(number_float))
input>>3.141593
```

## 填充数字

```
number_integer = 42
print('%4d' % number_integer)
print('{:4d}'.format(number_integer))
input>>  42
```

对于浮点数，填充值表示完整的长度。

```
number_float = 3.141592653589793
print('%06.2f' % number_float)
print('{:06.2f}'.format(number_float))
input>>003.14
```

## 有符号的整数

默认来说，只有负数会有符号。当然，我们可以显式给正数也加上。

```
number_integer = 42
print('%+d' % number_integer)
print('{:+d}'.format(number_integer))
input>>+42
```

使用空格字符表示负数应该以负号作为前缀，而正数应该使用前导空格。

```
print('% d' % (- 23))
print('{: d}'.format(- 23))
input>>-23
```

新风格也能给负整数填充。

```
print('{:=-5d}'.format(- 23))
print('{:=+5d}'.format(23))
input>>-  23
input>>+  23
```

## 名称占位符

介绍完了对于数字的操作，接下来介绍一些关于占位符的案例。

```
data = {'first': 'Hodor', 'last': 'Hodor!'}
print('%(first)s %(last)s' % data)
print('{first} {last}'.format(**data))
print('{first} {last}'.format(first='Hodor', last='Hodor!'))  # only available with new style formatting.
input>>Hodor Hodor!
```

## Getitem和Getattr

新风格允许对数据结构进行更灵活的操作。
它可以操作拥有__getitem__方法的对象。

```
person = {'first': 'Jean-Luc', 'last': 'Picard'}
print('{p[first]} {p[last]}'.format(p=person))
input>>Jean-Luc Picard
data = [4, 8, 15, 16, 23, 42]
print('{d[4]} {d[5]}'.format(d=data))
input>>23 42
```

它也支持操作拥有getattr()属性的对象。

```
class Plant(object):
    type = 'tree'
print('{p.type}'.format(p=Plant()))
input>>tree
```

它也支持二者的同时操作。

```
class Plant(object):
    type = 'tree'
    kinds = [{'name': 'oak'}, {'name': 'maple'}]
print('{p.type}: {p.kinds[0][name]}'.format(p=Plant()))
input>>tree: oak
```

## 时间

新风格也允许日期时间对象内联格式化。

```
from datetime import datetime
print('{:%Y-%m-%d %H:%M:%S}'.format(datetime.now()))
input>>2018-01-29 13:38:40
```

## 参数格式化

新风格格式化允许使用参数来动态指定。参数化格式是大括号中的嵌套表达式，可以在冒号后的父级格式的任何地方出现。

```
print('{:{align}{width}}'.format('test', align='>', width='10'))
input>>      test
print('{:.{prec}} = {:.{prec}f}'.format('Gibberish', 2.7182, prec=3))
input>>Gib = 2.718
print('{:{width}.{prec}f}'.format(2.7182, width=5, prec=2))
input>> 2.72
print('{:{prec}} = {:{prec}}'.format('Gibberish', 2.7182, prec='.3'))
input>>Gib = 2.72
from datetime import datetime
dt = datetime.now()
print('{:{dfmt} {tfmt}}'.format(dt, dfmt='%Y-%m-%d', tfmt='%H:%M'))
input>>2018-01-29 13:59
```

## 参考
[https://pyformat.info/](https://pyformat.info/)

[https://github.com/ulope/pyformat.info](https://github.com/ulope/pyformat.info)
## 最后

本文代码连接： [% and .format()](https://github.com/Miksztowi/CorePython/blob/master/format_and_%25_/demo.py)


如有纰漏之处，还请您不吝赐教。 

Have fun :)
