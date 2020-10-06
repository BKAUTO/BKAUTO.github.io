---
layout: post
title: "CSAPP 读书笔记（一）"
subtitle: "进程，I/O多路复用，线程"
date: 2020-10-05
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
- 并发(concurrency)——逻辑控制流在时间上重叠，出现在计算机系统许多不同层面上
![concurrency](https://techdifferences.com/wp-content/uploads/2017/12/Untitled.jpg "并发和并行")

并发不止是操作系统内核用来运行多个应用程序的机制，应用级并发在许多情况十分有用：
1. 访问慢速I/O设备
2. 通过推迟工作以降低延迟
3. 服务多个网络客户端
4. 并行计算
5. 进程
- 每个逻辑控制流都是一个进程，由**内核进行调度与维护**。进程有**独立的虚拟地址空间**，可进行进程间通信（IPC, interprocess communication）。
6. I/O多路复用
- 应用程序在一个进程的上下文中显式地调度它们自己的逻辑流。逻辑流被模型化为状态机，主程序显式地从一个状态转换到另一个状态。程序是一个**单独的进程**，所有的流都共享同一个地址空间。
7. 线程
- 线程是运行在一个**单一进程**上下文中的逻辑流。像进程流一样**由内核进行调度**，而像I/O多路复用一样**共享同一个虚拟地址空间**。

## 基于进程的并发编程
使用如下函数：
```
fork
exec
waitpid
```
构造并发服务器：
- 在父进程中接受客户端连接请求，然后创建一个新的子进程来为每个新客户端提供服务。

### 进程的优劣
- 父、子进程间共享状态信息：共享文件表，但不共享用户地址空间，模型清晰。
- 进程共享状态信息困难，进程控制和IPC的开销很高。

## 基于I/O多路复用的并发编程
这一部分我觉得这本书上写的并不是很清晰，当然也可能是翻译的问题。可以先看一下[IO 多路复用是什么意思？ - 罗志宇的回答 - 知乎](https://www.zhihu.com/question/32163005/answer/55772739)，对I/O多路复用的概念有一个大概的认知。
![I/O多路复用](https://pic2.zhimg.com/80/18d8525aceddb840ea4c131002716221_1440w.jpg "I/O多路复用")

假如要求你编写一个echo服务器，响应**两个独立的I/O事件**：
- 能对用户从标准输入键入的交互命令做出响应
- 网络客户端发起连接请求
> 我们先等待哪个事件呢？

### select函数与迭代echo服务器
在大家熟悉的epoll(nginx)之前，最早的I/O多路复用是基于select函数实现的。  
基本的思路是要求内核挂起进程，只有一个或多个I/O事件发生后，才将控制返回给应用程序。
```c
#include "csapp.h"
void echo(int connfd);
void command(void);

int main(int argc, char **argv) 
{
    int listenfd, connfd;
    socklen_t clientlen;
    struct sockaddr_storage clientaddr;
    fd_set read_set, ready_set;

    if (argc != 2) {
	fprintf(stderr, "usage: %s <port>\n", argv[0]);
	exit(0);
    }
    listenfd = Open_listenfd(argv[1]);  //line:conc:select:openlistenfd

    FD_ZERO(&read_set);              /* Clear read set */ //line:conc:select:clearreadset
    FD_SET(STDIN_FILENO, &read_set); /* Add stdin to read set */ //line:conc:select:addstdin
    FD_SET(listenfd, &read_set);     /* Add listenfd to read set */ //line:conc:select:addlistenfd

    while (1) {
	ready_set = read_set;
	Select(listenfd+1, &ready_set, NULL, NULL, NULL); //line:conc:select:select
	if (FD_ISSET(STDIN_FILENO, &ready_set)) //line:conc:select:stdinready
	    command(); /* Read command line from stdin */
	if (FD_ISSET(listenfd, &ready_set)) { //line:conc:select:listenfdready
            clientlen = sizeof(struct sockaddr_storage); 
	    connfd = Accept(listenfd, (SA *)&clientaddr, &clientlen);
	    echo(connfd); /* Echo client input until EOF */
	    Close(connfd);
	}
    }
}

void command(void) {
    char buf[MAXLINE];
    if (!Fgets(buf, MAXLINE, stdin))
	exit(0); /* EOF */
    printf("%s", buf); /* Process the input command */
}
```
select函数处理一个描述符集合（实际是一个n位向量），该向量的一位就代表多个独立I/O事件的一路。select函数会一直阻塞，直到描述符集合中的某一描述符准备好可以读（比如键入回车，标准输入描述符变为可读）。  

可以看出select函数有一个明显的问题，为了获得准备好集合（ready_set），会修改函数的传入参数读集合（read_set）。对于上面这段程序，当服务器连接到某一客户端后，会连续echo输入行，直到从客户端断开连接，这期间服务器对标准输入没有响应。

### 基于I/O多路复用的并发事件驱动服务器
I/O多路复用可以用作并发事件驱动（event-driven）程序的基础，下面介绍一个多客户端并发事件驱动echo服务器的实现。
每个新的客户端k，基于I/O多路复用的并发服务器会创建一个新的状态机$s_k$，并将它和已连接描述符$d_k$联系起来。  
- 一个pool结构里维护着活动客户端的集合。在调用`init_pool`初始化池之后，服务器进入一个无限循环。
- 服务器调用select函数来检测两种不同类型的输入事件
    - 来自一个新客户端的连接请求到达
    - 一个已存在的客户端的已连接描述符准备好可以读了

### 有限状态模型
- select函数检测到输入事件
- add_client函数创建一个新的逻辑流（状态机）
- check_clients函数回送输入行，执行状态转移
- 当客户端完成文本行发送，check_clients删除状态机
> 事件驱动的Web服务器
>> 现代高性能服务器（如Node.js、nginx和Tornado）使用的都是基于I/O多路复用的事件驱动的编程方式，主要是因为相比于进程和线程的方式，它有明显的性能优势。

### I/O多路复用技术的优劣
- 优点
    - 更多对程序行为的控制
    - 每个逻辑流可访问全部地址空间
- 缺点
    - 编码复杂
    - 粒度

## 基于线程的并发编程
- 线程（thread）就是运行在进程上下文中的逻辑流。
- 一个进程里可同时运行多个线程， 线程由内核自动调动。
- 独立的线程上下文，包括线程ID、栈、栈指针、程序计数器等。
- 多个线程运行在单一进程的上下文中，因此**共享这个进程虚拟地址空间上的内容**，包括它的代码、数据、堆、共享库和打开的文件。
> 在一些重要的方面，线程执行是不同于进程的。
> - 线程上下文更小，切换快。
> - 线程不严格按照父子层次组织，和一个进程相关的线程组成一个对等线程池。
> - 主线程——第一个允许的线程。
> - **一个线程可以杀死它的对等线程，或者等待它的任意对等线程终止。**
> - 每个对等线程都能读写相同的共享数据。