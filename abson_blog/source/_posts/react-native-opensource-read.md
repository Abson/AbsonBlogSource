---
title: React-Native 源码学习
date: 2019-01-30 15:12:46
tags:
---



### 其他笔记

#### OC 利用 RunLoop 实现线程队列调用事件
```C++
# RCTMessageThread.mm

RCTMessageThread::RCTMessageThread(NSRunLoop *runLoop, RCTJavaScriptCompleteBlock errorBlock)
  : m_cfRunLoop([runLoop getCFRunLoop])
  , m_errorBlock(errorBlock)
  , m_shutdown(false) {
  CFRetain(m_cfRunLoop);
}

RCTMessageThread::~RCTMessageThread() {
  CFRelease(m_cfRunLoop);
}

// This is analogous to dispatch_async
void RCTMessageThread::runAsync(std::function<void()> func) {
  CFRunLoopPerformBlock(m_cfRunLoop, kCFRunLoopCommonModes, ^{ func(); });
  CFRunLoopWakeUp(m_cfRunLoop);
}

// This is analogous to dispatch_sync
void RCTMessageThread::runSync(std::function<void()> func) {
  if (m_cfRunLoop == CFRunLoopGetCurrent()) {
    func();
    return;
  }

  dispatch_semaphore_t sema = dispatch_semaphore_create(0);
  runAsync([func=std::make_shared<std::function<void()>>(std::move(func)), &sema] {
    (*func)();
    dispatch_semaphore_signal(sema);
  });
  dispatch_semaphore_wait(sema, DISPATCH_TIME_FOREVER);
}
```

#### OC 判断现在的执行队列是否在主队列
```C++
BOOL RCTIsMainQueue()
{
  static void *mainQueueKey = &mainQueueKey;
  static dispatch_once_t onceToken;
  dispatch_once(&onceToken, ^{
    dispatch_queue_set_specific(dispatch_get_main_queue(),
                                mainQueueKey, mainQueueKey, NULL);
  });
  return dispatch_get_specific(mainQueueKey) == mainQueueKey;
}
```


