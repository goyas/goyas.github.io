---
layout: post
title: enable_shared_from_this用法分析
date: 2019-12-01 19:29
categories: network
tag: Reactor Proacotr
excerpt: 异步编程为什么要用enable_shared_from_this及它的原理分析
---

# 一、背景
---
在[为什么需要异步编程](https://goyas.github.io/async-program/)文章末尾提到，"为了使socket和缓冲区（read或write）在整个异步操作的生命周期一直保持活动，我们需要采取特殊的保护措施。你的连接类需要继承自enabled_shared_from_this，然后在内部保存它需要的缓冲区，而且每次异步调用都要传递一个智能指针给this操作"。本文就详细介绍为什么使用enabled_shared_from_this就能保证对象的生命周期，以及enabled_shared_from_this内部的具体实现分析。   

# 二、为什么需要保证对象生命周期
---
首先想象下同步编程，比如socket建立connect后，read或者write数据，因为是同步阻塞的，数据传输完后，socket对象就已经完成了此次任务，此时就算对象销毁，也并不会引起异常。但是异步编程就不一样了，当一个线程调用一个异步函数（例如：该函数还是socket写文件任务），该函数会立即返回，尽管规定的任务还没有完成，这样线程就会执行异步函数的下一条语句，而不会被挂起。只有当"写文件任务"完成后，由新的线程发送完成消息来执行结果同步，但是当新的线程完成"写文件任务"后，再发送过来，此时异步函数调用方对象是否还存在，这就是个需要解决的问题，这也就是为什么需要保证对象的生命周期。   

更加直白一点的例子，假设你需要做下面的操作：
```c
io_service service;
ip::tcp::socket sock(service);
char buff[512];
...
read(sock, buffer(buff));
```
在这个例子中，sock和buff的存在时间都必须比read()调用的时间要长。也就是说，在调用read()返回之前，它们都必须有效。你传给一个方法的所有参数在方法内部都必须有效。当我们采用异步方式时，事情会变得比较复杂。
```c
io_service service;
ip::tcp::socket sock(service);
char buff[512];
void on_read(const boost::system::error_code &, size_t) {}
...
async_read(sock, buffer(buff), on_read);
```
在这个例子中，sock和buff的存在时间都必须比async_read()操作本身时间要长，但是read操作持续的时间我们是不知道的，因为它是异步的。当socket满足条件，有数据可读时，此时操作系统会把数据发送到缓冲区，触发async_read的回调函数on_read执行，on_read执行来通过socket读取数据到buffer，所以必须socket和buffer的生命周期要能得到保证。那究竟用什么方法呢？

# 三、实践中使用方法
---
异步编程时，我们在传入回调函数的时候，通常会想要其带上当前类对象的上下文，或者回调本身就是类成员函数，那这个工作自然非this指针莫属了，像这样：   
```c++
void sock_sender::post_request_no_lock()
{
    Request &req = requests_.front();
    boost::asio::async_write(
		*sock_ptr_, 
		boost::asio::buffer(req.buf_ptr->get_content()),
        boost::bind(&sock_sender::self_handler, this, _1, _2));
}
```
**然而回调执行的时候并一定对象还存在。**为了确保对象的生命周期大于回调，我们可以使类继承自enable_shared_from_this，然后回调的时候使用bind传入shared_from_this()返回的智能指针。由于bind保存的是参数的副本，**bind构造的函数对象会一直持有一个当前类对象的智能指针而使其引用计数不为0，这就确保了对象的生命周期大于回调中构造的函数对象的生命周期**，像这样：   
```c++
class sock_sender : public boost::enable_shared_from_this<sock_sender>
{
    //...
};
void sock_sender::post_request_no_lock()
{
    Request &req = requests_.front();
    boost::asio::async_write(
		*sock_ptr_,
        boost::asio::buffer(req.buf_ptr->get_content()),
        boost::bind(&sock_sender::self_handler, shared_from_this(), _1, _2));
}
```
“实际上边已经提到了，延长资源的生命周期防止使用它时已经被释放。这种问题绝大部分出现在异步调用的时候。因为异步函数的执行时间点无法确定。异步函数可能会使用异步调用之前的变量（比如类对象），这样就必须保证该变量在异步执行期间有效。如何做到这一点呢？**<font color="#dd0000">只需要传递一个指向自身的shared_ptr（必须使用shared_from_this()）给异步函数。因为这个拷贝过程使得对资源的引用计数加一。</font><br />**          

# 四、关于enable_shared_from_this的原理分析    
---
首先要说明的一个问题是：如何安全地将this指针返回给调用者。<font color="#dd0000">一般来说，我们不能直接将this指针返回。</font><br /> 想象这样的情况，该函数将this指针返回到外部某个变量保存，然后这个对象自身已经析构了，但外部变量并不知道，此时如果外部变量使用这个指针，就会使得程序崩溃。  

使用智能指针shared_ptr看起来是个不错的解决方法。但问题是如何去使用它呢？我们来看如下代码：  
```c++
#include <iostream>
#include <boost/shared_ptr.hpp>
class Test
{
public:
    //析构函数
    ~Test() { std::cout << "Test Destructor." << std::endl; }
    //获取指向当前对象的指针
    boost::shared_ptr<Test> GetObject()
    {
        boost::shared_ptr<Test> pTest(this);
        return pTest;
    }
};
int main(int argc, char *argv[])
{
    {
        boost::shared_ptr<Test> p( new Test( ));
        std::cout << "q.use_count(): " << q.use_count() << std::endl; 
        boost::shared_ptr<Test> q = p->GetObject();
    }
    return 0;
}
```
运行后，程序输出：
```c
　　Test Destructor.
　　q.use_count(): 1
　　Test Destructor.
```
可以看到，对象只构造了一次，但却析构了两次。并且在增加一个指向的时候，shared_ptr的计数并没有增加。<font color="#dd0000">也就是说，这个时候，p和q都认为自己是Test指针的唯一拥有者，这两个shared_ptr在计数为0的时候，都会调用一次Test对象的析构函数，所以会出问题。</font><br />    

那么为什么会这样呢？给一个shared_ptr<Test>传递一个this指针难道不能引起shared_ptr<Test>的计数吗？

答案是：<font color="#dd0000">对的，shared_ptr<Test>根本认不得你传进来的指针变量是不是之前已经传过。</font><br />       

看这样的代码：
```c
int main()
{
    Test* test = new Test();
    shared_ptr<Test> p(test);
    shared_ptr<Test> q(test);
    std::cout << "p.use_count(): " << p.use_count() << std::endl;
    std::cout << "q.use_count(): " << q.use_count() << std::endl;
    return 0;
}
```
运行后，程序输出：
```c
p.use_count(): 1
q.use_count(): 1
Test Destructor.
Test Destructor.
```
也证明了刚刚的论述：shared_ptr<Test>根本认不得你传进来的指针变量是不是之前已经传过。  

事实上，类对象是由外部函数通过某种机制分配的，而且一经分配立即交给 shared_ptr管理，而且以后<font color="#dd0000">凡是需要共享使用类对象的地方，必须使用这个 shared_ptr当作右值来构造产生或者拷贝产生（shared_ptr类中定义了赋值运算符函数和拷贝构造函数）另一个shared_ptr ，从而达到共享使用的目的。</font><br />       

解释了上述现象后，现在的问题就变为了：<font color="#dd0000">**如何在类对象（Test）内部中获得一个指向当前对象的shared_ptr 对象？（之前证明，在类的内部直接返回this指针，或者返回return shared_ptr<Test> pTest(this);）不行，因为shared_ptr根本认不得你传过来的指针变量是不是之前已经传过，你本意传个shared_ptr<Test> pTest(this)是想这个对象use_count=2，就算this对象生命周期结束，但是也不delete，因为你异步回来还要用对象里面的东西。)**</font><br />   如果我们能够做到这一点，直接将这个shared_ptr对象返回，就不会造成新建的shared_ptr的问题了。    

