---
title: iOS高效图片 IO 框架是如何炼成的
date: 2018-04-10 18:39:04
tags: 
			- iOS 
categories: 
			- 计算机系统
			- iOS
---

当我们使用图片存储的时候，难免会涉及到文件IO，GPU渲染等问题，文章注重从计算机操作系统方面深入浅析地讲解如何优化图片IO的速度，提高 iOS 中 UIImageView 的渲染效率和内存优化，这对我们做多图片相册等应用会非常有帮助，而且让我们把[阅读CASPP——进程篇](https://abson.github.io.com/2018/04/03/process-from-csapp/)和[阅读CSAPP——虚拟内存篇](https://abson.github.io.com/2018/04/03/virtual-memory-from-csapp/)这两篇文章学到的内容进行实战应用。

<!-- more -->

### 图像数据拷贝？
当我们使用以下 Object-C 代码从网络中获取图片并加载到 UIImageView 上
```objectivec 
   NSURL* url = [NSURL URLWithString:@"https://img.alicdn.com/bao/uploaded/i2/2836521972/TB2cyksspXXXXanXpXXXXXXXXXX_!!0-paimai.jpg"];
    __weak typeof(self) weakSelf = self;
    NSURLSessionDataTask *task = [[NSURLSession sharedSession] dataTaskWithRequest:[[NSURLRequest alloc] initWithURL:url]
                                                                 completionHandler:^(NSData *data, NSURLResponse *response, NSError *error) {
                                                                     UIImage* image = [UIImage imageWithData:data];
                                                                     dispatch_async(dispatch_get_main_queue(), ^{
                                                                         [weakSelf.imageView setImage:image];
                                                                     });
                                                                 }];
    [task resume];
```
运行上面代码，通过 Instrument 的 TimeProfile 查看 CPU 消耗情况：
![Alt text](/images/ios-efficient-image-io/1523343490248.jpg)

上述图片中发现了两个问题：

1. 应用程序使用了 `CA::Render::copy_image`, 这是因为 Core Animation 在图像数据非字节对齐的情况下渲染前会先拷贝一份图像数据，当我们使用 `imageWithContentsOfFile` 也会发生这种情况。
2. 应用程序使用了 `CA::Render::create_image_from_provider`, 这个方法实际上是对图片进行解码，原因是 UIImage 在加载的时候实际上并没有对图片进行解码，而是延迟到图片被显示或者其他需要解码的时候。这种策略节约了内存，但是却会在显示的时候占用大量的主线程CPU时间进行解码，导致界面卡顿。

那么如果解决上述两个问题，我们使用 **FastImageCache** 这个第三方库加载图片，官方Demo一开始的 `FICDPhoto` 加载图片的方法为使用 `imageWithContentsOfFile`, 这会导致 `CA::Render::copy_image` 的图像数据拷贝，所以更改为以下方法：
```objectivec 
- (UIImage *)sourceImage {
    __block UIImage *sourceImage = [UIImage imageWithContentsOfFile:[_sourceImageURL path]];
    if (!sourceImage) {
        pthread_mutex_lock(&_mutex);
        NSURLSessionDataTask *task = [[NSURLSession sharedSession] dataTaskWithRequest:[[NSURLRequest alloc] initWithURL:_sourceImageURL]
                                        completionHandler:^(NSData *data, NSURLResponse *response, NSError *error) {
                                            sourceImage = [UIImage imageWithData:data];
                                            pthread_cond_signal(&_signal);
                                            pthread_mutex_unlock(&_mutex);
                                        }];
        [task resume];
        pthread_cond_wait(&_signal, &_mutex);
        pthread_mutex_unlock(&_mutex);
    }
    return sourceImage;
}
```
当图片被渲染到 ImageView 的时候，利用 Instrument 查看并没有发生 `CA::Render::copy_image` 和 `CA::Render::create_image_from_provider` 的情况。

#### 如何优化？
**FastImageCache是如何解决上述两个问题的？**
先看第一个问题，为何  Core Animation 在渲染之前要拷贝一份数据呢？
在此之前，我们先看一些计算机系统的理论知识，等看完后再回头看看，答案便更加明朗了。
我们都知道，图片说白了就是一段字节数据所组成的文件数据而已，也就是说我们把图片显示到界面上，不过是把一堆字节加载到 CPU 的寄存器当中，然后通过 GPU 将字节变成红(R)、绿(G)、蓝(B)的三原色的数组，然后通过界面显示出来(也就是我们所说的渲染)。所以，我们先了解一下，字节数据是如何通过内存加载到 CPU 的。

学过计算机操作系统的我们都知道，所有的字节数据都是通过总线来传送的，总线连接着 CPU 到主存，主存到磁盘等主要硬件设备的传输路线，而主要的传输单位就是[字(word)](https://zh.wikipedia.org/wiki/%E5%AD%97_(%E8%AE%A1%E7%AE%97%E6%9C%BA))，由于数字信号分为高频和低频，所以我们的计算机信号只有 0 和 1 两种来区分，所以我们采用了二进制的数据形式来表述数据，因此信号的量化精度一般以比特（bits）来衡量，这就是字节数据在总线中传输的本质。

![关于计算机硬件组成](/images/ios-efficient-image-io/CSAPP_C1_P1.png)

而 64 位系统当中，**字(word)** 是 8 个字节的大小。
内存的存储单元被称为[块(Chunk)](https://zh.wikipedia.org/wiki/%E5%9D%97_(%E6%95%B0%E6%8D%AE%E5%AD%98%E5%82%A8))，而块的大小因硬件设备而定的，大多数[文件系统](https://zh.wikipedia.org/wiki/%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F)都是基于块设备，即存取规定数据块的硬件抽象层。
假如32位的文件系统中，[高速缓存(Cache)](https://zh.wikipedia.org/wiki/CPU%E7%BC%93%E5%AD%98)以4个字节的块大小为传送到 CPU 当中，下图说明了 CPU 如何以4字节内存访问粒度访问4字节的数据查找：

![Alt text](/images/ios-efficient-image-io/align01.jpg)

如果我们们获取的数据为未对齐的4个字节，CPU 就会执行额外的工作去访问数据：加载两个字节块(Chunk)的数据，移出不需要的字节，然后将它们组合在一起。这个过程肯定会降低性能并浪费CPU周期，以便从内存中获取正确的数据。

![Alt text](/images/ios-efficient-image-io/align02.jpg)

所以我们存储数据的时候，需要进行 **[字节对齐(Byte Alignment)](https://zh.wikipedia.org/wiki/%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84%E5%AF%B9%E9%BD%90)** ，其实称为字节块对齐更为合适，也就是说我们所取的数据，都以上图第一种的形式在内存中读写，避免发生第二种情况，这样就能节省 CPU 周期，加快存取速度。

再回头看看当我们从内存中读取图片数据的时候，也是一堆的字节块，如果图像字节并没有经过如何处理，那么，就会出现以下情况：

![Alt text](/images/ios-efficient-image-io/fastImageCache2.png)

当我们数据传输的时候，由**字**来传输，存储设备中读取内存却以**块**为单位, 所以从内存读取到的图像数据势必会带上其他"杂质字节"。所以便发生了——"通常情况"，但是 GPU 所需要的数据，是理想状态下的数据，因为这些"杂质字节"会影响图像生成。所以 Core Animation 就需要把"杂质字节"去除，然后变成我们的——"理想状态"。其实，不是 Core Animation 规定这么做，是我们接粗到图像处理的时候，都必须这么做，不然图像数据就乱了，只不过 Core Animation 帮我们封装了底层处理而已。
通过我们学习到的知识，我们意识到，我们需要对图片的存入进行字节对齐。
**对于高速缓存(Cache)来说，存取都是以字节块的形式，而块的大小跟 CPU 的高速缓存存储器有关，ARMv7是 32 Byte，A9是 64 Byte，在 A9 下CoreAnimation 应该是按 64 Byte(也就是8个字，8Byte/字) 作为一块数据去读取、存储和渲染，让图像数据对齐64 Byte 就可以避免CoreAnimation再拷贝一份数据。能节约内存和进行copy的时间。(因为图片存入的时候已经对齐过了，获取的时候自然也是字节对齐的)，**

**如何字节块对齐避免 Core Animation 进行图像数据复制？**
以下是代码形式进行实操：
**计算图像所需字节大小**
```objectivec
/** FICImageTable.m */

CGSize pixelSize = [_imageFormat pixelSize]; // 想要展示的图片大小
NSInteger bytesPerPixel = [_imageFormat bytesPerPixel]; // 该图像个是中的字节每像素, 例如 FICImageFormatStyle32BitBGRA 为32位4个字节
_imageRowLength = (NSInteger)FICByteAlignForCoreAnimation((size_t) (pixelSize.width * bytesPerPixel));
_imageLength = _imageRowLength * (NSInteger)pixelSize.height;
```
通过 `FICByteAlignForCoreAnimation` 函数对图片数据进行字节对齐然后计算， 得到 `_imageRowLength` 图像每行的字节数, **图像所需字节 = 图像的高度 * _imageRowLength(字节块对齐的图像每行字节数)**。 

**通过实际图像每行所需的字节进行字节块对齐**
```cpp
inline size_t FICByteAlignForCoreAnimation(size_t bytesPerRow) {
    return FICByteAlign(bytesPerRow, 64); // 跟 CPU 的高速缓存器有关
}
```

**让为 width 成为 alignment 的倍数计算**
```cpp

inline size_t FICByteAlign(size_t width, size_t alignment) {
    return ((width + (alignment - 1)) / alignment) * alignment;
}
```

**创建Entry所对应的 Chunk，而 Chunk 是页对齐的**
```objectivec
// 设置每一个 entry 的字节长度，因为除了图像数据外，fastImageCache 还额外为图像添加了两个 UUID 的 32 个字节
_entryLength = (NSInteger)FICByteAlign(_imageLength + sizeof(FICImageTableEntryMetadata), (size_t) [FICImageTable pageSize]);

entryData = [[FICImageTableEntry alloc] initWithImageTableChunk:chunk bytes:mappedEntryAddress length:(size_t) _entryLength];
```
为什么要进行页对齐？因为对于磁盘来讲，磁盘中的字节块大小就是页，因为`分页`就是磁盘和物理内存的存储方式，这样做就可以节省读取 `entryData`时CPU周期, 这是跟字节对齐一样的道理。

**通过`_imageRowLength`来创建位图，由于图像字节块已经对齐, 避免 `CA::Render::copy_image` 图像数据的拷贝发生了.**
```objectivec
// 创建 CGDataProviderRef 用于图像上下文创建，提供图像数据和数据结构的 Release 函数
CGDataProviderRef dataProvider = CGDataProviderCreateWithData((__bridge_retained void *)entryData, [entryData bytes], [entryData imageLength], _FICReleaseImageData);

CGSize pixelSize = [_imageFormat pixelSize]; // 想要展示的图片大小
CGBitmapInfo bitmapInfo = [_imageFormat bitmapInfo]; // 位图数据的信息，例如是大小端，计算位数等
NSInteger bitsPerComponent = [_imageFormat bitsPerComponent]; // 每个组成的位数，32位RGBA、RGB和8位Gray都为8bit，而16位的RGB为5bit
NSInteger bitsPerPixel = [_imageFormat bytesPerPixel] * 8; // bit每个像素
CGColorSpaceRef colorSpace = [_imageFormat isGrayscale] ? CGColorSpaceCreateDeviceGray() : CGColorSpaceCreateDeviceRGB();
 
CGImageRef imageRef = CGImageCreate((size_t) pixelSize.width, (size_t) pixelSize.height, (size_t) bitsPerComponent, (size_t) bitsPerPixel,(size_t) _imageRowLength, colorSpace, bitmapInfo, dataProvider, NULL, false, (CGColorRenderingIntent)0);
CGDataProviderRelease(dataProvider);
CGColorSpaceRelease(colorSpace);
```

### 文件读取，资源消耗？
当我们从磁盘当中获取图像数据时，必须调用`read()`函数来从磁盘中读取图像字节数据，那么一起看看 read() 函数的调用过程:

图3-1
![1523520326722](/images/ios-efficient-image-io/1523520326722.jpg)

图3-1展示了当应用程序调用`read()`函数的时候：

1. CPU 接受到中断信号，进入了内核模式。
2. 内核模式中利用内核模式程序访问[高速缓存(Cache)](https://zh.wikipedia.org/wiki/CPU%E7%BC%93%E5%AD%98)，查看高速缓存是否存在图像数据，如果又则返回，没有则继续访问物理内存。
3. 内核模式程序读取物理内存，查看是否存在对应的`物理页`(由于磁盘跟物理内存的存储方式都是以`分页`的方式划分数据块的，通常64位系统为 4KB/页)，如果存在则将物理页数据，则返回相对应图像数据的物理页没有则发生异常行为(页错误)。
4. **缺页异常处理程序**访问磁盘，找到磁盘中对应的图像数据加载成`磁盘页`, 并磁盘页作为新页替换物理内存中的物理页，然后将物理页数据作为`字节块`缓存到高速缓存当中
5. 异常处理程序发出中断信号将控制返回内核程序，内核程序再次加载高速缓存字节块返回放置`内核缓冲区`。
6. 由于内核程序跟用户程序的内存地址空间是完全不同的，所以对于虚拟内存来讲，内核程序要降内核缓冲区中的数据字节进行一次拷贝，才能返回给用户程序.
**(PS: 用户程序跟内核程序的概念中，可能两者都为同一程序，只是由于CPU切换模式而转换，也有可能是两个不同的程序)**

CPU 读取内存页过程:
>![](/images/virtual-memory-from-csapp/1522750928874.jpg)


当然，上面的分析是针对逻辑层面跟硬件层面的讲解，实际上软件层面上，对于一次磁盘请求如下：

图3-2显示了 read 系统调用在核心空间中所要经历的层次模型。从图中看出, 对于磁盘的一次读请求:

* 首先经过虚拟文件系统层（vfs layer）
* 其次是具体的文件系统层（例如 ext2）
* 接下来是 cache 层（page cache 层）
* 通用块层（generic block layer）
* IO 调度层（I/O scheduler layer）
* 块设备驱动层（block device driver layer）
* 最后是物理块设备层（block device layer）

图3-2 read 系统调用在核心空间中的处理层次
![](/images/ios-efficient-image-io/15235197670131.gif)

(对于这部分目前先留个悬念，以后分享在详细讲解文集系统)

通过上面对`read()`函数的分析，我们知道从磁盘中读取一次文件的操作是非常繁碎而且非常消耗资源的(特别是大文件)，而且由于物理内存和高速缓存的资源是有限的，当我们不再访问图像数据的时候，图像数据就会被当做`牺牲页`换出物理内存和高速缓存，当我们应用程序后面再次`read()`访问的时候，还得再次重新走上述流程。
那么这个时候我们可以怎么优化来加快我们对图片的IO呢？

#### 如何优化？
对于我们iOS这种封闭系统来讲，优化手段其实很有限，因为我们不能直接操作内核，但是在是不是就无法优化呢？而操作系统为我们提供了一个用户级的内核函数，`mmap/ummap`，这是一个实现[内存映射](https://en.wikipedia.org/wiki/Memory-mapped_file)做法，那么内存映射能为我们读写文件的操作带来什么？
答案就是优化了上述流程的 1 和 6 中所产生的内存拷贝过程，我们首先来看看内存映射是什么。

操作系统通过将一个虚拟内存区域与一个磁盘上的对象(object)关联起来，以初始化这个虚拟内存区域的内容，这个过程称为内存映射(memory mapping).

下图展示了内存映射的做法：
![](/images/ios-efficient-image-io/15235244516302.jpg)

下图展示了内存映射区域在进程中的位置：
![](/images/ios-efficient-image-io/20160114201303391)

当磁盘文件通过内存映射到应用程序的时候，是直接跟用户空间的地址相关联的，也就是说当我们读取磁盘文件数据的时候，CPU 不用在切换用户空间和内核空间，随之**字节拷贝也不会再发生，所有读取操作都能在用户空间中进行**。

好了，说了这么多，具体做法怎么做呢？
在 FastImageCache 当中，创建 Chunk 直接文件中的对一个 Chunk 内存区域部分进行内存映射：
```objectivec
// FICImageTableChunk.m

- (instancetype)initWithFileDescriptor:(int)fileDescriptor index:(NSInteger)index length:(size_t)length {
    self = [super init];
    
    if (self != nil) {
        _index = index;
        _length = length;
        _fileOffset = _index * _length;
        // 通过内存映射设置为共享内存文件
        _bytes = mmap(NULL, _length, (PROT_READ|PROT_WRITE), (MAP_FILE|MAP_SHARED), fileDescriptor, _fileOffset);

        if (_bytes == MAP_FAILED) {
            NSLog(@"Failed to map chunk. errno=%d", errno);
            _bytes = NULL;
            self = nil;
        }
    }
    
    return self;
}
```
这里就有一个疑问了，通过[FastImageCache 架构分析](https://abson.github.io.com/2018/04/10/ios-fastimagecache-opensource-interpretation/)文章中知道，一个图片文件应该对应为一个`Entry`才对呀，为什么现在内存映射要映射`Chunk`呢？
因为内存映射文件越大越有效果呀，不然小数据通过`read()`函数直接进入内核拷贝字节这种做法就跟内存映射没有对比了。

为了让映射文件越大， FastImageCache 甚至直接在存储图片的时候就直接降图片解码了：
```objectivec
- (void)setEntryForEntityUUID:(NSString *)entityUUID sourceImageUUID:(NSString *)sourceImageUUID imageDrawingBlock:(FICEntityImageDrawingBlock)imageDrawingBlock {
    if (entityUUID != nil && sourceImageUUID != nil && imageDrawingBlock != NULL) {
        [_lock lock]; // 递归锁
        // 创建 Entry
        NSInteger newEntryIndex = [self _indexOfEntryForEntityUUID:entityUUID];
        if (newEntryIndex == NSNotFound) {
            newEntryIndex = [self _nextEntryIndex];
            
            if (newEntryIndex >= _entryCount) {
                // Determine how many chunks we need to support new entry index.
                // Number of entries should always be a multiple of _entriesPerChunk
                NSInteger numberOfEntriesRequired = newEntryIndex + 1;
                NSInteger newChunkCount = _entriesPerChunk > 0 ? ((numberOfEntriesRequired + _entriesPerChunk - 1) / _entriesPerChunk) : 0; 
                NSInteger newEntryCount = newChunkCount * _entriesPerChunk;
                [self _setEntryCount:newEntryCount]; 
            }
        }
        
        if (newEntryIndex < _entryCount) {
            CGSize pixelSize = [_imageFormat pixelSize];
            CGBitmapInfo bitmapInfo = [_imageFormat bitmapInfo];
            CGColorSpaceRef colorSpace = [_imageFormat isGrayscale] ? CGColorSpaceCreateDeviceGray() : CGColorSpaceCreateDeviceRGB();
            NSInteger bitsPerComponent = [_imageFormat bitsPerComponent];
            
            // Create context whose backing store *is* the mapped file data
            FICImageTableEntry *entryData = [self _entryDataAtIndex:newEntryIndex]; // 创建内存映射区域
            if (entryData != nil) {
                [entryData setEntityUUIDBytes:FICUUIDBytesWithString(entityUUID)];
                [entryData setSourceImageUUIDBytes:FICUUIDBytesWithString(sourceImageUUID)];
                
                // Update our book-keeping
                _indexMap[entityUUID] = @((NSUInteger) newEntryIndex);
                [_occupiedIndexes addIndex:(NSUInteger) newEntryIndex];
                _sourceImageMap[entityUUID] = sourceImageUUID;
                
				 // 用于内存最近使用策略来装载和释放内存
                [self _entryWasAccessedWithEntityUUID:entityUUID];
                [self saveMetadata];
                
                // Unique, unchanging pointer for this entry's index
                NSNumber *indexNumber = [self _numberForEntryAtIndex:newEntryIndex];
                
                // Relinquish the image table lock before calling potentially slow imageDrawingBlock to unblock other FIC operations
                [_lock unlock];
                // 利用创建位图，将图图象数据draw到位图当中，然后再保存位图字节数据
                CGContextRef context = CGBitmapContextCreate([entryData bytes], (size_t) pixelSize.width, (size_t) pixelSize.height,
                        (size_t) bitsPerComponent, (size_t) _imageRowLength, colorSpace, bitmapInfo);
                
                CGContextTranslateCTM(context, 0, pixelSize.height);
                CGContextScaleCTM(context, _screenScale, -_screenScale);
                
                @synchronized(indexNumber) {
                    // Call drawing block to allow client to draw into the context
                    // 解码
                    imageDrawingBlock(context, [_imageFormat imageSize]);
                    CGContextRelease(context);
                
                    // Write the data back to the filesystem
                    [entryData flush];
                }
            } else {
                [_lock unlock];
            }
            
            CGColorSpaceRelease(colorSpace);
        } else {
            [_lock unlock];
        }
    }
}
```
这里涉及到了[递归锁](https://blog.csdn.net/zouxinfox/article/details/5838861)，主要是防止多次调用`lock`造成死锁，有机会再次跟大家分享递归锁的神奇用法。

代码中`-_entryDataAtIndex`方法便创建了`Chunk`，但此时`Chunk`并没有数据，只是做了一个文件的映射区域.
利用`-_indexOfEntryForEntityUUID`创建了`Entry`, 分配了图像所需的字节和UUID等metaData所需的字节内存空间。然后我们使用`-CGBitmapContextCreate`利用内存空间创建位图，通过`-imageDrawingBlock`将图片字节全部`draw`到位图，然后通过`Entry flush` 就图像数据同步到磁盘当中, 这就完成了图片的存储了。

#### 存在的问题
但是内存映射是不是就没有缺陷呢？
通过上文学习我们知道，内存映射是直接对应这虚拟内存区域的，也就是说是占用这我们虚拟内存的地址空间的，而且是一块常驻内存，那么当映射内存非常大的时候，甚至会影响我们程序的堆内存创建反而导致性能更差。所以 FastImageCahce 中甚至给`Entry`做了内存限制，一个`Entry`只能存储两M的数据.

```objectivec
NSInteger goalChunkLength = 2 * (1024 * 1024);
NSInteger goalEntriesPerChunk = goalChunkLength / _entryLength;
_entriesPerChunk = (NSUInteger) MAX(4, goalEntriesPerChunk); // 最少也要存在 4Entry/Chunk
_chunkLength = (size_t)(_entryLength * _entriesPerChunk); // Chunk 的内存字节大小
```
实际大小要跟着图像的大小改变，但是跟 2M 不会相差太多。

未完待续...(后续学到新方法，会持续更新)

通过上述方法，就能有效的加快我们图片的文件IO，特别当前我们的女性用户，手机里面有几十G图图片，当我们要做一个图片相册的精美应用的时候，这些性能便不可忽视了。根据摩尔定律，计算机性能**约每隔18-24个月便会增加一倍，性能也将提升一倍**，但是用户的要求和使用方式也会随着时间不断提高的！所以，平常培养对计算机原理的深入理解，才能写出高性能的代码，才不会在面对高并发高内存的情况下素手无策。

参考文献：
[《Linux驱动mmap内存映射》](https://blog.csdn.net/jsn_ze/article/details/53561010)

