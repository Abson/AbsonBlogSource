---
title: C++项目中遇到的一些方法归纳(持续更新)
date: 2018-04-02 18:24:27
tags: C++,
      代码片段
---

浏览 C++ 项目中收集到的代码学习片段
<!-- more -->

### 将 string 类型，转换为模板类型
```cpp
 template <typename P0>
  bool FromString(const std::string& s, P0* p) {
    std::istringstream iss(s);
    iss >> std::boolalpha >> *p;
    return !iss.fail();
  }
```

---

### 将模板类型 ，转换为string 类型
```cpp
template <class T>
static bool ToString(const T &t, std::string* s) {
  RTC_DCHECK(s);
  std::ostringstream oss;
  oss << std::boolalpha << t;
  *s = oss.str();
  return !oss.fail();
}
```

---

### Mach内核获得Cpu使用率百分比
```cpp
NSInteger ARDGetCpuUsagePercentage() {
  // Create an array of thread ports for the current task.
  const task_t task = mach_task_self();
  thread_act_array_t thread_array;
  mach_msg_type_number_t thread_count;
  if (task_threads(task, &thread_array, &thread_count) != KERN_SUCCESS) {
    return -1;
  }

  // Sum cpu usage from all threads.
  float cpu_usage_percentage = 0;
  thread_basic_info_data_t thread_info_data = {};
  mach_msg_type_number_t thread_info_count;
  for (size_t i = 0; i < thread_count; ++i) {
    thread_info_count = THREAD_BASIC_INFO_COUNT;
    kern_return_t ret = thread_info(thread_array[i],
                                    THREAD_BASIC_INFO,
                                    (thread_info_t)&thread_info_data,
                                    &thread_info_count);
    if (ret == KERN_SUCCESS) {
      cpu_usage_percentage +=
          100.f * (float)thread_info_data.cpu_usage / TH_USAGE_SCALE;
    }
  }

  // Dealloc the created array.
  vm_deallocate(task, (vm_address_t)thread_array,
                sizeof(thread_act_t) * thread_count);
  return lroundf(cpu_usage_percentage);

```
> 题外话，更具体的虚拟内存、物理内存、CPU 使用等等获取方法在以下[链接](https://stackoverflow.com/questions/63166/how-to-determine-cpu-and-memory-consumption-from-inside-a-process?rq=1)

---
 
### 字节对齐
**8个字节对齐**
```cpp
#   define WORD_MASK 7UL
static inline uint32_t word_align(uint32_t x) {
    return static_cast<uint32_t>((x + WORD_MASK) & ~WORD_MASK);
}
static inline size_t word_align(size_t x) {
    return (x + WORD_MASK) & ~WORD_MASK;
}
```

---

### 指针的二进制、十进制、八进制输出

**二进制**
```cpp
const void* p
std::cout << std::bitset<sizeof(int32_t)*8>((uintptr_t)p) << std::endl;
```
**十进制**
```cpp
const void* p
std::cout << (uintptr_t)p << std::endl;
```
**十六进制**
```cpp
const void* p
std::cout << std::hex << (uintptr_t)p << std::endl;
```
**八进制**
```cpp
const void* p
std::cout << std::oct << (uintptr_t)p << std::endl;
```

---

### 利用模板函数将数字转为字符串
```cpp
const char digits[] = "9876543210123456789";const char* zero = digits + 9;

template<typename T>size_t covert(char buf[], T value){  T i = value;  char* p = buf;  /*从 value 中每个位转为  0-9 的字符串*/  do   {    int lsd = static_cast<int>( i % 10 );    i /= 10;    *p++ = zero[lsd];  } while(i != 0);  /*添加正负号*/  if (value < 0)  {    *p++ = '-';  }  *p = '\0';
  /* 由于是从个位开始添加字符串的，需要反转容器元素顺序 */  std::reverse(buf, p);  return p - buf;}
```

---

### 模板静态检测传入参数是否符合规格
```cpp
template <typename T, std::ptrdiff_t Size>class ArrayViewBase{static_assert(Size > 0, "ArrayView Size must be variable or non-negative");};
```

---

### 内存屏障
例子中使用内存屏障，防止编译器进行优化，让读取 `ptr` 的操作要在 memset 方法后才进行
```cpp
void ExplicitZeroMemory(void* ptr, size_t len) {  memset(ptr, 0, len);  // 内存屏障 __asm__ __voilate__("" ::: "memory");  /* As best as we can tell, this is sufficient to break any optimisations that     might try to eliminate "superfluous" memsets. If there's an easy way to     detect memset_s, it would be better to use that. */  __asm__ __volatile__("" : : "r"(ptr) : "memory");}
```
PS: 题外话
>C与C++语言中，volatile关键字意图允许内存映射的I/O操作。这要求编译器对此的数据读写按照程序中的先后顺序执行，不能对volatile 内存的读写重排序。因此关键字 volatile 并不保证是一个内存屏障。
>对于Visual Studio 2003，编译器保证对 volatile 的操作是有序的，但是不能保证处理器的乱序执行

---

### SFINAE && std::enable_if<>
SFINAE是英文 Substitution failure is not an error 的缩写，意思是匹配失败不是错误。
我们可以利用 `std::enable_if<>` 来腿短模板成不成立

```cpp
// 模板方法1
template <typename T>
typename std::enable_if<std::is_trivial<T>::value>::type SFINAE_test(T value) { std::cout<<"T is trival"<<std::endl; } 

// 模板方法2
template <typename T> 
typename std::enable_if<!std::is_trivial<T>::value>::type SFINAE_test(T value) { std::cout<<"T is none trival"<<std::endl; }

```
当传入类型 T 的 std::is_trivial 为 false 的时候，就会调用`模板方法2`, 否则为 `模板方法1`

---

### 如何模拟 GCD 中的 dispatch_async 和 dispatch_sync
```cpp
CFRunLoopRef m_cfRunLoop([runLoop getCFRunLoop])
 
// This is analogous to dispatch_async
void runAsync(std::function<void()> func) {
  CFRunLoopPerformBlock(m_cfRunLoop, kCFRunLoopCommonModes, ^{ func(); });
  CFRunLoopWakeUp(m_cfRunLoop);
}

// This is analogous to dispatch_sync
void runSync(std::function<void()> func) {
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


