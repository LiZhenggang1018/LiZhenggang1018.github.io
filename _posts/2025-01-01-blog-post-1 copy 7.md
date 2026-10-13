# 其他

C++20协程&C++26 std::execution相比传统thread是有部分优势的，如下：

1. 协程+执行器的方案有C++标准委员会加Nvidia官方背书，进入了std标准库，后续有（较大）概率成为（统一CPU+GPU编程范式）并发编程的基础框架设施。（可能在2032年==2026（C++26发布）+6（所有编译器支持+技术铺开）成为并发编程主流）。
2. 结构化并发的各种同步原语（stdexec中的sender&receiver模型+函数）可以非常方便地描述任务执行依赖（以stdexec::when\_all()为例，（可以思考下）仅凭std::thread（且不使用C++20 std::latch std::barrier时）如何表述等N个并行的结果算出来之后合并汇总？同理stdexec::transfer从一个执行上下文（如一个线程池或调度器）切换到另一个，也很难在std::thread中找到类似的），异步代码写起来（跟同步代码一样）很简单。

```cpp
// 尝试不借助stdexec实现如下语义？？
sender auto composed_cancellation_example(auto query) {
  return stop_when( // 等待2个事件timeout()跟on_click()，任何一个事件完成则取消另一个
    timeout( // 设置5s超时，超过5s则取消
      when_all( // 等first_successful 和 load_file都跑完。一个操作失败时立即取消其他操作
        first_successful( // query_server_a(query) 成功完成时发送停止请求
          query_server_a(query),
          query_server_b(query)),
        load_file("some_file.jpg")),
      5s),
    cancelButton.on_click());
}
```

1. （部分）公司把所有自研拍照算法都改成了协程+std::execution，尽可能将所有算法子模块都按照结构化并发的方式做了并行，实测性能有收益。
2. 后处理算法比HAL更适合做这样的结构化并发，原因：算法（复杂的数学计算+简单的软件设计）天然容易拆分成输入输出依赖关系明确的原子化纯函数且收益显著，HAL（复杂的依赖关系+软件逻辑）等软件侧则拆分要稍复杂些。

# 更多资料

《[Using Senders/Receivers](https://accu.org/journals/overload/33/185/teodorescu/)》  By Lucian Radu Teodorescu  更多使用std::execution实现各类并发任务的实例&代码

[overload185\_sr\_examples](https://github.com/lucteo/overload185_sr_examples)

[Concurrency Approaches: Past, Present, and Future](https://accu.org/conf-docs/PDFs_2023/XConcurrencyapproachespastpresentandfuture.pdf)

[Senders/Receivers: An Introduction](https://accu.org/journals/overload/32/184/teodorescu/)

* 📎 [cppcon\_20\_Taskflow\_Parallel and Heterogeneous Task Programming in Modern C++.pdf](https://docs.vivo.xyz/detail/pdf/100119653354?appName=knowledge-server&spaceId=null)
  # 其他

[Vas并发框架优化讨论
