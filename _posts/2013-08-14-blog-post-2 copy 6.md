其他问题

1. 上文涉及到2种时间戳：
   1. 视频编解码系统（MediaCodec）从视频中解码出的yuv图像帧显示时间戳/**PTS**（Presentation Time Stamp/显示时间戳，将压缩帧解码后得到的原始帧的显示时间，指示某一帧（视频帧或音频帧）应该在什么时候被呈现（显示或播放）给用户的时间值，以时间基/time\_base为单位）
   2. 传统相机HAL使用的（附在随帧raw或yuv的metadata上的，HAL通用的ANDROID\_SENSOR\_TIMESTAMP）**图像时间戳**（本质是相机驱动在图像sensor首行曝光结束readout开始/SoF时刻打上的壁钟时间戳，参考：[手机相机sensor曝光示意图](https://zhuanlan.zhihu.com/p/1905190906446979557)）

两者基准不一致，（取决于编解码系统自己的实现，可能）**相差了一个手机系统休眠时间**（该时间差Mono2BootTimeOffset等价于Posix系统中CLOCK\_MONOTONIC（去掉手机休眠时间的单调时钟）与CLOCK\_BOOTTIME（包含手机休眠时间的壁钟）两类时间戳之差，可多次采样求差加权得到），要做对齐（可允许一定误差，如abs(transformed\_pts - image\_sof\_timestamp) < 0.5 \* ANDROID\_SENSOR\_FRAME\_DURATION），不然解码一张yuv图像得到的PTS信息无法映射到该帧在一阶段的存盘文件信息的标识id（后者由HAL的帧图像时间戳指定）。

将一阶段序列化的每帧metadata信息如果随yuv帧一起编码到mp4文件的Metadata Track(metadata编码可采用独立元数据轨/**MetaTrack**（与yuv帧的时间戳精确匹配）)中（而不是单独存盘到文件系统中），可避免上述两类时间戳不匹配的问题（代价：（用户可感知的）mp4文件size变大）。

2、二阶段任务执行时，安卓系统可能**休眠**（休眠时整体软件体系被freeze/冻结了，虽然物理时间在流逝但代码会休眠住（此时软件现场环境上下文是保存完好的），直至解除休眠才开始恢复往下执行），解决方法：

1. 图库进程延时子服务在二阶段任务执行期间持续**持有安卓休眠锁WakeLock**（[https://developer.android.com/develop/background-work/background-tasks/awake/wakelock?hl=zh-cn](https://link.zhihu.com/?target=https%3A//developer.android.com/develop/background-work/background-tasks/awake/wakelock%3Fhl%3Dzh-cn)）阻止系统休眠（PARTIAL\_WAKE\_LOCK类型），等到任务执行结束再释放WakeLock准许系统休眠。
2. 在HAL分段式框架的代码中，如超时监控TimeoutMonitor模块（C++超时代码实现参考：[超时回调的一种C++实现](https://zhuanlan.zhihu.com/p/1930919514746360155) ）要监控一段(不包含休眠时间，只含类似有效cpu运行时间片段的)真实软件执行时间段 是否超时，C++代码中**要使用std::steady\_clock**（跟壁钟无关的，适用于度量有效时间间隔的，且度量间隔会自动减掉休眠时间的 单调时钟），而**不要使用std::system\_clock**(简单的壁钟时间)。（参考：[https://en.cppreference.com/w/cpp/chrono.html](https://link.zhihu.com/?target=https%3A//en.cppreference.com/w/cpp/chrono.html) ）

两个方案中如果做到了1（推荐，阻止休眠）也可以不用关注2。

3、一阶段将原始yuv编码进入80分.mp4，二阶段再从80.mp4中解码出yuv，此时：二阶段解码后得到的yuv相比与一阶段HAL写入的原始yuv有**效果回退**（对视频编解码了解不深，不太懂这个问题在工程维度是否有必要（能否肉眼感知的效果变化）/是否可以/如何能解决，可适当**提高编码Bitrate**缓解（优势：更多的细节、更少的压缩瑕疵（如模糊、色块、噪点）；劣势：mp4文件size变大。需权衡））。

4、序列化时可以合理**选择哪些数据被序列化**，一些全局的/持久的/不变的/已经写入硬件的数据可以不参与序列化（因为后续的二阶段本身也是运行在CameraDaemon进程中，传统实时相机一阶段能够访问的上述数据，在分段式二阶段也可访问到）。如：

1. 如器件\\相机\\软件系统不变属性值，e.g: constexpr常量。
2. metadata中的静态能力值StaticCapability
3. 模组EEPROM/OTP中烧录的相机标定数据

一些后处理业务用不到的子图像Buffer（一般相机中非常多的子图像buffer都会attach其智能指针到主图像sensor raw/yuv buffer上）也不应该序列化。

**5、编解码模块的位置**：（可能）**放在fwk模块**比放在hal模块更合理。

原因：后处理算法并不是都在HAL进程中，在fwk中也部署了一些（跟芯片组件能力无关的，相对简单些的、时序在HAL图像算法之后的）后处理算法（如滤镜、水印、偏安卓图形显示渲染方向、如RGBA域的算法）。如果在HAL就完成了mp4封包再传给fwk，那么fwk的后处理算法（在fwk不二次解码的情况下）就难以生效了。

综上：编解码上移到fwk中（可能）更合理。

6、如果编解码上移到fwk，那么fwk与HAL将不再通过文件句柄fd来跨进程交互数据，而是通过（轮转的）一系列的图像共享内存来交互数据，这种跨进程的多帧共享内存的生产者消费者模型就涉及到安卓通用的**BufferQueue机制**（一种基于状态机管理、Fence 同步和跨进程共享内存的在生产者（Producer）和消费者（Consumer）之间高效安全地传递图形缓冲区的安卓机制）。（BufferQueue参考：[BufferQueue 和 Gralloc | Android Open Source Project](https://link.zhihu.com/?target=https%3A//source.android.google.cn/docs/core/graphics/arch-bq-gralloc%3Fhl%3Dnb) ）

编解码上移fwk时分段式视频1层示意图

此时一种（可能的）分段式相关的fwk-hal交互实现为：

1. fwk通过avcodec的接口获取到它的Surface子对象（**ConsumerSurface**）,将对象的IBufferProducer取出来，放入BufferProducerSequenceable,通过HIDL接口传给HAL
2. HAL（离线图谱OfflineGraphCtrlServer）取出BufferProducerSequenceable中的IBufferProducer,并基于此创建一个Surface子对象（**ProducerSurface**）,生产者ProducerSurface先调用RequstBuffer()申请到一块buffer，
3. HAL将该buffer enque到阻塞队列**IveBufferBlockingQueue**中，分段式视频框架驱动后处理OfflineIps，从阻塞队列中取出buffer，依次运行各yuv域后处理算法，最后一个后处理算法组件执行结束后，通过注册的回调将buffer返回给HAL中OfflineGraphCtrlServer的Sink端。
4. OfflineGraphCtrlServer的Sink端调用FlushBuffer()将buffer返回给**SurfaceBufferQueue**。HAL的FlushBuffer()函数中会跨进程地触发fwk中注册的回调函数OnBufferAvailable()，通知消费者buffer已经被生产者填好数据了。
5. fwk的OnBufferAvailable()（注：avcodec中把这个函数封装成了avcodec自己的OnOutputBufferAvailable(),两者等价)中，将buffer编码封包进mp4，然后release该buffer。