下面来看看enable_shared_from_this类的威力。    
enable_shared_from_this 是一个以其派生类为模板类型参数的基类模板，继承它，派生类的this指针就能变成一个 shared_ptr。   
有如下代码：
```c++
#include <iostream>
#include <memory>

class Test : public std::enable_shared_from_this<Test>        //改进1
{
public:
    //析构函数
    ~Test() { std::cout << "Test Destructor." << std::endl; }
    //获取指向当前对象的指针
    std::shared_ptr<Test> GetObject()
    {
        return shared_from_this();      //改进2
    }
};
int main(int argc, char *argv[])
{
    {
        std::shared_ptr<Test> p( new Test( ));
        std::shared_ptr<Test> q = p->GetObject();
        std::cout << "p.use_count(): " << p.use_count() << std::endl;
        std::cout << "q.use_count(): " << q.use_count() << std::endl;
    }
    return 0;
}
```
运行后，程序输出：
```c
	p.use_count(): 2
	q.use_count(): 2
	Test Destructor.
```
可以看到，问题解决了！只有一次new对象，那么释放的时候也就一次，不会出现两次而引起程序崩溃。但是要说明的是，这里举的例子是两个shared_ptr<Test> p和q都离开作用域时，Test对象才调用了析构函数，真正释放对象。但是我们在异步函数里面其目的是:
```c++
struct connection : boost::enable_shared_from_this<connection> {
	typedef boost::shared_ptr<connection> ptr;
	void start(ip::tcp::endpoint ep) {
        sock_.async_connect(ep, boost::bind(&connection::on_connect, shared_from_this(), _1));
    }
};

int main(int argc, char* argv[]) {
    ip::tcp::endpoint ep( ip::address::from_string("127.0.0.1"), 8001);
    connection::ptr(new connection)->start(ep);
} 
```
1、这里的connection::ptr(new connection)->start(ep);能否用普通new的指针，而没有被shared_ptr托管的指针？ 答案是不能，原因见后面**说明2。**      
2、这段server端的代码，每当有不同client连过来，就会触发on_connect回调函数执行。在所有异步调用中，我们传递一个boost::bind仿函数当作参数。这个仿函数内部包含了一个智能指针，指向connection实例。**只要有一个异步操作等待时**，Boost.Asio就会保存boost::bind仿函数的拷贝，这个拷贝保存了指向连接实例的一个智能指针，从而保证connection实例保持活动。问题解决！  
  
