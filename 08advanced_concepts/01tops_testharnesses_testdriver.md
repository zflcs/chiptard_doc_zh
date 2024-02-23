# Tops, Test-Harnesses, and the Test-Driver

Chipyard SoC 中三个最高层次结构是 `ChipTop` (DUT)`、TestHarness` 和 `TestDriver`。 `ChipTop` 和 `TestHarness` 均由 Chisel 生成器发出。`TestDriver` 作为我们的测试平台，是 Rocket Chip 中的 Verilog 文件。

## ChipTop/DUT

`ChipTop` 是实例化 `System` 子模块的顶级模块，通常是具体类 DigitalTop 的实例。绝大多数设计都位于 `System` 中。`ChipTop` 层内存在的其他组件通常是 IO 单元、时钟接收器和多路复用器、复位同步器以及需要存在于 `System` 外部的其他模拟 IP。`IOBinder` 负责实例化与 `System` IO 相对应的 `ChipTop` IO 的 IO 单元。`HarnessBinder` 负责实例化连接到 `ChipTop` 端口的测试工具。大多数类型的设备和测试工具都可以使用自定义 `IOBinder` 和 `HarnessBinder` 进行实例化。

## Custom ChipTops

默认标准 `ChipTop` 为 `IOBinder` 提供了一个最小的准系统模板，以围绕 `DigitalTop` 特征生成 IOCell。对于流片、集成模拟 IP 或其他非标准用例，Chipyard 支持使用 `BuildTop` key 指定自定义 `ChipTop`。 [generators/chipyard/src/main/scala/example/CustomChipTop.scala](https://github.com/ucb-bar/chipyard/blob/main/generators/chipyard/src/main/scala/example/CustomChipTop.scala) 中提供了使用非标准 IOCell 的自定义 ChipTop 示例。

## System/DigitalTop

Rocket Chip SoC 的系统模块是通过 cake pattern 组成的。具体来说，`DigitalTop` 扩展了一个 `System`，`System` 又扩展了一个 `Subsystem`，而 `Subsystem` 又扩展了一个 `BaseSubsystem`。

### BaseSubsystem

`BaseSubsystem` 定义在 `generators/rocketchip/src/main/scala/subsystem/BaseSubsystem.scala` 中。查看 `BaseSubsystem` 抽象类，我们看到该类实例化了顶层总线（frontbus、systembus、peripheralbus 等），但没有指定拓扑。我们还看到此类定义了几个 `ElaborationArtefacts`，即 Chisel 描述后发出的文件（例如设备树字符串和 diplomacy 图可视化 GraphML 文件）。

### Subsystem

查看 [generators/chipyard/src/main/scala/Subsystem.scala](https://github.com/ucb-bar/chipyard/blob/main/generators/chipyard/src/main/scala/Subsystem.scala)，我们可以看到 Chipyard 的 `Subsystem` 如何扩展 `BaseSubsystem` 抽象类。`Subsystem` 混合了 `HasBoomAndRocketTiles` 特征，根据指定的参数定义和实例化 BOOM 或 Rocket tile。我们还为这里的每个 tile 连接一些基本 IO，特别是 hartids 和 reset vector。

### System

`generators/chipyard/src/main/scala/System.scala` 完成了 `System` 的定义。

1. `HasHierarchicalBusTopology` 在 Rocket Chip 中定义，指定顶层总线之间的连接。
2. `HasAsyncExtInterrupts` 和 `HasExtInterruptsModuleImp` 添加用于外部中断的 IO，并将它们适当地连接到 tile。
3. `CanHave...AXI4Port` 添加各种 Master 和 Slave AXI4 端口，添加 TL 到 AXI4 转换器，并将它们连接到适当的总线。
4. `HasPeripheryBootROM` 添加 BootROM 设备。

### Tops

然后，SoC Top 使用自定义组件的特征扩展 `System` 类。在 Chipyard 中，这包括添加 NIC、UART 和 GPIO 以及为启动方法设置硬件等内容。有关这些启动方法的更多信息，请参阅 [Communicating with the DUT](https://chipyard.readthedocs.io/en/stable/Advanced-Concepts/Chip-Communication.html#communicating-with-the-dut)。

## TestHarness

`TestHarness` 和 Top 之间的连接是通过添加到 Top 的特征中定义的方法执行的。当从 `TestHarness` 调用这些方法时，它们可以实例化 harness 范围内的模块，然后将它们连接到 DUT。例如，在 `CanHaveMasterAXI4MemPortModuleImp` 特征中定义的 `connectSimAXIMem` 方法，当从 `TestHarness` 调用时，将实例化 `SimAXIMems` 并将它们连接到顶部的正确 IO。

虽然这种附加到顶部 IO 的迂回方式可能看起来不必要地复杂，但它允许设计者将自定义特征组合在一起，而不必担心任何特定特征的实现细节。

## TestDriver

`TestDriver` 定义在 `generators/rocketchip/src/main/resources/vsrc/TestDriver.v` 中。该 Verilog 文件通过实例化 `TestHarness`、驱动时钟和复位信号以及解释成功输出来执行仿真。该文件使用 `TestHarness` 和 `Top` 生成的 Verilog 进行编译，以生成仿真器。