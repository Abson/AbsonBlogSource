---
title: RAC 源码分析(一)
date: 2018-04-02 15:26:25
tags: 源码分析
---

> RecativeCoCoa objectivec 版本的分析，第一部分主要阐述 `rac_signalForSelector:` 该方法的实现，观摩一下 RAC 是如何监听方法回调的.

![](/images/rac-source-analyze/logo.png) 
<!-- more -->

1. `SEL aliasSelector = RACAliasForSelector(selector);` 这个方法得到了字符串拼接 `rac_alias_ + @selector` 后的方法 `aliasSelector` 用于后面替换原方法，实现方法监听。
2.  `RACSubject *subject` 为热信号，检测是否已经在监听改方法，如果有，把信号返回。
3.  [RACSwizzleClass](#####static Class RACSwizzleClass(NSObject *self)) 动态创建好这个类的 RAC 关联类，为 `class + _RACSelectorSignal` 并把这个映射类的符号添加到 OC 动态类符号当中，然后将 RAC 关联类中的方法转发 `forwardInvocation:` 、方法响应 `respondsToSelector:`
4.  创建热信号 `RACSubject`
5.  先查看一下对象 object 是否存在被监听的实例方法，如果不存在而查看被监听方法是否一个协议方法.
6. ` class_replaceMethod` 方法把参数 `selector` 方法替换成 OC消息转发方法`_objc_msgForward`，这样的话所有被监听的方法都会调用 `forwardInvocation:` 方法了
**结合NSObject文档可以知道，_objc_msgForward 消息转发做了如下几件事：**
>1.调用resolveInstanceMethod:方法，允许用户在此时为该Class动态添加实现。如果有实现了，则调用并返回。如果仍没实现，继续下面的动作。
>2.调用forwardingTargetForSelector:方法，尝试找到一个能响应该消息的对象。如果获取到，则直接转发给它。如果返回了nil，继续下面的动作。
>3.调用methodSignatureForSelector:方法，尝试获得一个方法签名。如果获取不到，则直接调用doesNotRecognizeSelector抛出异常。
>4.调用forwardInvocation:方法，将地3步获取到的方法签名包装成Invocation传入，如何处理就在这里面了。

![](/images/rac-source-analyze/2017-02-25-Message-Forwarding.png) 

### static Class RACSwizzleClass(NSObject *self) 
```objectivec
static Class RACSwizzleClass(NSObject *self) 
{
  // 1.
  Class statedClass = self.class;
  Class baseClass = object_getClass(self);

  Class knownDynamicSubclass = objc_getAssociatedObject(self, RACSubclassAssociationKey);
  if (knownDynamicSubclass != Nil) return knownDynamicSubclass;

  NSString *className = NSStringFromClass(baseClass);

  if (statedClass != baseClass) {

    @synchronized (swizzledClasses()) {
      if (![swizzledClasses() containsObject:className]) {
        RACSwizzleForwardInvocation(baseClass);
        RACSwizzleRespondsToSelector(baseClass);
        RACSwizzleGetClass(baseClass, statedClass);
        RACSwizzleGetClass(object_getClass(baseClass), statedClass);
        RACSwizzleMethodSignatureForSelector(baseClass);
        [swizzledClasses() addObject:className];
      }
    }

    return baseClass;
  }

  const char *subclassName = [className stringByAppendingString:RACSubclassSuffix].UTF8String;
  Class subclass = objc_getClass(subclassName);

  if (subclass == nil) {
    subclass = [RACObjCRuntime createClass:subclassName inheritingFromClass:baseClass];
    if (subclass == nil) return nil;

    RACSwizzleForwardInvocation(subclass);
    RACSwizzleRespondsToSelector(subclass);

    RACSwizzleGetClass(subclass, statedClass);
    RACSwizzleGetClass(object_getClass(subclass), statedClass);

    RACSwizzleMethodSignatureForSelector(subclass);

    objc_registerClassPair(subclass);
  }

  object_setClass(self, subclass);
  objc_setAssociatedObject(self, RACSubclassAssociationKey, subclass, OBJC_ASSOCIATION_ASSIGN);
  return subclass;
}
```
1. `object.class` 由于KVO重写了class 方法，所以不能准确的找到类.
`object_getClass()` 方法可以准确的找到 isa 指针.
`object.class` 与 `object_getClass(object)` 进行判断 来防止KVO导致的AOP无效.
为什么会发生这种情况？
因为当你使用 KVO 的时候，系统会帮你重写类的 class 方法，生成 `NSKVONotifying_ + Class`。例如如下代码
```
[self.view addObserver:self forKeyPath:@"frame" options:NSKeyValueObservingOptionNew context:nil];

- (void)observeValueForKeyPath:(nullable NSString *)keyPath ofObject:(nullable id)object change:(nullable NSDictionary<NSKeyValueChangeKey, id> *)change context:(nullable void *)context {
  Class statedClass = self.view.class;
  Class baseClass = object_getClass(self.view);

  NSLog(@"%@  %@", NSStringFromClass(statedClass), NSStringFromClass(baseClass));
}
```
得到的 log 如下：
`RACLearning[3657:478602] UIView  NSKVONotifying_UIView`
2. `swizzledClasses()` 专门用来存储 isa 被修改过的类，这些类不用重新生成为 RAC 的类.
3. 重新生成 RAC 的类 `Class + _RACSelectorSignal`，如果类符号中不存在，则利用运行时粗行家改类符号，并继承  Class。
4. [RACSwizzleForwardInvocation](#####  static void RACSwizzleForwardInvocation(Class class)) 利用运行时替换消息转发的方法 
`forwardInvocation:` ，由 [NSObjectRACSignalForSelector](##### static RACSignal *NSObjectRACSignalForSelector(NSObject *self, SEL selector, Protocol *protocol)) 代码中可以看到，所传入的方法参数 `selector` 都被换成了 `_objc_msgForward` 方法，所以，当方法 `selector` 调用是，自然就会调用 `forwardInvocation:` 方法了。

###  static void RACSwizzleForwardInvocation(Class class)
```objectivec
static void RACSwizzleForwardInvocation(Class class) {
  SEL forwardInvocationSEL = @selector(forwardInvocation:);
  Method forwardInvocationMethod = class_getInstanceMethod(class, forwardInvocationSEL);

  // Preserve any existing implementation of -forwardInvocation:.
  void (*originalForwardInvocation)(id, SEL, NSInvocation *) = NULL;
  if (forwardInvocationMethod != NULL) {
    originalForwardInvocation = (__typeof__(originalForwardInvocation))method_getImplementation(forwardInvocationMethod);
  }

  id newForwardInvocation = ^(id self, NSInvocation *invocation) {
    BOOL matched = RACForwardInvocation(self, invocation);
    if (matched) return;

    if (originalForwardInvocation == NULL) {
      [self doesNotRecognizeSelector:invocation.selector];
    } else {
      originalForwardInvocation(self, forwardInvocationSEL, invocation);
    }
  };

  class_replaceMethod(class, forwardInvocationSEL, imp_implementationWithBlock(newForwardInvocation), "v@:@");
}
```
1. 定义 IMP 函数指针指针 `originalForwardInvocation` ，指向  `class` 类的 `forwardInvocation` 内部函数实现，
2. 然后创建新的  `forwardInvocation:` block 函数 `newForwardInvocation`, block 函数内部先调用 [RACForwardInvocation](##### static BOOL RACForwardInvocation(id self, NSInvocation *invocation)) 函数，以调用映射过后的 `rac_alias_ + @selector`  函数(实际上是 `@selector`的复制体)，然后给热信号 `subject `发送信号 Next 以调用订阅 block `nexblock`.
3. 利用 `class_replaceMethod` 方法替换了 class 的对象方法 `forwardInvocation:` 为新的 `newForwardInvocation` block 函数.

### static BOOL RACForwardInvocation(id self, NSInvocation *invocation)
```objectivec 
static BOOL RACForwardInvocation(id self, NSInvocation *invocation) {
  SEL aliasSelector = RACAliasForSelector(invocation.selector);
  RACSubject *subject = objc_getAssociatedObject(self, aliasSelector);

  Class class = object_getClass(invocation.target);
  BOOL respondsToAlias = [class instancesRespondToSelector:aliasSelector];
  if (respondsToAlias) {
    invocation.selector = aliasSelector;
    [invocation invoke];
  }

  if (subject == nil) return respondsToAlias;

  [subject sendNext:invocation.rac_argumentsTuple];
  return YES;
}
```
由 [NSObjectRACSignalForSelector](##### static RACSignal *NSObjectRACSignalForSelector(NSObject *self, SEL selector, Protocol *protocol)) 中知道，热信号 subject 已经被创建了， 而且利用了 `class_addMethod ` 方法完成了被监听方法 `@selector` 的复制体 `rac_alias_ + @selector` 方法，所以这里直接利用  `[invocation invoke]`调用  `rac_alias_ + @selector` 方法，然后再像热信号 `subject` 发送 `next` 信号并且带上 `rac_alias_ + @selector` 方法的参数数组`rac_argumentsTuple`以调用 nextblock.

### - (RACTuple *)rac_argumentsTuple
```objectivec 
- (RACTuple *)rac_argumentsTuple {
  NSUInteger numberOfArguments = self.methodSignature.numberOfArguments;
  NSMutableArray *argumentsArray = [NSMutableArray arrayWithCapacity:numberOfArguments - 2];
  for (NSUInteger index = 2; index < numberOfArguments; index++) {
    [argumentsArray addObject:[self rac_argumentAtIndex:index] ?: RACTupleNil.tupleNil];
  }

  return [RACTuple tupleWithObjectsFromArray:argumentsArray];
}
```
1.  参数个数 `methodSignature.numberOfArguments` 默认有一个 _cmd 一个 target 所以要 -2
2.  获取该方法的参数 ，`rac_argumentAtIndex` 函数内通过 methodSignature 的  `getArgumentTypeAtIndex` 方法来判断获取 OC 对象或 基础类型(如 int，char，bool 等)转成的 NSNumber 对象。
3.  `?: ` 新写法，如果有则添加返回的 OC 对象，没有就把  `RACTupleNil.tupleNil` 单例对象添加进去（为什么添加单例对象？可以节省内存！）
4.  利用  `RACTuple` 对象吧参数数组包装一下返回。
i



