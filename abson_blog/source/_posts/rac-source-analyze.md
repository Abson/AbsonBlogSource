---
title: RAC 源码分析(一)
date: 2018-04-02 15:26:25
tags: 源码分析
categories: 
			- 源码分析
			- iOS
---

> RecativeCoCoa objectivec 版本的分析，第一部分主要阐述 `rac_signalForSelector:` 该方法的实现，观摩一下 RAC 是如何监听方法回调的.

![](/images/rac-source-analyze/logo.png) 
<!-- more -->
### rac_signalForSelector
当我们监听某个变量的变化，很自然我们就想到了 KVO 的方式， 但是我们有想过监听方法吗？在 RAC 当中，我们需要监听某个方法的被调用，就使用到了`rac_signalForSelector`这个函数。先看一下这个函数的实现方法。

```objectivec
// NSobject+RACSelectorSiganl.m

- (RACSignal *)rac_signalForSelector:(SEL)selector {
	NSCParameterAssert(selector != NULL);

	return NSObjectRACSignalForSelector(self, selector, NULL);
}
```
方法 `rac_signalForSelector` 直接调用了静态函数 `NSObjectRACSignalForSelector`!

### NSObjectRACSignalForSelector
```objectivec
// NSobject+RACSelectorSiganl.m

static RACSignal *NSObjectRACSignalForSelector(NSObject *self, SEL selector, Protocol *protocol) {
	/** 1.*/
	SEL aliasSelector = RACAliasForSelector(selector);

	@synchronized (self) {
		/** 2.*/
		RACSubject *subject = objc_getAssociatedObject(self, aliasSelector);
		if (subject != nil) return subject;
		/** 3.*/
		Class class = RACSwizzleClass(self);
		NSCAssert(class != nil, @"Could not swizzle class of %@", self);
		/** 4.*/
		subject = [[RACSubject subject] setNameWithFormat:@"%@ -rac_signalForSelector: %s", self.rac_description, sel_getName(selector)];
		objc_setAssociatedObject(self, aliasSelector, subject, OBJC_ASSOCIATION_RETAIN);

		[self.rac_deallocDisposable addDisposable:[RACDisposable disposableWithBlock:^{
			[subject sendCompleted];
		}]];
		/** 5.*/
		Method targetMethod = class_getInstanceMethod(class, selector);
		if (targetMethod == NULL) {
			const char *typeEncoding;
			if (protocol == NULL) {
				typeEncoding = RACSignatureForUndefinedSelector(selector);
			} else {
				// Look for the selector as an optional instance method.
				struct objc_method_description methodDescription = protocol_getMethodDescription(protocol, selector, NO, YES);

				if (methodDescription.name == NULL) {
					methodDescription = protocol_getMethodDescription(protocol, selector, YES, YES);
					NSCAssert(methodDescription.name != NULL, @"Selector %@ does not exist in <%s>", NSStringFromSelector(selector), protocol_getName(protocol));
				}

				typeEncoding = methodDescription.types;
			}

			RACCheckTypeEncoding(typeEncoding);
			/** 6.*/
			if (!class_addMethod(class, selector, _objc_msgForward, typeEncoding)) {
				NSDictionary *userInfo = @{
					NSLocalizedDescriptionKey: [NSString stringWithFormat:NSLocalizedString(@"A race condition occurred implementing %@ on class %@", nil), NSStringFromSelector(selector), class],
					NSLocalizedRecoverySuggestionErrorKey: NSLocalizedString(@"Invoke -rac_signalForSelector: again to override the implementation.", nil)
				};

				return [RACSignal error:[NSError errorWithDomain:RACSelectorSignalErrorDomain code:RACSelectorSignalErrorMethodSwizzlingRace userInfo:userInfo]];
			}
		} else if (method_getImplementation(targetMethod) != _objc_msgForward) {
			// Make a method alias for the existing method implementation.
			const char *typeEncoding = method_getTypeEncoding(targetMethod);

			RACCheckTypeEncoding(typeEncoding);
			/** 6.*/
			BOOL addedAlias __attribute__((unused)) = class_addMethod(class, aliasSelector, method_getImplementation(targetMethod), typeEncoding);
			NSCAssert(addedAlias, @"Original implementation for %@ is already copied to %@ on %@", NSStringFromSelector(selector), NSStringFromSelector(aliasSelector), class);

			// Redefine the selector to call -forwardInvocation:.
			/** 7.*/
			class_replaceMethod(class, selector, _objc_msgForward, method_getTypeEncoding(targetMethod));
		}

		return subject;
	}
}
```
总体上方法做了以下这些步骤：

