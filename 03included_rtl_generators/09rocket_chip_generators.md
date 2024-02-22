# Rocket-Chip Generators

Chipyard 包括几个由 [SiFive](https://www.sifive.com/) 开发的开源生成器，现在作为 Chips Alliance 的一部分公开维护。目前它们被组织在两个名为 `Rocket-chip-blocks` 和 `Rocket-chip-inclusive-cache` 的子模块中。

## Last-Level Cache Generator

`Rocket-chip-inclusive-cache` 包括最后一级缓存生成器。 Chipyard 框架使用最后一级缓存作为二级缓存。要使用此 L2 缓存，您应该将 `freechips.rocketchip.subsystem.WithInclusiveCache` 配置片段添加到您的 SoC 配置中。要了解有关配置此 L2 缓存的更多信息，请参阅 [Memory Hierarchy](https://chipyard.readthedocs.io/en/stable/Customization/Memory-Hierarchy.html#memory-hierarchy) 部分。

## Peripheral Devices Overview

`Rocket-chip-blocks` 包括多个外围设备生成器，例如 UART、SPI、PWM、JTAG、GPIO 等。

这些外围设备通常会影响 SoC 的内存映射及其顶层 IO。所有外设模块都带有一个不会相互冲突的默认内存地址，但如果需要在 SoC 中集成多个重复的模块，则需要为该设备显式指定适当的内存地址。

此外，如果设备需要顶级 IO，您将需要定义一个配置片段来更改 SoC 的顶级配置。添加顶级 IO 时，您还应该注意它是否与测试工具交互。

此示例实例化一个包含 GPIO 端口的顶层模块，然后将 GPIO 端口输入连接到 0 (`false.B`)。

最后，将相关配置片段添加到 SoC 配置中。例如：

```Scala
class GPIORocketConfig extends Config(
  new chipyard.config.WithGPIO ++                           // add GPIOs to the peripherybus
  new freechips.rocketchip.subsystem.WithNBigCores(1) ++
  new chipyard.config.AbstractConfig)
```

## General Purpose I/Os (GPIO) Device

GPIO 设备是 `rocket-chip-blocks` 提供的外围设备。每个通用 I/O 端口有 5 个 32 位配置寄存器、2 个控制引脚输入和输出值的 32 位数据寄存器以及 8 个用于信号电平和边沿触发的 32 位中断控制/状态寄存器。此外，所有 GPIO 都可以有两个 32 位复用功能选择寄存器。

### GPIO main features

1. 输出状态：推挽或开漏，带有可选的上拉/下拉电阻。
2. 从输出值寄存器 (GPIOx_OUTPUT_VAL) 或外设（复用功能输出）输出数据。
3. 每个 I/O 的 3 位驱动强度选择。
4. 输入状态：浮动、上拉或下拉。
5. 输入数据到输入值寄存器 (GPIOx_INPUT_VAL) 或外设（复用功能输入）。
6. 复用功能选择寄存器。
7. 用于快速输出反转的位反转寄存器（GPIOx_OUTPUT_XOR）。

### Including GPIO in the SoC

```Scala
class ExampleChipConfig extends Config(
  // ...

  // ==================================
  //   Set up Memory Devices
  // ==================================
  // ...

  // Peripheral section
  new chipyard.config.WithGPIO(address = 0x10010000, width = 32) ++

  // ...
)
```

## Universal Asynchronous Receiver/Transmitter (UART) Device

UART 设备是 `rocket-chip-blocks` 提供的外围设备。 UART 提供了一种与外部设备执行全双工数据交换的灵活方法。通过小数波特率发生器可以实现非常广泛的波特率。UART 外设不支持其他调制解调器控制信号或同步串行数据传输。

### UART main features

1. 全双工异步通信。
2. 波特率发生器系统。
3. 16× Rx 过采样，每比特 2/3 多数表决。
4. 两个内部 FIFO，用于通过可编程水印中断发送和接收数据。
5. 通用的可编程发送和接收波特率。
6. 可配置停止位（1 或 2 个停止位）。
7. 发送器和接收器的单独使能位。
8. 带标志的中断源。
9. 可配置的硬件流控制信号。

### Including UART in the SoC

```Scala
class ExampleChipConfig extends Config(
  // ...

  // ==================================
  //   Set up Memory Devices
  // ==================================
  // ...

  // Peripheral section
  new chipyard.config.WithUART(address = 0x10020000, baudrate = 115200) ++

  // ...
)
```

## Inter-Integrated Circuit (I2C) Interface Device

I2C 设备是 `rocket-chip-blocks` 提供的外围设备。 I2C（内部集成电路）总线接口处理与串行 I2C 总线的通信。它提供多主机功能，并控制所有 I2C 总线特定的排序、协议、仲裁和时序。它支持标准模式 (Sm)、快速模式 (Fm) 和快速模式增强版 (Fm+)。

### I2C main features

I2C 总线规范兼容性：

1. Slave 和 master 模式。
2. 多主控能力。
3. 标准模式（高达 100 kHz）。
4. 快速模式（高达 400 kHz）。
5. 增强型快速模式（高达 1 MHz）。
6. 7 位寻址模式。

### Including I2C in the SoC

```Scala
class ExampleChipConfig extends Config(
  // ...

  // ==================================
  //   Set up Memory Devices
  // ==================================
  // ...

  // Peripheral section
  new chipyard.config.WithI2C(address = 0x10040000) ++

  // ...
)
```

## Serial Peripheral Interface (SPI) Device

SPI 设备是 `rocket-chip-blocks` 提供的外围设备。 SPI 接口可用于与使用 SPI 协议的外部设备进行通信。

串行外设接口 (SPI) 协议支持与外部设备的半双工、全双工和单工同步串行通信。该接口可配置为主设备，在这种情况下，它向外部从设备提供通信时钟 (SCLK)。

### SPI main features

1. master 操作。
2. 全双工同步传输。
3. 4 至 16 位数据大小选择。
4. 主模式波特率预分频器高达 fPCLK/2。
5. 通过硬件或软件进行 NSS 管理。
6. 可编程时钟极性和相位。
7. 可编程数据顺序，具有 MSB 优先或 LSB 优先移位。
8. 具有中断功能的专用发送和接收标志。
9. 两个具有 DMA 功能的 32 位嵌入式 Rx 和 Tx FIFO。

### Including SPI in the SoC

```Scala
class ExampleChipConfig extends Config(
  // ...

  // ==================================
  //   Set up Memory Devices
  // ==================================
  // ...

  // Peripheral section
  new chipyard.config.WithSPI(address = 0x10031000) ++

  // ...
)
```
