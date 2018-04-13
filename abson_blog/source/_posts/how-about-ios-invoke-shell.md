---
title: how-about-ios-invoke-shell
date: 2018-04-13 10:36:35
tags: 
			- 计算机基础
			- 思考
			- iOS
			- Unix
categories: 
			- 计算机系统
			- iOS
---

一开始认为iOS是Unix系统，肯定是可以调用Shell命令的。但是后面发觉事情并不是那么简单。
>确定是否能调用Shell命令的要项：
>1. 是否存在 Shell 程序
>2. 是否能使用多进程 (因为 shell 命令都是 fork 出一个进程进行处理的)

<!-- more -->

**首先明白什么是 Shell?**
>Unix shell，一种壳层与命令行界面，是Unix操作系统下传统的用户和计算机的交互界面。第一个用户直接输入命令来执行各种各样的任务。
>普通意义上的shell就是可以接受用户输入命令的程序。它之所以被称作shell是因为它隐藏了操作系统低层的细节。

>意思就是 Shell 命令会执行系统的底层 API 进行，让用户通过简单得命令执行复杂的系统操作。

首先确定iOS是否存在 Shell 程序 (这个还真的要确认一下)，但是就目前的情况来看，iOS 并不存在任何 Shell 程序。

一开始我上网查找，找到最多的都是使用 `system` 函数
```
int	 system(const char *)

system("ls -al")
```
后来一看，真机上毛输出都没有，返回结果是 `0x7f00`, 意思就是 没有权限操作，真是坑了个爹。

然后看一下系统，发觉这个函数在 iOS8 被抛弃了，系统建议用 `posix_spawn` 好吧，可能跟这个有关系

```
pid_t pid;
char* argv[]  =
{
	"ls",
	NULL
};

int result = posix_spawn(&pid, argv[0], NULL, NULL, argv, environ);
perror("posix_spawn");
waitpid(pid, NULL, 0);
```
等到的输出，一直是 `posix_spawn: No child processes`。很是绝望。

没办法，后来在 Stack Overflow 上面找到一个[帖子](https://stackoverflow.com/questions/14170416/executing-a-bash-command-in-an-ios-app)。

>Yes, you can but it is extremely limited, and ping will probably not work... Regardless use the system() and check gdb.
>But Quentin is right about using PING.
>NOTE: This is only useful for debugging and shouldn't be used for actual apps.

一看，好东西，原来真机上不行，但是在模拟器上可以搞，原以为很开心的，因为起码能用，结果模拟器上使用`system` 函数输出如下：
```
dyld: dyld_sim cannot be loaded in a restricted process
```
很是无语呀，iOS 应用上无法使用多进程。

想了想，越狱行不行，通过越狱的话，就可以使用多进程了，如果没有 Shell 的话，直接使用 [OpenSSH(OpenBSD Secure Shell)](https://zh.wikipedia.org/wiki/OpenSSH),这样我们就可以通过远程连接来操作 iPhone了， 然后再通过 pc 进行 ssh 连接过去，然后就可以使用命令行了。
找到了[文章](http://blog.sina.com.cn/s/blog_51d3553f0100xrxz.html)证实了想法。

