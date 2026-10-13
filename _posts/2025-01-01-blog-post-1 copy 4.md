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

### [结构化并发Structured Concurrency](https://ericniebler.com/2020/11/08/structured-concurrency/)

定义：一种编程模式，它确保**并发操作具有明确的结构和生命周期**，使得并发任务能够以类似于**结构化编程**的方式组织和管理。在结构化并发中，并发任务的生命周期被限制在一个明确的作用域内（[异步作用域与词法作用域重合](https://ericniebler.com/2021/08/29/asynchronous-stacks-and-scopes/)），**子任务的生存期不应超过其父任务的生存期**，确保并发操作有明确的开始和结束点。当作用域退出时，所有并发任务都必须已经完成（确保子操作在父操作之前完成的方式）。这有助于避免诸如任务泄漏、未定义行为以及复杂的生命周期管理等问题。

**典型应用**：C++20协程

**优势**（相比异步回调）：

* 资源安全：自动清理并发资源
* 异常安全：异常传播更可靠
* 可读性：并发结构更清晰
* 调试友好：调用栈更易理解

1. 传统同步代码（不使用goto，通过条件分支、循环、函数调用等结构化编程的方式）（默认）都是结构化编程：对象的生命周期由严格嵌套的作用域指定。函数激活嵌套，作用域嵌套，对象生命周期也嵌套。
2. 异步代码（不使用结构化并发时）中使用**异步回调**，带来坏处：业务逻辑与对象生命周期的非局部的不连续性。需手动维护状态（异步回调闭包捕获的变量，需要通过智能指针或值语义捕获，否则容易发生生命周期问题），难以阅读（逻辑跨越多个函数和几个对象，控制和数据流并不明显）。
3. 结构化并发，用同步编程风格表达异步操作，协程保证了编写过程中的思维连贯性，使得函数（闭包）体本身就无缝保持了程序状态。逻辑紧凑，可读性高，不易写出错的代码，可调试性强。

添加 co\_await 使得同步函数变为异步，而**不干扰计算的结构**（可以继续使用我们熟悉的所有习惯用法：使用异常处理错误，状态保存在局部变量中，使用析构函数释放资源，按值或按引用传递参数，以及所有其他良好、安全和符合现代 C++习惯的特征）

被等待的异步操作**必然**在调用函数之前完成，就像普通的函数调用一样。**作用域和生命周期仍然像往常一样嵌套**，只是现在作用域在时间上是不连续的。

但假如使用原始回调和future，这种结构就会丧失。

// 1：同步代码，OK

void computeResult(State & s);

int doThing() {

State s;

computeResult(s);

return s.result;

}

// 2.1：异步回调，捕获裸指针导致悬挂引用，不OK

boost::future computeResult(State & s);

boost::future doThing() {

State s;

auto fut = computeResult(s); // 逻辑和生命周期分散在多个线程中时，程序的层次结构// 就会丧失。特别是与嵌套作用域相关的嵌套生命周期

return fut.then(

[&](auto&&) { return s.result; }); // OOPS
}

// 2.2：异步回调，捕获智能指针，OK

// 异步回调函数所访问的任何数据都必须满足以下条件之一：**全局变量、线程局部变量，或使用引用计数**，因为这两个操作的激活过程并不嵌套。——[《Working\_with\_Asynchrony\_Generically\_Parts\_1\_2\_Eric\_Niebler\_1》](https://github.com/CppCon/CppCon2021)

boost::future computeResult(shared\_ptr s); // addref the state

boost::future doThing() {

auto s = std::make\_shared();

auto fut = computeResult(s);

return fut.then(

[s](auto&&) { return s.result; }); // addref the state
}

// 2.3：异步+结构化并发，OK

cppcoro::task<> computeResult(State & s);

cppcoro::task doThing() {

State s;

co\_await computeResult(s);

// [协程是对回调的语法糖](https://ericniebler.com/2021/08/29/asynchronous-stacks-and-scopes/)；在协程中的 co\_await 之后的所有内容都是回调。编译器就是这样处理的。

co\_return s.result;

}

"The popular concurrency primitives – \*\*go statements, thread spawning functions, callbacks, futures, promises, ... they're all variants on **goto**, in theory and in practice. \**And not even the modern domesticated goto, but the old-testament fire-and-brimstone goto, that could leap across function boundaries. These primitives are dangerous even if we don't use them directly, because they undermine our ability to reason about control flow and compose complex systems out of abstract modular parts, and they interfere with useful language features like automatic resource cleanup and error propagation. Therefore, like goto, they have no place in a modern high-level language." \**"流行的并发原语 – go 语句、线程生成函数、回调、\*\*\*\*futures, promises ... \*\*\*\*它们在理论和实践中都是\*\*\*\*goto\*\*\*\*的变体。\*\*甚至不是现代驯化的 goto ，而是旧约中的火与硫磺 goto ，\*\*能够跨越函数边界。**这些原语是**危险**的，即使我们不直接使用它们，因为它们**削弱了我们对控制流的推理能力\*\*，并使我们**无法将复杂系统组合成抽象的模块化部分**，同时它们还干扰了有用的语言特性，如**自动资源清理**和**错误传播**。因此，像 goto 一样，它们在现代高级语言中没有立足之地。"

——《[Notes on structured concurrency, or: Go statement considered harmful](https://vorpus.org/blog/notes-on-structured-concurrency-or-go-statement-considered-harmful/)》
