---
title: 'C++20协程与C++26 std::execution结构化并发在相机性能优化中的应用'
date: 2025-01-01
author: 李正刚
permalink: /posts/2025/01/blog-post-1/
tags:
  - C++
  - C++26
  - C++20
---
## 3. 执行策略与并行算法 (C++17/20)

C++17/20通过执行策略将并发能力直接集成到STL算法中，是数据并行编程的重大进步。

执行策略类型：共有四种策略，用于指定算法的执行方式。许多STL算法（如 std::sort, std::for\_each, std::transform）的重载版本将执行策略作为第一个参数。

* std::**execution::seq**：强制顺序执行。
* std::**execution::par**：允许跨多个线程并行执行，是最常用的并行策略。（多个线程可以同时运行，但同一个线程内的迭代是顺序进行的（不会交叉执行））
* std::**execution::par\_unseq**：在par基础上，允许在单个线程内进一步向量化（使用SIMD指令）和无序执行（同一个线程内的多个迭代可能以交错/非顺序的方式执行（例如同时处理 N个元素，且它们的指令可以交织））。（par\_unseq相比par有额外限制，par\_unseq不允许使用thread\_local变量，且不允许调用任何可能导致阻塞的函数（如mutex、condition\_variable、sleep()等函数））
* std::**execution::unseq** (C++20)：（不启用多线程）允许在单个线程内进行向量化（编译器使用如 AVX、SSE 等 SIMD 指令）和无序执行（相当于par\_unseq 去掉并行的版本）。

C++20继续修复了std::thread中遗留的部分设计问题，包括：

* RAII式析构自动join()的std::**jthread**，
* 线程的取消操作&停止令牌std::**stop\_token**。（当用户不再对线程运行结果感兴趣时，可以取消此线程执行）

C++20同时引入了一些重要的同步原语用于协调多个线程间的同步：

* std::**latch**：一个**倒计时锁存器**，它是一种**一次性使用**的同步机制，允许线程等待直到计数器减为零（用于实现类似stdexec::when\_all()语义）。适用于一次性的等待（如所有子线程跑完）场景，如初始化、任务完成等。
* std::**barrier**：一个**可重复使用的屏障**，用于协调多个线程在（多阶段的）某个执行点的同步，支持多阶段重复使用+自定义回调。允许多个（一般是含有多个阶段的）线程在\*\*某个执行点同步（**如 所有线程都到达了阶段A，并（可选地、可分离地）等其他线程到达阶段A**）\*\*，然后继续执行下一阶段。适用于需要多次同步的复杂并行算法。
* std::\*\*counting\_semaphore：\*\*用于控制并发访问共享资源的线程数量。它允许多个线程同时访问一个资源（相对比std::mutex仅允许单个线程访问），但限制最大并发数（至少支持 `LeastMaxValue` 个并发访问者。）。
* std::**atomic\_ref**: 用于为满足std::is\_trivially\_copyable的T类型对象提供 原子型API的引用语义。

部分三方库在上述标准库并发工具之上，基于**任务及依赖关系**描述进一步完善了并行框架，

Task-parallel Programming - 《[**Dynamic\_Asynchronous\_Tasking\_with\_Dependencies**](https://github.com/CppCon/CppCon2025/blob/main/Presentations/Dynamic_Asynchronous_Tasking_with_Dependencies.pptx)》 CppCon2025

如：

* [**Taskflow**](https://github.com/taskflow/taskflow)**：**

三方库**Taskflow**基于任务的依赖关系，通过taskGraph/有向无环图 ([DAG](https://github.com/ChunelFeng/CGraph)，节点表示任务，有向边表示依赖关系) 来管理任务依赖（内置通过 precede()/succeed() 等方法建立**声明式依赖关系**），并内置高性能的**工作窃取（work-stealing）调度器**（线程池中的空闲线程去别的线程的列表尾部窃取任务来执行，便于动态负载均衡），对 C++ 标准库并发工具进行补充和高级抽象。（C++17, header-only, MIT license）

（据称 荣耀影像部门在研究taskflow落地camera影像框架，TBD...）

Taskflow核心流程

菱形任务管线示例：先A，再并行B\\C;最后D

4中代码实现上述菱形任务示例：1. taskflow   2. OpenMP   3.TBB   4. Kokkos

* [**FFRT**](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/ffrt-overview)**：**

Function Flow Runtime Kit：HarmonyOs上通过任务及其依赖关系描述的并发编程框架。

注：1\~3在已有的相机HAL代码（C++17及之前代码常用的）较常见，参考《C++ Concurrency in Action》
