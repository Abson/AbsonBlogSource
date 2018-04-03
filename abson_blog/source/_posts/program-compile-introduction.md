---
title: 计算机程序编译过程 
date: 2018-04-02 18:28:42
tags: 计算机基础
---

>现在这个社会充斥着太多的水货程序员了，他们不懂任何计算机原理，但是他们依旧做着公司的业务，很好的完成老板交代的任务，但是这些东西永远对他们来说都说一个黑盒子，他们不会知道为什么会产生段错误，为什么会有悬挂指针，不知道为什么会产生链接错误，什么是缺失符号，更不知道为什么函数 return 一个局部变量就会段错误，返回一个字面量常量就不会，当发生这些问题来只能谷歌，stackoverflow，甚至只能是百度。所以，这里一步一步给大家科普，不断普及一些“常识”让大家更好的理解我们编写的软件程序。

<!-- more -->

计算机程序编译过程分为4个步骤：
* 预处理
* 编译
* 汇编
* 链接

##### 预处理
```
$gcc -E hello.c -o hello.i
```
或
```
$cpp hello.c > hello.i
```

预编译过程主要：
1. 处理预编译指令，例如 #define，#if，#ifdefine 等。
2. 将 #include 包含文件插入到该预编译指令的位置
3. 删除所有的注析 // 、/**/

##### 编译
```
$gcc –S hello.i –o hello.s
```
或
```
$gcc –S hello.c –o hello.s
```

编译代表了一整个过程：
* 词法分析
* 语法分析
* 语义分析
* 源代码优化
* 代码生成
* 目标代码优化

###### 词法分析
扫描字节序并产生记号。
词法分析产生的记号一般可以分为如下几类：关键字、标识符、字面量（包含数字、字符串等）和特殊符号（如加号、等号）。

![](http://upload-images.jianshu.io/upload_images/1073278-13008206cdcbbb1b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

###### 语法分析
语法分析器（Grammar Parser）将对由扫描器产生的记号进行语法分析，从而产生语法树（Syntax Tree）。由语法分析器生成的语法树就是以表达式（Expression）为节点的树。

![](http://upload-images.jianshu.io/upload_images/1073278-0eabaceefb127822.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


###### 语义分析

语法分析仅仅是完成了对表达式的语法层面的分析，但是它并不了解这个语句是否真正有意义。
编译器所能分析的语义是静态语义（Static Semantic），所谓静态语义是指在编译期可以确定的语义，与之对应的动态语义（Dynamic Semantic）就是只有在运行期才能确定的语义。
静态语义通常包括声明和类型的匹配，类型的转换。

![](http://upload-images.jianshu.io/upload_images/1073278-0b62dffd774b7645.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


**动态语义和静态语义?**
> 比如将一个浮点型赋值给一个指针的时候，语义分析程序会发现这个类型不匹配，编译器将会报错。动态语义一般指在运行期出现的语义相关的问题，比如将0作为除数是一个运行期语义错误。

###### 中间语言生成
中间代码使得编译器可以被分为前端和后端。编译器前端负责产生机器无关的中间代码，编译器后端将中间代码转换成目标机器代码。这样对于一些可以跨平台的编译器而言，它们可以针对不同的平台使用同一个前端和针对不同机器平台的数个后端。

![](http://upload-images.jianshu.io/upload_images/1073278-831ed60a601c1561.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


######  目标代码生成与优化
 代码级优化器产生中间代码标志着下面的过程都属于编译器后端。编译器后端主要包括代码生成器（Code Generator）和目标代码优化器（Target Code Optimizer）。
 代码生成器将中间代码转换成目标机器代码，这个过程十分依赖于目标机器。
 
 对于上面例子中的中间代码，代码生成器可能会生成下面的代码序列
```
movl index, %ecx        ; value of index to ecx
addl $4, %ecx           ; ecx = ecx + 4
mull $8, %ecx             ; ecx = ecx * 8
movl index, %eax        ; value of index to eax
movl %ecx, array(,eax,4)  ; array[index] = ecx
```

##### 汇编

```
$as hello.s –o hello.o
```
或
```
$gcc –c hello.c –o hello.o
```

汇编器(as)将汇编代码翻译成机器语言指令，把这些指令打包成一种叫做`可重定位目标程序(relocatable)`的格式，并将结果保持在目标文件 hello.o 中。hello.o 是一个二进制文件。


##### 链接

```
$ld -static /usr/lib/crt1.o /usr/lib/crti.o /usr/lib/gcc/i486-linux-gnu/4.1.3/crtbeginT.o -L/usr/lib/gcc/i486-linux-gnu/4.1.3 -L/usr/lib -L/lib hello.o --start-group -lgcc -lgcc_eh -lc --end-group /usr/lib/gcc/i486-linux-gnu/4.1.3/crtend.o /usr/lib/crtn.o
```

链接阶段最重要的工作就是重定位，将所有的目标文件都链接起来，在 #include 文件中的函数声明原本只会生成一个没有跳转地址的指令，重定位的工作在其他目标文件中找到这些目标地址，并把地址填补进去。

链接分为静态链接和动态链接，也就是我们通常说的私有对象和共享对象。这里得展开另外一篇来讲了，内容太多。

![](http://upload-images.jianshu.io/upload_images/1073278-7f70cb9cf938d2fa.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

链接就像将所有的组件合成一起，组成一个整体。
