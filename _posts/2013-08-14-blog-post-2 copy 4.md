## 分段式视频

分段式视频与分段式拍照共用一套分段式框架，流程相似，较大的不同点在于：

### 一阶段

分段式视频（相比拍照）在一阶段时的差异点：

1、图像数据是否序列化的选择：

1. **不需要存储、序列化YUV**图像数据，因为原始的YUV数据已经编码进入了图库中的原始（没有经过任何后处理算法的，效果80分的）mp4文件中。通过**编解码**，可以从mp4文件中获得（几乎）原始的YUV420数据。
2. 可能会/可选地存储、序列化若干张RAW图（大小\~MB），这块取决于视频的后处理算法需求：一些视频处理算法可能需要一些参考RAW图才能取得更好的降噪或者去模糊的效果，仅靠YUV图效果无法最优。但这里不可能为每个视频帧都保存RAW否则存储ROM占用/功耗（主要是一阶段视频录制对功耗极为敏感）开销太大，可能**每秒存一/几帧RAW**。
3. 但是需要存储每帧图像数据上的**随帧metadata**，视频帧数比较多时需要严格限制（只存必须存的，一般每帧存几十KB）将存取的随帧metadata数据大小。否则的话同样会导致存储ROM占用/功耗高。

同时，多个（以2min 30fps NonSensorHdr视频文件为例，含**3600帧yuv**）视频帧（相对应的一次分段式拍照只需要存**7\~9张输入RAW图**的metadata数据）的metadata数据一般是通过O\_APPEND/追加的模式持续写入同一文件，此模式下io\_uring需要特殊写法（liburing在O\_APPEND模式下写入时不需要指定偏移量，由内核自动追加）才能支持**O\_APPEND**模式做并发读写。

### 二阶段


分段式视频-1层设计图

分段式视频（相比拍照）在二阶段时的差异点（上图黄色标注模块）：

**重点模块：分段式HIDL接口+客户端&服务端，分段式任务引擎IVEE，编解码+反序列化，离线后处理OfflineIps软件框架，暂停&恢复机制，实时渲染视频帧。**

1、Framework通过HIDL接口将图库打开的user.mp4的文件句柄fd（read\_only属性）（还包含一个dest.mp4文件句柄（read write get\_attr属性）用于供HAL写入最终生成mp4文件）下发给HAL的分段式框架（需要图库进程来打开mp4文件，因为：从设计上来说由HAL进程来打开是不合理的，HAL不应该有（访问用户媒体资源/user.mp4文件）这一类权限，这种Selinux权限（触发最严格的neverallow规则级别安全约束）也不会被评审通过）。涉及如下点：

1. 该文件句柄的跨进程传递涉及到[**native\_handle\_t**](https://zhida.zhihu.com/search?content_id=260819415&content_type=Article&match_order=1&q=native_handle_t&zhida_source=entity)( 即该fd并不能简单地将int值赋值到HIDL接口中传递，不同进程中的相同的int数值对应的fd并不指向同一个底层文件。必须通过安卓的**native\_handle\_t**（实现为IPC底层驱动自动dup()该fd到新进程）来包装传递），包括：
2. IPC通信中的文件句柄的创建
3. 跨进程传递
4. 解析（native\_handle\_t的data[]数组延伸字段）
5. 校验（句柄有效性，文件size一致性）
6. 克隆（fd的dup()操作，以及dup后的新旧fd共享同一个offset，部分场景需考虑lseek(fd, 0, SEEK\_SET)重置offset到文件头）
7. 异步保活（fd通过一个同步HIDL接口下发，通过一个异步HIDL接口返回）
8. 资源释放

参考文章 [【native\_handle&fd】跨进程通信中的文件句柄方案](https://zhuanlan.zhihu.com/p/1903121571981271897) )

1. **Selinux权限**的配置。

（.te等配置文件）不进行配置的话，HAL无法有效地访问这个句柄，Selinux的设计初衷就是为了拦截一些可能不合理的（主体对客体的）跨进程资源。（在分段式方案中）包括：

1. 文件句柄
2. FUSE文件系统create read open write search getattr...
3. 共享内存
4. 休眠锁WakeLock
5. 驱动设备节点：如壳温、cpu频点、android props

Selinux参考 ：[https://source.android.com/docs/security/features/selinux?hl=zh-cn](https://link.zhihu.com/?target=https%3A//source.android.com/docs/security/features/selinux%3Fhl%3Dzh-cn)

1. 考虑到该fd在全流程中会被（跨进程、异步回调式地）使用，还需考虑实现**fd的RAII**资源管理类FdWrapper/UniqueFd（参考：[https://libcamera.org/api-html/classlibcamera\_1\_1UniqueFD.html](https://link.zhihu.com/?target=https%3A//libcamera.org/api-html/unique__fd_8h_source.html) ）来确保（根因：典型资源类对象fd的原始代码是用C写的没有C++ RAII机制）：在任何代码分支执行结束后，（客户端跟服务端两个进程中的）fd指向的资源最终都能成功释放无泄露（软件本质是fd被创建+dup()的次数，是否与fd（及dupped fd）的close()次数相等，大于则泄露，等于或小于则OK）。同时也要求hidl接口需明确其传递的native\_handle\_t中的fd的资源所有权（参考：[https://source.android.com/docs/core/architecture/hidl/types?hl=zh-cn#handle\_t](https://link.zhihu.com/?target=https%3A//source.android.com/docs/core/architecture/hidl/types%3Fhl%3Dzh-cn%23handle_t) 《句柄和文件描述符所有权》 节描述）。同时可考虑在C++代码中打开类似asan的google fdsan（参考：[https://android.googlesource.com/platform/bionic/+/master/docs/fdsan.md](https://link.zhihu.com/?target=https%3A//android.googlesource.com/platform/bionic/%2B/master/docs/fdsan.md)）的运行时检查来捕获这一类的fd泄漏问题。
2. 分段式框架接收到mp4文件句柄后，涉及：
