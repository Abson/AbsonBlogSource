---
title: 静态库中Class的分类问题和符号冲突问题 (Xcode other Link Flags)
date: 2018-04-13 14:55:57
tags: 
			- 计算机基础
			- iOS
			- xcode
categories: 
			- 计算机系统
			- 编译链接
---

>other linker flags 是 xcode 这个集成开发环境所特有的，目的是让连接器器 ld 除了默认参数外再根添加额外参数进行链接工作。

Object-C 链接特性:
>The "selector not recognized" runtime exception occurs due to an issue between the implementation of standard UNIX static libraries, the linker and the dynamic nature of Objective-C. Objective-C does not define linker symbols for each function (or method, in Objective-C) - instead, linker symbols are only generated for each class. If you extend a pre-existing class with categories, the linker does not know to associate the object code of the core class implementation and the category implementation. This prevents objects created in the resulting application from responding to a selector that is defined in the category.

Object-C的链接器并不会为每个方法建立符号表，而是为每个类建立链接符号。这样的话静态库中定义了已存在的类的分类，链接器就以为这个类存在了，不会将分类和核心类代码关联（合并）起来，这样在最后可执行文件中，就会找不到分类里所定义的方法。

例如如下错误：

![Alt text](/images/xcode-link-symbol-conflict/1523602861218.jpg)

就看 log 可以看出，是 NSString 的一个分类方法 `designByOhterLinker` 找不到实现了，而这个方法，确实是一个静态库里面的一个分类方法。

![Alt text](/images/xcode-link-symbol-conflict/1523602923047.jpg)

---

**如何解决这个问题？**

三个Linker 参数：
* -ObjC
* -all_load
* -force_load
* -dead_strip (8.27日更新)

#####-ObjC ：
>This flag causes the linker to load every object file in the library that defines an Objective-C class or category. While this option will typically result in a larger executable (due to additional object code loaded into the application), it will allow the successful creation of effective Objective-C static libraries that contain categories on existing classes.

加入这个参数后，链接器会将静态库中的每个类和分类加载到最后的可执行文件，当然，这个参数会导致可执行文件比较大，原因是加载了更多的额外对象的代码到可执行文件当中去，**但是这会解决 Objec-C 中静态库中找不到分类方法的问题。**

上面说得很清楚，-ObjC 会解决静态库中已存在的类的分类问题，那么，如果分类存在与静态库，**但是类并不在静态库的这种情况，该怎么办呢？**

>Important: For 64-bit and iPhone OS applications, there is a linker bug that prevents -ObjC from loading objects files from static libraries that contain only categories and no classes. The workaround is to use the -allload or -forceload flags.

说得很清楚，使用-all_load 或 -force_load 就可以解决上述问题。

#####-all_load：
该参数把所找到的目标文件都加载到可执行文件当中去，但是这就存在一个问题了，如果两个静态库中，都使用了同一份目标文件（这是一个很常见的问题，例如大家的目标文件都使用了用以名字 base64.o）就会发生 ld: duplicate symbol 符号冲突问题，所以不太建议使用。

#####-force_load：
该参数的作用跟 -all_load 其实是一样的，但是 -force_load 需要指定要进行全部加载的库文件的路径，这样的话，只要完全加载一个库文件，不影响其余库的可重定位目标文件的按需加载。

但是也有一种最头痛，就是当两个静态库中使用了相同的目标文件

![Alt text](/images/xcode-link-symbol-conflict/1523602956363.jpg)

上图的两个上图的两个 libMyOtherStaticLibrary.a 和 libMyStaticLibrary 中的 MyClass.o 类发生了冲突
那么，这个时候有两种解决方法：

>1、利用 -force_load 让链接器指定编译把其中一个静态库的目标文件，不加载另一个静态库的重复目标文件

具体做法：

![Alt text](/images/xcode-link-symbol-conflict/1523602988319.jpg)

但是这么做有一个弊端，如果这两个静态库同时都使用到了分类（基本上都会使用吧）那么如果只让编译器加载其中一个静态库的目标文件 （-force_load），而不将另一个静态库中的分类合并加载到目标文件的话，也是会导致运行的时候导致上述的崩溃问题。但是如果 -foce_load 两个静态库，又会有符号冲突，那么，怎么办呢？

----
>2、简单来说就是去除某个静态库中的重复目标文件，然后再打包

具体做法：
1）通过使用压缩工具命令 ar -t 去查看两个静态库文件里的目标文件那些存在冲突
如下：
![Alt text](/images/xcode-link-symbol-conflict/1523603024403.jpg)

![Alt text](/images/xcode-link-symbol-conflict/1523603052436.jpg)

很明显就是 MyClass.o 这个目标文件发生符号冲突了， 其实不这样看也行，反正编译的时候 Clang 编译器就就会有符号冲突的报错，上图 Xcode 报错的那个图就是很好的例子，可以看错那些目标文件重复了。

2）将其中一个静态库中的重复目标文件去掉，然后再次打包成静态库使用

1.  首先利用 lipo 命令将其中一个iOS静态库的文件解压出来（因为iOS的静态库文件是一个将不同 CPU 架构静态库合并的一个打包文件）。
![Alt text](/images/xcode-link-symbol-conflict/1523603084163.jpg)

可以看出 libMyOtherStaticLibrary.a 中包含了 armv7 跟 arm64 两种架构的静态库文件

2. 分别将两种不同架构的静态库文件提取出来

![Alt text](/images/xcode-link-symbol-conflict/1523603131501.jpg)

![Alt text](/images/xcode-link-symbol-conflict/1523603160464.jpg)

3.  使用 ar 压缩工具分别将这两个不同架构的静态库文件与另一个发生冲突的静态库文件中的目标文件剔除出去。

![Alt text](/images/xcode-link-symbol-conflict/1523603185423.jpg)

通过上面命令看出已成功将 MyClass.o 剔除出静态库

4. 利用 lipo 将两个不同架构的静态库重新打包封装成 iOS 的静态库文件

![Alt text](/images/xcode-link-symbol-conflict/1523603205737.jpg)

![Alt text](/images/xcode-link-symbol-conflict/1523603228087.jpg)

然后在 libMyOtherStaticLibraryOut.a 这个静态库重新放到工程当中去替换原来的 libMyOtherStaticLibrary.a

Other linker flags 只需用 -ObjC 就可以了

编译，Successful!
运行，完美！

#####-dead_strip （2017.8.27 更新）
参数的作用在于解决我们上面可重定位目标文件（.o）中类符号的冲突问题，如果发生了这种情况，**使用该参数就是一个非常快捷的办法了，让 Clang 编译器帮助我们去除重复符号的可重定位目标文件问题。**
但是使用这个参数却有一个问题，就是如果我们使用了改参数，就不能使用 -all_load 或 -force_load，认真想想也知道，如果我们指定了让编译器帮我们决定哪些目标文件该被链接，哪些不被链接（-dead_strip），那么我们就不能手动的强制地让所有目标文件都进行链接了（-all_load 或  -force_load）。如果是这样的话，我们又回到最初的问题了，-ObjC 会解决静态库中已存在的类的分类问题，那么，如果分类存在与静态库，但是类并不在静态库的这种情况，该怎么办呢？


