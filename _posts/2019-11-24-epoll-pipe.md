---
layout: post
title: epoll���ܼ�ʹ��
date: 2019-11-24 16:32
categories: C++������
tag: network
excerpt: epoll���ܼ�ʹ��
---

С�����ܣ��򵥵ĸ��ӽ���֮���ͨѶ���ӽ��̸���ÿ��1s���Ϸ���"message"�������̣�����Ҫ�ܶ��Ӧ��ʵ��������Ҫ�û����롣

## �����ϴ���
```
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
    tick.it_value.tv_sec    =   1;   // 1s��������ʱ��
    tick.it_interval.tv_sec =   1;   // ��ʱ��������ÿ��1s��ִ����Ӧ�ĺ���

    // setitimer������SIGALRM�źţ��˴�ʹ�õ���ITIMER_REAL�����Զ�Ӧ����SIGALRM�ź�
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


## �������ܷ���  

***1��int setitimer(int which, const struct itimerval *value, struct itimerval *ovalue);***
which����Ъ��ʱ�����ͣ�������ѡ�� 
ITIMER_REAL //��ֵΪ0����ʱ����ֵʵʱ�ݼ������͵��ź���SIGALRM��  
ITIMER_VIRTUAL //��ֵΪ1������ִ��ʱ�ݼ���ʱ����ֵ�����͵��ź���SIGVTALRM��  
ITIMER_PROF //��ֵΪ2�����̺�ϵͳִ��ʱ���ݼ���ʱ����ֵ�����͵��ź���SIGPROF��  

**���ܣ�**��linux�������ʱ���Ҫ��̫��ȷ�Ļ���ʹ��alarm()��signal()�����ˣ���ȷ���룩�����������Ҫʵ�־��ȽϸߵĶ�ʱ���ܵĻ�����Ҫʹ��setitimer������setitimer()ΪLinux��API������C���Ե�Standard Library��setitimer()���������ܣ�һ��ָ��һ��ʱ��󣬲�ִ��ĳ��function������ÿ���һ��ʱ���ִ��ĳ��function��it_intervalָ�����ʱ�䣬it_valueָ����ʼ��ʱʱ�䡣���ָֻ��it_value������ʵ��һ�ζ�ʱ�����ͬʱָ�� it_interval����ʱ��ϵͳ�����³�ʼ��it_valueΪit_interval��ʵ���ظ���ʱ�����߶����㣬��������ʱ���� ��Ȼ���������setitimer�ṩ�Ķ�ʱ�������ߣ�ֻ�������ȴ���ʱ���źžͿ����ˡ�  

***2��int sigaction(int signum, const struct sigaction *act, struct sigaction *oldact);***    
signum��Ҫ�������źš�  
act��Ҫ���õĶ��źŵ��´���ʽ��  
oldact��ԭ�����źŵĴ���ʽ��  

***3��int pipe(int filedes[2]);***
����ֵ���ɹ�������0�����򷵻�-1�������������pipeʹ�õ������ļ�����������fd[0]:���ܵ���fd[1]:д�ܵ���������fork()�е���pipe()�������ӽ��̲���̳��ļ����������������̲��������Ƚ��̣��Ͳ���ʹ��pipe�����ǿ���ʹ�������ܵ����ܵ���һ�ְ���������֮��ı�׼����ͱ�׼������������Ļ��ƣ��Ӷ��ṩһ���ö�����̼�ͨ�ŵķ����������̴����ܵ�ʱ��ÿ�ζ���Ҫ�ṩ�����ļ��������������ܵ�������һ���Թܵ�����д��������һ���Թܵ����ж��������Թܵ��Ķ�д��һ���IOϵͳ����һ�£�ʹ��write()����д�����ݣ�ʹ��read()�������ݡ�




