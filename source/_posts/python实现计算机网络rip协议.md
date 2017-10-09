---
title: python实现计算机网络rip协议

date: 2017-06-15 16:32:39

tags: python
---

## 一、实验目的 ##

- 了解RIP协议的原理和应用等相关知识，通过距离矢量算法来实现最短传输路径的路由选择。通过本次课程设计，可以对RIP协议的工作原理和实现机制，路由表的建立和路由信息的更新等有更直观和清晰的认识。


## 二、打印结果 ##

![](http://i.imgur.com/0bSBuIP.png)

- 输入路由器数量，之后输入路由器邻接矩阵，表示路由器的连接关系，1为相邻，0为不相邻。 

![](http://i.imgur.com/PB4j3Av.png)

- 输入第一个路由器的路由表，首先输入表的长度，之后输入数据，数据格式为目的网络，距离长度，下一跳路由器。

![](http://i.imgur.com/pBGc8y5.png)

- 输入第二个路由器的路由表。

![](http://i.imgur.com/FbVF3pc.png)
![](http://i.imgur.com/B4IXjAA.png)

- 遍历打印每一个当前路由器的修改时间，修改前的当前表，和接受的相邻路由表以及更新后的当前路由表。

## 三、程序代码 ##
  
> github:[https://github.com/Miksztowi/CorePython/blob/master/rip.py](https://github.com/Miksztowi/CorePython/blob/master/rip.py)