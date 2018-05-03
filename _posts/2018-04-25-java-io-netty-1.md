---
layout: post
title:  Netty入门篇（1）之I/O基础
date:   2018-04-25 16:19:00 +0800
categories: Java
tag: netty
---

* content
{:toc}
之前有研究过一些NIO的内容，但是并没有深入的了解。所以现在希望可以系统的学习一下相关的知识。从最基本的I/O到NIO一直到Netty。Netty是由[JBOSS](https://baike.baidu.com/item/JBOSS)提供的一个java开源框架。Netty提供异步的、事件驱动的网络应用程序框架和工具，用以快速开发高性能、高可靠性的网络服务器和客户端程序。

1.I/O基础入门
------------------------------------

Java1.4之前的早期版本对I/O的支持并不完善，开发人员在开发高性能I/O程序时，会面临一些巨大的挑战和困难，主要问题如下：

- 没有数据缓冲区，I/O性能存在问题；
- 没有C/C++的channel的概念，只有输入输出流；
- 同步阻塞式I/.O通信（BIO），通常会导致通信线程被长时间阻塞；
- 支持的字符集有限，硬件可移植性不好。

### 1.1 linux网络I/O模型

Linux内核将所有的外部设备都看做一个文件来操作，对一个文件的读写操作通常会内核提供的系统命令，返回一个file descriptor（fd，文件描述符）。而对一个socket的读写也会有相应的描述符，称为socketfd。描述符就是一个数字，它指向内核中的一个结构体（文件路径，数据取等一些属性）。

根据Unix网络编程对I/O模型的分类，共分为以下5种：

1. 阻塞I/O模型：最常用的I/O模型就是阻塞I/O模型，缺省情况下，所有文件操作都是阻塞的。我们以套接字接口为例来讲解此模型：在进程空间中调用recvfrom，其系统调用直到数据包到达且被复制到应用程序的缓冲区中或者中间发生错误时才返回，在此期间一直会等待，进程在从调用recvfrom开始到它返回的整段时间内都是被阻塞的，因此被称为阻塞I/O模型。

2. 非阻塞I/O模型：recvfrom从应用层到内核的时候，如果该缓冲区没有数的话，就直接返回一个EWOULDBLOCK错误，一般都是对非阻塞I/O模型进行轮询检查这个状态，看内核是不是有数据到来。

3. I/.O复用模型：linux提供select/poll，进程通过将一个或多个fd传递给select或poll系统调用，阻塞在select操作上，这样select/poll可以帮助我们侦测多个fd是否处于继续状态。select/poll是顺序扫描fd是否就绪，而且支持的fd数量有限。因此他的使用受到了一些制约。Linux还提供了一个人epoll系统调用，epoll使用基于时间驱动方式代替顺序扫描，因此性能更高。当有fd就绪时，立即回调函数rollback。

4. 信号驱动I/O模型：首先开启套接字接口信号驱动I/O功能，并通过系统调用sigaction执行一个信号处理函数（此系统调用立即返回，进程继续工作，非阻塞）。当数据准备就绪时，就为该进程生成一个SIGIO信号，通过信号回调通知应用程序调用recvfrom来读取数据，并通知主循环函数处理数据。

5. 异步I/O：告知内核启动某个操作，并让内核在整个操作完成以后（包括将数据复制到用户自己的缓冲区）通知我们。这种模型与信号驱动模型的主要区别是：信号驱动I/O由内核通知我们何时可以开始一个I/O操作；异步I/O模型通知我们操作何时已经完成。

   ​

### 1.2 I/O多路复用技术

## 2.Java的I/O演进

## 3.总结













<hr>
​最后的最后，老婆我爱你。







