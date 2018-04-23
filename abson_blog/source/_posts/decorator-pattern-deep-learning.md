---
title: 设计模式系列——装饰者模式(DecoratorPattern)
date: 2018-04-19 19:06:22
tags:
			- 设计模式
			- 存储结构
			- C/C++
categories: 
			- 设计模式
---

所谓的设计模式，其实是对面向对象编程思想中的一个转变，是在繁重需求任务中做到可扩展，高度灵活，并且适应业务开发而产生的一种思想。
今天我们说的修饰者模式，是一种动态地往一个类中添加新的行为的设计模式。就功能而言，修饰模式相比生成子类更为灵活，这样可以给某个对象而不是整个类添加一些功能。
<!--more-->

当有几个相互独立的功能需要扩充时，这个区别就变得很重要。在有些面向对象的编程语言中，类不能在运行时被创建，通常在设计的时候也不能预测到有哪几种功能组合。这就意味着要为每一种组合都得创建一个新类。

好吧，现在把我当做一个小偷，我需要对大师的画进行临摹，我临摹的画有很多类型的 可能是油画，可能是水墨画，也可能是沙画等等。我临摹完了这些画之后，需要对画进行装饰一下才能卖个好价钱，比如对画添加一个画框，这个画框可能是木框，可能是钻石框(好吧，我承认我比较奢侈)。但是添加完了画框之后，我可能又想为其添加一些签名，例如齐白石老先生的签名，例如梵高的签名，这样才能卖个好价钱呀！反正我会想尽一切办法去添加画本身内容外的装饰。那么我们传统手法怎么做呢？
我们学过OOP思想——类的继承的话，就会想着不断继承下去，例如我的油画：`class OilPicture`，然后添加了木框：`class WoodFrameOilPicture : public OilPicture`, 然后添加了齐白石老先生的签名:`class BaiShiWoodFrameOilPicture : public WoodFrameOilPicture`。想一想，不同的组合竟然会产生各种不同的子类，这种不断继承来添加新特性的继承地狱是在是太愚蠢了。
现在我们可不可以想出一个新方法，让我的油画可以自由组合，又可以脱离这种继承地狱的方法呢？这就是我们今天要说的主题——修饰者模式。
**修饰模式是类继承的另外一种选择。类继承在编译时候增加行为，而装饰模式是在运行时增加行为。**

首先我们定义图画类：
```cpp
class Picture {
public:
    Picture() {}
    virtual ~Picture() {};
    
    virtual int32_t getWorth() = 0; // 
    virtual std::string Description() = 0;
    virtual bool Contain(const std::type_info&) = 0;
private:
    Picture(const Picture&) = delete;
    void operator=(const Picture&) = delete;
};
```
这个图画类我们定义了四个接口：
`int32_t getWorth()`: 通过添加不同的装饰品后获取图画的价值.
`std::string Description()`: 这幅图画的描述文字.
`bool Contain(const std::type_info&)`: 用于判断图画是否添加了某种装饰品.

我们临摹的图画主要有两种，一种是**油画**，一种是**水墨画**
```cpp
class OilPicture : public Picture {
public:
    ~OilPicture() {}
    
    int32_t getWorth() override {
        return 500; 
    };
    
    std::string Description() override {
        return "一副好看油画!";
    }
    
    bool Contain(const std::type_info& info) override {
        return typeid(OilPicture).hash_code() == info.hash_code();
    }
};
```
```cpp
class InkPicture : public Picture {
public:
    ~InkPicture() {}
    
    int32_t getWorth() override {
        return 500; 
    }
    
    std::string Description() override {
        return "一副好看的水墨画!";
    }
   
    bool Contain(const std::type_info& info) override {
        return typeid(InkPicture).hash_code() == info.hash_code();
    }
};
```
我们首先模仿的油画和水墨画都定价为500块钱，然后添加他们的描述。好了现在这样我们也可以去卖了，但是这个价值不高，我们需要添加一点装饰品，让画看上去高大上一点。

