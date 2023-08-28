# mini-muduo
| **Part Ⅰ**            | **Part Ⅱ**            | **Part Ⅲ**            | **Part Ⅳ**            | **Part V**            | **Part VI**           |
| --------------------- | --------------------- | --------------------- | --------------------- | --------------------- | --------------------- |
| [项目介绍](#项目介绍) | [项目特点](#项目特点) | [开发环境](#开发环境) | [并发模型](#并发模型) | [运行案例](#运行案例) | [模块讲解](#模块讲解) |

# 项目介绍

本项目是基于**《Linux多线程服务端编程》**（陈硕著）的知识模拟实现muduo。Mymuduo基于事件驱动和事件回调的epoll+线程池面向对象编程，采用Multi-Reactor架构以及One Loop Per Thread设计思想的多线程C++网路库。Mymuduo重写了muduo核心组件，去除了对boost库的依赖。重要组件包括Channel、Poll、EventLoop、TcpSever等、同时还实现了Buffer缓冲区。

项目已经实现了 Channel 模块、Poller 模块、事件循环模块、HTTP 模块、定时器模块、异步日志模块、内存池模块、数据库连接池模块。 



# 项目特点

- 完全去掉了Muduo库中的Boost依赖，全部使用C++11新语法，如智能指针、function函数对象、lambda表达式等
- 没有单独封装Thread，使用C++11引入的std::thread搭配lambda表达式实现工作线程，没有直接使用pthread库
- 底层使用 Epoll + LT 模式的 I/O 复用模型，并且结合非阻塞 I/O  实现主从 Reactor 模型。
- 采用「one loop per thread」线程模型，并向上封装线程池避免线程创建和销毁带来的性能开销。
- 采用 eventfd 作为事件通知描述符，方便高效派发事件到其他线程执行异步任务。
- 基于自实现的双缓冲区实现异步日志，由后端线程负责定时向磁盘写入前端日志信息，避免数据落盘时阻塞网络服务。
- 参照 Nginx 实现了内存池模块，更好管理小块内存空间，减少内存碎片。



## 开发环境

- 操作系统：`Ubuntu 18.04.6 LTS`
- 编译器：`g++ 7.5.0`
- 编辑器：`vscode`
- 版本控制：`git`
- 项目构建：`cmake 3.10.2`



# 并发模型

![1693209722871.png](https://img1.imgtp.com/2023/08/28/Av1NNUYt.png)



项目采用主从 Reactor 模型，MainReactor 只负责监听派发新连接，在 MainReactor 中通过 Acceptor 接收新连接并轮询派发给 SubReactor，SubReactor 负责此连接的读写事件。

调用 TcpServer 的 start 函数后，会内部创建线程池。每个线程独立的运行一个事件循环，即 SubReactor。MainReactor 从线程池中轮询获取 SubReactor 并派发给它新连接，处理读写事件的 SubReactor 个数一般和 CPU 核心数相等。使用主从 Reactor 模型有诸多优点：

1. 响应快，不必为单个同步事件所阻塞，虽然 Reactor 本身依然是同步的；
2. 可以最大程度避免复杂的多线程及同步问题，并且避免多线程/进程的切换；
3. 扩展性好，可以方便通过增加 Reactor 实例个数充分利用 CPU 资源；
4. 复用性好，Reactor 模型本身与具体事件处理逻辑无关，具有很高的复用性；



# 模块讲解

