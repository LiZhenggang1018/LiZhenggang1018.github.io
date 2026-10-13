## 代码优化

1. 循环优化：只做最必要的，不做非必须的（声明/构造变量，无用的重复计算）
2. 循环展开Loop Unrolling：将循环体复制多次，每次迭代执行更多工作，从而减少循环分支判断次数。如：for里面同时做4个数据的处理，然后每次跳4个数据（Cache友好）（-O2编译器会自动优化）
3. 移动语义避免拷贝/copy elision，函数参数传递与返回时注意对象传递方式，典型如：
   1. std::**string\_view**（某些场景）代替std::string作为函数形参。（不涉及元素所有权时）用视图**view**（或std::**span**）替代容器来传参。（C++20）
   2. 函数输出数值时，使用返回值而非输出参数。（编译器保证**RVO**/Return Value Optimise, **URVO**/ Unnamed Return Value Optimise，没有不必要拷贝）。

## 并行化

1. C++标准库：
   1. std::thread
   2. Coroutines （C++20）
   3. std::execution （C++26）
2. 并行软硬件基础设施：
   1. OpenMP
   2. CUDA
   3. OpenCL
   4. SYCL
   5. SIMD

## 多线程优化

1. 锁的颗粒度（只保护必要的数据竞争条件下的临界区共享数据），不在锁中调用未知的回调。
2. atomic替代锁，lock-free programming。（但情况复杂时，由于编译器会在保证结果不变的情况下自由地乱序调整，消除中间变量等，加锁会更简单可靠）
3. std::shared\_mutex等共享锁（区分读写case，多个读操作不互斥）替代简单锁std::mutex
4. thread\_local线程本地变量（避免在多个线程之间共享数据）

## 容器优化

1. C++标准库容器类型选择（是否随机访问，是否首/任意/尾需插入/删除，是否排序，是否键唯一，缓存友好性，LIFO/FIFO）(CppCon中有测试文章表示vector总是好)
2. **emplace\_back**(arg)（有时候）替换push\_back(Obj(arg))
3. 用**reserve**来保留大小，减少内存动态分配。
4. 选择合适的算法和数据结构（复杂度优化）

## 内存管理&内存池

**PMR**：多态内存资源 (Polymorphic Memory Resources) ：C++标准库中提供的高性能（优化后的STL标准库实现[3.5x faster](https://www.youtube.com/watch?v=q6A7cKFXjY0)）的预定义内存资源分配策略。它提供了一种标准化、可扩展的内存管理框架，可高效实现**内存池**功能。(在关键路径上避免反复申请/释放堆内存。)

1. 多线程[**synchronized\_pool\_resource**](https://en.cppreference.com/w/cpp/memory/synchronized_pool_resource.html)
2. 单线程 [**unsynchronized\_pool\_resource**](https://en.cppreference.com/w/cpp/memory/unsynchronized_pool_resource.html)

参考《[C++17 - The Complete Guide](https://www.josuttis.com/welcomee.html)》

## 缓存友好性

优化内存访问、提高缓存命中率：

1. 顺序内存访问：如行优先遍历(访问内存时尽量顺序访问，避免随机访问。相关联/易被一起访问的数据尽量放一起/紧凑排布)
2. 声明结构体时，用位字段压缩数据，成员按大小递减的顺序声明减少padding。
3. 消除多线程中的伪共享（两个不同的处理器修改位于同一缓存行上的不同变量时）问题：
   1. 多线程共享的数据 声明 **alignas(std::hardware\_destructive\_interference\_size) **对齐/填充内存对象。
   2. 用局部变量替代多线程共享的数据 消除数据竞争
4. 冷热数据分区，冷热数据使用不同的内存分配器，热数据尽量排布在一起共享缓存行。
5. 内存预取Pre-fetch，指令预加载。

## 文件IO

1. 异步IO替代同步IO：io\_uring&lib\_uring
2. 压缩数据：zlib，图像视频文件的编解码软硬件算法（MediaCodec、libjpeg\_turbo）。
3. 共享内存方案：DMA 、Gralloc、 ION 、Ashmem等共享内存规避不同进程间通信或不同设备间通信的内存拷贝。

## 优化so大小

1. 优化代码大小：**-Os**（比-O2保守，关注的是产生最小so文件）；**-Oz** 更激进
2. 导出符号可见性控制：
   1. .mk文件声明默认不导出**-fvisibility=hidden**
   2. .h/.cpp文件中**attribute((visibility("default")))**定向导出符号
   3. 使用版本脚本精确控制导出符号。
3. 链接器移除未使用的代码：**-ffunction-sections -fdata-sections**
4. 编译时去除符号信息（等同于strip）：**-s -DNDEBUG**
5. 涉及模板时，[**thin-template**](https://github.com/xiaoweiChen/Cpp-Templates-2nd)用于减少模板代码膨胀（将模板的公共部分提取到非模板基类中，减少模板实例化产生的冗余代码量）。
6. **EBO**（Empty Base Optimization）空基类优化、**[no\_unique\_address](no_unique_address) ** 等（C++20）

## 三方数学库

1. BLAS：基础线性代数子程序
2. LAPACK：线性代数包
3. Intel MKL
4. eigen：矩阵计算

# 优化原则

1. 测量优先：先profiling，找到瓶颈，再优化热点。（"过早优化是万恶之源"）
2. 80/20法则：80%的时间花在20%的代码上（阿姆达尔定律Amdahl's Law，系统优化中"短板效应"的本质）

# 参考：

[**《**](https://weedge.github.io/perf-book-cn/zh/)[**Performance Analysis and Tuning on Modern CPU**](https://weedge.github.io/perf-book-cn/zh/)[**》**](https://weedge.github.io/perf-book-cn/zh/)书

**《**[**Performance Hints**](https://abseil.io/fast/hints#performance-hints)**》 **by [Jeff Dean](https://research.google/people/jeff/), [Sanjay Ghemawat](https://research.google/people/sanjayghemawat/)

《 Optimizing software in C++:An optimization guide for Windows,Linux adn Mac》

《[**Algorithms for Modern Hardware**](https://en.algorithmica.org/hpc/)》 by [Sergey Slotin](http://sereja.me/).

《[**CppCoreGuidelines**](https://github.com/lynnboy/CppCoreGuidelines-zh-CN/blob/master/CppCoreGuidelines-zh-CN.md)》 Per: 性能 章节