这里我们可能需要为画添加一个签名，这个签名统一命名为签名装饰品，它是图画的一部分
```cpp
class AuthorPictureDecorator : public Picture {
public:
    AuthorPictureDecorator(Picture* p, std::string author) : picture_(p), author_(author) {}
    
    virtual ~AuthorPictureDecorator() {
        delete picture_;
    }
    
    std::string Description() override {
        return picture_->Description() + PetPhrase();
    }
    
    int32_t getWorth() override {
        return picture_->getWorth() + AuthorWorth();
    };
    
    virtual int32_t AuthorWorth() = 0;
    virtual std::string PetPhrase() = 0;
protected:
    Picture* picture_;
    std::string author_; // 作者的名字
};
```
`std::string PetPhrase()`: 作者的口头禅
`Picture* picture_`: 这个变量用于记住我们要修饰的图画对象.
`int32_t getWorth() `: 图画的价值加上签名的价值，就是我们的图画真实的价值了.
`int32_t AuthorWorth()` 作者签名的价值

好了，但是我只会模仿**齐白石**老先生跟**梵高**的签名：
```cpp
class AuthorQiBaishi : public AuthorPictureDecorator {
public:
    AuthorQiBaishi(Picture* p) : AuthorPictureDecorator(p, "QiBaiShi") {
    }
    
    int32_t AuthorWorth() override { 
        if (Contain(typeid(InkPicture))) { // 齐白石老先生只会画水墨画
            return 1000;
        }
        else {
            return -500; // 假的画
        }
    }
    
    bool Contain(const std::type_info& info) override {

        if (typeid(AuthorQiBaishi).hash_code() != info.hash_code()) {
            return picture_->Contain(info);
        }
        return true;
    }

    std::string PetPhrase() override {
        return "我是" + author_ + "皮皮虾我们走！";
    }
};
```
```cpp
class AuthorFanGao : public AuthorPictureDecorator {
public:
    AuthorFanGao(Picture* p) : AuthorPictureDecorator(p, "FanGao") {
    }
    
    int32_t AuthorWorth() override {
        if (Contain(typeid(OilPicture))) { // 梵高大石只会画油画呀
            return 2000;
        }
        else {
            return -500; // 假的画
        }
    }
    
    bool Contain(const std::type_info& info) override {
        
        if (typeid(AuthorFanGao).hash_code() != info.hash_code()) {
            return picture_->Contain(info);
        }
        return true;
    }
    
    std::string PetPhrase() override {
        return "I am " + author_ + " 不要跟我谈钱，我就是穷!";
    }
};
```

恩，很好，添加了签名貌似更值钱了，不过添加一个画框那就更完美了，顺便挣一波画框的钱，哈哈哈，我真是生意天才!
这里我们可能需要为画添加一个画框，这个画框统一命名为**画框装饰品**，它是图画的一部分：
```cpp
class PictureFrameDecorator : public Picture {
public:
    PictureFrameDecorator(Picture* p) : picture_(p) {}
    virtual ~PictureFrameDecorator() {
        delete picture_;
    }
    
    int32_t getWorth() override {
        return picture_->getWorth() + FrameWorth();
    }
    
    std::string Description() override {
        return picture_->Description() + Features();
    } 
    
    virtual int32_t FrameWorth() = 0;
    virtual std::string Features() const = 0;
protected:
    Picture* picture_;
};
```
`std::string Features()`: 画框的描述
`Picture* picture_`: 这个变量用于记住我们要修饰的图画对象.
`int32_t getWorth() `: 图画目前的价值加上画框的价值，就是我们的图画真实的价值了.
`int32_t FrameWorth()` 画框的真实的价值


