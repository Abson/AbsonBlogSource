---
title: 面向对象：设计中的思考
date: 2018-08-14 21:32:14
tags:
			- 设计模式
---

设计模式说白了，就是接口编程与组合对象所延伸出来得变异。d

## 如何设计接口？

### 学会针对接口编程，而不是针对实现编程
**设计接口需传入某类型变量的时候，不将变量声明为某个特定得具体类的实例对象，而是让它遵从抽象类所定义的接口。**
如下:
比如我们现在需要设计一个 `doSomething` 接口，需要接受一个变量，去为我们做一些不确定的操作。
```c++
// 设计
class ConcreteClass {
};

class InterfaceClass {
public:	
  static void doSomething(ConcreteClass*);
};

// 使用
ConcreteClass instance = new ConcreteClass;
InterfaceClass::doSomething(instance);
```
以上设计具有局限性，首先，对于 `doSomething` 我们并不知道内部如何实现，因为面向对象的思想就是封装，意思就是具有封闭性，当我们传入具体对象 `ConcreteClass` 的时候就绑定了该接口，以后只能在 `ConcreteClass` 进行拓展开发，我们无法传入其他类对象实例。也就是说，我们只能使针对 `ConcreteClass` 的实现进行编程的。

那么，针对接口编程又是什么意思？
```c++
class AbstractClass {
public:
  virtual void abstractFeature() = 0;
};

class ConcreteClass : public AbstractClass {
};

class InterfaceClass {
public:
  static void doSomething(AbstractClass*);
}

// 使用
ConcreteClass instance = new ConcreteClass;
InterfaceClass::doSomething(instance);
```
以上这段代码，我们改动了小许，通过一个抽象基类，也就是一个接口类来为我们提供接口，我们设计的 `doSomething` 接口所做的操作，就是依赖于这个接口类中的接口去操作。
这样写，虽然有了小许面向接口编程的概念，但是从中我们还是依赖于必须实例化 `ConcreteClass` 对象，当客户操作时，必须投入精力去理解 `ConcreteClass` 的"真正含义"，还会存在由于用户操作不当破坏 `instance` 的结构性的可能。

如何更面向接口编程？
```c++
class AbstractClass {
public:
  virtual void abstractFeature() = 0;
};

class ConcreteClass : public AbstractClass {
};

class Factory {
public:
  static AbstractClass createInstance(int type);
};

class InterfaceClass {
public:
  static void doSomething(AbstractClass*);
}

// 使用
AbstractClass instance = Factory::createInstance(1);
InterfaceClass::doSomething(instance);
```
以上代码通过加入创建型模式中的工厂模式来为用户提供创建 `doSomething` 接口所需对象的接口，用户只需要通过具体类型 `type` 来控制所需要的事件类型，便可以调用接口。留给客户操作得越少，保证出错得几率就约小。

那么定义接口有什么法则？

1. 客户无须知道它们使用对象的特定类型，只须对象有客户所期望的接口。
2. 客户无须知道他们使用的对象使用什么类来实现，他们只须知道定义接口的抽象类。

好处：
极大地减少子系统实现之间的相互依赖关系，也产生了可复用的面向对象设计原则

> 针对接口编程，而不是针对实现编程

## 如何选择设计模式？
### 考虑设计模式是怎么解决设计问题的
### 浏览模式的意图部分


### 研究模式怎样相互关联
设计模式之间的关系：

![IMG_0319](/images/IMG_0319.jpg)


### 研究目的相似的模式


### 检查重新设计的原因



