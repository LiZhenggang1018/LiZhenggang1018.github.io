7、广义的分段式视频除了处理mp4文件，还可用于处理**动态照片/**[**LivePhoto**](https://zhida.zhihu.com/search?content_id=260819415&content_type=Article&match_order=1&q=LivePhoto&zhida_source=entity)文件(简单理解为一张**封面照片帧**（此时除了视频编解码，可能额外涉及 **jpeg/heif** 等（sdr/hdr）**图像编解码**，可通过三方库**libjpeg\_turbo**实现。参考：[jpeg&exif&heif-学习笔记](https://zhuanlan.zhihu.com/p/1903049446050203400)）+**几十个视频帧**的组合文件形式)（可能的LivePhoto二阶段后处理算法：LivePhoto视频帧**插帧**、LivePhoto视频帧**EIS防抖**）。

8、当分段式架构尚未完成对照片/视频文件的分段式处理时，如果用户此时进入图库浏览照片文件/播放视频文件时（用户拍完照片/视频立即进入图库查看/播放）：

1. 对于照片，由于单张照片的分段式处理时间较短（CPU提频绑核时约\~s），可以调用分段式（CPU提频到最高）完成单张照片处理，将回显帧替换原始低质量帧，并最终送显给用户，此时用户看到是后处理优化过的图片。
2. 对于视频，由于整个mp4文件的分段式处理完成耗时高（一般约束在视频时长的2倍以内），本文所示方案还无法支持处理后的视频帧及时送视频播放组件显示，此时用户看到的是未经后处理的视频帧。

一种（可能的）优化方案是采用**实时渲染视频帧**的分段式方案：

1. 将图库/媒体库的**视频播放组件接入**（编解码上移framework的）**分段式架构**（播放组件framework中注册+调用分段式HIDL服务），
2. 当播放视频时触发分段式视频处理：此时，由于用户播放视频时可能存在快进、慢放、跳转（Seek）等（打破正常的、顺序的解码和渲染流程）操作，对分段式后处理单帧/整体性能耗时就有严格约束了，也仅支持单帧YUV（而非多帧融合）的后处理算法）
3. 将分段式处理的帧实时送给播放组件送显，确保用户看到的是经过后处理的优化帧。

如何在实时渲染视频帧的分段式方案中，同时**兼顾用户播放体验+最终处理视频效果**（后者要求部署更复杂（多帧融合的、单帧计算量较大无法确保实时性的）的二阶段后处理算法），还需考虑更多设计细节。

9、视频封面帧yuv图像（一阶段录制时）需考虑运行后处理算法后再上报。

10、（如果需要）在**非安卓操作系统**（e.g: HarmonyOS）实现上述分段式框架，一些依赖安卓基础组件的模块需要与**安卓解耦**，采用对应操作系统下的自研组件，包括：

1. 类HIDL通信机制、Client&Service、Proxy&Stub、DeathRecepient （设计一套独立IPC通信机制、IDL(Interface Definition Language)语言接口&编译器、sequenceable数据类型，可参考：[https://gitee.com/openharmony/docs/blob/master/zh-cn/application-dev/IDL/idl-guidelines.md](https://gitee.com/openharmony/docs/blob/master/zh-cn/application-dev/IDL/idl-guidelines.md) ）
2. native\_handle\_t以及共享内存等支持IPC传输的组件/数据结构
3. MediaCodec编解码（参考：[https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/audio-video-codec](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/audio-video-codec) ）
4. android::sp android::wp android::RefBase侵入式智能指针等基础C++组件（android ::Thread Mutex Condition这些完可以通过std库中C++组件替代）
5. 不同的Selinux权限管控
6. 构建系统&脚本：android.mk -> build.gn (GN (Generate Ninja) 构建系统)

11、自研（非安卓）操作系统下，分段式框架（主要体现在应用侧）的优势有：

1. 自研OS中，三方APP的分享&访问照片或视频等（一阶段）分段式媒体资源时，可以做到全量拦截+先触发完成分段式处理（在安卓下，网盘等三方应用的上传分享目前还无法拦截，其余的大部分可以拦截），确保所有场景下用户通过**三方应用最终分享/上传的媒体文件经过分段式二阶段处理**（而不会将一阶段的低质量媒体文件上传）。
2. 自研OS支持将**分段式能力开放给三方应用**使用。

（ArkTS参考：

[https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/camera-deferred-capture](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/camera-deferred-capture)

C/C++参考：

[https://developer.huawei.com/consumer/cn/doc/best-practices/bpta-camera-shot2see](https://developer.huawei.com/consumer/cn/doc/best-practices/bpta-camera-shot2see) ）
