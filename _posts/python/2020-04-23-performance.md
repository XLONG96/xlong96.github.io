---
layout: post
title:  "Python性能优化"
date:   2020-04-23 10:45:12
categories: python
comments: false
---

* content
{:toc}

本篇将从 CPU 资源优化和内存优化两个角度讲讲 Python 的性能优化


# CPU资源优化

## 1. 解决过多的select

* 替换thread中用到select的模块，这些模块会使用增量select的逻辑，导致过多的select空转，消耗一定的CPU

## 2. json替换成性能更好的ujson

## 3. 使用C重写部分逻辑作为so库导入

## 4. 降低gc频率



# 内存优化

## 1. 虚拟内存占用优化

python内存分配默认使用系统的malloc，在glibc 2.10版本开始引入了 thread arena，线程在申请内存的时候，glibc 为他创建一个 thread arena，这个内存池的大小一般是64M。环境变量 MALLOC_ARENA_MAX 用来控制进程可以创建的 thread arena 数量上限（默认值为0，则数量等于 cpu core*8）。当线程数量很多时，该进程就会看到下面这种情况：

```
# 通过pmap可以看到进程有多个大小为 64M（64*1024=65536）的匿名内存，导致进程的VMS非常高。
$pmap -x [pid] | sort -nrk2

00007fb2dc021000   65404       0       0 -----   [ anon ]
00007fb2d8021000   65404       0       0 -----   [ anon ]
00007fb2c8021000   65404       0       0 -----   [ anon ]
00007fb2c4021000   65404       0       0 -----   [ anon ]
00007fb2b4021000   65404       0       0 -----   [ anon ]
00007fb2a0021000   65404       0       0 -----   [ anon ]
00007fb294021000   65404       0       0 -----   [ anon ]
00007fb288021000   65404       0       0 -----   [ anon ]
00007fb280021000   65404       0       0 -----   [ anon ]
00007fb27c021000   65404       0       0 -----   [ anon ]
00007fb278021000   65404       0       0 -----   [ anon ]
```

解决这个问题可以在进程启动时设置环境变量`export MALLOC_ARENA_MAX=1`(具体可以查看 `man mallopt`手册，有些glibc版本并没有该环境变量)，此时相当于禁用了thread arena，这样VMS就不会很高了。

另外也可以换成tcmalloc、jemalloc等其他内存管理器，由于分配策略的不同，替换后VMS不会很高。同时性能上也会有提升。

## 2. 使用gc调试找出不能被回收的对象

gc.set_debug(flags)，flags如下：

* gc.DEBUG_COLLETABLE： 打印能够被垃圾回收器回收的对象
* gc.DEBUG_UNCOLLETABLE： 打印没法被垃圾回收器回收的对象，即定义了__del__的对象
* gc.DEBUG_SAVEALL：当设置了这个选项，能够被拉起回收的对象不会被真正销毁（free），而是放到gc.garbage这个列表里面，利于在线上查找问题

## 3. 使用objgraph

几个实用函数介绍：

* show_most_common_types(limits = 10)

打印实例最多的前N（limits）个对象

* show_growth()

统计自上次调用以来增长得最多的对象，这个函数很是有利于发现潜在的内存泄露

* show_backrefs(obj，max_depth, filename)

生产一张有关objs的引用图

* find_backref_chain(obj, predicate, max_depth=20, extra_ignore=())

找到一条指向obj对象的最短路径，且路径的头部节点须要知足predicate函数 （返回值为True），能够快捷、清晰指出对象的被引用的状况

* show_chain()

将find_backref_chain 找到的路径画出来, 该函数事实上调用show_backrefs，只是排除了全部不在路径中的节点

* by_type(typename)

返回该类型的对象列表，能够用这个函数很方便找到一个单例对象

* count(typename)

返回该类型对象的数目，其实就是经过gc.get_objects()拿到所用的对象，而后统计指定类型的数目

**查看引用**

在线Graphviz渲染

* http://www.webgraphviz.com/
* http://dreampuf.github.io/GraphvizOnline


## 4. 使用weakref

实用API：

* weakref.ref(object, callback = None)

建立一个对object的弱引用，返回值为weakref对象，callback: 当object被删除的时候，会调用callback函数，在标准库logging （__init__.py）中有使用范例。使用的时候要用()解引用，若是referent 已经被删除，那么返回None。

* weakref.proxy(object, callback = None)

建立一个代理，返回值是一个weakproxy对象，callback的做用同上。使用的时候直接用 和object同样，若是object已经被删除 那么跑出异常   ReferenceError: weakly-referenced object no longer exists。


**weakref 工具集合:**

* WeakKeyDictionary:这是一种可变映射，里面的值是对象的弱引用。被引用的对象在程序中的其他地方被当作垃圾回收后，对应的键会自动从 WeakValueDictionary 中删除。因此，WeakValueDictionary 经常用于缓存。

* WeakSet: 保存元素弱引用的集合类。元素没有强引用时，集合会把它删除。

* finalize (内部使用弱引用)

**实例**

```python
class Connection(object):

    MSG_TYPE_CHAT = 0X01
    MSG_TYPE_CONTROL = 0X02

    def __init__(self):

        self.msg_handlers = {
            self.MSG_TYPE_CHAT : self.handle_chat_msg,
            self.MSG_TYPE_CONTROL : self.handle_control_msg
        }
 
    def on_msg(self, msg_type, *args):
        self.msg_handlers[msg_type](*args)

    def handle_chat_msg(self, msg):
        pass

    def handle_control_msg(self, msg):
        pass
```


上面的代码很是常见，代码也很简单，初始化函数中为每种消息类型定义响应的处理函数，当消息到达(on_msg)时根据消息类型取出处理函数。但这样的代码是存在循环引用的，msg_handlers 中的bound method 会有到 Connection类实例的引用。如何手动解决呢，为Connection增长一个destroy（或者叫clear）函数，该函数将 self.msg_handlers 清空（self.msg_handlers.clear()）。当Connection理论上不在被使用的时候调用destroy函数便可。

对于多个对象间的循环引用，处理方法也是同样的，就是在“适当的时机”调用destroy函数，难点在于什么是适当的时机。

另一种更方便的方法，就是使用弱引用weakref， weakref是Python提供的标准库，旨在解决循环引用。

```python
import objgraph
import weakref


def callback(msg):
    print msg


class Connection(object):
    MSG_TYPE_CHAT = 0X01
    MSG_TYPE_CONTROL = 0X02

    def __init__(self):
        self.msg_handlers = {
            self.MSG_TYPE_CHAT: weakref.proxy(self, callback).handle_chat_msg,
            self.MSG_TYPE_CONTROL:  weakref.proxy(self, callback).handle_control_msg,
        }

    def on_msg(self, msg_type, *args):
        self.msg_handlers[msg_type](*args)

    def handle_chat_msg(self, msg):
        print "handle_chat_msg" + msg

    def handle_control_msg(self, msg):
        print "handle_control_msg" + msg


c = Connection()
c.on_msg(Connection.MSG_TYPE_CHAT, "")
objgraph.show_refs(c, filename="a.png")
````