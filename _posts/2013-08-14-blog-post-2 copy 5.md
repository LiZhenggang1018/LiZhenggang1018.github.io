FFmpeg编解码整体流程示例-来源：《深入理解FFmpeg》刘歧 著

* 1. **解包/Demuxing**（从容器格式（如MP4）中分离出压缩的音频和视频流，获取图像信息如width、height、stride、size、format/yuv420\_semi\_planar/NV12、colorSpace、pts转换归一化后的帧时间戳，以及视频信息如视频时长&总帧数、编码格式、帧率、编码时间、数据轨道、比特率、编码时间、HDR信息等）
  2. **解码/Decoding**：将压缩的视频或音频数据（如H.264、HEVC）还原为原始数据（YUV）
  3. YUV图像处理优化/图像**后处理算法**（ai-deblur/ai-denoise 等yuv域多帧降噪、去模糊算法）
  4. **编码/Encoding**：原始视频数据（如YUV像素数据）通过编码器（Encoder）按照某种视频编码标准（如H.264、HEVC、AV1）进行压缩
  5. **封包/Muxing**：（含视频流、音频流（音频数据不经过编解码和算法，不做修改继承原有信息，直接透传给封包模块写入二阶段视频文件）、metadata及视频维测数据（一般追加在视频文件末尾，继承HDRVivid，继承原有维测信息）等）
  6. 最终效果优化后的mp4写入dest.mp4中（120.mp4），上报给APK，APK将120.mp4替换原始的80.mp4；
  7. 如果处理失败（失败次数超过最大处理次数），则保留原视频不动，fwk记录的PendingRequest队列中删除本次任务的uuid，同时通知底层HAL删除本次对应的序列化存盘文件。

编解码流程（安卓通过**mediacodec**服务的VNDK标准接口基于底层硬件做高效编解码，此处只是用跨平台通用的FFmpeg三方软件库做示例）可类似参考：[基于FFmpeg的视频编解码实战](https://zhuanlan.zhihu.com/p/1932035630709343496)。

1. 由于视频有大量连续帧序列，所以分段式视频的处理时间一般也较长（X s\~ X min）。视频的单次二阶段处理任务过程极大可能会被用户的操作（如亮屏）打断。需要设计软件框架能够实现分段式视频的**中断&恢复**：
2. **中断**：操作被打断/中断时能够快速（有时间性能要求地，X百ms内）**保存好现场/已处理结果**+**收尾/让出相机硬件资源**(如离线isp/后处理软件中一二阶段共用的、不支持多实例的资源类等，因为被中断后用户可能会立马使用相机)，
3. **恢复：**任务再次下发/恢复时 能够从上次中断时刻的处理帧往下**接续解码处理**mp4文件，而不是从头开始又从视频文件首帧开始处理（可通过将暂停前的已处理帧写入芯片组件路径下（HAL进程可直接访问）的data/vendor/camera/.../temp文件的形式实现（同样temp文件等（或者说所有分段式缓存文件）建议不要添加如.mp4等任何文件后缀名，以免被图库/各类应用文件系统扫描到））。当已知帧号/帧时间戳PTS时，从某一帧开始编解码在MediaCodec中是支持的。
4. 解码器解出来若干帧，送后处理YUV算法，然后新YUV图送回编码器编码进mp4，涉及到（同一个进程中多线程的）多个类似**生产者－消费者模型**的**图像帧/metadata/反序列化Buffer**的生产消费案例，涉及如下软件模块：
5. 线程安全的**阻塞队列**实现（参考：[阻塞双端队列的一种C++实现](https://zhuanlan.zhihu.com/p/1932043303915847799) ）
6. 基于消息队列的**线程池**实现
7. AndroidOS：基于std::thread、std::packaged\_task等实现；
8. HarmonyOS：基于FFRT/Function Flow Runtime Kit并发编程框架实现，参考：[https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/ffrt-overview](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/ffrt-overview) ）
9. 注：后续建议统一使用C++26中基于无栈协程的std::execution中提供的线程池（stdexec::static\_thread\_pool），简单易用且性能最优。

(参考[https://github.com/NVIDIA/stdexec/blob/main/include/exec/static\_thread\_pool.hpp](https://github.com/NVIDIA/stdexec/blob/main/include/exec/static_thread_pool.hpp) )

基于上述模块实现（HAL同一进程中）图像buffer的轮转机制：

1. 从内存池中申请帧内存（BlockingQueue<CameraBuffer>已提前申请好固定队列深度的CameraBuffer）
2. 填充解码图像内容（填充80.mp4的YUV图像内容）
3. 图像后处理(各类YUV后处理算法，如美肤，AiDeblur去震颤,AiDenoise降噪)
4. 编入编码器
5. 帧内存复用给下一帧或最终释放
6. 不断循环此过程，直至解包模块上报解包到EoS帧/最后一个待处理帧。
