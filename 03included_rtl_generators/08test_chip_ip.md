# Test Chip IP

Chipyard 包含一个测试芯片 IP 库，该库提供了设计 SoC 时可能有用的各种硬件小部件。其中包括 [SimTSI](https://chipyard.readthedocs.io/en/stable/Generators/TestChipIP.html#simtsi)、[Block Device Controller](https://chipyard.readthedocs.io/en/stable/Generators/TestChipIP.html#block-device-controller)、[TileLink SERDES](https://chipyard.readthedocs.io/en/stable/Generators/TestChipIP.html#tilelink-serdes)、[TileLink Switcher](https://chipyard.readthedocs.io/en/stable/Generators/TestChipIP.html#tilelink-switcher)、[TileLink Ring Network](https://chipyard.readthedocs.io/en/stable/Generators/TestChipIP.html#tilelink-ring-network) 和 [UART Adapter](https://chipyard.readthedocs.io/en/stable/Generators/TestChipIP.html#uart-adapter)。

## SimTSI

tethered test chips 使用 SimTSI 和 TSIToTileLink 与主机处理器进行通信。在主机 CPU 上运行的 RISC-V 前端服务器实例可以向 TSIToTileLink 发送命令，以从内存系统读取和写入数据。前端服务器使用此功能将测试程序加载到内存中并轮询程序是否完成。有关这方面的更多信息可以在 [Chipyard Boot Process](https://chipyard.readthedocs.io/en/stable/Customization/Boot-Process.html#chipyard-boot-process) 中找到。

## Block Device Controller

块设备控制器为辅助存储提供通用接口。该设备主要在 FireSim 中用于与块设备软件仿真模型连接。[firesim-software](https://github.com/firesim/firesim-software) 中的默认 Linux 配置。

要将块设备添加到您的设计中，请将 `WithBlockDevice` 配置片段添加到您的配置中。

## TileLink SERDES

测试芯片 IP 库中的 TileLink SERDES 允许对 TileLink 内存请求进行序列化，以便可以通过串行链路将它们带出芯片。五个 TileLink 通道复用在两个 SERDES 通道上，每个方向一个。

该库提供了三种不同的变体，`TLSerdes` 向芯片公开管理器接口，在其出站链路上隧道 A、C 和 E 通道，在其入站链路上隧道 B 和 D 通道。 `TLDesser` 向芯片公开客户端接口，在其入站链路上公开隧道 A、C 和 E，在其出站链路上公开隧道 B 和 D。最后，`TLSerdesser` 向芯片公开客户端和管理器接口，并且可以双向隧道传输所有通道。

有关如何使用 SERDES 类的示例，请查看 [the Test Chip IP unit test suite](https://github.com/ucb-bar/testchipip/blob/master/src/main/scala/Unittests.scala) 中的 `SerdesTest` 单元测试。

## TileLink Switcher

当芯片具有多个可能的内存接口并且您希望在启动时选择将内存请求映射到哪些通道时，请使用 TileLink switcher。它公开一个客户端节点、多个管理器节点和一个选择信号。根据选择信号的设置，来自客户端节点的请求将被定向到管理器节点之一。选择信号必须在发送任何 TileLink 消息之前设置，并在其余操作中保持稳定。一旦 TileLink 消息开始发送，更改选择信号是不安全的。

有关如何使用切换器的示例，请查看 [Test Chip IP unit tests](https://github.com/ucb-bar/testchipip/blob/master/src/main/scala/Unittests.scala) 中的 `SwitcherTest` 单元测试。

## TileLink Ring Network

TestChipIP 提供了一个 TLRingNetwork 生成器，它具有与 RocketChip 提供的 TLXbar 类似的接口，但内部使用环形网络而不是交叉开关。这对于具有非常宽的 TileLink 网络（许多内核和 L2 组）的芯片非常有用，这些网络可以牺牲横截面带宽来缓解布线拥塞。有关如何使用环形网络的文档可以在 [The System Bus](https://chipyard.readthedocs.io/en/stable/Customization/Memory-Hierarchy.html#the-system-bus) 中找到。实现本身可以在[这里](https://github.com/ucb-bar/testchipip/blob/master/src/main/scala/Ring.scala)找到，并且可以作为如何使用不同拓扑实现您自己的 TileLink 网络的示例。

## UART Adapter

UART 适配器是 TestHarness 中的一个设备，连接到 DUT 的 UART 端口以模拟通过 UART 的通信（例如，在 Linux 启动期间打印到 UART）。除了使用主机的 `stdin/stdout` 之外，它还能够在仿真期间使用 `+uartlog=<NAME_OF_FILE>` 将 UART 日志输出到特定文件。

默认情况下，通过添加 `WithUART` 和 `WithUARTAdapter` 配置，将此 UART 适配器添加到 Chipyard 内的所有系统中。

## SPI Flash Model

SPI 闪存模型是对简单 SPI 闪存设备进行建模的设备。目前仅支持单读、四读、单写和四写指令。存储器由使用 `+spiflash#=<NAME_OF_FILE>` 提供的文件支持，其中 `#` 是 SPI 闪存 ID（通常为 `0`）。

## Chip ID Pin

芯片 ID 引脚设置其所添加到的芯片的芯片 ID。这在多芯片配置中最有用。引脚值由线束绑定器中设置的芯片 ID 值驱动，默认可以通过 MMIO 在地址 `0x2000` 处读取芯片 ID 值。

可以使用 `testchipip.soc.WithChipIdPin` 配置将该引脚添加到系统中。引脚宽度和 MMIO 地址是可参数化的，可以通过将 `ChipIdPinParams` 作为参数传递给配置来设置。还可以使用 `testchipip.soc.WithChipIdPinWidth` 配置来设置宽度。