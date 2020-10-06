---
layout: post
title: "CSAPP 读书笔记（二）"
subtitle: "基于线程的并发编程"
date: 2020-10-06
author: "BKAUTO"
header-img: "img/post-bg-2020.jpg"
mathjax: true
tags:
  - Computer System
  - 并发
  - 读书笔记
---

这部分内容主要来自《Computer Systems——A Programmer's Perspective》Chapter 12。

## 概述 
前面一篇笔记简单介绍了并发编程的常见形式：
- 多进程
- I/O多路复用
- 多线程

我们已经知道，线程同时继承了进程和I/O多路复用的特点：
- 独立的线程上下文
- 对等线程共享虚拟内存

### Posix线程
Posix线程（Pthreads）是C程序中处理线程的一个标准接口。  
以下程序在主线程创建一个对等线程，然后等待它的终止。
```c
#include "csapp.h"
void *thread(void *vargp);                    //line:conc:hello:prototype

int main()                                    //line:conc:hello:main
{
    pthread_t tid;                            //line:conc:hello:tid
    Pthread_create(&tid, NULL, thread, NULL); //line:conc:hello:create
    Pthread_join(tid, NULL);                  //line:conc:hello:join
    exit(0);                                  //line:conc:hello:exit
}

void *thread(void *vargp) /* thread routine */  //line:conc:hello:beginthread
{
    printf("Hello, world!\n");                 
    return NULL;                               //line:conc:hello:return
}                                              //line:conc:hello:endthread
```

## 多线程程序中的共享变量
线程的出现使得在并发中共享数据更加方便，那究竟哪些数据是可以被多个线程共享的？数据的共享又会带来哪些问题？
> 为了编写正确的多线程程序，我们必须对所谓的共享以及它是如何工作的有很清楚的了解。

### 线程内存模型
一组并发线程运行在一个进程的上下文中。  
- 每个线程都有它自己独立的线程上下文：
    - 线程ID
    - 栈
    - 栈指针
    - 程序计数器
    - 条件码
    - 通用目的寄存器
- 每个线程和其他线程共享进程上下文的剩余部分：
    - 整个用户虚拟地址空间
        - 只读文本（代码）
        - 读/写数据
        - 堆
        - 共享库代码和数据
    - 打开文件的集合  

> 各自独立的线程栈的内存模型不是那么整齐清楚的。不同的线程栈不对其他线程设防，若一个线程得到一个指向其他线程栈的指针，那么它就可以读写这个栈的任何部分。

### 将变量映射到内存
- ***全局变量***    
定义在函数之外的变量。    
运行时，虚拟内存的读/写区域只包含每个全局变量的一个实例，**任何线程都可以引用**。

- ***本地自动变量***  
定义在函数内部但是没有static属性的变量。  
运行时，每个线程的栈都包含它自己的所有本地自动变量的实例。  

- ***本地静态变量***
定义在函数内部并有static属性的变量。
和全局变量一样，虚拟内存的读/写区域只包含每个本地静态变量的一个实例，**任何线程都可以引用（但只初始化一次）**。

## 用信号量同步线程
共享变量十分方便，但它们也引入了*同步错误（synchronization error）*的可能性。  
举一个十分经典的例子，以下程序创建两个线程，每个线程都对共享计数变量cnt加1。  
```c
#include "csapp.h"

void *thread(void *vargp);  /* Thread routine prototype */

/* Global shared variable */
volatile long cnt = 0; /* Counter */

int main(int argc, char **argv) 
{
    long niters;
    pthread_t tid1, tid2;

    /* Check input argument */
    if (argc != 2) { 
	printf("usage: %s <niters>\n", argv[0]);
	exit(0);
    }
    niters = atoi(argv[1]);

    /* Create threads and wait for them to finish */
    Pthread_create(&tid1, NULL, thread, &niters);
    Pthread_create(&tid2, NULL, thread, &niters);
    Pthread_join(tid1, NULL);
    Pthread_join(tid2, NULL);

    /* Check result */
    if (cnt != (2 * niters))
	printf("BOOM! cnt=%ld\n", cnt);
    else
	printf("OK cnt=%ld\n", cnt);
    exit(0);
}

/* Thread routine */
void *thread(void *vargp) 
{
    long i, niters = *((long *)vargp);

    for (i = 0; i < niters; i++) //line:conc:badcnt:beginloop
	cnt++;                   //line:conc:badcnt:endloop

    return NULL;
}
```
cnt的最终值并没有像预期一样得到$2\times{niters}$，这是因为两个线程操作共享变量cnt内容的指令交替执行。比如线程2加载cnt发生在线程1加载cnt之后、线程1存储cnt之前，**这是错误的！**

### 使用信号量来实现互斥——互斥锁
我们想要确保每个线程在执行它的临界区中的指令时，拥有对共享变量的互斥的访问，这种现象称为互斥（mutual exclusion）。

- 二元信号量（binary semaphore）  
值总是0或1，以提供互斥为目的的二元信号量称为互斥锁（mutex）。  

- P操作（互斥锁加锁）   
如果信号量s是非零的，那么就挂起这个线程，直到s变为非零，而一个V操作会重启这个线程。

- V操作（互斥锁解锁）  
V操作将s加1。如果有任何线程阻塞在P操作等待s变成非零，那么V操作会重启这些线程中的一个，然后该线程将s减1，完成它的P操作。

```c
volatile long cnt = 0;
sem_t mutex;

Sem_init(&mutex, 0, 1);

for (i = 0; i < niters; i++) {
    P(&mutex);
    cnt++;
    V(&mutex);
}
```
(未完待续。。。)