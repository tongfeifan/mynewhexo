---
title: Python3中的AsyncIO库阅读后思考
date: 2016-02-28 13:29:17
tags: [Python, 并发编程, 源码阅读]
---

## 何谓异步

要搞清楚AsyncIO的实现原理，首先需要明白一些基本概念，异步（Asynchrony），详细的描述在[维基百科](https://en.wikipedia.org/wiki/Asynchrony_%28computer_programming%29)中有。大致翻译：

> 异步是一种对于独立于主程序流的事件的处理方法。这些事件一般是外部事件，如信号到达，动作触发，而且异步不会阻塞等待结果。
> 一种处理异步的常规方式便是提供一个方法，该方法返回给调用者一个对象（一般称为[future or promise](https://en.wikipedia.org/wiki/Futures_and_promises)）用来表示一个持续进行的事件。
*<!--more-->

恩，大致翻译如此，引用来自[知乎](http://zhihu.com/question/19732473/answer/20851256?utm_campaign=webshare&amp;utm_source=weibo&amp;utm_medium=zhihu)@[卢毅](https://www.zhihu.com/people/tianyishengshui)的一个通俗例子便是：

> 你打电话问书店老板有没有《分布式系统》这本书，如果是同步通信机制，书店老板会说，你稍等，”我查一下"，然后开始查啊查，等查好了（可能是5秒，也可能是一天）告诉你结果（返回结果）。
> 而异步通信机制，书店老板直接告诉你我查一下啊，查好了打电话给你，然后直接挂电话了（不返回结果）。然后查好了，他会主动打电话给你。在这里老板通过“回电”这种方式来回调。

## 何谓协程(coroutine)

在更细的粒度上执行的子程序，协程在同一线程上并行执行，由程序级控制其调度。在python中，通过生成器(generator)来并行执行多个协程。即通过yield关键字实现。

引用[该处](http://www.jackyshen.com/2015/05/21/async-operations-in-form-of-sync-programming-with-python-yielding/)的解释便是：

> 任何包含`yield`关键字的函数都会自动成为生成器(generator)对象,里面的代码一般是一个有限或无限循环结构，每当第一次调用该函数时，会执行到yield代码为止并返回本次迭代结果，yield指令起到的是return关键字的作用。然后函数的堆栈会自动冻结(freeze)在这一行。当函数调用者的下一次利用next()或generator.send()或for-in来再次调用该函数时，就会从yield代码的下一行开始，继续执行，再返回下一次迭代结果。通过这种方式，迭代器可以实现无限序列和惰性求值。

## Python 中的AsyncIO

好了，至此我们可以看一下Python3.4、3.5中的AsyncIO的实现
打开`asyncio`库，进入`__init__`文件，可以看到AsyncIO使用了python3.4的selectors库用于系统级别的IO切换，同时也兼顾了win32平台，为win32平台另外引入了其他模块

``` python
if sys.platform == 'win32':
    # Similar thing for _overlapped.
    try:
        from . import _overlapped
    except ImportError:
        import _overlapped  # Will also be exported.
```

``` python
if sys.platform == 'win32':  # pragma: no cover
    from .windows_events import *
    __all__ += windows_events.__all__
else:
    from .unix_events import *  # pragma: no cover
    __all__ += unix_events.__all__
```

除此之外，这个package包括了其他一系列用于执行方便执行协程的模块。在此可以着重分析一下`base_events`、 `coroutines`、 `futures`、 `task`模块。

从`coroutines`模块，我们可以看出关键是`coroutine`装饰器，该装饰器将generator类型标记为 corcoutine类型。

``` python
def coroutine(func):
    """Decorator to mark coroutines.

    If the coroutine is not yielded from before it is destroyed,
    an error message is logged.
    """
   if _inspect_iscoroutinefunction(func):
        # In Python 3.5 that's all we need to do for coroutines
        # defiend with "async def".
        # Wrapping in CoroWrapper will happen via
        # 'sys.set_coroutine_wrapper' function.
        return func

    if inspect.isgeneratorfunction(func):
        coro = func
    else:
        @functools.wraps(func)
        def coro(*args, **kw):
            res = func(*args, **kw)
            if isinstance(res, futures.Future) or inspect.isgenerator(res):
                res = yield from res
            elif _AwaitableABC is not None:
                # If 'func' returns an Awaitable (new in 3.5) we
                # want to run it.
                try:
                    await_meth = res.__await__
                except AttributeError:
                    pass
                else:
                    if isinstance(res, _AwaitableABC):
                        res = yield from await_meth()
            return res
```

另外该模块中还有一个`debug_wrapper`， 会在开启debug模式的情况下被调用，我猜测通过这种方法，便可方便在debug模式下断点调试时也可及时看到目前`coroutine`中的值。

接下来，`futures`模块即为一个常规的futures模块，其中实现了future类，而在`tasks`模块中，则实现了对coroutine对象的wrap，将coroutine封装为future类。如此一来，便将其统一为一个抽象的future类，可被统一调度。
最后，结合`base_events`模块，我们来看一段典型的示例代码，并讨论一些，这段代码背后是怎么跑的。

``` python
import threading
import asyncio

@asyncio.coroutine
def hello():
    print('Hello world! (%s)' % threading.currentThread())
    yield from asyncio.sleep(1)
    print('Hello again! (%s)' % threading.currentThread())

loop = asyncio.get_event_loop()
tasks = [hello(), hello()]
loop.run_until_complete(asyncio.wait(tasks))
loop.close()
```

以上为一段经典的，及其简单的协程调用代码。通过`@asyncio.coroutine`与`yield from`， hello程序被定义成了一个coroutine，（python3.5开始，推荐`asnyc/await`关键字）然后返回一个event loop实例。通过`asyncio.wait()`，tasks（coroutine列表）被包装为Task（继承自Future的wrapper），然而作为参数传递给`loop.run_unitl_complite()`方法。在该方法中，Future中的任务被不断地并行来回切换执行(yield from导致的协程特性)，直到全部完成，关闭该event loop。