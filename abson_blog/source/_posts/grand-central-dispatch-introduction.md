---
title: 深入浅出 GCD 线程使用 
date: 2018-04-02 19:50:49
tags: iOS 
---

![](/images/cpp-project-new-learning/1-1406291220161Z.jpeg)
<!-- more -->

#### 串行与并行
同步和异步针对的是**线程队列**，所谓的线程队列可以理解为一组线程的数组。

**串行队列：**
队列中是事件有序执行，遵循 FIFO（first in first out）的原则，先进入队列的事件先执行。

串行队列创建：
```objectivec
dispatch_queue_t queue = dispatch_queue_create("com.queue.serial", DISPATCH_QUEUE_SERIAL);

dispatch_get_main_queue() // 主队列，也是串行队列
```

**并行队列**
并行队列中的事件在逻辑上是一起执行的，但是这是要根据机器 CPU 的情况而定，在 C++ 线程库中，`std::thread::hardware_concurrency()` 能获取到当前机器最大能并发的线程数量，iPhone6P 中为 2，也就是说最大同时能处理两个并发线程任务，其他后面添加的任务都得等待两个任务中的其中一个执行完了，才可以执行。

```objectivec
dispatch_queue_t queue = dispatch_queue_create("com.queue.concurrent", DISPATCH_QUEUE_CONCURRENT);

dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0); // 全局并发队列
```

#### 同步和异步 
同步和异步针对的是**线程**，那么什么是同步线程，什么是异步线程。

**同步线程：**
阻塞当前线程，要等待同步线程内的任务执行完了并且返回以后，才可以继续执行被阻塞线程的事件。

同步线程创建：
```objectivec
dispatch_sync(queue, block);
```

**异步线程：**
不阻塞当前线程，等当前线程完成时间片（完成当前事件）切换后再执行异步线程。

异步线程创建：
```objectivec
dispatch_async(queue, block);
```

![](/images/cpp-project-new-learning/dispatch_queue.jpeg)

#### 线程问题

##### 主线程中的死锁
```
NSLog(@"1");
dispatch_sync(dispatch_get_main_queue(), ^(){
	NSLog(@"2");
});
NSLog(@"3");
```
>输出：1

如果上面代码是在**主线程**当中执行的，那么就会造成我们的死锁问题，注意是主线程当中，后面我们还有一个测试说明。
假定上面代码为主线程中执行的代码，如果不造成死锁的情况是输出应该是 1，2，3，但现在事件只执行了 1，那么死锁就很明显了，我们现在对它进行分析。

 dispatch_sync **同步线程**，将当前线程阻塞，先执行block（@"2") 然后解放线程
dispatch_get_main_queue **主线程队列**，也可以叫做**串行队列**，将 dispatch_sync 同步线程放到队列后，先执行 ( @"3") 再执行同步线程，遵循 FIFO 的原则。
当时因为 dispatch_sync 是在主线程创建的，所以主线程被阻塞，主线程的事件(@"3") 要等待 dispatch_sync 的 block 执行完后才能执行
所以事件(@"3")无法执行，事件(@"2")更无法执行，相互等待造成死锁。

**`dispatch_sync(dispatch_get_main_queue(), block)`是否一定会造成死锁呢？上面问题如果并不是放在主线程中有会怎么样？**

```
NSLog(@"1");
dispatch_queue_t queue = dispatch_queue_create("com.queue.concurrent", DISPATCH_QUEUE_CONCURRENT);
dispatch_async(queue), ^(){
	NSLog(@"2");
	dispatch_sync(dispatch_get_main_queue(), ^(){
		NSLog(@"3");
	});
	NSLog(@"4");
});
NSLog(@"5");
```

>输出: 1，5，2，3，4

输出中，可以看得出所有事件全部都执行完成，没有造成死锁，但是明明使用了 `dispatch_sync(dispatch_get_main_queue(), block);`这个经常被说成会造成死锁的方法，但是为什么这里没有造成死锁呢，我们来分析一下。

`dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT,block)` 中， dispatch_async **异步线程**，将其放在了 dispatch_get_global_queue **全局队列**，也可以叫**并行队列**中，主线程不用等待异步 dispatch_async 内的事件（block）执行完成，所以直接执行了事件(@“1”)和事件(@"5")。当线程时间片切换出来，异步线程内的事件(block)便开始执行了，所以事件(@"2") 便执行了。
当运行到 `dispatch_sync(dispatch_get_main_queue(),block)` 中，dispatch_sync 阻塞当前线程，细想一下，**当前线程是一个异步线程并不是主线程**，事件(@"4")又是在这个异步线程中的事件，所以要等待 dispatch_sync 同步线程内的事件执行完了，才可以执行。同步线程放在 dispatch_get_main_queue 主线程队列中，主线程队列同时也是一个串行队列，所以事件(@"3") 一定会在事件@("1")和事件(@"5")之后，当执行完事件(@"3")便可以执行事件(@"4")了。

上面例子说明一件事，**dispatch_async 同步线程会阻塞当前线程直至同步线程内的事件(block)执行完，至于是否会发生死锁，就得看同步线程所阻塞的线程是否存在它的线程队列（queue）中**。

```
current thread

dispatch_sync(queue), block)
```

第一个例子中，`current thread` 为主线程，`queue` 主线程队列，主线程属于主线程队列，所以造成死锁。

第二个例子中，`current thread` 为我们所开启的异步线程 dispatch_async，并且放在我们自己所创建的 `dispatch_queue_t queue = dispatch_queue_create("com.queue.concurrent", DISPATCH_QUEUE_CONCURRENT);` 异步线程队列中，`queue ` 为主线程队列，异步线程  dispatch_async 并不属于主线程队列中，所以并没有造成死锁。


##### 异步串行队列和同步串行队列

首先我们做一个比较，在串行队列中开启一个异步线程，然后再异步线程的事件中再开启一个同步线程。（默认下面例子都是在主线程中运行）

```objectivec
dispatch_queue_t queue = dispatch_queue_create("com.queue.CONCURRENT", DISPATCH_QUEUE_CONCURRENT);

NSLog(@"1");
dispatch_async(queue, ^() {
	NSLog(@"2");
    dispatch_sync(queue, ^(){
      NSLog(@"3");
    });
    NSLog(@"4");
});

NSLog(@"5");
```

>输出：1，5，2，3，4

然后将 queue 换成一个串行队列，看看效果如何

```objectivec
dispatch_queue_t queue2 = dispatch_queue_create("com.queue.SERIAL", DISPATCH_QUEUE_SERIAL);

NSLog(@"1");
dispatch_async(queue2, ^() {
	NSLog(@"2");
    dispatch_sync(queue2, ^(){
      NSLog(@"3");
    });
    NSLog(@"4");
});

NSLog(@"5");
```

>输出：1，5，2

第一个例子使用 `DISPATCH_QUEUE_CONCURRENT` 并发队列，输出正常，而第二个例子中使用了 `DISPATCH_QUEUE_SERIAL` 串行队列，发生了死锁，后面的事件 (@"3") 和事件 (@"4")便无法执行。

我们首先分析一下第一个例子，为什么并没有发生死锁，首先我们往**并发队列** queue 中添加了dispatch_async 异步线程 ，主线程并不等待异步线程的执行，所以事件 (@"1") 后便马上执行事件 (@"5")，当内核线程空闲，加载并发队列 queue 中的 dispatch_async 异步线程 并执行线程中的事件(block) 的，事件 (@"2") 马上就会被执行。
当遇到了 dispatch_sync 同步线程的时候，当前线程，也就是 dispatch_async 这个异步线程会进入阻塞，等待 dispatch_sync 同步线程内的事件(block) 执行完，才可以往下执行事件(@"4")，我们并将dispatch_sync 同步线程放进了 queue 并发队列当中去，**并发队列的特点就是逻辑上是一起执行的，所以 dispatch_sync 同步线程加入 queue 后就马上被执行了，当事件(@"3")执行完后并且返回，阻塞放开，事件(@"4")并马上被执行。全过程并没有发生死锁**。

