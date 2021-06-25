---
layout: post
title: "浅谈python GLI锁"
subtitle: '对于GLI的一些分析'
author: "odin"
header-style: text
tags:
  - python
---

![]({{site.baseurl}}/img/in-post/post-python/python.png)

关于本文内容，我们需要达成以下共识:
* 1.cpu是用于计算的，而不是用于I/O
* 2.多个cpu意味着可以同时进行多个核的并行计算处理，所以多核意味着计算性能的提升
* 3.每个cpu一旦遇到I/O阻塞是需要等待的，所以多核对于I/O并没有提升

## 1.什么是GLI

官方文档对于GLI的解释如下:

> In CPython, the global interpreter lock, or GIL, is a mutex that protects access to Python objects, 
preventing multiple threads from executing Python bytecodes at once. This lock is necessary mainly 
because CPython's memory management is not thread-safe. (However, since the GIL exists, other features 
have grown to depend on the guarantees that it enforces.)

即: `在CPython解释器中，全局解释锁GLI是在执行python字节码时，为了保护访问python对象而组织多线程的一个互斥锁。设计这个锁的主要原因是CPython解释器不是线性安全的。
然而直到今天GLI锁还存在的原因是很多功能已经依赖GLI作为执行的保障了。`

对官方提供的说明我们可以得到一些关于GLI的重要信息:

* GLI锁是对于python解释器上的一种全局锁。属于解释器层面的而并非是python语言特性。
    > python最主要的解释器CPython存在GLI锁，其他一些解释器有些不存在GLI锁，如: `JPython`、`IronPython`
* GLI锁的存在使得python多线程在同一时间只有一个线程处于工作中，以此保证内存管理是安全的。
* GLI锁是设计上的历史遗留问题。现在很多python项目和库依赖于GLI。

## 2.GLI会带来什么问题

分计算密集型和I/O密集型两个场景来看

### 2.1).计算密集型

使用串行方式，计算两次

```python
import time
import threading

def GetCount():
    count = 0
    while count <= 1000000000:
        count += 1

# 单线程执行 2 次 CPU 密集型任务
start = time.time()
GetCount()
GetCount()
end = time.time()
print("execution time: %s" % (end - start))

# execution time: 111.21523690223694
```

改为2个线程执行计算

```python
import time
import threading

def GetCount():
    count = 0
    while count <= 1000000000:
        count += 1


# 2个线程同时执行CPU密集型任务
start = time.time()

t1 = threading.Thread(target=GetCount)
t2 = threading.Thread(target=GetCount)
t1.start()
t2.start()
t1.join()
t2.join()

end = time.time()
print("execution time: %s" % (end - start))
# execution time: 107.53772020339966
```

开启2个线程并没有比单线程效率高，甚至有时候效率还不如单线程

### 2.2)IO密集型

```python
from threading import Thread
import time

def work():
    time.sleep(2)
    print('===>')


l = []
start = time.time()
for i in range(400):
    p = Thread(target=work)  # 耗时2s多
    l.append(p)
    p.start()
for p in l:
    p.join()
stop = time.time()
print('execution time: %s' % (stop - start))

# execution time: 2.1608619689941406
```

多个线程的执行时间都是2s左右，若是使用单线程的话执行消耗时间可想而知。

## 3.GLI的原理和为什么存在

### 3.1)GLI存在原因
在 2000 年以前，cpu各厂家的效率方向都是在提升cpu的计算性能，努力在提升单核cpu的运行频率上。到后期研发提升上遇到了瓶颈，故而开始将单核转向多核来提升cpu计算性能。
为了有更多的更有效的利用多核cpu，很多编程语言就出现了多线程的编程方式，但也是由于多线程的编程方式出现，随之带来的就是多线程之间对于维护数据和状态一致性的困难。
python设计者在设计解释器之初可能没有考虑到cpu会往多核方向发展，所以在当时的状态下设计了全局解释锁的机制来保护多线程资源一致性的这一种最简单经济的方案。
而随着多核心时代来临，当大家试图去拆分和去除 GIL 的时候，发现大量库的代码和开发者已经重度依赖 GIL（默认认为 Pythonn 内部对象是线程安全的，无需在开发时额外加锁），所以这个去除 GIL 的任务变得复杂且难以实现。

所以，GIL 的存在更多的是历史原因，在 Python 3 的版本，虽然对 GIL 做了优化，但依旧没有去除掉，Python 设计者的解释是，在去除 GIL 时，会破坏现有的 C 扩展模块，因为这些扩展模块都严重依赖于 GIL，去除 GIL 有可能会导致运行速度会比 Python 2 更慢。
Python 走到现在，已经有太多的历史包袱，所以现在只能背负着它们前行，如果一切推倒重来，想必 Python 设计者会设计得更加优雅一些。

### 3.2)GLI原理

python的线程就是C语言的`pthread`，他是通过操作系统调度算法调度执行的。
在python2.X的时候，python的代码执行是基于opcode数量的调度方式，每执行一定数量的字节码，或者遇到系统IO时会强制释放GLI锁，然后触发一次系统的线程调度。
在python3.X对于其进行了优化，改为基于固定执行间隔时间，每执行固定时间的字节码，或者是遇到系统IO时会强制释放GLI锁，触发系统的线程调度。

基于GLI这种调度方式下，故而同一时间只会有一个线程处于工作状态。而线程在调度时又依赖于cpu的工作环境，也就是在单核和多核的情况下，多线程在调度的消耗时间上是不一样的。
* 如果是在单核 CPU 环境下，多线程在执行时，线程 A 释放了 GIL 锁，那么被唤醒的线程 B 能够立即拿到 GIL 锁，线程 B 可以无缝接力继续执行
* 而如果在在多核 CPU 环境下，当多线程执行时，线程 A 在 CPU0 执行完之后释放 GIL 锁，其他 CPU 上的线程都会进行竞争。

但 CPU0 上的线程 B 可能又马上获取到了 GIL，这就导致其他 CPU 上被唤醒的线程，只能眼巴巴地看着 CPU0 上的线程愉快地执行着，而自己只能等待，直到又被切换到待调度的状态，这就会产生多核 CPU 频繁进行线程切换，消耗资源，这种情况也被叫做「CPU颠簸」

## 4.总结GLI的优缺点和使用场景

优点：避免大量的加锁解锁操作

缺点：由于GLI锁的存在，同一时间只会有一个cpu处于工作状态，多核心退化为单核心处理器

适用场景：IO密集型场景下使用多线程，计算密集型场景下使用多进程
