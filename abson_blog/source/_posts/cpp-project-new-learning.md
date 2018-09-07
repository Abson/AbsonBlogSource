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

### 利用模板函数将数字转为字符串
```
const char digits[] = "9876543210123456789";const char* zero = digits + 9;

template<typename T>size_t covert(char buf[], T value){  T i = value;  char* p = buf;  /*从 value 中每个位转为  0-9 的字符串*/  do   {    int lsd = static_cast<int>( i % 10 );    i /= 10;    *p++ = zero[lsd];  } while(i != 0);  /*添加正负号*/  if (value < 0)  {    *p++ = '-';  }  *p = '\0';
  /* 由于是从个位开始添加字符串的，需要反转容器元素顺序 */  std::reverse(buf, p);  return p - buf;}
```


