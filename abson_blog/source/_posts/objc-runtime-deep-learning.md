---
title: 线程池
date: 2018-04-24 18:57:35
tags:
---

动态创建子线程来实现服务器有以下缺点：

* 动态创建进程（或线程）是比较耗费时间的，这将导致较慢的客户响应。
* 动态创建的子进程（或子线程）通常只用来为一个客户服务（除非我们做特殊的处理），这将导致系统上产生大量的细微进程（或线程）。进程（或线程）间的切换将消耗大量CPU时间。
* 动态创建的子进程是当前进程的完整映像。当前进程必须谨慎地管理其分配的文件描述符和堆内存等系统资源，否则子进程可能复制这些资源，从而使系统的可用资源急剧下降，进而影响服务器的性能。

至于主进程选择哪个子进程来为新任务服务，则有两种方式：

* 主进程使用某种算法来主动选择子进程。最简单、最常用的算法是随机算法和Round Robin（轮流选取）算法，但更优秀、更智能的算法将使任务在各个工作进程中更均匀地分配，从而减轻服务器的整体压力。
* 主进程和所有子进程通过一个共享的工作队列来同步，子进程都睡眠在该工作队列上。当有新的任务到来时，主进程将任务添加到工作队列中。这将唤醒正在等待任务的子进程，不过只有一个子进程将获得新任务的“接管权”，它可以从工作队列中取出任务并执行之，而其他子进程将继续睡眠在工作队列上。


`__thread` 是 `GCC` 内置的线程局部存储设施，存取效率可以和全局变量相比。`__thread` 变量每一个线程有一份独立实体，各个线程的值互不干扰。可以用来修饰那些带有全局性且值可能变，但是又不值得用全局变量保护的变量。

![](/images/thread_pool/15300252206086.jpg)


![](/images/thread_pool/15300254931346.jpg)


