---
title: Python中列表、迭代器、生成器的关系
date: 2016-03-04 13:32:40
tags: [Python, 编程语言]
---

对于Python初学者来说，列表（list）迭代器、生成器，是比较容易混淆的概念，我之前也是处于一种半迷糊的状态，所以打算好好整理一番，搞清楚其异同。
*<!--more-->

## 列表

列表是python的内置对象，可用来存储有序序列，并且可通过索引、切片等方式，获取任意位置的值，一旦创建，则所有元素都会进入内存。

## 迭代器

在Python3中，迭代器是定义了`__next__()`和`__iter()__`方法的对象。迭代器，在迭代过程中，会不断的调用`__next()__`方法来获取下一个元素，因此迭代器所有元素并不需要完全进入内存。因此相对列表，性能得到优化，比如在读取一个几G的大文件时，使用迭代器，可以只将在读的文件内容放入内存中。然而，迭代器也有不可回溯，不可通过索引、切片获取元素这些特点。

## 生成器

生成器的定义形式类似于函数，然而与函数不同之处是，在其中使用了`yield`关键字。一旦在函数中使用了`yiled`关键字，该函数定义便转化成了一个生成器对象的定义。生成器在运行时，一旦遇到`yield`就会带着yield后的值/对象，切换回调用方，直到调用其`__next__()`或者`send()`才会回到生成器继续执行。

``` python
>>> def g():
...     yield 1
...     yield 2
...     yield 3
...
>>> a = g()
>>> a
<generator object g at 0x1033b1f10>
>>> dir(a)
['__class__', '__del__', '__delattr__', '__dir__', '__doc__', '__eq__', '__format__', '__ge__', '__getattribute__', '__gt__', '__hash__', '__init__', '__iter__', '__le__', '__lt__', '__name__', '__ne__', '__new__', '__next__', '__qualname__', '__reduce__', '__reduce_ex__', '__repr__', '__setattr__', '__sizeof__', '__str__', '__subclasshook__', 'close', 'gi_code', 'gi_frame', 'gi_running', 'gi_yieldfrom', 'send', 'throw']
```

## 互相之间关系

从以上代码，我们可以看到生成器对象是有`__iter__()`和`__next__()`方法的。因此生成器也是迭代器，**反之不成立**。这便是生成器与迭代器之间的关系，那么迭代器与列表是什么关系呢？相信这是一个困扰了许多人的问题。我们先来查看一些列表的方法。

``` python
>>> list_obj = [1,2,3]
>>> dir(list_obj)
['__add__', '__class__', '__contains__', '__delattr__', '__delitem__', '__dir__', '__doc__', '__eq__', '__format__', '__ge__', '__getattribute__', '__getitem__', '__gt__', '__hash__', '__iadd__', '__imul__', '__init__', '__iter__', '__le__', '__len__', '__lt__', '__mul__', '__ne__', '__new__', '__reduce__', '__reduce_ex__', '__repr__', '__reversed__', '__rmul__', '__setattr__', '__setitem__', '__sizeof__', '__str__', '__subclasshook__', 'append', 'clear', 'copy', 'count', 'extend', 'index', 'insert', 'pop', 'remove', 'reverse', 'sort']
```

可以看到，我们定义的列表对象`list_obj`只有`__iter__()`方法而没有`__next__()`方法，所以我们可以断定列表对象只是一个有`__iter()__`方法的可迭代对象，并不是一个迭代器。那么列表和迭代器究竟是怎样的关系。我们更近一步的查看

``` python
>>> list_obj.__iter__()
<list_iterator object at 0x1033c1be0>
>>> dir(list_obj.__iter__())
['__class__', '__delattr__', '__dir__', '__doc__', '__eq__', '__format__', '__ge__', '__getattribute__', '__gt__', '__hash__', '__init__', '__iter__', '__le__', '__length_hint__', '__lt__', '__ne__', '__new__', '__next__', '__reduce__', '__reduce_ex__', '__repr__', '__setattr__', '__setstate__', '__sizeof__', '__str__', '__subclasshook__']
```

不难发现，列表对象的`__iter__()`方法所返回的对象是一个迭代器。
其实也正是因为这个原因，使得类似于列表这样许多可迭代对象（如set，tuple等）都可以使用`for...in...`语句进行迭代。

## for...in...

实际上`for...in...`语句是一个对可迭代对象进行遍历的语法糖，其内部实现是首先调用可迭代对象的`__iter()__`方法获取其相应迭代器对象，然后不断调用该迭代器的`__next__()`方法，直到遇到`StopIteration`异常，然后停止迭代，完成遍历。
过程如下

``` python
>>> list_obj = [1,2,3]
>>> list_iter = list_obj.__iter__()
>>> list_iter.__next__()
1
>>> list_iter.__next__()
2
>>> list_iter.__next__()
3
>>> list_iter.__next__()
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
StopIteration
>>>
```

---

最后，我们来优雅地写一个斐波那契数列吧。

``` python
def fab(max):
    a = 0
    b = 1
    for _ in range(max):
        yield b
        a, b = b, a + b
```
