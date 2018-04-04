---
title: 阅读CASPP——进程篇
date: 2018-04-03 21:22:22
tags: 笔记与理解
---

在现代系统上运行一个程序时，我们会得到一个假象，就好像我们的程序是系统中当前运行的唯一的程序一样。我们的程序好像是独立地使用处理器和内存。处理器就好像是系统中当前运行的唯一程序一样。我们的程序就好像是独占地使用处理器和内存。处理器就好像是无间断地一条一条地执行我们程序中的指令。最后，我们程序中的代码和数据好像是系统内存唯一对象。而这些假象都是通过进程的概念提供给我们的。

未完待续...

<!-- more -->

学前疑问:
1. 什么是进程？为什么需要进程？
2. 进程的组成部分？
3. 什么是进程的上下文？
4. 什么是上下文切换，为什么需要上下文切换?
5. 模式切换？上下文切换？
6. 什么是 I/O 操操?

### 什么是进程?
进程的经典定义就是一个执行中程序的实例。当我们使用 shell 输入命令行 './helloWorld.out' 以运行 helloWorld 可执行文件的时候，shell 就会创建一个新的进程，然后再这个新进程的上下文中运行这个可执行文件。
也可以说进程、线程、异常处理程序等是一个逻辑。

### 进程的地址空间
进程为每个程序提供给它自己的私有空间，这就制造了进程独占地使用系统地址空间。
![进程地址空间](/images/process-from-csapp/1438747148_9194.png)

上图展示了程序的地址空间分配图，在 Linux 下，一个进程的地址空间一般为4GB(内核地址空间1G，用户地址空间3GB)，这里所说的地址空间都为虚拟内存地址(关于什么是虚拟内存，查看<a href="https://simplecodesky.com/2018/04/03/virtual-memory-from-csapp/"><span style="color:red">阅读CSAPP——虚拟内存篇</span></a>)。
可以看出一个程序的内存空间分了**内核空间**和**用户空间**，而用户空间是无法访问内核空间的地址的。如果一个用户进程必须通过系统调用接口(例如read，write 函数)间接地访问内核代码和数据。

**系统调用**
系统调用在类Unix系统中指的是活跃的进程对内核所提供的服务进行请求。例如输入/输出(I/O和进程创建)。

**I/O 操作**
I/O 可以被定义为任何信息流入或流出 CPU 与主内存（RAM）。也就是说，一台电脑的 CPU和内存与该电脑的用户（通过键盘或鼠标）、存储设备（硬盘或磁盘驱动）还有其他电脑的任何交流都是 I/O。

### 进程的上下文
内核为每个进程维持着一个上下文，上下文由一些对象的值组成，这些对象包括通用目的的寄存器，浮点寄存器，程序计数器、用户栈、状态寄存器、内核栈和各种内核数据结构(比如描述地址空间的页表，包含有关当前进程信息进程表，以及包含进程已打开文件的信息的文件表)。
简而言之，进程的上下文就是保存了进程的状态和堆栈内存的数据结构等等。

### 进程的上下文切换
当某些时刻，例如读写文件等操作时进程发生了阻塞的行为，内核中的调度器(代码实现, 也就是说上下文切换只能发生在内核模式中),会让当前进程A休眠，切换到另外一个进程B，这个时候就会发生以下三步操作：
 * 保存进程A的所有状体，也就是进程的上下文。
 * 恢复进程B被保存上下文。
 * 将控制传递给进程B。

这三个步骤就称为**上下文切换**。当然，即使是系统调用没有发生阻塞时，内核也可以决定执行上下文切换。例如中断行为，系统会有一个周期性的中断定时器，每隔一段时间当定时器发生中断时，内核就能判断当前进程运行的时间足够长，并切换到一个新的进程。(这也是多任务并发的思想，叫做时间分片)。

### 用户模式和内核模式
用户模式和内核模式一般被称为**用户态**和**内核态**。是 CPU 提供给的一种机制，是 CPU 的一种模式状态,用于限制一个应用可以执行的指令以及它可以访问的地址空间范围。
<p><span style= "color:red">注意：这是一种模式切换，而不是上下文切换，因为它不一定引起进程状态的转换，在大多数情况下，也不一定引起进程切换。</span></p>
* **用户态**这样做可以将每个用户进程都能独立开来,使其无法更改属于其他应用程序的数据。每个应用程序都孤立运行，如果一个应用程序损坏，则损坏会限制到该应用程序。其他应用程序和操作系统不会受该损坏的影响。
* **内核态**下运行的所有代码都共享单个虚拟地址空间, 进程可以执行指令集中的任何指令，并且可以访问系统中的任何内存位置。但这表示内核态进程未从其他进程和操作系统自身独立开来，如果内核态进程意外写入错误的虚拟地址，则属于操作系统或其他驱动程序的数据可能会受到损坏。

整个操作系统分为两层：**用户态**和**内核态**，这种分层的架构极大地提高了资源管理的可扩展性和灵活性，而且方便用户对资源的调用和集中式的管理，带来一定的安全性。以下这幅图详尽的分析了内核所做的事情:

![](/images/process-from-csapp/431521-20160523181544475-414696764.jpg)

很多程序开始时运行于**用户态**，但在执行的过程中，一些操作需要在内核权限下才能执行，这就涉及到一个从**用户态**切换到**内核态**的过程。比如C函数库中的内存分配函数`malloc()`，它具体是使用`sbrk()`系统调用来分配内存，当`malloc`调用`sbrk()`的时候就涉及一次从**用户态**到**内核态**的切换，类似的函数还有`printf()`，调用的是`wirte()`系统调用来输出字符串，等等。

**何时切换？**
到底在什么情况下会发生从用户态到内核态的切换，一般存在以下三种情况：

1. 当然就是系统调用：原因如上的分析。

2. 异常事件： 当CPU正在执行运行在用户态的程序时，突然发生某些预先不可知的异常事件，这个时候就会触发从当前用户态执行的进程转向内核态执行相关的异常事件，典型的如缺页异常。

3. 外围设备的中断：当外围设备完成用户的请求操作后，会像CPU发出中断信号，此时，CPU就会暂停执行下一条即将要执行的指令，转而去执行中断信号对应的处理程序，如果先前执行的指令是在用户态下，则自然就发生从用户态到内核态的转换。

从触发方式和效果上来看，这三种切换方式是完全一样的，都相当于是执行了一个中断响应的过程。但是从触发的对象来看，系统调用是进程主动请求切换的，而异常和硬中断则是被动的。

![](/images/process-from-csapp/1522851295053.jpg)