接着来看看enable_shared_from_this 是如何工作的，以下是它的源码：   
```c++
template<class T> class enable_shared_from_this
{
protected:

    BOOST_CONSTEXPR enable_shared_from_this() BOOST_SP_NOEXCEPT
    {
    }

    BOOST_CONSTEXPR enable_shared_from_this(enable_shared_from_this const &) BOOST_SP_NOEXCEPT
    {
    }

    enable_shared_from_this & operator=(enable_shared_from_this const &) BOOST_SP_NOEXCEPT
    {
        return *this;
    }

    ~enable_shared_from_this() BOOST_SP_NOEXCEPT // ~weak_ptr<T> newer throws, so this call also must not throw
    {
    }

public:

    shared_ptr<T> shared_from_this()
    {
        shared_ptr<T> p( weak_this_ );
        BOOST_ASSERT( p.get() == this );
        return p;
    }

    shared_ptr<T const> shared_from_this() const
    {
        shared_ptr<T const> p( weak_this_ );
        BOOST_ASSERT( p.get() == this );
        return p;
    }

    weak_ptr<T> weak_from_this() BOOST_SP_NOEXCEPT
    {
        return weak_this_;
    }

    weak_ptr<T const> weak_from_this() const BOOST_SP_NOEXCEPT
    {
        return weak_this_;
    }

public: // actually private, but avoids compiler template friendship issues

    // Note: invoked automatically by shared_ptr; do not call
    template<class X, class Y> void _internal_accept_owner( shared_ptr<X> const * ppx, Y * py ) const BOOST_SP_NOEXCEPT
    {
        if( weak_this_.expired() )
        {
            weak_this_ = shared_ptr<T>( *ppx, py );
        }
    }

private:

    mutable weak_ptr<T> weak_this_;
};

} // namespace boost

#endif  // #ifndef BOOST_SMART_PTR_ENABLE_SHARED_FROM_THIS_HPP_INCLUDED
```
其中shared_from_this()函数的实现为：
```c
    shared_ptr<T> shared_from_this()
    {
        shared_ptr<T> p( weak_this_ );
        BOOST_ASSERT( p.get() == this );
        return p;
    }
```
可以看见，这个函数使用了weak_ptr对象(weak_this)来构造一个shared_ptr对象，然后将shared_ptr对象返回。注意这个weak_ptr是实例对象的一个成员变量，所以对于一个对象来说，它一直是同一个，每次调用shared_from_this()时，就会根据weak_ptr来构造一个临时shared_ptr对象。   

**也许看到这里会产生疑问，这里的shared_ptr也是一个临时对象，和前面有什么区别？还有，为什么enable_shared_from_this 不直接保存一个 shared_ptr 成员？**  

