---
title: FastImageCache 架构分析
date: 2018-04-10 21:15:54
tags:
- 第三方框架
categories: 
- 第三方框架
- iOS
---
### 文章介绍
本文章注重分析 [FastImageCache](https://github.com/path/FastImageCache) 这个 Github 第三方图片IO库的架构和部分分析等等。
<!-- more -->
对于 [FastImageCache](https://github.com/path/FastImageCache) 很多同学或多或少都会听过，但是网上很多人说这是一个网络图片库，我只想说一句——"你们是来搞笑的吗？有用过吗？有看过一点点源码吗？", 这个图片库跟网络半毛钱关系都没有，如果真的用过一点点示例的，根本不会说出这句话。

[FastImageCache](https://github.com/path/FastImageCache)是Path团队开发的一个开源库，用于提升图片的加载和渲染速度，让基于图片的列表滑动起来更顺畅，来看看它是怎么做的。

### 加载图片过程
iOS从磁盘加载一张图片，使用UIImageVIew显示在屏幕上，需要经过以下步骤：

1. 用户应用程序调用 read() 函数尝试从物理内存/高速缓存中读取图像数据
2. CPU 模式切换为内核模式，内核应用程序尝试从物理内存/高速缓存中读取图像数据，如果存在则走步骤 3，如果不存在则引起缺页异常，由内核中的缺页异常处理程序确定物理内存中的牺牲页，将磁盘页作为新页调入到物理内存中，更新内存中的页表条目PTE，缺页异常处理程序返回用户应用程序，重新走第 1 步。(关于这部分知识可查看[阅读CSAPP——虚拟内存篇](https://simplecodesky.com/2018/04/03/virtual-memory-from-csapp/))
3. 内核应用程序将图片数据放在内核缓冲区，从内核缓冲区复制数据到用户空间。(关于内核态和用户态的知识，可查看[阅读CASPP——进程篇]())
4. 创建 UIImageView，把图像数据赋值给 UIImageView
5. 如果图像数据为未解码的PNG/JPG，解码为位图数据
6. CATransaction捕获到UIImageView layer树的变化
7. 主线程Runloop提交CATransaction，开始进行图像渲染
	6.1 如果数据没有字节对齐，Core Animation会再拷贝一份数据，进行字节块对齐。
	6.2 GPU处理位图数据，进行渲染。
	
FastImageCache 分别优化了2,4,6.1三个步骤：

1.	使用`mmap`内存映射函数，省去了上述第2步数据从内核空间拷贝到用户空间和内核态的操作和用户态之间的切换消耗。
2.	缓存解码后的位图数据到磁盘，下次从磁盘读取时省去第4步解码的操作。
3.	生成字节块对齐的数据，防止上述第6.1步CoreAnimation在渲染时再拷贝一份数据。

接下来具体介绍这三个优化点以及它的实现。

### 内存映射
什么是用户态和内核态？
[用户模式和内核模式](https://simplecodesky.com/2018/04/03/process-from-csapp/###用户模式和内核模式)
上面这篇博客讲解了用户态和用户态不过是为了防止用户恶意操作操作系统中的数据和硬件而建立的CPU模式机制。这样当用户应用程序想要访问硬件设备的时候，都要进入内核态(一个内存地址空间跟用户态完全不一样的环境，这里可能会发生进程切换，但大部分情况是不会的)。

那么使用了 `mmap` 函数后会优化了什么？
[关于mmap, 内存映射](https://simplecodesky.com/2018/04/03/virtual-memory-from-csapp/)
说白了 `mmap` 就是将磁盘内存地址跟应用程序的虚拟地址一一对应的操作，在真正使用到这些数据前却不会消耗物理内存，也不会有读写磁盘的操作,只有真正使用这些数据时，也就是图像准备渲染在屏幕上时，虚拟内存管理系统VMS才根据缺页加载的机制从磁盘加载对应的数据块到物理内存，再进行渲染，让我们对文件的读写都可以直接在用户态进行，省去了用户态和内核态的切换和缓冲区之间的内存拷贝。(在 Linux 中通常用于多进程访问同一数据的时候进行共享内存的创建，这样就不用多次拷贝内核缓冲区字节到不同的应用程序当中，而且写入数据都能同步到不同的进程)

### 解码图像
一般我们使用的图像是JPG/PNG，这些图像数据不是位图，而是是经过编码压缩后的数据，使用它渲染到屏幕之前需要进行解码转成位图数据，这个解码操作是比较耗时的，并且没有GPU硬解码，只能通过CPU，iOS默认会在主线程对图像进行解码。很多库都解决了图像解码的问题，不过由于解码后的图像太大，一般不会缓存到磁盘，SDWebImage的做法是把解码操作从主线程移到子线程，让耗时的解码操作不占用主线程的时间。
FastImageCache也是在子线程解码图像，不同的是它会缓存解码后的图像到磁盘。因为解码后的图像体积很大，FastImageCache对这些图像数据做了系列缓存管理，详见下文实现部分。另外缓存的图像体积大也是使用内存映射读取文件的原因，小文件使用内存映射无优势，内存拷贝的量少，拷贝后占用用户内存也不高，文件越大内存映射优势越大。

### Data Alignment 字节块对齐
[iOS高效图片 IO 框架是如何炼成的](https://simplecodesky.com/2018/04/10/ios-efficient-image-io/) 中有讲到为何需要字节对齐，而且字节对齐的实现等等。

### FastImageCache 架构实现
通过阅读 [FastImageCache](https://github.com/path/FastImageCache) 后，粗略的画出了以下这幅流程图，途中标识着这个框架的要获取图片的整体流程如何实现:  
![](/images/ios-fastimagecache-opensource-interpretation/1523356546052.jpg)


**那么 [FastImageCache](https://github.com/path/FastImageCache) 是如何实现上述流程的呢？**
关于的总体结构如下图：
![](/images/ios-fastimagecache-opensource-interpretation/fastImageCache3.png)

上图中 ImageCache 是一个单例，用户管理整个图片框架系统。

FastImageCache 为了更好的读存图片数据，将所有字节相同的图片都放在一起，也就是所有格式相同的图片都放在同一个文件系统当中，而在 FastImageCache 中，`imageFormat` 分为 4 种格式：
```
FICImageFormatStyle32BitBGRA,
FICImageFormatStyle32BitBGR,
FICImageFormatStyle16BitBGR,
FICImageFormatStyle8BitGrayscale,
```

* ImageTable：每一种格式都对应一个 `ImageTable` 结构体，一个 `ImageTable` 更是一个真实的文件，可以叫做一个真实的文件系统，命名为 `xxx.table` 文件放在沙盒的 Cache 文件夹当中。
* Chunk：`ImageTable`拥有多个 `Chunk`，而 `Chunk` 被设置为 2M 大小的内存块，存在于 ImageTable 文件系统中，获取的时候通过内存映射 `mmap` 作为共享内存映射到用户空间的虚拟内存当中。
* Entry： `Chunk`存在着多个`Entry`, 因为操作系统当中访问磁盘，都是以磁盘页和内存页的方式存在于磁盘和物理内存当中，在 64 位 Unix操作系统中，一般为 **4KB**，这个大小可以通过设置操作系统更改，所以 `Entry` 为了加快访问数据字节访问获取速度和节省由于页不对齐而导致的 CPU 周期增加的消耗，就采用页对齐的方式。
`Entry` 中存储着一个图片的字节数组，和 `Meta(ImageCache用到的一些 UUID 的唯一图片标识)`，而 `ImageData` 为了避免发生 copy_images 的发生，更是采用了字节块对齐。

而这些实现细节，可以查看[iOS高效图片 IO 框架是如何炼成的](https://simplecodesky.com/2018/04/10/ios-efficient-image-io/)。

文献参考：
[《iOS图片加载速度极限优化—FastImageCache解析》](https://blog.cnbang.net/tech/2578/)