1. `SEL aliasSelector = RACAliasForSelector(selector);` 这个方法得到了字符串拼接 `rac_alias_ + @selector` 后的方法 `aliasSelector` 用于后面替换原方法，`aliasSelector`实际上是被监听方法`selector`的复制体，因为下面步骤会将监听方法`selector`替换，所以这里首先要保存一下。
2.  `RACSubject *subject` 为热信号，检测是否已经在监听改方法，如果有，把信号返回。(关于冷热信号的概念可以查看美团的[细说ReactiveCocoa的冷信号与热信号（三）：怎么处理冷信号与热信号](https://tech.meituan.com/talk-about-reactivecocoas-cold-signal-and-hot-signal-part-3.html))
3.  [RACSwizzleClass](#RACSwizzleClass构建映射类) 动态创建好`NSObject* Self`对象的 RAC 关联类, 这个关联类是继承自`NSObject* Self`的`isa`的，类名为 `class + _RACSelectorSignal`，并把这个映射类的符号添加到 OC 动态类符号当中，然后让对象`NSObject* Self`的`isa`指向这个映射类, 然后将 RAC 关联类中的方法转发 `forwardInvocation:`，完成方法监听.
4.  创建热信号 `RACSubject`, 并跟 `aliasSelector` 方法设置为映射关系，方便我们后面直接通过映射获取`RACSubject`来发送信号
5.  先查看一下对象 object 是否存在被监听的实例方法，如果不存在而查看被监听方法为协议方法
6. 通过 `class_addMethod` 为关联类增添监听方法`selector`的复制体`aliasSelector`, 这样才能方便后面能后调用监听方法`selector`
7. ` class_replaceMethod` 方法把参数 `selector` 方法替换成 OC消息转发方法`_objc_msgForward`，由于`selector` 方法被替换了，掉用的时候自然都会调用 `forwardInvocation:` 方法了，但是`selector` 的实现也消失了，不过我们之前已将创建了复制体——`aliasSelector`，难道不是吗？
**结合NSObject文档可以知道，`_objc_msgForward` 消息转发做了如下几件事：**
>1.调用resolveInstanceMethod:方法，允许用户在此时为该Class动态添加实现。如果有实现了，则调用并返回。如果仍没实现，继续下面的动作。
>2.调用forwardingTargetForSelector:方法，尝试找到一个能响应该消息的对象。如果获取到，则直接转发给它。如果返回了nil，继续下面的动作。
>3.调用methodSignatureForSelector:方法，尝试获得一个方法签名。如果获取不到，则直接调用doesNotRecognizeSelector抛出异常。
>4.调用forwardInvocation:方法，将地3步获取到的方法签名包装成Invocation传入，如何处理就在这里面了。

![](/images/rac-source-analyze/2017-02-25-Message-Forwarding.png) 
通过上面这幅图，可以看出，如果一个 OC 方法在类符号中查找不到的时候，就进行了 `_objc_msgForward`，而 `_objc_msgForward` 到最后都会调用到 `-forwardInvocation`这个 OC 方法，那么RAC的意图就显然易见了，它需要做的，就是利用方法调用时，将所有被监听的方法都运行到`-forwardInvocation`，通过改造`-forwardInvocation`方法，利用 RAC 的信号，通知外层监听实现方法。

### RACSwizzleClass构建映射类
```objectivec
static Class RACSwizzleClass(NSObject *self) 
{
  /** 1.*/
  Class statedClass = self.class;
  Class baseClass = object_getClass(self);

  Class knownDynamicSubclass = objc_getAssociatedObject(self, RACSubclassAssociationKey);
  if (knownDynamicSubclass != Nil) return knownDynamicSubclass;

  NSString *className = NSStringFromClass(baseClass);

  if (statedClass != baseClass) {
	/** 2.*/
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
  
  /** 3.*/
  const char *subclassName = [className stringByAppendingString:RACSubclassSuffix].UTF8String;
  Class subclass = objc_getClass(subclassName);

  if (subclass == nil) {
  	 /** 4.*/
    subclass = [RACObjCRuntime createClass:subclassName inheritingFromClass:baseClass];
    if (subclass == nil) return nil;
	 /** 5.*/
    RACSwizzleForwardInvocation(subclass);
    /** 6.*/
    RACSwizzleRespondsToSelector(subclass);
	 /** 7.*/
    RACSwizzleGetClass(subclass, statedClass);
    RACSwizzleGetClass(object_getClass(subclass), statedClass);
	 /** 8.*/
    RACSwizzleMethodSignatureForSelector(subclass);

    objc_registerClassPair(subclass);
  }
  /** 9.*/
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
```objectivec
[self.view addObserver:self forKeyPath:@"frame" options:NSKeyValueObservingOptionNew context:nil];

- (void)observeValueForKeyPath:(nullable NSString *)keyPath ofObject:(nullable id)object change:(nullable NSDictionary<NSKeyValueChangeKey, id> *)change context:(nullable void *)context {
  Class statedClass = self.view.class;
  Class baseClass = object_getClass(self.view);

  NSLog(@"%@  %@", NSStringFromClass(statedClass), NSStringFromClass(baseClass));
}
```
得到的 log：
> RACLearning[3657:478602] UIView  NSKVONotifying_UIView

2. `swizzledClasses()` 专门用来存储 isa 被修改过的类，这些类不用重新生成为 RAC 的类.
3. 检查OC类符号表中是否存在经过RAC改造的类 —— `Class + _RACSelectorSignal`。
4. 不存在改造类的情况下，利用运行时方法`objc_allocateClassPair`创建继承Class类的类符号——`Class + _RACSelectorSignal`。
5. [RACSwizzleForwardInvocation](#RACSwizzleForwardInvocation) 利用运行时替换方式`class_replaceMethod`将经过改造的类`Class + _RACSelectorSignal`的消息转发的方法`forwardInvocation:` 的实现替换成 RAC 的新实现.
6. 由于我们在方法[NSObjectRACSignalForSelector](#NSObjectRACSignalForSelector)将被监听方法`selector`替换成了`_objc_msgForward`函数了，所以当我们使用外层API调用`respondsToSelector`去判断`selector`是否有实现的时候，很明显会返回false, 而[RACSwizzleRespondsToSelector](#RACSwizzleRespondsToSelector) 将经过改造的类`Class + _RACSelectorSignal`的`respondsToSelector:`方法实现转而判断`aliasSelector`是否实现的前提了，前文也说过`aliasSelector`实际上是被监听方法`selector`的复制体。
7. [RACSwizzleGetClass](#RACSwizzleGetClass)让 RAC 让新创建的类`Class + _RACSelectorSignal`的 `class`方法的实现全部返回对象`NSObject* self`的类，也就是类`Class + _RACSelectorSignal`变成了一个真正的伪装类，为了让对象对自己的身份『说谎』，所有的方法都转发到了这个子类上，如果不修改 class 方法，那么当开发者使用它自省时就会得到错误的类，而这是我们不希望看到的
8. 通过上面图中可以知道，必须为`forwardInvocation:`提供一个方法签名，才能走运行时转发，所以[RACSwizzleMethodSignatureForSelector](#RACSwizzleMethodSignatureForSelector)自然是为类`Class + _RACSelectorSignal`创建方法签名了.
9. 将对象`NSObject *self`的isa强制指向经过RAC改造的类 —— `Class + _RACSelectorSignal`了，这些下来左右调用该对象的所有方法都会访问`Class + _RACSelectorSignal`类结构体中的`MethodList`.

###  RACSwizzleForwardInvocation
```cpp
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
2. 然后创建新的  `forwardInvocation:` block 函数 `newForwardInvocation`, block 函数内部先调用 [RACForwardInvocation](#RACForwardInvocation) 函数，以调用映射过后的 `rac_alias_ + @selector`  函数(实际上是 `@selector`的复制体)，然后给热信号 `subject `发送信号 Next 以调用订阅 block `nexblock`.
3. 利用 `class_replaceMethod` 方法替换了 class 的对象方法 `forwardInvocation:` 为新的 `newForwardInvocation` block 函数.

### RACForwardInvocation
```cpp 
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
由 [NSObjectRACSignalForSelector](#NSObjectRACSignalForSelector) 中知道，热信号 subject 已经被创建了， 而且利用了 `class_addMethod ` 方法完成了被监听方法 `@selector` 的复制体 `rac_alias_ + @selector` 方法，所以这里直接利用  `[invocation invoke]`调用  `rac_alias_ + @selector` 方法，然后再像热信号 `subject` 发送 `next` 信号并且带上 `rac_alias_ + @selector` 方法的参数数组`rac_argumentsTuple`以调用 nextblock.

### rac_argumentsTuple
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

### RACSwizzleRespondsToSelector
```cpp
static void RACSwizzleRespondsToSelector(Class class) {
	SEL respondsToSelectorSEL = @selector(respondsToSelector:);
	
	Method respondsToSelectorMethod = class_getInstanceMethod(class, respondsToSelectorSEL);
	BOOL (*originalRespondsToSelector)(id, SEL, SEL) = (__typeof__(originalRespondsToSelector))method_getImplementation(respondsToSelectorMethod);

	id newRespondsToSelector = ^ BOOL (id self, SEL selector) {
		Method method = rac_getImmediateInstanceMethod(class, selector);

		if (method != NULL && method_getImplementation(method) == _objc_msgForward) {
			SEL aliasSelector = RACAliasForSelector(selector);
			if (objc_getAssociatedObject(self, aliasSelector) != nil) return YES;
		}

		return originalRespondsToSelector(self, respondsToSelectorSEL, selector);
	};

	class_replaceMethod(class, respondsToSelectorSEL, imp_implementationWithBlock(newRespondsToSelector), method_getTypeEncoding(respondsToSelectorMethod));
}
```
更换改造类`class + _RACSelectorSignal`的`respondsToSelector`方法，让`aliasSelector`方法成为真正的被监听方法`selector`的复制体


实际上，整个调用过程就是如下图所示:
![](/images/rac-source-analyze/1523456808092.jpg)