但是我这边材料只有两种，一种是**木头**，一种是**钻石**，额确实有点奢侈，有什么办法呢~：
```cpp
class DiamondFrame : public PictureFrameDecorator {
public:
    DiamondFrame(Picture* p) : PictureFrameDecorator(p) {}
    ~DiamondFrame() {}
    
    int32_t FrameWorth() override {
        return 30000; // 钻石框定价三万块钱，恩，这肯定能好好挣一波
    }
    
    bool Contain(const std::type_info& info) override {
        
        if (typeid(DiamondFrame).hash_code() != info.hash_code()) {
            return picture_->Contain(info);
        }
        return true;
    }
    
    std::string Features() const override {
        return "闪闪发光!";
    }
};
```
```cpp
class WoodFrame : public PictureFrameDecorator {
public:
    WoodFrame(Picture* p) : PictureFrameDecorator(p) {}
    ~WoodFrame() {}
    
    int32_t FrameWorth() override {
        return 5000; // 木头我也要定价为 5000 块钱，我不管！
    }
    
    std::string Features() const override {
        return "久远大自然的味道!";
    }
    
    bool Contain(const std::type_info& info) override {
        if (typeid(WoodFrame).hash_code() != info.hash_code()) {
            return picture_->Contain(info);
        }
        return true;
    }
};
```
好了，先定这两个装饰品吧，后面根据用户的要求，再去添加其他装饰品吧。

先看看客户A的要求："你好，我要一副**油画**，有**梵高**大师的签名的"
我：好的，没问题老板，嘻嘻嘻！
```cpp
Picture* oil_pic = new OilPicture();
oil_pic = new AuthorFanGao(oil_pic);
```
我：好了，老板，你要的作品~`oil_pic`，价值为2500块钱
客户A：嗷，看上去加个图框貌似更好，我要个钻石框吧.
我：好的，老板，老板大气！
```cpp
Picture* oil_pic = new OilPicture();
oil_pic = new AuthorFanGao(oil_pic);
oil_pic = new DiamondFrame(oil_pic);
```
我：好了，老板，你要的作品~`oil_pic`，32500块钱
客户A:嗷，完美~

先看看客户B的要求："你好，我要一副**水墨画**，有**梵高**大师和**齐白石**老先生的签名的"
我：老板，你的口味比较独特呀，不过没问题~
```cpp
Picture* ink_pic = new InkPicture();
ink_pic = new AuthorFanGao(ink_pic);
ink_pic = new AuthorFanGao(ink_pic);
```
我：好了，老板，你要的作品~`ink_pic`，价值为1000块钱，唉，往水墨画上加梵高大石的名字这种品位必须给你打折~
客户B：不错不错~

通过这样的装饰，我们可以随意添加各种各样并且不同种类的装饰品与此同时又不会影响到**油画**类和**水墨画**类本身的特性。

优点：通过使用修饰模式，可以在运行时扩充一个类的功能，例如签名功能，画框功能。原理是：增加一个修饰类包裹原来的类，包裹的方式一般是通过在将原来的对象作为修饰类的构造函数的参数。**装饰类实现新的功能，但是，在不需要用到新功能的地方，它可以直接调用原来的类中的方法。修饰类必须和原来的类有相同的接口。**

具体结构如下图：
![](/images/decorator-pattern-deep-learning/123.jpg)


* **Component**:抽象类的接口，例如我们的**图画类**`Picture`
* **ConcreteComponent**:是 Component 的子类，具体的需要使用类，实现了相应的方法，它充当了“被装饰者”的角色。例如我们的**油画类**和**水墨画类**
* **Decorator**：也是 Component 的子类，抽象装饰者类的接口，内部有一个 Component 对象被持有用于被装饰，例如我们的**画框装饰品类**
* **ConcreteDecorator**：是Decorator的子类，是具体的装饰者。由于它同时也是Component的子类，因此它能方便地拓展Component的状态（比如添加新的方法）。每个装饰者都应该有一个实例变量用以保存某个Component的引用，这也是利用了组合的特性。在持有Component的引用后，由于其自身也是Component的子类，那么，相当于ConcreteDecorator包裹了Component，不但有Component的特性，同时自身也可以有别的特性，也就是所谓的装饰。相当于我们的**木头类**和**钻石类**。

