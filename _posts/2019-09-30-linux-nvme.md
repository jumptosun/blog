---
layout: post
title:  "linux ssd bug"
date:   2019-09-30
categories: hardware
---

# 1. bug 描述
安装了 manjaro 的 xps 9350，睡眠长时间放电后，无法启动系统。

现象：

- 电脑可以正常启动 bios, 可以进入 grub, 选择 manjaro 后无法找到安装硬盘。
- 外接 linux usb liveCD 启动，无法看见固态硬盘，linux 无法看见 ssd。
- win10 pe 进入可以看见硬盘，工具 diskgenius 无法拷贝 ext4 数据。

# 2. 原因
xps 硬盘可以设置三种连接方式:

- disable
- ACHI
- NVMe

简单来说就是 linux 不支持 NVMe 的方式来读取。
之所以 xps bios 未经设置而改变硬盘读取方式，猜测原因为电池耗尽 bios 恢复出场设置。

# 3. 总线知识

总线分类:   

- 数据总线（Data Bus）：在CPU与RAM之间来回传送需要处理或是需要储存的数据。
- 地址总线（Address Bus）：用来指定在RAM（Random Access Memory）之中储存的数据的地址。
- 控制总线（Control Bus）：将微处理器控制单元（Control Unit）的信号，传送到周边设备。
- 扩展总线（Expansion Bus）：外部设备和计算机主机进行数据通信的总线，例如ISA总线，PCI总线。
- 局部总线（Local Bus）：取代更高速数据传输的扩展总线

AHCI:  
本质是一种PCI类设备，在系统内存总线和串行ATA设备内部逻辑之间扮演一种通用接口的角色（即它在不同的操作系统和硬件中是通用的）。这类设备描述了一个含控制和状态区域、命令序列入口表的通用系统内存结构；每个命令表入口包含SATA设备编程信息，和一个指向（用于在设备和主机传输数据的）描述表的指针。

NVM Express（NVMe):  
或称非易失性内存主机控制器接口规范(Non-Volatile Memory express),是一个逻辑设备接口规范。他是与AHCI类似的、基于设备逻辑接口的总线传输协议规范（相当于通讯协议中的应用层），用于访问通过PCI-Express（PCIe）总线附加的非易失性内存介质，虽然理论上不一定要求 PCIe 总线协议。