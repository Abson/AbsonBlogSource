---
title: Posix多线程之旅——生产者-消费者 
date: 2018-04-03 17:40:44
tags:
---

生产者消费者问题（英语：Producer-consumer problem），也称有限缓冲问题（英语：Bounded-buffer problem），是一个多线程同步问题的经典案例。该问题描述了共享固定大小缓冲区的两个线程——即所谓的“生产者”和“消费者”——在实际运行时会发生的问题。生产者的主要作用是生成一定量的数据放到缓冲区中，然后重复此过程。与此同时，消费者也在缓冲区消耗这些数据。该问题的关键就是要保证生产者不会在缓冲区满时加入数据，消费者也不会在缓冲区中空时消耗数据。

<!-- more -->

![](/images/posix-producer-consumer-problem/problem.jpg)
 
但是在 leveldb 中有这么一段代码，很好的写出了一个后台线程传入事件异步并发执行执行。
```cpp
#include <iostream>
#include <deque>
#include <unistd.h>

class PosixEnv
{
public:
    PosixEnv() : started_bgthread_(false)
    {
        PthreadCall("mutex_init", pthread_mutex_init(&mu_, NULL));
        PthreadCall("cvar_init", pthread_cond_init(&bgsignal_, NULL));
    }

    void Schedule(void(*function)(void*), void* arg)
    {
        PthreadCall("lock", pthread_mutex_lock(&mu_));

        // Start background thread if necessary
        if (!started_bgthread_)
        {
            started_bgthread_ = true;
            PthreadCall(
                    "create thread",
                    pthread_create(&bgthread_, NULL,  &PosixEnv::BGThreadWrapper, this));
        }

        if (queue_.empty())
        {
            // 如果当前队列为空，那么后台线程此时可能为信号等待状态，激活一个等待该条件的线程
            PthreadCall("signal", pthread_cond_signal(&bgsignal_));
        }

        BGItem item = BGItem();
        item.arg = arg;
        item.function = function;
        queue_.push_back(item);

        // 当 unlock 后，pthread_cond_wait 获取到条件信号和锁，就会执行之后的代码
        PthreadCall("unlock", pthread_mutex_unlock(&mu_));
    }

    static void* BGThreadWrapper(void* arg)
    {
        reinterpret_cast<PosixEnv*>(arg)->BGThread();
        return NULL;
    }

#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Wmissing-noreturn"
    void* BGThread()
    {
        while (true)
        {
            // Wait until there is an item that is ready to run
            PthreadCall("lock", pthread_mutex_lock(&mu_));// 这个mutex主要是用来保证 pthread_cond_wait的并发性
            while (queue_.empty())
            {
               // 等待条件信号，并将锁交出去，当获取到信号量
                PthreadCall("wait", pthread_cond_wait(&bgsignal_, &mu_));
                // pthread_cond_wait 会先解除之前的 pthread_mutex_lock 锁定的 mtx，然后阻塞在等待对列里休眠，
                // 直到再次被唤醒（大多数情况下是等待的条件成立而被唤醒，唤醒后，该进程会先锁定先 pthread_mutex_lock(&mtx);
            }

            void (*function)(void*) = queue_.front().function;
            void* arg = queue_.front().arg;
            queue_.pop_front();

            PthreadCall("unlock", pthread_mutex_unlock(&mu_));
            // 异步并行执行方法，如果将这句代码放在 unlock 之前那么就是异步串行了
            (*function)(arg);
        }
    }
#pragma clang diagnostic pop

private:
    void PthreadCall(const char* label, int result)
    {
        std::cout << label << std::endl;
        if (result != 0) {
            fprintf(stderr, "pthread %s: %s\n", label, strerror(result));
            abort();
        }
    }

    pthread_mutex_t mu_;
    pthread_cond_t bgsignal_;
    pthread_t bgthread_;
    bool started_bgthread_;

    struct BGItem { void* arg; void (*function)(void*); };
    typedef std::deque<BGItem> BGQueue;
    BGQueue queue_;
};

void speak(void* pstr)
{
    std::string* str = reinterpret_cast<std::string*>(pstr);
    printf("我要说话啦：%s\n", str->c_str());
}

int main(int argc, const char * argv[]) {

    std::string str("哇哈哈哈啊哈哈哈啊哈哈");
    std::string str2("你的豆腐的的地方水电费违反诶我去翁");

    PosixEnv env = PosixEnv();
    env.Schedule(&speak, &str);
    sleep(10);

    env.Schedule(&speak, &str2);

    sleep(5);

    return 0;
}
```

上面例子，把所有的事件模型包装成一个`struct BGItem` 的结构体，而通过 `BGQueue` 对消息队列进行处理，这里没有用到延时执行的消息队列，也没有优先队列的处理，只是一个简单的生产者——消费者的模型。
通过创建一个工作线程 `bgthread_` 来开启线程运作，通过信号量 `pthread_cond_wait` 和 `pthread_cond_signal` 来进行控制工作线程的阻塞和运作。
该例子中，主线程负责生成事件模型，而工作线程负责消费处理事件模型，而消息队列，就是我们所说的缓冲区了。


