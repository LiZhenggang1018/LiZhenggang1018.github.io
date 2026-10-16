# 《Back_To_Basics_Memory_Alignment》

跨cache_line才是导致访问速度下降的重要原因

std::offset用来计算成员变量地址到结构体首地址的地址偏移

std::**start_lifetime_as**<T>(ptr) （C++23）提供了一种标准化的方式，显式地通知编译器：从 ptr 处开始，存在一个 T 类型的对象。它不会在运行时执行任何操作，不复制字节，也不调用构造函数——纯粹是对象模型层面的"记账"，但正是这个记账使得后续访问成为良构的。从而消除因对象生命周期不匹配而导致的未定义行为。

去padding可通过：

```
struct **\__attribute_\_((packed))** PackedData {} // GCC/Clang
```

```
# pragma pack(push, 1) // MSVC
```

struct PackedData {

用于：

网络协议——精确匹配线路格式

文件格式——二进制文件头

硬件寄存器——内存映射 I/O

内存受限系统——嵌入式设备

大型数组——当节省内存很重要时

std::vector<bool>里面的坑

使用 alignas、alignof、sizeof、std::align 来检查和控制

《Writing_Low_Latency_Cpp》

```
# 《Can_I_memcpy_This_Type_Across_a_Boundary》

问题：

IPC/跨进程（进程运行在不同的软件环境：如Linux+Clang & MacOs+Clang）通信传递C++对象时，如何保证C++对象的序列化&反序列化安全。

1. 字节
2. 内存布局一致

方案：

用C++26反射来保证：

编译期获取：结构体有哪些成员，各成员size，成员距离结构体首地址的offset。来组成签名，编译期校验不同软件环境下的签名一致。

开源地址：

<https://github.com/ximicpp/XOffsetDatastructure>
```
