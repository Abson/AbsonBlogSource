---
title: 白话并发——死锁
date: 2018-04-13 10:28:48
tags: 
			- 计算机基础
			- 线程锁
			- 并发
categories: 
			- 计算机系统
			- 并发
---

> 文章主要带大家理解什么是死锁，死锁什么情况下会发生，还有解决死锁的方法。文章主要用 C++11 标准库中的 std::thread 来讲解。std::thread 底层上还是调用的POSIX的线程标准的 pthread。
> 文中由头到尾通过孩子玩耍玩具的实力去带你理解死锁。
<!-- more -->

##### waht is Deadlock(什么是死锁)？
>试想有一个玩具，这个玩具由两部分组成，必须拿到这两个部分，才能够玩。例如，一个玩具鼓，需要一个鼓锤和一个鼓才能玩。现在有两个小孩，他们都很喜欢玩这个玩具。当其中一个孩子拿到了鼓和鼓锤时，那就可以尽情的玩耍了。当另一孩子想要玩，他就得等待另一孩子玩完才行。再试想，鼓和鼓锤被放在不同的玩具箱里，并且两个孩子在同一时间里都想要去敲鼓。之后，他们就去玩具箱里面找这个鼓。其中一个找到了鼓，并且另外一个找到了鼓锤。现在问题就来了，除非其中一个孩子决定让另一个先玩，他可以把自己的那部分给另外一个孩子；但当他们都紧握着自己所有的部分而不给予，那么这个鼓谁都没法玩。 
>——出自 <<C++并发编程>> 一书

上面只是对线程死锁的一个概念描述，文中的孩子和玩具分别对应如下：
>孩子——>线程
>鼓和鼓锤——>两个互斥量

**当一对线程需要对他们拥有的互斥量做一些操作，其中每个线程都有一个互斥量，且等待另外一个解锁** 。这样互相等待的线程就像孩子们一样，谁都无法开心的敲响这个鼓，因为他们都在等待对方释放互斥量。这种情况就是死锁。

我们对上面的例子进行代码形象化！

##### 构建对象例子说明

首先是对玩具的定义：

```
class Tool {
public:
  Tool(const std::string& tool_nam) : tool_name_(tool_nam) {}

  friend void PlayDrum(Tool& lhs, Tool& rhs);
private:
  std::mutex mutex_;
  std::string tool_name_;
};

/*
 * 尝试玩耍鼓锤
 * */
void PlayDrum(Tool& lhs, Tool& rhs) {
  std::lock_guard<std::mutex> guard_a(lhs.mutex_); // 获取到了鼓或锤
  std::lock_guard<std::mutex> guard_b(rhs.mutex_); // 获取到了鼓或锤
  std::cout << "完成组成出玩具: " << tool_name << std::endl; // 成功组成玩具，终于可以玩耍了。
}
```

玩具类中，每个玩具都拥有一个自己的名称和玩具的实体，而 mutex_ 互斥量就是对应的鼓或锤了，就是玩具的实体，而 tool_name_ 为玩具的名称，这样孩子们才知道他们究竟拿到的是什么玩具，毕竟孩子还小。

下面我们就实例化出鼓和锤这两个玩具
```
Tool drum("鼓");
Tool hammer("锤");
```
嗯，看上去鼓和锤都有了，那么孩子又对应什么呢？

上面说过孩子对应着线程实例，那么我们就必须有两个孩子（childen）, 而且两个孩子都想玩鼓和锤。

```
// 非常非常地想玩鼓和锤
void WantToPlayDrum() {
  PlayDrum(drum, hammer);
}

void LeanDeadLock(){

  std::thread childen1(WantToPlayDrum);
  std::thread childen2(WantToPlayDrum);

  childen1.join();
  childen2.join();
};
```

上面例子实例化了两条线程（两个孩子），他们都只想去玩鼓锤（WantToPlayDrum）。
我们不妨再看看 PlayDrum 这个方法，

```
void PlayDrum(Tool& lhs, Tool& rhs) {
  std::lock_guard<std::mutex> guard_a(lhs.mutex_); // 获取到了鼓或锤
  std::lock_guard<std::mutex> guard_b(rhs.mutex_); // 获取到了鼓或锤
  std::cout << "完成组成出玩具: " << tool_name << std::endl; // 成功组成玩具，终于可以玩耍了。
}
```
上面整个例子这些写是没有问题的，因为他们都是非常有顺序的先拿鼓，再拿锤。不管是线程1（孩子1）还是线程2（孩子2）先拿到鼓，另外一个孩子必须等待拿到鼓的孩子先玩耍完，才能拿到鼓。

##### 如何形成死锁

但是这里出现了另外一个问题，当两个孩子想要获取的玩具顺序不一样的时候，又会怎么样 ？

