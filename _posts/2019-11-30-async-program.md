---
layout: post
title: 为什么需要异步编程
date: 2019-11-30 16:15
categories: network
tag: boost.Asio
excerpt: 为什么需要异步编程及异步编程实现
---

# 一、背景  
---  
在[Reactor和Proactor模型](https://goyas.github.io/reactor-proactor/)一文中讲到，Reactor模型提供了一个比较理想的I/O编程框架，让程序更有结构，用户使用起来更加方便，比裸API调用开发效率要高。另外一方面，如果希望每个事件通知之后，做的事情能有机会被代理到某个线程里面去单独运行，而线程完成的状态又能通知回主任务，那么”异步”的机制就必须被引入。本文以boost.Asio库（其设计模式为Proactor）为基础，讲解为什么需要异步编程以及异步编程的实现。  

# 二、举例   
---  
## 跑步
设想你是一位体育老师，需要测验100位同学的400米成绩。你当然不会让100位同学一起起跑，因为当同学们返回终点时，你根本来不及掐表记录各位同学的成绩。  

如果你每次让一位同学起跑并等待他回到终点你记下成绩后再让下一位起跑，直到所有同学都跑完。恭喜你，你已经掌握了***同步阻塞***模式。你设计了一个函数，传入参数是学生号和起跑时间，返回值是到达终点的时间。你调用该函数100次，就能完成这次测验任务。这个函数是同步的，因为只要你调用它，就能得到结果；这个函数也是阻塞的，因为你一旦调用它，就必须等待，直到它给你结果，不能去干其他事情。   

如果你一边每隔10秒让一位同学起跑，直到所有同学出发完毕；另一边每有一个同学回到终点就记录成绩，直到所有同学都跑完。恭喜你，你已经掌握了***异步非阻塞***模式。你设计了两个函数，其中一个函数记录起跑时间和学生号，该函数你会主动调用100次；另一个函数记录到达时间和学生号，该函数是一个事件驱动的callback函数，当有同学到达终点时，你会被动调用。你主动调用的函数是异步的，因为你调用它，它并不会告诉你结果；这个函数也是非阻塞的，因为你一旦调用它，它就马上返回，你不用等待就可以再次调用它。但仅仅将这个函数调用100次，你并没有完成你的测验任务，你还需要被动等待调用另一个函数100次。  

当然，你马上就会意识到，同步阻塞模式的效率明显低于异步非阻塞模式。那么，谁还会使用同步阻塞模式呢？不错，异步模式效率高，但更麻烦，你一边要记录起跑同学的数据，一边要记录到达同学的数据，而且同学们回到终点的次序与起跑的次序并不相同，所以你还要不停地在你的成绩册上查找学生号。忙乱之中你往往会张冠李戴。你可能会想出更聪明的办法：你带了很多块秒表，让同学们分组互相测验。恭喜你！你已经掌握了***多线程同步模式***！   

每个拿秒表的同学都可以独立调用你的同步函数，这样既不容易出错，效率也大大提高，只要秒表足够多，同步的效率也能达到甚至超过异步。   

***可以理解，你现的问题可能是：既然多线程同步既快又好，异步模式还有存在的必要吗？***   

***很遗憾，异步模式依然非常重要，因为在很多情况下，你拿不出很多秒表。你需要通信的对端系统可能只允许你建立一个SOCKET连接，很多金融、电信行业的大型业务系统都如此要求。***    

# 三、背景知识介绍   
---  
## 3.1 同步函数VS异步函数
以下部分主要来自于：https://www.cnblogs.com/balingybj/p/4780442.html  
依据微软的MSDN上的解说：  
（1）、同步函数：当一个函数是同步执行时，那么当该函数被调用时不会立即返回，直到该函数所要做的事情全都做完了才返回。  
（2）、异步函数：如果一个异步函数被调用时，该函数会立即返回尽管该函数规定的操作任务还没有完成。  
（3）、在一个线程中分别调用上述两种函数会对调用线程有何影响呢？  
* 当一个线程调用一个同步函数时（例如：该函数用于完成写文件任务），如果该函数没有立即完成规定的操作，则该操作会导致该调用线程的挂起（将CPU的使用权交给系统，让系统分配给其他线程使用），直到该同步函数规定的操作完成才返回，最终才能导致该调用线程被重新调度。  
* 当一个线程调用的是一个异步函数（例如：该函数用于完成写文件任务），该函数会立即返回尽管其规定的任务还没有完成，这样线程就会执行异步函数的下一条语句，而不会被挂起。那么该异步函数所规定的工作是如何被完成的呢？当然是通过另外一个线程完成的了啊；那么新的线程是哪里来的呢？可能是在异步函数中新创建的一个线程也可能是系统中已经准备好的线程。    

（4）、一个调用了异步函数的线程如何与异步函数的执行结果同步呢？  
* 为了解决该问题，调用线程需要使用“等待函数”来确定该异步函数何时完成了规定的任务。因此在线程调用异步函数之后立即调用一个“等待函数”挂起调用线程，一直等到异步函数执行完其所有的操作之后，再执行线程中的下一条指令。  

我们是否已经发现了一个有趣的地方呢？！就是我们可以使用等待函数将一个异步执行的函数封装成一个同步函数。

## 3.2 同步调用VS异步调用  
操作系统发展到今天已经十分精巧，线程就是其中一个杰作。操作系统把 CPU 处理时间划分成许多短暂时间片，在时间 T1 执行一个线程的指令，到时间 T2 又执行下一线程的指令，各线程轮流执行，结果好象是所有线程在并肩前进。这样，编程时可以创建多个线程，在同一期间执行，各线程可以“并行”完成不同的任务。   

在单线程方式下，计算机是一台严格意义上的冯·诺依曼式机器，一段代码调用另一段代码时，只能采用同步调用，必须等待这段代码执行完返回结果后，调用方才能继续往下执行。有了多线程的支持，可以采用异步调用，调用方和被调方可以属于两个不同的线程，调用方启动被调方线程后，不等对方返回结果就继续执行后续代码。被调方执行完毕后，通过某种手段通知调用方：结果已经出来，请酌情处理。　　

计算机中有些处理比较耗时。调用这种处理代码时，调用方如果站在那里苦苦等待，会严重影响程序性能。例如，某个程序启动后如果需要打开文件读出其中的数据，再根据这些数据进行一系列初始化处理，程序主窗口将迟迟不能显示，让用户感到这个程序怎么等半天也不出来，太差劲了。借助异步调用可以把问题轻松化解：把整个初始化处理放进一个单独线程，主线程启动此线程后接着往下走，让主窗口瞬间显示出来。等用户盯着窗口犯呆时，初始化处理就在背后悄悄完成了。程序开始稳定运行以后，还可以继续使用这种技巧改善人机交互的瞬时反应。用户点击鼠标时，所激发的操作如果较费时，再点击鼠标将不会立即反应，整个程序显得很沉重。借助异步调用处理费时的操作，让主线程随时恭候下一条消息，用户点击鼠标时感到轻松快捷，肯定会对软件产生好感。        

异步调用用来处理从外部输入的数据特别有效。假如计算机需要从一台低速设备索取数据，然后是一段冗长的数据处理过程，采用同步调用显然很不合算：计算机先向外部设备发出请求，然后等待数据输入；而外部设备向计算机发送数据后，也要等待计算机完成数据处理后再发出下一条数据请求。双方都有一段等待期，拉长了整个处理过程。其实，计算机可以在处理数据之前先发出下一条数据请求，然后立即去处理数据。如果数据处理比数据采集快，要等待的只有计算机，外部设备可以连续不停地采集数据。如果计算机同时连接多台输入设备，可以轮流向各台设备发出数据请求，并随时处理每台设备发来的数据，整个系统可以保持连续高速运转。编程的关键是把数据索取代码和数据处理代码分别归属两个不同的线程。数据处理代码调用一个数据请求异步函数，然后径自处理手头的数据。待下一组数据到来后，数据处理线程将收到通知，结束 wait 状态，发出下一条数据请求，然后继续处理数据。        

异步调用时，调用方不等被调方返回结果就转身离去，**因此必须有一种机制让被调方有了结果时能通知调用方**。在同一进程中有很多手段可以利用，笔者常用的手段是回调、event 对象和消息。        

**回调**：回调方式很简单：调用异步函数时在参数中放入一个函数地址，异步函数保存此地址，待有了结果后回调此函数便可以向调用方发出通知。如果把异步函数包装进一个对象中，可以用事件取代回调函数地址，通过事件处理例程向调用方发通知。  　　

**event** ： event 是 Windows 系统提供的一个常用同步对象，以在异步处理中对齐不同线程之间的步点。如果调用方暂时无事可做，可以调用 wait 函数等在那里，此时 event 处于 nonsignaled 状态。当被调方出来结果之后，把 event 对象置于 signaled 状态，wait 函数便自动结束等待，使调用方重新动作起来，从被调方取出处理结果。这种方式比回调方式要复杂一些，速度也相对较慢，但有很大的灵活性，可以搞出很多花样以适应比较复杂的处理系统。        

**消息**：借助 Windows 消息发通知是个不错的选择，既简单又安全。程序中定义一个用户消息，并由调用方准备好消息处理例程。被调方出来结果之后立即向调用方发送此消息，并通过 WParam 和 LParam 这两个参数传送结果。消息总是与窗口 handle 关联，因此调用方必须借助一个窗口才能接收消息，这是其不方便之处。另外，通过消息联络会影响速度，需要高速处理时回调方式更有优势。  

如果调用方和被调方分属两个不同的进程，由于内存空间的隔阂，一般是采用 Windows 消息发通知比较简单可靠，被调方可以借助消息本身向调用方传送数据。event 对象也可以通过名称在不同进程间共享，但只能发通知，本身无法传送数据，需要借助 Windows 消息和 FileMapping 等内存共享手段或借助 MailSlot 和 Pipe 等通信手段。       

如果你的服务端的客户端数量多，你的服务端就采用异步的，但是你的客户端可以用同步的，客户端一般功能比较单一，收到数据后才能执行下面的工作，所以弄成同步的在那等。    

## 3.3 同步异步与阻塞和非阻塞
同步异步指的是通信模式，而阻塞和非阻塞指的是在接收和发送时是否等待动作完成才返回。   

首先是通信的同步，主要是指客户端在发送请求后，必须得在服务端有回应后才发送下一个请求。所以这个时候的所有请求将会在服务端得到同步。  
其次是通信的异步，指客户端在发送请求后，不必等待服务端的回应就可以发送下一个请求，这样对于所有的请求动作来说将会在服务端得到异步，这条请求的链路就象是一个请求队列，所有的动作在这里不会得到同步的。  

阻塞和非阻塞只是应用在请求的读取和发送。   
在实现过程中，如果服务端是异步的话，客户端也是异步的话，通信效率会很高，但如果服务端在请求的返回时也是返回给请求的链路时，客户端是可以同步的，这种情况下，服务端是兼容同步和异步的。相反，如果客户端是异步而服务端是同步的也不会有问题，只是处理效率低了些。   

### 举个打电话的例子  
阻塞 block 是指，你拨通某人的电话，但是此人不在，于是你拿着电话等他回来，其间不能再用电话。同步大概和阻塞差不多。   
非阻塞 nonblock 是指，你拨通某人的电话，但是此人不在，于是你挂断电话，待会儿再打。至于到时候他回来没有，只有打了电话才知道。即所谓的“轮询 / poll”。   
异步是指，你拨通某人的电话，但是此人不在，于是你叫接电话的人告诉那人(leave a message)，回来后给你打电话（call back）。

一、同步阻塞模式   
在这个模式中，用户空间的应用程序执行一个系统调用，并阻塞，直到系统调用完成为止(数据传输完成或发生错误)。    

二、同步非阻塞模式    
同步阻塞 I/O 的一种效率稍低的。非阻塞的实现是 I/O 命令可能并不会立即满足，需要应用程序调用许多次来等待操作完成。这可能效率不高，因为在很多情况下，当内核执行这个命令时，应用程序必须要进行忙碌等待，直到数据可用为止，或者试图执行其他工作。因为数据在内核中变为可用到用户调用 read 返回数据之间存在一定的间隔，这会导致整体数据吞吐量的降低。但异步非阻塞由于是多线程，效率还是高。   

```c
/* create the connection by socket 
* means that connect "sockfd" to "server_addr"
* 同步阻塞模式 
*/
if (connect(sockfd, (struct sockaddr *)&server_addr, sizeof(struct sockaddr)) == -1)
{
perror("connect");
exit(1);
}

/* 同步非阻塞模式 */
while (send(sockfd, snd_buf, sizeof(snd_buf), MSG_DONTWAIT) == -1)
{
sleep(1);
printf("sleep\n");
}
```

# 四、异步的需求    
---  
前面说了那么多，现在终于可以回到我们的正题，介绍异步编程了。就像之前所说的，同步编程比异步编程简单很多。这是因为，线性的思考是很简单的（调用A，调用A结束，调用B，调用B结束，然后继续，这是以事件处理的方式来思考）。后面你会碰到这种情况，比如：五件事情，你不知道它们执行的顺序，也不知道他们是否会执行！这部分主要参考：https://mmoaay.gitbooks.io/boost-asio-cpp-network-programming-chinese/content/Chapter2.html     

尽管异步编程更难，但是你会更倾向于选择使用它，比如：写一个需要处理很多并发访问的服务端。并发访问越多，异步编程就比同步编程越简单。  

假设：你有一个需要处理1000个并发访问的应用，从客户端发给服务端的每个信息都会再返回给客户端，以‘\n’结尾。  

同步方式的代码，1个线程：
```c++
using namespace boost::asio;
struct client {
    ip::tcp::socket sock;
    char buff[1024]; // 每个信息最多这么大
    int already_read; // 你已经读了多少
};
std::vector<client> clients;
void handle_clients() {
    while ( true)
        for ( int i = 0; i < clients.size(); ++i)
            if ( clients[i].sock.available() ) on_read(clients[i]);
}
void on_read(client & c) {
    int to_read = std::min( 1024 - c.already_read, c.sock.available());
    c.sock.read_some( buffer(c.buff + c.already_read, to_read));
    c.already_read += to_read;
    if ( std::find(c.buff, c.buff + c.already_read, '\n') < c.buff + c.already_read) {
        int pos = std::find(c.buff, c.buff + c.already_read, '\n') - c.buff;
        std::string msg(c.buff, c.buff + pos);
        std::copy(c.buff + pos, c.buff + 1024, c.buff);
        c.already_read -= pos;
        on_read_msg(c, msg);
    }
}
void on_read_msg(client & c, const std::string & msg) {
    // 分析消息，然后返回
    if ( msg == "request_login")
        c.sock.write( "request_ok\n");
    else if ...
}
```

有一种情况是在任何服务端（和任何基于网络的应用）都需要避免的，就是代码无响应的情况。在我们的例子里，我们需要**handle_clients()**方法尽可能少的阻塞。如果方法在某个点上阻塞，任何进来的信息都需要等待方法解除阻塞才能被处理。   

为了保持响应，只在一个套接字有数据的时候我们才读，也就是说，**if ( clients[i].sock.available() ) on_read(clients[i])**。在**on_read**时，我们只读当前可用的；调用**read_until(c.sock, buffer(...),  '\n')**会是一个非常糟糕的选择，因为直到我们从一个指定的客户端读取了完整的消息之前，它都是阻塞的（我们永远不知道它什么时候会读取到完整的消息）  

这里的瓶颈就是**on_read_msg()**方法；当它执行时，所有进来的消息都在等待。一个良好的**on_read_msg()**方法实现会保证这种情况基本不会发生，但是它还是会发生（有时候向一个套接字写入数据，缓冲区满了时，它会被阻塞）
同步方式的代码，10个线程   
```c++
using namespace boost::asio;
struct client {
　  // ... 和之前一样
    bool set_reading() {
        boost::mutex::scoped_lock lk(cs_);
        if ( is_reading_) return false; // 已经在读取
        else { is_reading_ = true; return true; }
    }
    void unset_reading() {
        boost::mutex::scoped_lock lk(cs_);
        is_reading_ = false;
    }
private:
    boost::mutex cs_;
    bool is_reading_;
};
std::vector<client> clients;
void handle_clients() {
    for ( int i = 0; i < 10; ++i)
        boost::thread( handle_clients_thread);
}
void handle_clients_thread() {
    while ( true)
        for ( int i = 0; i < clients.size(); ++i)
            if ( clients[i].sock.available() )
                if ( clients[i].set_reading()) {
                    on_read(clients[i]);
                    clients[i].unset_reading();
                }
}
void on_read(client & c) {
    // 和之前一样
}
void on_read_msg(client & c, const std::string & msg) {
    // 和之前一样
}
```

为了使用多线程，我们需要对线程进行同步，这就是**set_reading()**和**set_unreading()**所做的。**set_reading()**方法非常重要，比如你想要一步实现“判断是否在读取然后标记为读取中”。但这是有两步的（“判断是否在读取”和“标记为读取中”），你可能会有两个线程同时为一个客户端判断是否在读取，然后你会有两个线程同时为一个客户端调用**on_read**，结果就是数据冲突甚至导致应用崩溃。  

你会发现代码变得极其复杂。  

同步编程有第三个选择，就是为每个连接开辟一个线程。但是当并发的线程增加时，这就成了一种灾难性的情况。  

然后，让我们来看异步编程。我们不断地异步读取。当一个客户端请求某些东西时，**on_read**被调用，然后回应，然后等待下一个请求（然后开始另外一个异步的read操作）。

异步方式的代码，10个线程
```c++
using namespace boost::asio;
io_service service;
struct client {
    ip::tcp::socket sock;
    streambuf buff; // 从客户端取回结果
}
std::vector<client> clients;
void handle_clients() {
    for ( int i = 0; i < clients.size(); ++i)
        async_read_until(clients[i].sock, clients[i].buff, '\n', boost::bind(on_read, clients[i], _1, _2));
    for ( int i = 0; i < 10; ++i)
        boost::thread(handle_clients_thread);
}
void handle_clients_thread() {
    service.run();
}
void on_read(client & c, const error_code & err, size_t read_bytes) {
    std::istream in(&c.buff);
    std::string msg;
    std::getline(in, msg);
    if ( msg == "request_login")
        c.sock.async_write( "request_ok\n", on_write);
    else if ...
    ...
    // 等待同一个客户端下一个读取操作
    async_read_until(c.sock, c.buff, '\n', boost::bind(on_read, c, _1, _2));
}
```

发现代码变得有多简单了吧？client结构里面只有两个成员，**handle_clients()**仅仅调用了**async_read_until**，然后它创建了10个线程，每个线程都调用**service.run()**。这些线程会处理所有来自客户端的异步read操作，然后分发所有向客户端的异步write操作。另外需要注意的一件事情是：**on_read()**一直在为下一次异步read操作做准备（看最后一行代码）。

## 4.1 持续运行
再一次说明，如果有等待执行的操作，**run()**会一直执行，直到你手动调用**io_service::stop()**。为了保证**io_service**一直执行，通常你添加一个或者多个异步操作，然后在它们被执行时，你继续一直不停地添加异步操作，比如下面代码：
```c++
using namespace boost::asio;
io_service service;
ip::tcp::socket sock(service);
char buff_read[1024], buff_write[1024] = "ok";
void on_read(const boost::system::error_code &err, std::size_t bytes);
void on_write(const boost::system::error_code &err, std::size_t bytes)
{
    sock.async_read_some(buffer(buff_read), on_read);
}
void on_read(const boost::system::error_code &err, std::size_t bytes)
{
    // ... 处理读取操作 ...
    sock.async_write_some(buffer(buff_write,3), on_write);
}
void on_connect(const boost::system::error_code &err) {
    sock.async_read_some(buffer(buff_read), on_read);
}
int main(int argc, char* argv[]) {
    ip::tcp::endpoint ep( ip::address::from_string("127.0.0.1"), 2001);
    sock.async_connect(ep, on_connect);
    service.run();
}
```

1. 当*service.run()*被调用时，有一个异步操作在等待。
2. 当socket连接到服务端时，*on_connect*被调用了，它会添加一个异步操作。
3. 当*on_connect*结束时，我们会留下一个等待的操作（*read*）。
4. 当*on_read*被调用时，我们写入一个回应，这又添加了另外一个等待的操作。
5. 当*on_read*结束时，我们会留下一个等待的操作*（write*）。
6. 当*on_write*操作被调用时，我们从服务端读取另外一个消息，这也添加了另外一个等待的操作。
7. 当*on_write*结束时，我们有一个等待的操作（read）。
8. 然后一直继续循环下去，直到我们关闭这个应用。

## 4.2 保持活动
假设你需要做下面的操作：
```
io_service service;
ip::tcp::socket sock(service);
char buff[512];
...
read(sock, buffer(buff));
```

在这个例子中，*sock*和*buff*的存在时间都必须比*read()*调用的时间要长。也就是说，在调用*read()*返回之前，它们都必须有效。这就是你所期望的；你传给一个方法的所有参数在方法内部都必须有效。当我们采用异步方式时，事情会变得比较复杂。
```
io_service service;
ip::tcp::socket sock(service);
char buff[512];
void on_read(const boost::system::error_code &, size_t) {}
...
async_read(sock, buffer(buff), on_read);
```

在这个例子中，*sock*和*buff*的存在时间都必须比*read()*操作本身时间要长，但是read操作持续的时间我们是不知道的，因为它是异步的。

当使用socket缓冲区的时候，你会有一个*buffer*实例在异步调用时一直存在（使用*boost::shared_array<>*）。在这里，我们可以使用同样的方式，通过创建一个类并在其内部管理socket和它的读写缓冲区。然后，对于所有的异步操作，传递一个包含智能指针的*boost::bind*仿函数给它：
```c++
using namespace boost::asio;
io_service service;
struct connection : boost::enable_shared_from_this<connection> {
    typedef boost::system::error_code error_code;
    typedef boost::shared_ptr<connection> ptr;
    connection() : sock_(service), started_(true) {}
    void start(ip::tcp::endpoint ep) {
        sock_.async_connect(ep, boost::bind(&connection::on_connect, shared_from_this(), _1));
    }
    void stop() {
        if ( !started_) return;
        started_ = false;
        sock_.close();
    }
    bool started() { return started_; }
private:
    void on_connect(const error_code & err) {
        // 这里你决定用这个连接做什么: 读取或者写入
        if ( !err) do_read();
        else stop();
    }
    void on_read(const error_code & err, size_t bytes) {
        if ( !started() ) return;
        std::string msg(read_buffer_, bytes);
        if ( msg == "can_login") do_write("access_data");
        else if ( msg.find("data ") == 0) process_data(msg);
        else if ( msg == "login_fail") stop();
    }
    void on_write(const error_code & err, size_t bytes) {
        do_read(); 
    }
    void do_read() {
        sock_.async_read_some(buffer(read_buffer_), boost::bind(&connection::on_read, shared_from_this(),   _1, _2)); 
    }
    void do_write(const std::string & msg) {
        if ( !started() ) return;
        // 注意: 因为在做另外一个async_read操作之前你想要发送多个消息, 
        // 所以你需要多个写入buffer
        std::copy(msg.begin(), msg.end(), write_buffer_);
        sock_.async_write_some(buffer(write_buffer_, msg.size()), boost::bind(&connection::on_write, shared_from_this(), _1, _2)); 
    }

    void process_data(const std::string & msg) {
        // 处理服务端来的内容，然后启动另外一个写入操作
    }
private:
    ip::tcp::socket sock_;
    enum { max_msg = 1024 };
    char read_buffer_[max_msg];
    char write_buffer_[max_msg];
    bool started_;
};

int main(int argc, char* argv[]) {
    ip::tcp::endpoint ep( ip::address::from_string("127.0.0.1"), 8001);
    connection::ptr(new connection)->start(ep);
} 
```

在所有异步调用中，我们传递一个**boost::bind**仿函数当作参数。这个仿函数内部包含了一个智能指针，指向**connection**实例。只要有一个异步操作等待时，Boost.Asio就会保存**boost::bind**仿函数的拷贝，这个拷贝保存了指向连接实例的一个智能指针，从而保证**connection**实例保持活动。问题解决！  

当然，**connection**类仅仅是一个框架类；你需要根据你的需求对它进行调整（它看起来会和当前服务端例子的情况相当不同）。   

你需要注意的是创建一个新的连接是相当简单的：**connection::ptr(new connection)- >start(ep)**。这个方法启动了到服务端的（异步）连接。当你需要关闭这个连接时，调用**stop()**。   

当实例被启动时（**start()**），它会等待客户端的连接。当连接发生时。**on_connect()**被调用。如果没有错误发生，它启动一个read操作（**do_read()**）。当read操作结束时，你就可以解析这个消息；当然你应用的**on_read()**看起来会各种各样。而当你写回一个消息时，你需要把它拷贝到缓冲区，然后像我在**do_write()**方法中所做的一样将其发送出去，因为这个缓冲区同样需要在这个异步写操作中一直存活。最后需要注意的一点——当写回时，你需要指定写入的数量，否则，整个缓冲区都会被发送出去。    

# 总结
网络api实际上要繁杂得多，这个章节只是做为一个参考，当你在实现自己的网络应用时可以回过头来看看。  

Boost.Asio实现了端点的概念，你可以认为是IP和端口。如果你不知道准确的IP，你可以使用**resolver**对象将主机名，例如**www.yahoo.com**转换为一个或多个IP地址。   

我们也可以看到API的核心——socket类。Boost.Asio提供了**TCP、UDP**和 **ICMP**的实现。而且你还可以用你自己的协议来对它进行扩展；当然，这个工作不适合缺乏勇气的人。  

异步编程是刚需。你应该已经明白为什么有时候需要用到它，尤其在写服务端的时候。调用**service.run()**来实现异步循环就已经可以让你很满足，但是有时候你需要更进一步，尝试使用**run_one()、poll()**或者**poll_one()**。   

当实现异步时，你可以异步执行你自己的方法；使用**service.post()**或者**service.dispatch()**。   

最后，为了使socket和缓冲区（read或者write）在整个异步操作的生命周期中一直活动，我们需要采取特殊的防护措施。你的连接类需要继承自**enabled_shared_from_this**，然后在内部保存它需要的缓冲区，而且每次异步调用都要传递一个智能指针给**this**操作。  
