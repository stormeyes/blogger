# 聊聊异步(二)

## 回到yield

回到之前失败的那个yield, 关键在于这里

```python
yield urllib2.urlopen(url)
```

实际上对于yield来说, 它能做的只是**在yield后面的语句返回后**再冻结上下文状态并将控制权返回给调用者. urllib2.urlopen一直hold住直到请求成功拿到数据后才返回, 而urllib2.urlopen()本身是基于python的底层同步IO接口包装实现的, 这个同步接口本身就是hold住才会返回, 然后逐层返回到上层包装起来的urllib2.urlopen(), 因此yield会被阻塞住. 想要实现异步的效果, 只有调用Python提供异步的IO接口, 一调用就立马返回(返回的不是数据, 一般是一个handler), yield拿到返回后得以立马冻结上下文, 脱离环境后返还执行权限给调用者.

说到底, 还是要底层提供异步的接口, yield只是一种调度的手段而已.

## epoll

其实我不是很想花费篇幅介绍epoll, 作为一个如此出名的东西, 网上的介绍早就成吨了. 然而考虑到epoll的实现与异步相关太紧密, 这里还是要做一下介绍.

在epoll出现之前, 先后出现了select和poll这两个异步IO接口, 但是它们表现得让人失望, 直到epoll的出现.poll/select和epoll的调用接口不同，select和poll都只提供了一个函数——select或者poll函数。而epoll提供了三个函数，epoll_create,epoll_ctl和epoll_wait，epoll_create是创建一个epoll句柄；epoll_ctl是注册要监听的事件类型；epoll_wait则是等待事件的产生。epoll克服了SELECT/POLL的三个重要缺点:

+   每次注册新的事件到epoll句柄中时（在epoll_ctl中指定EPOLL_CTL_ADD），会把所有的fd拷贝进内核，而不是在epoll_wait的时候重复拷贝。epoll保证了每个fd在整个过程中只会拷贝一次。
+   epoll的解决方案不像select或poll一样每次都把current轮流加入fd对应的设备等待队列中，而只在epoll_ctl时把current挂一遍（这一遍必不可少）并为每个fd指定一个回调函数，当设备就绪，唤醒等待队列上的等待者时，就会调用这个回调函数，而这个回调函数会把就绪的fd加入一个就绪链表）。epoll_wait的工作实际上就是在这个就绪链表中查看有没有就绪的fd（利用schedule_timeout()实现睡一会，判断一会的效果，和select实现中的第7步是类似的）。
+   epoll没有文件描述符数量限制，它所支持的FD上限是最大可以打开文件的数目，这个数字一般远大于2048,举个例子,在1GB内存的机器上大约是10万左右，具体数目可以cat /proc/sys/fs/file-max察看,一般来说这个数目和系统内存关系很大。

epoll只是关于网络IO的异步实现, 和磁盘IO无关, 因为epoll中, 触发回调的信号来自网卡硬件的ready事件.

从epoll的实现中你可以发现, 无论是select, poll还是epoll(这些称为IO多路复用)所谓异步的实现, 本质上都是在同步阻塞轮询.

## 异步磁盘IO

IO 包括两种, 网络IO和磁盘IO, 对于网络IO即socket, 我们有epoll这个工具实现异步; 对于磁盘IO, Linux提供了AIO, 悲伤的是, AIO有以下问题:

+   AIO需要引入一个Linux系统默认没有的外部库
+   AIO在高并发操作的情况下无法有效扩展

其实我并没有明白上面这句话的意思, 如果你有兴趣, 可以阅读[这篇文章](https://github.com/python/asyncio/wiki/ThirdParty#filesystem)的FIleSystem一节并和我分享一下你的想法.

因此, Python的asyncio并不支持file async IO. 但是, 为什么Nodejs可以实现文件的异步读写呢? 因为Nodejs实现异步的基础--libuv通过多线程来模拟了异步读写IO, 而通过多线程来模拟异步IO正是大多是异步文件IO库所采用的方法.

等等?! Nodejs? 多线程? Nodejs不是单线程吗? 为什么会有多线程?

如果你有这个疑问, 说明你并不真正理解Nodejs的运行机制, 在Nodejs中, 除了你自己的代码, 所有的东西都是并行的! 这句话来自Nodejs的作者的文章, 你可以点击[这里](http://debuggable.com/posts/understanding-node-js:4bd98440-45e4-4a9a-8ef7-0f7ecbdd56cb)进行阅读.

Python社区的一个贡献者提供了一个基于多线程的异步文件IO实现, 见[这里](https://github.com/Tinche/aiofiles/)

## uvloop

还记得Nodejs的基石--libuv吗? 就是这个神奇的东西给js带来了异步IO的能力, 在Nodejs发布后, 这个使用C++实现的东西被独立出来,可以使用在任何语言上! 有人将libuv转到了Python下, 命名为uvloop. uvloop提供了和asyncio一致的接口并带来了异步磁盘IO的实现, 并且从数据上来看, uvloop表现得非常优秀!

## Reference

[这里](https://pymotw.com/3/asyncio/control.html)有一个关于Python各个模块的教程的指引, 通过阅读asyncio相关的模块, 能获取更多与异步有关的信息