```
void WantToPlayDrum1() {
  PlayDrum(drum, hammer); // 先拿到鼓再拿到锤
}

void WantToPlayDrum2() {
  PlayDrum(hammer, drum); // 先拿到锤再拿到鼓
}

void LeanDeadLock(){

  std::thread child1(WantToPlayDrum1);
  std::thread child2(WantToPlayDrum2);

  child1.join();
  child2.join();
};
```
上面例子中，线程1（孩子1）想先拿到 drum（鼓）再拿到 hammer（锤），而线程2（孩子2）则相反，那么上面整个例子又有什么问题呢？为什么会有死锁产生？

这个时候我们还是得看 PlayDrum 这个方法，我们假设线程1（孩子1）先进入了 PlayDrum 这个方法，并且成功的获取到了鼓（lhs.mutex_），但与此同时，线程2（孩子2）又进入了 PlayDrum 这个方法并且获取到了锤（lhs.mutex_）。当线程1（孩子1）想要去获取锤（rhs.mutex_）的时候，发现线程2（孩子2）已经握在手上了，没办法，只能等待线程2（孩子2）玩耍完不要这个锤的时候，线程1（孩子1）才能愉快的玩耍。而线程2（孩子2）的情况则相反，想要去获取鼓（rhs.mutex_）的时候，发现线程1（孩子1）已经握在手上了，并且怒气冲冲的喊他给他锤，没办法线程2（孩子2）只能也怒气冲冲的喊线程1（孩子1），给他鼓。


图解：
![Alt text](/images/what-is-deadlock/1523586690442.jpg)

孩子们都发现各自所需要的第二个玩具都给持有了，两个孩子唯有等待大家让出来了，当然，两个孩子都是倔强的孩子，到最后谁到拿不到鼓锤，大家最后都只能傻傻的站在原地。

##### 解决上述死锁的方法

这就是死锁的产生了，那么，这个时候就会有人站出来问，为什么孩子们这么笨，要一个玩具一个玩具的拿，他不是有两只手吗，不行还有两条腿，在不行就霸坑，直接抱着玩具箱找玩具，不让别人的孩子一起找。
是的，我们这就顺着这种思路去实现代码——霸坑！
我们重写孩子们想要玩玩具的过程：

```
void PlayDrum(Tool& lhs, Tool& rhs) {
  std::lock(lhs.mutex_, rhs.mutex_);
  std::lock_guard<std::mutex> guard_a(lhs.mutex_); // 获取到了鼓
  std::lock_guard<std::mutex> guard_b(rhs.mutex_); // 获取到了锤
  std::string tool_name = lhs.tool_name_ + rhs.tool_name_; // 成功组成玩具，终于可以玩耍了。
  std::cout << "完成组成出玩具: " << tool_name << std::endl;
}
```

细心的看客就会看到我们仅仅在玩耍玩具的代码中添加了 ` std::lock(lhs.mutex_, rhs.mutex_);` 这一句代码，那么这句代码的作用是什么呢？
实现固有顺序，当一个线程（孩子），想要玩耍鼓锤的时候，先把装着鼓锤的箱子拿走，然后再慢慢找鼓锤——std::lock，那么另外一个孩子也没办法，只能慢慢等他找到鼓锤并玩耍玩后才能尝试去获取鼓锤了。

可以看 std::lock 的内部实现：
>先获取一个锁，然后再调用std::try_lock去获取剩下的锁，如果失败了，则下次先获取上次失败的锁。
>重复上面的过程，直到成功获取到所有的锁。

##### 重复的玩具？

看官看到这里心想，这人终于要说完了，但是万万没想到上面例子还存在一个坑。
试想一下，当一个孩子想要玩耍两个锤或鼓的时候，会怎么样？
不妨来看一看实现代码：
```
void WantToPlayDrum() {
  PlayDrum(drum, drum);
}
```

当线程（孩子）要玩耍两个鼓的时候，我们就要调用 PlayDrum 方法了，通过 `std::lock_guard<std::mutex> guard_a(lhs.mutex_)` 获取了鼓，但是当调用 ` std::lock_guard<std::mutex> guard_b(rhs.mutex_);` 获取鼓的时候，发现鼓没有了（因为由始至终都只有一个鼓），所以线程（孩子）只能等待或者说是期待吧，手中的鼓能够神迹的复制一个出来，但是这当然是不可能发生的啦， 所以线程（孩子）只能默默的等待了。

怎么解决这个问题？
当然是不能让这种无中生有的事情发生啦！
优化 PlayDrum 函数：
```
void PlayDrum(Tool& lhs, Tool& rhs) {
  if (&lhs == &rhs) { // 1、防止两个相同的对象相互量相互等待，造成死锁
    return;
  }
  std::lock(lhs.mutex_, rhs.mutex_);
  std::lock_guard<std::mutex> guard_a(lhs.mutex_); // 获取到了鼓
  std::lock_guard<std::mutex> guard_b(rhs.mutex_); // 获取到了锤
  std::string tool_name = lhs.tool_name_ + rhs.tool_name_; // 成功组成玩具，终于可以玩耍了。
  std::cout << "完成组成出玩具: " << tool_name << std::endl;
}
```

这样我们就大功告成了，孩子们终于可以愉快“排队”的玩耍了，毕竟是法治，人人都得有素质嘛。



