---
layout: post
title: enable_shared_from_this�÷�����
date: 2019-12-01 19:29
categories: network
tag: Reactor Proacotr
excerpt: �첽���ΪʲôҪ��enable_shared_from_this������ԭ�����
---

# һ������
---
��[Ϊʲô��Ҫ�첽���](https://goyas.github.io/async-program/)����ĩβ�ᵽ��"Ϊ��ʹsocket�ͻ�������read��write���������첽��������������һֱ���ֻ��������Ҫ��ȡ����ı�����ʩ�������������Ҫ�̳���enabled_shared_from_this��Ȼ�����ڲ���������Ҫ�Ļ�����������ÿ���첽���ö�Ҫ����һ������ָ���this����"�����ľ���ϸ����Ϊʲôʹ��enabled_shared_from_this���ܱ�֤������������ڣ��Լ�enabled_shared_from_this�ڲ��ľ���ʵ�ַ�����   

# ����Ϊʲô��Ҫ��֤������������
---
����������ͬ����̣�����socket����connect��read����write���ݣ���Ϊ��ͬ�������ģ����ݴ������socket������Ѿ�����˴˴����񣬴�ʱ����������٣�Ҳ�����������쳣�������첽��̾Ͳ�һ���ˣ���һ���̵߳���һ���첽���������磺�ú�������socketд�ļ����񣩣��ú������������أ����ܹ涨������û����ɣ������߳̾ͻ�ִ���첽��������һ����䣬�����ᱻ����ֻ�е�"д�ļ�����"��ɺ����µ��̷߳��������Ϣ��ִ�н��ͬ�������ǵ��µ��߳����"д�ļ�����"���ٷ��͹�������ʱ�첽�������÷������Ƿ񻹴��ڣ�����Ǹ���Ҫ��������⣬��Ҳ����Ϊʲô��Ҫ��֤������������ڡ�   

����ֱ��һ������ӣ���������Ҫ������Ĳ�����
```c
io_service service;
ip::tcp::socket sock(service);
char buff[512];
...
read(sock, buffer(buff));
```
����������У�sock��buff�Ĵ���ʱ�䶼�����read()���õ�ʱ��Ҫ����Ҳ����˵���ڵ���read()����֮ǰ�����Ƕ�������Ч���㴫��һ�����������в����ڷ����ڲ���������Ч�������ǲ����첽��ʽʱ��������ñȽϸ��ӡ�
```c
io_service service;
ip::tcp::socket sock(service);
char buff[512];
void on_read(const boost::system::error_code &, size_t) {}
...
async_read(sock, buffer(buff), on_read);
```
����������У�sock��buff�Ĵ���ʱ�䶼�����async_read()��������ʱ��Ҫ��������read����������ʱ�������ǲ�֪���ģ���Ϊ�����첽�ġ���socket���������������ݿɶ�ʱ����ʱ����ϵͳ������ݷ��͵�������������async_read�Ļص�����on_readִ�У�on_readִ����ͨ��socket��ȡ���ݵ�buffer�����Ա���socket��buffer����������Ҫ�ܵõ���֤���Ǿ�����ʲô�����أ�

# ����ʵ����ʹ�÷���
---
�첽���ʱ�������ڴ���ص�������ʱ��ͨ������Ҫ����ϵ�ǰ�����������ģ����߻ص�����������Ա�����������������Ȼ��thisָ��Ī���ˣ���������   
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
**Ȼ���ص�ִ�е�ʱ��һ�����󻹴��ڡ�**Ϊ��ȷ��������������ڴ��ڻص������ǿ���ʹ��̳���enable_shared_from_this��Ȼ��ص���ʱ��ʹ��bind����shared_from_this()���ص�����ָ�롣����bind������ǲ����ĸ�����**bind����ĺ��������һֱ����һ����ǰ����������ָ���ʹ�����ü�����Ϊ0�����ȷ���˶�����������ڴ��ڻص��й���ĺ����������������**����������   
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
��ʵ���ϱ��Ѿ��ᵽ�ˣ��ӳ���Դ���������ڷ�ֹʹ����ʱ�Ѿ����ͷš�����������󲿷ֳ������첽���õ�ʱ����Ϊ�첽������ִ��ʱ����޷�ȷ�����첽�������ܻ�ʹ���첽����֮ǰ�ı�������������󣩣������ͱ��뱣֤�ñ������첽ִ���ڼ���Ч�����������һ���أ�**<font color="#dd0000">ֻ��Ҫ����һ��ָ�������shared_ptr������ʹ��shared_from_this()�����첽��������Ϊ�����������ʹ�ö���Դ�����ü�����һ��</font><br />**          

# �ġ�����enable_shared_from_this��ԭ�����    
---
����Ҫ˵����һ�������ǣ���ΰ�ȫ�ؽ�thisָ�뷵�ظ������ߡ�<font color="#dd0000">һ����˵�����ǲ���ֱ�ӽ�thisָ�뷵�ء�</font><br /> ����������������ú�����thisָ�뷵�ص��ⲿĳ���������棬Ȼ��������������Ѿ������ˣ����ⲿ��������֪������ʱ����ⲿ����ʹ�����ָ�룬�ͻ�ʹ�ó��������  

ʹ������ָ��shared_ptr�������Ǹ�����Ľ�������������������ȥʹ�����أ������������´��룺  
```c++
#include <iostream>
#include <boost/shared_ptr.hpp>
class Test
{
public:
    //��������
    ~Test() { std::cout << "Test Destructor." << std::endl; }
    //��ȡָ��ǰ�����ָ��
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
���к󣬳��������
```c
����Test Destructor.
����q.use_count(): 1
����Test Destructor.
```
���Կ���������ֻ������һ�Σ���ȴ���������Ρ�����������һ��ָ���ʱ��shared_ptr�ļ�����û�����ӡ�<font color="#dd0000">Ҳ����˵�����ʱ��p��q����Ϊ�Լ���Testָ���Ψһӵ���ߣ�������shared_ptr�ڼ���Ϊ0��ʱ�򣬶������һ��Test������������������Ի�����⡣</font><br />    

��ôΪʲô�������أ���һ��shared_ptr<Test>����һ��thisָ���ѵ���������shared_ptr<Test>�ļ�����

���ǣ�<font color="#dd0000">�Եģ�shared_ptr<Test>�����ϲ����㴫������ָ������ǲ���֮ǰ�Ѿ�������</font><br />       

�������Ĵ��룺
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
���к󣬳��������
```c
p.use_count(): 1
q.use_count(): 1
Test Destructor.
Test Destructor.
```
Ҳ֤���˸ոյ�������shared_ptr<Test>�����ϲ����㴫������ָ������ǲ���֮ǰ�Ѿ�������  

��ʵ�ϣ�����������ⲿ����ͨ��ĳ�ֻ��Ʒ���ģ�����һ�������������� shared_ptr���������Ժ�<font color="#dd0000">������Ҫ����ʹ�������ĵط�������ʹ����� shared_ptr������ֵ������������߿���������shared_ptr���ж����˸�ֵ����������Ϳ������캯������һ��shared_ptr ���Ӷ��ﵽ����ʹ�õ�Ŀ�ġ�</font><br />       

������������������ڵ�����ͱ�Ϊ�ˣ�<font color="#dd0000">**����������Test���ڲ��л��һ��ָ��ǰ�����shared_ptr ���󣿣�֮ǰ֤����������ڲ�ֱ�ӷ���thisָ�룬���߷���return shared_ptr<Test> pTest(this);�����У���Ϊshared_ptr�����ϲ����㴫������ָ������ǲ���֮ǰ�Ѿ��������㱾�⴫��shared_ptr<Test> pTest(this)�����������use_count=2������this�����������ڽ���������Ҳ��delete����Ϊ���첽������Ҫ�ö�������Ķ�����)**</font><br />   ��������ܹ�������һ�㣬ֱ�ӽ����shared_ptr���󷵻أ��Ͳ�������½���shared_ptr�������ˡ�    

����������enable_shared_from_this���������    
enable_shared_from_this ��һ������������Ϊģ�����Ͳ����Ļ���ģ�壬�̳������������thisָ����ܱ��һ�� shared_ptr��   
�����´��룺
```c++
#include <iostream>
#include <memory>

class Test : public std::enable_shared_from_this<Test>        //�Ľ�1
{
public:
    //��������
    ~Test() { std::cout << "Test Destructor." << std::endl; }
    //��ȡָ��ǰ�����ָ��
    std::shared_ptr<Test> GetObject()
    {
        return shared_from_this();      //�Ľ�2
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
���к󣬳��������
```c
	p.use_count(): 2
	q.use_count(): 2
	Test Destructor.
```
���Կ������������ˣ�ֻ��һ��new������ô�ͷŵ�ʱ��Ҳ��һ�Σ�����������ζ�����������������Ҫ˵�����ǣ�����ٵ�����������shared_ptr<Test> p��q���뿪������ʱ��Test����ŵ��������������������ͷŶ��󡣵����������첽����������Ŀ����:
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
1�������connection::ptr(new connection)->start(ep);�ܷ�����ͨnew��ָ�룬��û�б�shared_ptr�йܵ�ָ�룿 ���ǲ��ܣ�ԭ�������**˵��2��**      
2�����server�˵Ĵ��룬ÿ���в�ͬclient���������ͻᴥ��on_connect�ص�����ִ�С��������첽�����У����Ǵ���һ��boost::bind�º�����������������º����ڲ�������һ������ָ�룬ָ��connectionʵ����**ֻҪ��һ���첽�����ȴ�ʱ**��Boost.Asio�ͻᱣ��boost::bind�º����Ŀ������������������ָ������ʵ����һ������ָ�룬�Ӷ���֤connectionʵ�����ֻ����������  
  
����������enable_shared_from_this ����ι����ģ�����������Դ�룺   
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
����shared_from_this()������ʵ��Ϊ��
```c
    shared_ptr<T> shared_from_this()
    {
        shared_ptr<T> p( weak_this_ );
        BOOST_ASSERT( p.get() == this );
        return p;
    }
```
���Կ������������ʹ����weak_ptr����(weak_this)������һ��shared_ptr����Ȼ��shared_ptr���󷵻ء�ע�����weak_ptr��ʵ�������һ����Ա���������Զ���һ��������˵����һֱ��ͬһ����ÿ�ε���shared_from_this()ʱ���ͻ����weak_ptr������һ����ʱshared_ptr����   

**Ҳ���������������ʣ������shared_ptrҲ��һ����ʱ���󣬺�ǰ����ʲô���𣿻��У�Ϊʲôenable_shared_from_this ��ֱ�ӱ���һ�� shared_ptr ��Ա��**  

���ڵ�һ�����⣬�����ÿһ��shared_ptr���Ǹ���weak_ptr������ģ���ÿ�ι���shared_ptr��ʱ��ʹ�õĲ�����һ���ģ��������������ͬ��weak_ptr����������ʱshared_ptr�ȼ�����**һ��shared_ptr**���������������ڲ�ͬ�ط�����shared_form_thisʱ���ǹ����ʼ����һ�����󣩣�PS����shared_ptr���У�����ʹ��weak_ptr����������shared_ptr����Ĺ��캯���ģ�
```c++
template<class Y>
explicit shared_ptr( weak_ptr<Y> const & r ): pn( r.pn )
```
���ڵڶ������⣬�����������ﴢ����һ��ָ�������shared_ptr����ô��� shared_ptr�ļ������ٶ�����1��Ҳ����˵�����������Զ�����������������������ǲ���ȡ�ġ�  

��enable_shared_from_this���У�û�п�������Ա����weak_this_��ʼ����ֵ�ĵط����Ǿ�������α�֤weak_this_ӵ����Test������ָ���أ�   

��������������Tʱ�������ε���enable_shared_from_this��Ĺ��캯��������Ϊprotected�����Լ���Test�Ĺ��캯�����ڵ���enable_shared_from_this�Ĺ��캯��ʱ�����ʼ��������enable_shared_from_this�е�˽�г�Ա����weak_this_��������Ĭ�Ϲ��캯��������ʱ��weak_this_����Ч�ģ�����˵��ָ���κζ��󣩡�   

���ţ����ⲿ�����ָ����Test�����ָ����Ϊ��ʼ����������ʼ��һ��shared_ptr��boost::shared_ptr<Test> p( new Test( ));����   

���������� shared_ptr����γ�ʼ���ģ�shared_ptr ���������¹��캯����    
```c++
template<class Y>
    explicit shared_ptr( Y * p ): px( p ), pn( p ) 
    {
        boost::detail::sp_enable_shared_from_this( this, p, p );
    }
```
��������� boost::detail::sp_enable_shared_from_this ��    
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
�����ֵ�����enable_shared_from_this �� _internal_accept_owner ��    
```c++
template<class X, class Y> void _internal_accept_owner( shared_ptr<X> const * ppx, Y * py ) const
    {
        if( weak_this_.expired() )
        {
            weak_this_ = shared_ptr<T>( *ppx, py );
        }
    }
```
���������enable_shared_from_this ��ĳ�Աweak_this_���п�����ֵ��ʹ��weak_this_��Ϊ����� shared_ptr ��һ���۲��ߡ�   
��ʱ�������������Ҫ�����shared_ptrʱ���Ϳ��Դ����weak_ptr������һ���ˣ�   
```c++
shared_ptr<T> shared_from_this()
    {
        shared_ptr<T> p( weak_this_ );
        BOOST_ASSERT( p.get() == this );
        return p;
    }
```
�������˵����������ҪС�ĵ���shared_from_this()����shared_ptr<T>�Ĺ��캯��������֮�����ʹ�ã�ԭ����enable_shared_from_this::weak_this_�����ڹ��캯�������ã�������shared_ptr<T>�Ĺ��캯�������á�   

### ˵��1��
���ԣ����´����Ǵ���ģ�   
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
ԭ������D�Ĺ��캯������Ȼ���Ա�֤enable_shared_from_this<D>�Ĺ��캯�������ã���weak_this_����Ч�ģ�����û���ӹܣ���  

### ˵��2��   
���´���Ҳ�Ǵ���ģ�   
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
ԭ��ͬ�ϡ�
�ܽ�Ϊ����Ҫ��ͼ��һ��û�б�shared_ptr�ӹܵ���������shared_from_this()����Ȼ�����δ������Ϊ�Ĵ���

# �ο����ף�    
[https://www.jianshu.com/p/4444923d79bd](https://www.jianshu.com/p/4444923d79bd)    
[https://blog.csdn.net/veghlreywg/article/details/89743605](https://blog.csdn.net/veghlreywg/article/details/89743605)    
[https://www.cnblogs.com/codingmengmeng/p/9123874.html](https://www.cnblogs.com/codingmengmeng/p/9123874.html)  
