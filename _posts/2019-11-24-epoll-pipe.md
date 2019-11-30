---
layout: post
title: epoll介绍及使用
date: 2019-11-24 16:32
categories: network
tag: epoll pipe
excerpt: epoll介绍及使用
---

小程序功能：简单的父子进程之间的通讯，子进程负责每隔1s不断发送"message"给父进程，不需要跑多个应用实例，不需要用户输入。

## 首先上代码
```c
#include<assert.h>
#include<signal.h>
#include<stdio.h>
#include<sys/epoll.h>
#include<sys/time.h>
#include<sys/wait.h>
#include<unistd.h>

int  fd[2];
int* write_fd;
int* read_fd;
const char msg[] = {'m','e','s','s','a','g','e'};

void SigHandler(int){
    size_t bytes = write(*write_fd, msg, sizeof(msg));
    printf("children process msg have writed : %ld bytes\n", bytes);
}

void ChildrenProcess() {
    struct sigaction sa;
    sa.sa_flags     =   0;
    sa.sa_handler   =   SigHandler;
    sigaction(SIGALRM, &sa, NULL);

    struct itimerval tick   =   {0};
    tick.it_value.tv_sec    =   1;   // 1s后将启动定时器
    tick.it_interval.tv_sec =   1;   // 定时器启动后，每隔1s将执行相应的函数

    // setitimer将触发SIGALRM信号，此处使用的是ITIMER_REAL，所以对应的是SIGALRM信号
    assert(setitimer(ITIMER_REAL, &tick, NULL) == 0);
    while(true) {
        pause();
    }
}

void FatherProcess() {
    epoll_event ev;
    epoll_event events[1];
    char buf[1024]  =   {0};
    int epoll_fd    =   epoll_create(1);
    ev.data.fd      =   *read_fd;
    ev.events       =   EPOLLIN | EPOLLET;
    epoll_ctl(epoll_fd, EPOLL_CTL_ADD, *read_fd, &ev);

    while (true) {
        int fds = epoll_wait(epoll_fd, events, sizeof(events), -1);
        if(events[0].data.fd == *read_fd) {
            size_t bytes = read(*read_fd, buf, sizeof(buf));
            printf("father process read %ld bytes = %s\n", bytes, buf);
        }
    }

    int status;
    wait(&status);
}

int main() {
    int ret = pipe(fd);
    if (ret != 0) {
        printf("pipe failed\n");
        return -1;
    }
    write_fd = &fd[1];
    read_fd  = &fd[0];

    pid_t pid = fork();
    if (pid == 0) {//child process
        ChildrenProcess();
    } else if (pid > 0) {//father process
        FatherProcess();
    }

    return 0;
}
```


## 函数功能分析  

***1、int setitimer(int which, const struct itimerval *value, struct itimerval *ovalue);***    
which：间歇计时器类型，有三种选择： 
ITIMER_REAL //数值为0，计时器的值实时递减，发送的信号是SIGALRM。  
ITIMER_VIRTUAL //数值为1，进程执行时递减计时器的值，发送的信号是SIGVTALRM。  
ITIMER_PROF //数值为2，进程和系统执行时都递减计时器的值，发送的信号是SIGPROF。  

**功能：**在linux下如果定时如果要求不太精确的话，使用alarm()和signal()就行了（精确到秒），但是如果想要实现精度较高的定时功能的话，就要使用setitimer函数。setitimer()为Linux的API，并非C语言的Standard Library，setitimer()有两个功能，一是指定一段时间后，才执行某个function，二是每间格一段时间就执行某个function。it_interval指定间隔时间，it_value指定初始定时时间。如果只指定it_value，就是实现一次定时；如果同时指定 it_interval，则超时后，系统会重新初始化it_value为it_interval，实现重复定时；两者都清零，则会清除定时器。 当然，如果是以setitimer提供的定时器来休眠，只需阻塞等待定时器信号就可以了。  

***2、int sigaction(int signum, const struct sigaction *act, struct sigaction *oldact);***    
signum：要操作的信号。  
act：要设置的对信号的新处理方式。  
oldact：原来对信号的处理方式。  

***3、int pipe(int filedes[2]);***    
返回值：成功，返回0，否则返回-1。参数数组包含pipe使用的两个文件的描述符。fd[0]:读管道，fd[1]:写管道。必须在fork()中调用pipe()，否则子进程不会继承文件描述符。两个进程不共享祖先进程，就不能使用pipe。但是可以使用命名管道。管道是一种把两个进程之间的标准输入和标准输出连接起来的机制，从而提供一种让多个进程间通信的方法，当进程创建管道时，每次都需要提供两个文件描述符来操作管道。其中一个对管道进行写操作，另一个对管道进行读操作。对管道的读写与一般的IO系统函数一致，使用write()函数写入数据，使用read()读出数据。




