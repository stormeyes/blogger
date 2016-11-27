# Python3.5 中async/await 特性的研究

## 从demo开始

从Python3.5起, Python语言增加了asnyc/await关键字, 正式宣告了Python拥有了语言层面官方的异步实现. 以下是一个示例demo:

```python
import time
import asyncio
import random

async def simulate_slow_io(index):
    random_time = random.randrange(1, 4)
    print('io {} start and sleep for {}'.format(index, random_time))
    await asyncio.sleep(random_time)
    print('io {} done'.format(index))


async def run_it():
    start_time = time.time()
    await asyncio.wait([
        simulate_slow_io(1),
        simulate_slow_io(2),
        simulate_slow_io(3)
    ])
    end_time = time.time()
    print('total use {}'.format(int(end_time-start_time)))

loop = asyncio.get_event_loop()
loop.run_until_complete(run_it())
print('Test if it can reach here')
```

 输出的结果如下:

```bash
io 2 start and sleep for 2
io 3 start and sleep for 2
io 1 start and sleep for 3
io 2 done
io 3 done
io 1 done
total use 3
```

这个输出结果告诉我们以下几件事：

+   start的顺序是2, 3, 1 这说明提供的asyncio.wait()中, 并不是按照传入的list的元素的顺序依次调用的.
+   总共花费的时间和最大的sleep时间一致, 说明的确是异步执行的. 如果是顺序同步阻塞执行, 总共的时间应该是2+2+3=7s.
+   run_until_complete和run_forever一样, 都是阻塞的, 不同的是run_until_complete会在执行完毕后往下执行, forever会永远hold住.

需要注意的是, 创建了一个async function后, 如果不将其用create_task/ensure_future包装成coroutine/future并加入事件循环调度, 而是直接执行, **该函数永远不会执行**

不过我觉得这个demo远远不够, 并没有解答我的疑问:

+   为什么即使在语言层面添加了async/await关键字, 依旧要声明一个event loop? 为什么不能像nodejs的异步调用那样不需要声明event loop, 直接就异步?
+   async.sleep对于实际使用没有任何意义, 我想知道的是await这个语法糖, 对于后面紧跟的那个IO调用, 到底是会自动把同步的转成异步, 还是只允许调用基于asyncio提供的异步IO实现的接口, 如果是后者, asyncio的io接口在哪里? 怎么使用?
+   如果要使用async/await发起http异步请求, 到底要怎么用?

不幸的是, 所有的中文资料都语焉不详互相抄袭或者是浮于表面, 不得不通过阅读Python的官方文档和一些异步框架的源码来解答这些疑问.

## 事件循环

我一直很喜欢Nodejs的异步, 因为和别的语言的异步相比, Nodejs的异步自然, 简洁, 直观, 标准的pub/sub模式, 只需要注册事件即可, 别的语言除了注册事件外还有一大坨别的代码. 我一直以为Node的异步非常简洁是因为js原本并没有IO能力, 是Node让js实现了IO的能力并且实现的接口默认全为异步所导致的结果, 现在看来事情并非如此简单.

