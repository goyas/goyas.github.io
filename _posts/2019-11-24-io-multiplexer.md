# 背景
在文章[《unix网络编程》（12）五种I/O模型](https://blog.csdn.net/u013074465/article/details/44876081)中提到了五种I/O模型，其中前四种：阻塞模型、非阻塞模型、信号驱动模型、I/O复用模型都是同步模型；还有一种是异步模型。  

对于从I/O多路复用到异步编程和RPC框架，想写一个系列的文章介绍这个演进过程，这一系列可能包括：  
1. IO多路复用模型
2. poll介绍与使用
3. Reactor和Proactor模型
4. 为什么需要异步编程
5. enable_shared_from_this用法分析
6. 网络通信库和RPC

# 为什么有多路复用？
多路复用技术要解决的是“通信”问题，解决核心在于“同步事件分离器”（de-multiplexer），linux系统带有的分离器select、poll、epoll网上介绍的比较多，大家可以看看这篇介绍的不错的文章：[我读过的最好的epoll讲解](https://zhuanlan.zhihu.com/p/36764771)。通信的一方想要知道另一方的状态(以决定自己做什么)，有两种方法: 一是轮询，二是消息通知。
## 轮询
轮询的一种典型的实现可能是这样的：当然这里的epoll_wait()也可以使用poll()或者select()替换。
```
while （true） {
    active_stream[] = epoll_wait(epollfd)
    for i in active_stream[] {
        read or write till
    }
}
```

轮询方式主要存在以下不足：
* 增加系统开销。无论是任务轮询还是定时器轮询都需要消耗对应的系统资源。
* 无法及时感知设备状态变化。在轮询间隔内的设备状态变化只有在下次轮询时才能被发现，这将无法满足对实时性敏感的应用场合。
* 浪费CPU资源。无论设备是否发生状态改变，轮询总在进行。在实际情况中，大多数设备的状态改变通常不会那么频繁，轮询空转将白白浪费CPU时间片。

## 消息通知
其实现方式通常是: "阻塞-通知"机制。阻塞会导致一个任务(task_struct，进程或者线程)只能处理一个"I/O流"或者类似的操作，要处理多个，就要多个任务(需要多个进程或线程)，因此灵活性上又不如轮询(一个任务足够)，很矛盾。  

矛盾的根源就是"一"和"多"的矛盾: 希望一个任务处理多个对象，同时避免处理阻塞-通知机制的内部细节。解决方案是多路复用(muliplex)。多路复用有3种基本方案，select()/poll()/epoll()，都是来解决这一矛盾的。  
* 通知代理: 用户把需要关心的对象注册给select()/poll()/epoll()函数。
* 一对多: 所有的被关心的对象，只要有一个对象有了通知事件，select()/poll()/epoll()就会结束阻塞状态。
* 方便性: 用户(程序员)不用再关心如何阻塞和被通知，以及哪些情况下会有通知产生。这件事情已经由上述几个系统调用做了，用户只需要实现"通知来了我该做什么"。

那么上面3个系统调用的区别是什么呢？  
第一个select()，结合了轮询和阻塞两种方式，没有问题，每次有一个对象事件发生的时候，select()只是知道有事件发生了，具体是哪个对象发生的，不知道，需要从头到尾轮询一遍，复杂度是O(n)。poll函数相对select函数变化不大，只是提升了最大的可轮询的对象个数。epoll函数把时间复杂度降到O(1)。

为什么select慢而epoll效率高？  
select()之所以慢，有几个原因: select()的参数是一个FD数组，意味着每次select调用，都是一次新的注册-阻塞-回调，每次select都要把一个数组从用户空间拷贝到内核空间，内核检测到某个对象状态变化并写入后，再从内核空间拷贝回用户空间，select再把这个数组读取一遍，并返回。这个过程非常低效。  

epoll的解决方案相当于是一种对select()的算法优化: 它把select()一个函数做的事情分解成了3步，首先epoll_create()创建一个epollfd对象(相当于一个池子)，然后所有被监听的fd通过epoll_ctrl()注册到这个池子，也就是为每个fd指定了一个内部的回调函数(这样，就没有了每次调用时的来回拷贝，用户空间的数组到内核空间只有这一次拷贝)。epoll_wait阻塞等待。在内核态有一个和epoll_wait对应的函数调用，把就绪的fd，填入到一个就绪列表中，而epoll_wait读取这个就绪列表，做到了快速返回(O(1))。

详细的对比可以参考[select、poll、epoll之间的区别总结](https://www.cnblogs.com/Anker/p/3265058.html?spm=ata.13261165.0.0.4ec468f3ruw05F)

有了上面的原理介绍，这里举例来说明下epoll到底是怎么使用的，加深理解。举两个例子：  
* 一个是比较简单的父子进程通信的例子，单个小程序，不需要跑多个应用实例，不需要用户输入。  
* 一个是比较实战的socket+epoll，毕竟现实案例中哪有两个父子进程间通讯这么简单的应用场景。  

# 有了多路复用，难道还不够？
有了I/O复用，有了epoll已经可以使服务器并发几十万连接的同时，维持高TPS了，难道这还不够吗？答案是，技术层面足够了，但在软件工程层面却是不够的。例如，总要有个for循环去调用epoll，总来处理epoll的返回，这是每次都要重复的工作。for循环体里面写什么----通知返回之后，做事情的程序最好能以一种回调的机制，提供一个编程框架，让程序更有结构一些。另一方面，如果希望每个事件通知之后，做的事情能有机会被代理到某个线程里面去单独运行，而线程完成的状态又能通知回主任务，那么"异步"的进制就必须被引入。

所以，还有两个问题要解决，一是"编程框架"，一是"异步"。我们先看几个目前流行的框架，大部分框架已经包含了某种异步的机制。我们接下来的篇章将介绍“编程框架”和“异步I/O模型”。
