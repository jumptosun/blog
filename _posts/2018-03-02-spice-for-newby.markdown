---
layout: post
title:  "spice架构"
date:   2018-03-02 
categories: spice
---

本文是对官方的 [https://www.spice-space.org/static/docs/spice_for_newbies.pdf](https://www.spice-space.org/static/docs/spice_for_newbies.pdf)
概要翻译，请参照阅读。

## 2.3  Spice Client 
跨平台的用户终端接口。

### 2.3.2  Client class
为了保持平台通用性，平台独立的实现被放在了不同的文件夹内。
Application 是最重要的类型，用来控制客户显示，监控等等。   

#### 2.3.2.1  channels
所有channel的原始继承关系如下:
- RedPeer:  对socket 的包装， 提供了诸如connect, disconnect, close, send, receive,
  socket swapping for migration 等基础操作。 定义了InMessages，OutMessage，
  (混合)CompoundInMessage。

- RedChannelBase: 继承了RedPeer， 提供了spice-protocol所描述的channel链接过程。        

- RedChannel: 继承自RedChannelBase，所有实际Channel的都继承自RedChannel。
  处理发送和接受。RedChannel线程是事件驱动，例如：发送或者退出。
  channel socket （事件）可以作为消息发送或接收的触发来源。

ChannelFactory 使用了工厂模式 来创建每一个Channel。

## 2.4 spice server
Spice server 是在libspice中实现的， 一个虚拟设备接口(Virtual Device Interface VDI)    
插件库。VDI 对外提供了统一的软件借口用来交互。spice server 一边通过 spice 协议和远    
程客户端交互，一边通过与VDI主机应用程序交互(例:QEMU)。    
    
为了远程显示，服务器维护一个命令队列和树用来管理当前对象依赖和隐藏。QXL命令被处理    
并转换为Spice协议命令发送给客户端。

spice 总是会将表现显示任务传送给客户端，以充分利用硬件加速。通过软解码或 GPU 解码
在主机端做最终显示。

spice server 保留构成当前图像的画图命令。只有当它被其他命令完全覆盖并且没有依赖它时，    
才会释放该命令, 或者刷新帧缓冲区的时候(render the command  to  the  frame  buffer)。    
两种情况会触发 render the command  to  the  frame  buffer
1.  资源耗尽的时候
2.  客户机读帧缓冲区。

###  2.4.1
服务器通过 Channel 与客户端进行通信。 每个类型的 Channel 都专用于一种特定的
数据类型。 每个通道有一个专用的 TCP 套接字，即可以使用加密（如SSL）也可以不使用加密。     
服务器通道类似于客户端通道包括：主，输入，显示，光标，播放和录制（有关更多信息，
请参阅频道2.3.2.1）

####  2.4.1.1    Red Server ( reds.c)
服务器监听客户端连接,接受并与之通信。
Reds 负责：

- Channel
  - 拥有和管理 channel（注册，注销，关闭）
  - 通知客户那些 channel 处于激活状态，以便客户端可以创建它们 
  - 处理 main 和 input channel 
  - 建立连接 (包括 main 和 其他)
  - 操作 socket 和 连接管理
  - 处理 SSL 和 ticketing
- VDI 接口的添加和移除
- 迁移过程协调 
- 处理用户命令
- 和客户代理通信
- 统计

#### 2.4.1.2    Graphic subsystem

Red Worker (red_worker.c)
spice server 为每个 QXL 设备启动不同的 red worker
每个 red worker 负责:

- 处理 QXL 设备发出的命令 (例如: 绘画，更新，光标)
- 处理从 dispatcher 发来的消息
- channel 管道 和 pipe items
- 显示 和 cursor channels ( display and cursor? channels)
- 图像压缩
- 视频流 - 识别，编码和流创建
- 缓存 - 客户端共享pixmap缓存，光标缓存，调色板缓存
- 动态图像的优化 - 使用树，容器，阴影，独立的区域，不透明的items
- Cario 或 OpenGL（pbuf和pixmap）渲染器,  画图 - 画板, 曲面等
- 命令队列处理

Red  Dispatcher (red_dispatcher.c)
-  调度程序 dispatcher，每个QXL设备实例一个
-  封装 worker 的接口，对 QXL 设备和 reds 提供一致的接口
-  为每个 QXL 设备实例 启动单独的 worker，并创建一个工作线程
- 为每个 worker 分配 socketpair channel
- QXL 使用 QXLWorker  讲 device call 转化为 独立异步消息， 并通过 red pipe
   传输。这样可以保持每部分的处理逻辑独立性。
- Reds 使用red_dispatcher.h中定义的接口来实现功能:初始化，图像压缩更改，
   视频流状态更改，鼠标模式设置渲染器添加。


## 3.2  Hardware Acceleration 硬件加速

2D 使用 Cairo库
3D linux OpenGl, windows GDI

## 3.3  Image Compression 图像压缩
三种算法 Quic, LZ, GLZ。 其中 GLZ 对连续图像压缩的更好。

##  3.4 Video Compression 视频压缩
对于静态图像  3.3 spice 使用无损压缩，为了避免重要显示对象的失真。
但是因为
1. 视频非常的占带宽，并且连续图像包括了很多冗余的信息。
2. 其内容并不是非常的重要，spice 对其采用有损压缩。
spice server 通过识别以较高速率更新的区域，试探性的地识别视频区域。
使用 有损 mpeg 进行压缩。 在公网环境下减少了阻塞的可能。
但是这种不确定的尝试可能导致画质降低，即识别不准确。
