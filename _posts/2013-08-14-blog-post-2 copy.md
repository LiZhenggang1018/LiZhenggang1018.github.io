## 分段式拍照

### 一阶段

相机apk -> 相机fwk -> 拍照HIDL -> 相机HAL -> 分段式信息采集存盘模块。

一阶段的变化（相比传统拍照）不大，主要是：

关键模块：**筛选、序列化、异步压缩存盘IO**

1. 参考如下信息，决策要做传统后处理还是分段式后处理：
   1. 相机UI界面拍照&视频模式、分辨率、帧率（视频还需考虑录制时长）等信息
   2. 手机ROM剩余空间 > XX% \* 总ROM空间
   3. 分段式已缓存文件独立占用总空间 < XX GB
2. （在走分段式处理的case下）去掉(所有的)后处理拍照算法模块，
3. 收集（仅后处理模块需要用到的）如下信息：
   1. 后处理算法需要的多张输入RAW图（一般7\~9张，含后处理需要的其他图像buffer）
   2. 随帧的metadata（场景检测、人脸检测、深度、3A&isp、产线等信息）
   3. 各类相机辅助器件gyro&ois&acc&flash&variableAperture的状态信息
   4. CaptureContext（描述一次拍照后处理的上下文的、后续可用于恢复整个离线后处理软件框架的一些状态信息/RuntimeInfo）（2、3、4下文统一用广义的metadata指代）

针对上述信息进行

1. **筛选**：仅挑选运行后处理软件+算法模块需要的状态信息最小子集。以metadata为例，会根据metadata的tagId、sectionName、tagName，选择仅后处理模块需要的字段信息而非全部metadata。
2. 深拷贝（深拷贝后，后续对拷贝内容执行序列化+存盘任务为异步执行，不阻塞当前相机运行）
3. **序列化**（序列化是指将数据结构或对象的状态转换为可以存储（例如，存入文件）或传输（例如，通过网络）的格式的过程，而反序列化则是从这种格式中恢复数据结构和对象的过程）为标准的TLV（Tag（标签）、Length（长度）和 Value（值））格式。
4. 存盘：序列化的结果（按照约定的数据结构排布）保存在（芯片组件下的，如data/vendor/camera/.../）文件系统中，其中：
   1. 存盘文件名仅留uuid(e.g: 20251203101010101010...)无后缀名，防止被手机清理助手等软件扫描到
   2. RAW图（含用户隐私图像信息）最终落盘位置（可能需）考虑隐私保护技术方案（HarmonyOS下存盘路径修改为专用信息加密文件路径）
5. 绑定能表征本次拍照任务的uuid/唯一用户id（如拍照时刻的时间戳精确到ms单位）。
6. 将本次拍照状态（是否为二阶段拍照）及本次uuid上报媒体库，媒体库保存待处理的uuid队列信息。
7. 将一阶段生成的低质量图（zsl选取的单张raw图（一般为传统后处理算法选帧需求中的参考帧），执行raw2yuv后将yuv送上层jpeg/heif编码）送图库存档。

其中：

1. **序列化**：

C++中对象序列化分为2种（本文仅讨论分段式技术所涉及的，序列化&反序列化发生在：同一软件操作系统+硬件平台、同一编程语言版本+编译器+ABI、同一进程内、同一数据结构体版本（不考虑新版本代码读取旧版本数据））：

1. 如仅涉及C++ **POD**（PlainOldData，等价于标准内存布局std::is\_standard\_layout\_v<T> && 可平凡拷贝std::is\_trivially\_copyable\_v<T>）类型的数据序列化较简单，可不依赖三方序列化库，直接(void\*首地址 +sizeof(T))逐字节拷贝/memcpy\_s+写入；

(注：这里未考虑T需满足更严格的std::has\_unique\_object\_representations\_v<T>/T必须具有唯一的对象/内存表示，因为我们的序列化&反序列化结果并不会跨平台、跨编译器或跨运行环境序列化，故不要求序列化后的数据/字节流具有一致性，也不关心C++结构体中填充位/PaddingBits的不确定性)

1. （更常见的）是**非POD**类型数据。需额外通过序列化组件执行序列化。

序列化组件可以通过自行编写C++序列化框架，其实现为针对下面（列举常用的）数据类型判断对应type\_traits并实现该类型序列化功能：

* 1. POD基础类型(或更准确的说仅需满足可平凡拷贝/std::is\_trivially\_copyable\_v<T>)
  2. C++标准库中的std::string，std::pair，std::tuple，STL容器（e.g. std::vector\\std::map\\std::array\\ std::deque\\ std::set, std::stack等常用序列容器+关联容器）
  3. std::shared\_ptr，std::unique\_ptr，android::sp等各类（强引用类型，不含裸指针及std::weak\_ptr等弱引用类型指针）智能指针（以及是否根据RTTI从基类指针中选取真正的运行时派生类型）
  4. std::variant，std::optional等类型
  5. 上述类型的继承&组合形式

也可以引用业界成熟三方库，包括：

* 1. google的protobuf ([https://github.com/protocolbuffers/protobuf](https://github.com/protocolbuffers/protobuf) )
  2. cap'n proto ([https://capnproto.org/](https://capnproto.org/) )
  3. cereal库（[https://github.com/USCiLab/cereal](https://github.com/USCiLab/cereal) C++11 header\_only）——简单轻量，**推荐**。
  4. nolman\_json等方案([https://github.com/nlohmann/json](https://github.com/nlohmann/json) nolman\_json部分可参考： [基于nlohmann\_json开源库的json序列化&反序列化](https://zhuanlan.zhihu.com/p/1903040263120721863) ）。

1. **文件IO**：
   1. 文件的写入如果采用Linux的write() / fwrite()等（同步阻塞的系统调用）方法，则默认是不可并发IO的。
   2. 可考虑使用io\_uring(Linux 内核在 5.1 版本引入的一种通过减少系统调用、支持零拷贝和内核轮询等机制实现高性能异步并发的 I/O 框架，可使用封装的**liburing**辅助库调用**io\_uring**)进行并发IO加速。（参考：[异步文件io框架io\_uring-学习笔记](https://zhuanlan.zhihu.com/p/1903138053830207348)）
2. **文件压缩**：文件写入前可以考虑压缩以节省硬盘空间。针对非图像的C++对象数据，考虑**zlib库**压缩，（参考：[https://android.googlesource.com/platform/external/zlib.git/+/master/zlib.h](https://link.zhihu.com/?target=https%3A//android.googlesource.com/platform/external/zlib.git/%2B/master/zlib.h)）（实测zlib压缩在分段式视频一阶段引入的功耗开销较大，最终（可以考虑）去掉该模块）。