偶然的一次机会, 我重新阅读了Node官网的[介绍](https://nodejs.org/zh-cn/about/) :

>   Node simply enters the event loop after executing the input script. Node exits the event loop when there are no more callbacks to perform. This behavior is like browser JavaScript — the event loop is hidden from the user.

这段话包含了三个信息

-   在每次执行脚本的时候, Node就开始进入了事件的轮询机制中.
-   直到所有的回调函数执行完毕后, Node才会退出事件轮询机制.
-   Node中的事件循环机制, 对开发者是**隐藏的, 不可见的** , 开发者是无感知的.

之前读的时候, 只是阅读了字面上的意思, 并没有理解到背后的深刻含义, 通过与Python的异步实现作比较, 我才开始明白这段话的含义.

异步的实现方式多种多样, Node选择了通过事件循环的方式来实现, 这意味着异步代码/函数的执行需要其所在的执行环境是事件循环环境, 解释器本身在一刻不停地轮询事件队列. 无论是Python, Ruby或者是别的什么语言, 语言本身都已经实现了IO并且默认都是阻塞的, 他们是天生同步的. 当为这些语言添加异步特性的时候, 要么显式声明一个事件循环并将异步事件放入该事件循环中. 要么解释器完全修改执行方式, 改为Node一样的事件循环方式. 很明显后者的主意是这个世界上最蠢的办法.

当然, 也可以让解释器在解释执行的时候, 发现异步函数调用就自动创建事件循环并将调用加入, 但是在状态不确定的动态语言中要做到这一点非常困难, 很容易引发奇怪的bug. 而对于Nodejs代码来说, 不需要声明一个事件循环, 因为**其本身就在一个巨大的事件循环环境中执行** . 一个明显的例子就是在最初版本的Node中, 即使内容只有一段console.log, 执行后node进程也会因为处于事件循环而hold住不退出. 而这个事件循环对开发者是**隐藏不可见**的. 在Python/Ruby/Java等语言中, 其代码本身就不是运行在事件循环机制中的, 因此需要显式声明一个事件循环并将异步函数加入. 

同时显式声明意味着你可以创建多个事件循环并为他们注册不同的异步事件, 这听起来很自由, 不过我还没有尝试过.

## 发起非阻塞的网络IO请求

首先, await后面必须要跟随IO操作吗? 我们用以下代码测试:

```python
import asyncio

async def test():
    await print('rc')
    
    
loop = asyncio.get_event_loop()
loop.create_task(test())
loop.run_forever()
```

output:

```bash
fun
Task exception was never retrieved
future: <Task finished coro=<test() done, defined at /home/kongkongyzt/bb.py:3> exception=TypeError("object NoneType can't be used in 'await' expression",)>
Traceback (most recent call last):
  File "/usr/lib/python3.5/asyncio/tasks.py", line 239, in _step
    result = coro.send(None)
  File "/home/kongkongyzt/bb.py", line 4, in test
    await print('fun')
TypeError: object NoneType can't be used in 'await' expression
```

解释器告诉我们, await后面不是一个awaitable 类型的object, 那么什么是awaitable呢?, 通过阅读stackoverflow上的这个[回答](http://stackoverflow.com/questions/37720563/how-to-make-python-awaitable-object), 我们可以知道

>   An awaitable object is an object that defines `__await__()` method returning an iterator

await后面必须跟着一个awaitable object, await会调用该object的 \__await\__方法, 得到一个迭代器, 随着迭代的深入不断地调用迭代器的next方法直到迭代器为空. 这意味着只要是一个awaitable的对象即可, 并不止于IO操作可以被await

其次, await后面如果调用传统的blocking IO, 比如urllib2.urlopen(), 会发生什么情况呢?

答案是整个loop(事件循环)会被阻塞住. 只有等到这个阻塞的IO事件结束, 整个事件循环才能继续工作, 回调的函数才能正常触发.await 无法自动将阻塞函数转化为异步的(*因为await只处理awaitable object, 而把Python原来的所有阻塞IO接口换成awaitable的将是一场灾难*), 因此你必须使用asyncio提供的所有IO异步接口(*这些接口返回Coroutine, which will be correctly handler by await*), 才能避免IO操作阻塞事件循环.

不过这个解释还是不够好, await其实就是yield from的封装, 对于yield来说, yield后面的代码如果不返回, yield也一样会卡住, 直到返回. 所以yield后面跟着的如果是阻塞的代码, 一样是阻塞的. 

那么, 如何基于asyncio的IO接口发起异步请求呢? 在Python的[官方文档](https://docs.python.org/3/library/asyncio-stream.html)中, 提供了两种异步操作IO最底层的调用, 一种是基于回调的, 一种是基于socket的, 我简单改造了文档里面的example如下:

```python
import sys
import asyncio
import urllib.parse

async def print_http_headers(url):
    url = urllib.parse.urlsplit(url)
    if url.scheme == 'https':
        connect = asyncio.open_connection(url.hostname, 443, ssl=True)
    else:
        connect = asyncio.open_connection(url.hostname, 80)
    reader, writer = await connect
    query = ('HEAD {path} HTTP/1.0\r\n'
             'Host: {hostname}\r\n'
             '\r\n').format(path=url.path or '/', hostname=url.hostname)
    writer.write(query.encode('latin-1'))
    while True:
        line = await reader.readline()
        if not line:
            break
        line = line.decode('latin1').rstrip()
        if line:
            print(url)

    # Ignore the body, close the socket
    writer.close()

def runit():
    loop = asyncio.get_event_loop()
    tasks = [
        asyncio.ensure_future(print_http_headers('http://www.github.com')),
        asyncio.ensure_future(print_http_headers('http://www.baidu.com')),
        asyncio.ensure_future(print_http_headers('http://hiwifi.com'))
    ]
    loop.run_until_complete(asyncio.gather(*tasks))
    loop.close()

runit()
```

output:

```bash
SplitResult(scheme='http', netloc='hiwifi.com', path='', query='', fragment='')
SplitResult(scheme='http', netloc='hiwifi.com', path='', query='', fragment='')
SplitResult(scheme='http', netloc='hiwifi.com', path='', query='', fragment='')
SplitResult(scheme='http', netloc='hiwifi.com', path='', query='', fragment='')
SplitResult(scheme='http', netloc='hiwifi.com', path='', query='', fragment='')
SplitResult(scheme='http', netloc='hiwifi.com', path='', query='', fragment='')
SplitResult(scheme='http', netloc='hiwifi.com', path='', query='', fragment='')
SplitResult(scheme='http', netloc='hiwifi.com', path='', query='', fragment='')
SplitResult(scheme='http', netloc='hiwifi.com', path='', query='', fragment='')
SplitResult(scheme='http', netloc='www.baidu.com', path='', query='', fragment='')
SplitResult(scheme='http', netloc='www.baidu.com', path='', query='', fragment='')
SplitResult(scheme='http', netloc='www.baidu.com', path='', query='', fragment='')
SplitResult(scheme='http', netloc='www.baidu.com', path='', query='', fragment='')
SplitResult(scheme='http', netloc='www.baidu.com', path='', query='', fragment='')
SplitResult(scheme='http', netloc='www.baidu.com', path='', query='', fragment='')
SplitResult(scheme='http', netloc='www.baidu.com', path='', query='', fragment='')
SplitResult(scheme='http', netloc='www.baidu.com', path='', query='', fragment='')
SplitResult(scheme='http', netloc='www.baidu.com', path='', query='', fragment='')
SplitResult(scheme='http', netloc='www.baidu.com', path='', query='', fragment='')
SplitResult(scheme='http', netloc='www.baidu.com', path='', query='', fragment='')
SplitResult(scheme='http', netloc='www.github.com', path='', query='', fragment='')
SplitResult(scheme='http', netloc='www.github.com', path='', query='', fragment='')
SplitResult(scheme='http', netloc='www.github.com', path='', query='', fragment='')
SplitResult(scheme='http', netloc='www.github.com', path='', query='', fragment='')
SplitResult(scheme='http', netloc='www.github.com', path='', query='', fragment='')
SplitResult(scheme='http', netloc='www.github.com', path='', query='', fragment='')
SplitResult(scheme='http', netloc='www.github.com', path='', query='', fragment='')
SplitResult(scheme='http', netloc='www.github.com', path='', query='', fragment='')
SplitResult(scheme='http', netloc='www.github.com', path='', query='', fragment='')
```

从输出中可以看出, 在该单线程中, 首先完成IO的是hiwifi,com(我的路由器的admin地址), 其次是baidu, 最后是github, 再一次证明了其是异步执行.

## 动态添加异步任务

之前的Demo都是在代码执行前确定了异步任务, 如果是代码在执行过程中动态地添加异步任务呢? 就好像js中给DOM节点绑定on事件后, 在on事件中又触发ajax请求一样.

首先可以想出的解决方案是,  把之前通过get_event_loop()得到的事件循环以函数参数的形式传给每个需要动态添加异步任务的函数. 通过阅读几个流行的Python asyncio框架(aiohttp, mugent, sanic, ..etc)源码, 我发现这也是主流的实现方式. 不过这种方法随之带来的一个问题是, 如果函数的调用层级过深, 那么每个调用层上的函数都必须携带事件循环参数(一般就叫loop), 即使该层的函数并不需要操作loop.

从语义上来说, get_event_loop的含义, 应该是指在每次调用中都返回当前的事件循环, 而应该有别的什么接口是用来创建新的事件循环的. 等等, Python支持在同一个运行环境下创建多个事件循环吗?

当对标准库有困惑的时候, 我一般会去看官方长长长长长长长的文档, 以下是[官方文档](https://docs.python.org/3/library/asyncio-eventloops.html)中对get_event_loop的解释:

>   class ``asyncio.AbstractEventLoopPolicy``
>   Event loop policy.
>
>   ``get_event_loop()``
>   ​	Get the event loop for the current context.
>
>   ​	Returns an event loop object implementing the AbstractEventLoop interface.
>
>   ​	Raises an exception in case no event loop has been set for the current context and the current policy does 	not specify to create one. It must never return None.
>
>   ``set_event_loop(loop)``
>   ​	Set the event loop for the current context to loop.
>
>   ``new_event_loop()``
>   ​	Create and return a new event loop object according to this policy’s rules.
>
>   If there’s need to set this loop as the event loop for the current context, set_event_loop() must be called explicitly.
>

在Python执行环境中可以通过asyncio.new_event_loop()的方式创建多个事件循环, 而每次asyncio.get_event_loop(), 会得到**当前上下文环境**的事件循环.所以下面是一个推荐的demo写法

```python
import asyncio

async def simulate_on_event(loop=None):
    loop = loop or asyncio.get_event_loop()
    print('simulating on event start')
    await simulate_ajax_event(loop)
    print('simlating on event end')
    
    
async def simulate_ajax_event(loop=None):
    loop = loop or asyncio.get_event_loop()
    print('simulating ajax event start')
    await asyncio.sleep(2)
    print('simulating ajax event stop')
    
    
if __name__ == '__main__':
    loop = asyncio.get_event_loop()
    loop.create_task(simulate_on_event())
    loop.run_forever()
```

## create_task VS ensure_future

从上面的几个demo可以看到, 有时候我们通过create_task()注册异步任务, 有时候我们通过ensure_future注册异步任务. 这有什么不同吗?

熟悉Node的应该知道Promise, Python的Future/task其实和Promise很像, Future/task返回的是携带异步状态的对象, 当状态变更为完成的时候, 触发iteror执行next()

阅读PEP492可以知道, asyncio.ensure_future就是之前的acyncio.async, 因为在3.5中添加了async关键字所以改名为ensure_future. create_task只能接受coroutine, 而ensure_future只需要是一个awaitable object即可, 在大多数情况下, 建议使用ensure_future

## 为什么是async/await

在看到Python支持async/await语法的新闻并阅读了demo后, 当时我产生了两个疑惑

+   为什么需要显式申明事件循环?
+   从语言设计的角度来说, 其实只需要await一个关键字即可, 为什么需要async关键字?

第一个疑问在本文``事件循环``一节已经解释, 第二个疑问官方pep492已经有了[解答](https://www.python.org/dev/peps/pep-0492/#importance-of-async-keyword)

>   ##### Importance of "async" keyword
>
>   While it is possible to just implement await expression and treat all functions with at least one await as coroutines, this approach makes APIs design, code refactoring and its long time support harder.
>
>   Let's pretend that Python only has await keyword:
>
>   ```python
>   def useful():
>       ...
>       await log(...)
>       ...
>
>   def important():
>       await useful()
>   ```
>
>   If useful() function is refactored and someone removes all await expressions from it, it would become a regular python function, and all code that depends on it, including important() would be broken. To mitigate this issue a decorator similar to @asyncio.coroutine has to be introduced.

有趣的是, 这份链接中同时指出了为什么不设计成 def async, 为什么不设计成await with/await for 之类的问题. 我喜欢Python的原因有很多, 其中一个就是Python社区永远是well documentation的, 官方文档不仅会事无巨细地告诉你怎么使用接口, 甚至是接口的实现细节, 还会告诉你语言特性的实现思路, 以及为什么不设计成这样/那样, You are not only supposed to be a user, but the designer who need to know the detail of the inner.

## 小礼物

Python的async确实是Python3中的一个亮点, 不少基于这个特性的第三方库和框架已经开始出现了. 前段时间做某个项目需要用到Python实现websocket server, 于是就顺手使用这个新出的异步特性实现了, 现在开源在这里: [moli](https://github.com/stormeyes/moli) , 算是给开源做出的小贡献吧, 欢迎star!