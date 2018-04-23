---
title: 一次进程的上下文切换需要多长时间
date: 2018-04-13 15:22:52
tags:
			- 翻译
			- 进程
categories: 
			- 计算机系统
			- 并发
			- 翻译
---

[原文](https://blog.tsunanet.net/2010/11/how-long-does-it-take-to-make-context.html).

这个标题不自觉的挑起了我的兴致，我决定付出时间去找出答案。StumbleUpon公司的发布了这样一个假设，即随着Nehalem架构的所有改进(Nehalem架构被用在 i7 处理器上面)，上下文切换将会变得更快。对于这个假设，换做是你，将会如何设计一个测试并且根据你的经验来找到这个问题的答案？究竟一次上下文切换有多昂贵呢？(直接告诉你答案：非常昂贵！)
<!--more-->

据我所知，所有的 CPU 都有一个恒定的时钟频率(定时器)。而所有的Linux内核都是由Ubuntu构建和发布的。

### 第一次尝试：利用系统调用(失败了)
我的第一个想法就是连续的调用多次廉价的`系统调用(system call)`, 计算出所有系统调用花费的总时间和平均时间。Linux 上最廉价的系统调用的触发函数貌似就是[gettid](http://man7.org/linux/man-pages/man2/gettid.2.html)。但是，得到的结果却证明了我的想法是多么的天真！因为现在系统调用实际上不会导致完全的上下文切换，Linux内核可用通过"模式切换"("从用户模式到内核模式，再回到用户模式")来完成系统调用。这就是为什么当我运行我的第一个测试程序时，vmstat不会显示上下文切换次数的明显增加。但是这个测试也很有趣，尽管结果这不是我最初想要的。
**什么是 vmstat？**
> vmstat: 命令是最常见的Linux/Unix监控工具，可以展现给定时间间隔的服务器的状态值,包括服务器的CPU使用率，内存使用，虚拟内存交换情况,IO读写情况。

源代码：[timesyscall.c](https://github.com/tsuna/contextswitch/blob/master/timesyscall.c) 的结果：

	* Intel 5150: 105ns/syscall
	* Intel E5440: 87ns/syscall
	* Intel E5520: 58ns/syscall
	* Intel X5550: 52ns/syscall
	* Intel L5630: 58ns/syscall
	* Intel E5-2620: 67ns/syscall
	
很好，从结果中看到，越是昂贵的 CPU 所表现的性能月好(但请注意，Sandy Bridge的成本略有增加)。但这并不是我们想要的结果。要测试上下文切换的成本，我们必须强制内核取消调度当前进程，然后调度另一个进程(进程切换)。为了对CPU进行基准测试，我们需要让内核在密集的循环中不做任何事情。**How would you do this?**

### 第二次尝试: 使用futex
这一次我使用了滥用[futex[RTFM]](https://zh.wikipedia.org/wiki/Futex)的方式。futex是大多数线程库用于实现阻塞操作（例如等待抢占互斥锁，信号量，条件变量等）的底层的Linux特定基元。如果你想了解更多关于futex的知识，可以阅读Ulrich Drepper所著的[Futexes Are Tricky](https://www.akkadia.org/drepper/futex.pdf)。不管怎么样，利用futex来暂停和恢复进程是非常容易的。我的做法就是fork出一个子进程，然后让父进程和子进程轮流等待futex。当父进程在等待状态的时候，子进程将其唤醒，然后子进程进入等待状态等待futex，直到父进程将子进程唤醒，然后父进程进入等待状态。就像乒乓球一样，你唤醒我，我唤醒你。

源代码：[timectxsw.c](https://github.com/tsuna/contextswitch/blob/master/timectxsw.c) 的结果：

	* Intel 5150: ~4300ns/context switch
	* Intel E5440: ~3600ns/context switch
	* Intel E5520: ~4500ns/context switch
	* Intel X5550: ~3000ns/context switch
	* Intel L5630: ~3000ns/context switch
	* Intel E5-2620: ~3000ns/context switch

注意：这些结果包含了futex系统调用的开销。

上述结果并不准确！在现实中，上下文切换是非常昂贵的，因为它与 CPU 缓存相关联(L1, L2, L3 (如果有的话), 还有 TLB!)。(ps:这些都是高速缓存了，TLB 是页表缓存)



### CPU 亲和力
在 SMP 的环境当中，这个测试是非常困难的，因为性能的表现对于一个任务(task)是否从一个核迁移到另外一个核的影响是非常大的(特别是如果迁移跨越了物理CPU)。我再次运行基准测试，但这次我将进程/线程固定在单个内核（或“硬件线程”）上。让性能提速受限。

源代码：[cpubench.sh](https://github.com/tsuna/contextswitch/blob/master/cpubench.sh) 结果：


未完待续...