我们再来看看第二个例子，首先我们往**串行队列** queue 中添加了dispatch_async 异步线程 ，其后过程跟第一个例子一样，直到遇到了  `dispatch_sync(queue2,  block)` ，**dispatch_sync` 同步线程 阻塞了 dispatch_async 异步线程**，并将同步线程放进了 queue2 串行队列中，**串行队列的特别是遵循 FIFO 特点，要必先执行完 dispatch_async 异步线程的事件(block)，才能执行同步线程 dispatch_sync 的事件 (block)，所以造成了死锁**。

##### AFNetWorking 怎么使用同步线程

```objectivec
self.synchronizationQueue = dispatch_queue_create([name cStringUsingEncoding:NSASCIIStringEncoding], DISPATCH_QUEUE_SERIAL);

- (nullable AFImageDownloadReceipt *)downloadImageForURLRequest:(NSURLRequest *)request
                                                  withReceiptID:(nonnull NSUUID *)receiptID
                                                        success:(nullable void (^)(NSURLRequest *request, NSHTTPURLResponse  * _Nullable response, UIImage *responseObject))success
                                                        failure:(nullable void (^)(NSURLRequest *request, NSHTTPURLResponse * _Nullable response, NSError *error))failure {
                                                        
	dispatch_sync(self.synchronizationQueue, ^{
	        NSString *URLIdentifier = request.URL.absoluteString;
	        if (URLIdentifier == nil) {
	            if (failure) {
	                NSError *error;
	                dispatch_async(dispatch_get_main_queue(), ^{
	                    failure(request, nil, error);
	                });
	            }
	            return;
	        }
        
	        ...
	});
}
```

上面一段代码才子 AFNetWorking 中的 AFImageDownloader.m 文件当中，作者创建了 synchronizationQueue 串行队列专门用作阻塞当前线程，限制性同步队列中的事件，判断 url 是否为空，但是为什么要这样做呢？

原因1：
因为对象方法 `downloadImageForURLRequest:withReceiptID:success:failure` 是同一个对象在多个异步线程的并发队列当中执行的，因为并发在逻辑上会同时触发异步线程，那么传进来的参数（request，receiptID，success，failure）会由于**资源竞争(condition race)** 的情况下会被覆盖，所以我们需要进行阻塞这个线程，先执行完一个请求后再执行另外一个请求

但是会有人问：为什么么不用 `@synchronized (<#lock#>) {}` ?
因为我们首先不确定调用对象方法 `downloadImageForURLRequest:withReceiptID:success:failure` 是否必定在异步线程中被调用，莫名的加锁会消耗资源，当我们使用了 ` dispatch_sync(self.synchronizationQueue,block)` 后，如果主线程当中被调用，也只会忽视这个方法，直接调用 block，因为阻塞主线程，往并不是主线程队列的线程队列中添加时间，是没有意义的。

**使用 dispatch_sync(self.synchronizationQueue,block) 需要注意什么问题？**
其实上面这么写，是有问题的，当方法 `downloadImageForURLRequest:withReceiptID:success:failure` 的调用上层，也是`dispatch_sync(self.synchronizationQueue,block)` 的情况下，就会造成死锁，就像下面一样

```
dispatch_sync(self.synchronizationQueue, ^(){
	NSLog(@"2");
    dispatch_sync(self.synchronizationQueue, ^(){
      NSLog(@"3");
    });
    NSLog(@"4");
});
```

或
```
dispatch_async(self.synchronizationQueue, ^(){
	NSLog(@"2");
    dispatch_sync(self.synchronizationQueue, ^(){
      NSLog(@"3");
    });
    NSLog(@"4");
});
```

至于怎么分析，为什么会发生死锁，各位看官，这就留给你们的作业，看了这么多，相信大家也会明白，特别是第二个例子，我们刚讲过，希望大家能在这篇博客中学到东西。

----
写在最后：
> 为什么要写这篇文章呢？主要今天在某公司面试的时候，被问到了关于 GCD 的线程问题，在我说出来答案后，面试官依然坚持已见，认为我是错的，写这篇博客的目的在于，不管这个面试官是否会游览博客，也让更多的面试官可以好好更新自己的知识储备库，不要做井底之蛙。其实在我看来，面试是一个双向交流的过程，我并不在意是否能你们公司工作，毕竟我也不想同事是一群无法交流的人，一个开心愉快并且能够助我成长的工作环境才是我真正需要的。
