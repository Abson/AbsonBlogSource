---
title: iOS高效图片 IO 框架是如何炼成的
date: 2018-04-10 18:39:04
tags: 
			- iOS 
categories: 
			- 计算机系统
			- iOS
---

当我们使用图片存储的时候，难免会设计到文件IO，GPU渲染等问题，文章注重从计算机操作系统方面深入浅析地讲解如何优化图片IO的速度，提高 iOS 中 UIImageView 的渲染效率和内存优化，这对我们做多图片相册等应用会非常有帮助，而且让我们把[阅读CASPP——进程篇](https://simplecodesky.com/2018/04/03/process-from-csapp/)和[阅读CSAPP——虚拟内存篇](https://simplecodesky.com/2018/04/03/virtual-memory-from-csapp/)这两篇文章学到的内容进行实战应用。

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

### 如何解决？
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
**通过`_imageRowLength`来创建位图，由于图像字节块已经对齐, 避免 `CA::Render::copy_image` 图像数据的拷贝发生了.**
```objectivec
// 创建 CGDataProviderRef 用于位图创建，提供图像数据和数据结构的 Release 函数
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

