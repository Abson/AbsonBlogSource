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