对于第一个问题，这里的每一个shared_ptr都是根据weak_ptr来构造的，而每次构造shared_ptr的时候，使用的参数是一样的，所以这里根据相同的weak_ptr来构造多个临时shared_ptr等价于用**一个shared_ptr**来做拷贝。（你在不同地方调用shared_form_this时，那管理的始终是一个对象）（PS：在shared_ptr类中，是有使用weak_ptr对象来构造shared_ptr对象的构造函数的：
```c++
template<class Y>
explicit shared_ptr( weak_ptr<Y> const & r ): pn( r.pn )
```
对于第二个问题，假设我在类里储存了一个指向自身的shared_ptr，那么这个 shared_ptr的计数最少都会是1，也就是说，这个对象将永远不能析构，所以这种做法是不可取的。  

在enable_shared_from_this类中，没有看到给成员变量weak_this_初始化赋值的地方，那究竟是如何保证weak_this_拥有着Test类对象的指针呢？   

首先我们生成类T时，会依次调用enable_shared_from_this类的构造函数（定义为protected），以及类Test的构造函数。在调用enable_shared_from_this的构造函数时，会初始化定义在enable_shared_from_this中的私有成员变量weak_this_（调用其默认构造函数），这时的weak_this_是无效的（或者说不指向任何对象）。   

接着，当外部程序把指向类Test对象的指针作为初始化参数来初始化一个shared_ptr（boost::shared_ptr<Test> p( new Test( ));）。   

现在来看看 shared_ptr是如何初始化的，shared_ptr 定义了如下构造函数：    
```c++
template<class Y>
    explicit shared_ptr( Y * p ): px( p ), pn( p ) 
    {
        boost::detail::sp_enable_shared_from_this( this, p, p );
    }
```
里面调用了 boost::detail::sp_enable_shared_from_this ：    
```c++
template< class X, class Y, class T >
 inline void sp_enable_shared_from_this( boost::shared_ptr<X> const * ppx,
 Y const * py, boost::enable_shared_from_this< T > const * pe )
{
    if( pe != 0 )
    {
        pe->_internal_accept_owner( ppx, const_cast< Y* >( py ) );
    }
}
```
里面又调用了enable_shared_from_this 的 _internal_accept_owner ：    
```c++
template<class X, class Y> void _internal_accept_owner( shared_ptr<X> const * ppx, Y * py ) const
    {
        if( weak_this_.expired() )
        {
            weak_this_ = shared_ptr<T>( *ppx, py );
        }
    }
```
而在这里，对enable_shared_from_this 类的成员weak_this_进行拷贝赋值，使得weak_this_作为类对象 shared_ptr 的一个观察者。   
这时，当类对象本身需要自身的shared_ptr时，就可以从这个weak_ptr来生成一个了：   
```c++
shared_ptr<T> shared_from_this()
    {
        shared_ptr<T> p( weak_this_ );
        BOOST_ASSERT( p.get() == this );
        return p;
    }
```
从上面的说明来看，需要小心的是shared_from_this()仅在shared_ptr<T>的构造函数被调用之后才能使用，原因是enable_shared_from_this::weak_this_并不在构造函数中设置，而是在shared_ptr<T>的构造函数中设置。   

### 说明1：
所以，如下代码是错误的：   
```c++
class D:public boost::enable_shared_from_this<D>
{
public:
    D()
    {
        boost::shared_ptr<D> p=shared_from_this();
    }
};
```
原因是在D的构造函数中虽然可以保证enable_shared_from_this<D>的构造函数被调用，但weak_this_是无效的（还还没被接管）。  

### 说明2：   
如下代码也是错误的：   
```c++
class D:public boost::enable_shared_from_this<D>
{
public:
    void func()
    {
        boost::shared_ptr<D> p=shared_from_this();
    }
};
void main()
{
    D d;
    d.func();
}
```
原因同上。
总结为：不要试图对一个没有被shared_ptr接管的类对象调用shared_from_this()，不然会产生未定义行为的错误。

# 参考文献：    
[https://www.jianshu.com/p/4444923d79bd](https://www.jianshu.com/p/4444923d79bd)    
[https://blog.csdn.net/veghlreywg/article/details/89743605](https://blog.csdn.net/veghlreywg/article/details/89743605)    
[https://www.cnblogs.com/codingmengmeng/p/9123874.html](https://www.cnblogs.com/codingmengmeng/p/9123874.html)  
