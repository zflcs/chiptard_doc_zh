# IceNet

IceNet 是与网络相关的 Chisel 设计库。 IceNet 的主要组件是 IceNIC，它是一个网络接口控制器，主要用于 [FireSim](https://fires.im/) 中进行多节点网络仿真。 IceNet 的微架构图如下所示。

![IceNet](../assets/icenet.png)

NIC 有四个基本部分：[Controller](https://chipyard.readthedocs.io/en/stable/Generators/IceNet.html#controller)，它从 CPU 接收请求以及向 CPU 发送响应；[Send Path](https://chipyard.readthedocs.io/en/stable/Generators/IceNet.html#send-path)，从内存读取数据并将其发送到网络；[Receive Path](https://chipyard.readthedocs.io/en/stable/Generators/IceNet.html#receive-path)，从网络接收数据并将其写入内存；以及可选的 [Pause Handler](https://chipyard.readthedocs.io/en/stable/Generators/IceNet.html#pause-handler)，它生成以太网暂停帧以进行流量控制。

## Controller

控制器向 CPU 公开一组 MMIO 寄存器。设备驱动程序写入寄存器以请求发送数据包或提供内存位置以写入接收到的数据。发送请求或数据包接收完成后，控制器向 CPU 发送中断，CPU 通过读取另一个寄存器来清除完成情况。

## Send Path

发送路径从读取器开始，读取器接收来自控制器的请求并从内存中读取数据。

由于 TileLink 响应可能会无序返回，因此我们使用保留队列对响应重新排序，以便可以按正确的顺序发送数据包数据。

然后，数据包数据进入仲裁器，仲裁器可以仲裁对 NIC 和一个或多个“接入”接口之间的出站网络接口的访问，这些接口来自可能想要发送以太网数据包的其他硬件模块。默认情况下，接口中没有分接头，因此仲裁器仅传递保留缓冲区的输出。

## Receive Path

接收路径从数据包缓冲区开始，它缓冲来自网络的数据。如果缓冲区空间不足，则会以数据包粒度丢弃数据，以确保 NIC 不会传送不完整的数据包。

数据可以选择从数据包缓冲区进入网络分路器，网络分路器检查以太网标头并选择要通过一个或多个“分路器输出”接口从 NIC 重定向到外部模块的数据包。默认情况下，没有 tap out 接口，因此数据将直接进入写入器，写入器将数据写入内存，然后将完成发送到控制器。

## Pause Handler

IceNIC 可以配置为具有暂停处理程序，该处理程序位于发送和接收路径以及以太网接口之间。该模块跟踪接收数据包缓冲区的占用情况。如果它发现缓冲区已满，它将向网络发送 [Ethernet pause frame](https://en.wikipedia.org/wiki/Ethernet_flow_control#Pause_frame) 以阻止发送更多数据包。如果 NIC 收到以太网暂停帧，暂停处理程序将阻止从 NIC 发送。

## Linux Driver

[firesim-software](https://github.com/firesim/firesim-software) 提供的默认 Linux 配置包含 IceNet 驱动程序。如果您启动带有 IceNIC 的 FireSim 映像，驱动程序将自动检测该设备，并且您将能够在用户空间中使用完整的 Linux 网络堆栈。

## Configuration

要将 IceNIC 添加到您的设计中，请将 `HasPeripheryIceNIC` 添加到您的惰性模块中，并将 `HasPeripheryIceNICModuleImp` 添加到模块实现中。如果您对惰性模块和模块实现之间的区别感到困惑，请参阅 [Cake Pattern / Mixin](https://chipyard.readthedocs.io/en/stable/Chipyard-Basics/Configs-Parameters-Mixins.html#cake-pattern-mixin)。

然后将 `WithIceNIC` 配置片段添加到您的配置中。这将定义 `NICKey`，IceNIC 使用它来确定其参数。配置片段有两个参数。 `inBufFlits` 参数是输入数据包缓冲区可以容纳的 64 位 flits 的数量，`usePauser` 参数确定 NIC 是否有暂停处理程序。