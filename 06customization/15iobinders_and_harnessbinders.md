# IOBinders and HarnessBinders

在 Chipyard 中，我们使用特殊的 `Parameters` keys、`IOBinders` 和 `HarnessBinders` 来弥合数字系统 IO 和 TestHarness collateral 之间的差距。

## IOBinders

`IOBinder` 函数负责实例化 `ChipTop` 层中的 IO 单元和 IOPort。

`IOBinder` 通常使用 `OverrideIOBinder` 或 `ComposeIOBinder` 宏来定义。 `IOBinder` 由一个与具有给定 trait 的 `System` 匹配的函数组成，该函数生成 IO 端口和 IOCell，并返回生成的端口和单元的列表。

例如，对于任何可能具有 UART 端口(`HasPeripheryUARTModuleImp`)的系统，`WithUARTIOCells` IOBinder 将在 ChipTop 内生成端口（端口）以及具有适当类型和方向的 IOCell（`cells2d`）。此函数返回生成的端口列表和生成的 IOCell 列表。生成的端口列表被传递到 `HarnessBinder`，以便它们可以连接到 `TestHarness` 设备。

```Scala
class WithUARTIOCells extends OverrideIOBinder({
  (system: HasPeripheryUARTModuleImp) => {
    val (ports: Seq[UARTPort], cells2d) = system.uart.zipWithIndex.map({ case (u, i) =>
      val (port, ios) = IOCell.generateIOFromSignal(u, s"uart_${i}", system.p(IOCellKey), abstractResetAsAsync = true)
      val where = PBUS // TODO fix
      val bus = system.outer.asInstanceOf[HasTileLinkLocations].locateTLBusWrapper(where)
      val freqMHz = bus.dtsFrequency.get / 1000000
      (UARTPort(() => port, i, freqMHz.toInt), ios)
    }).unzip
    (ports, cells2d.flatten)
  }
})
```

## HarnessBinders

`HarnessBinder` 函数确定将哪些模块绑定到 `TestHarness` 中 `ChipTop` 的 IO。 `HarnessBinder` 接口旨在在各种仿真/实现模式中重复使用，从而实现目标设计与仿真和测试问题的解耦。

1. 对于 SW RTL 或 GL 仿真，默认的 `HarnessBinder` 集会实例化各种设备（例如外部存储器或 UART）的软件仿真模型，并将这些模型连接到 `ChipTop` 的 IO。
2. 对于 FireSim 仿真，FireSim 特定的 `HarnessBinders` 会实例化 `Bridges`，这有助于在仿真芯片的 IO 上进行周期精确的仿真。有关更多详细信息，请参阅 FireSim 文档。
3. 将来，Chipyard FPGA 原型设计流程可能会使用 `HarnessBinders` 将 `ChipTop` IO 连接到 FPGA harness 中的其他设备或 IO。

与 `IOBinder` `一样，HarnessBinder` 也是使用宏（`OverrideHarnessBinder`、`ComposeHarnessBinder`）来定义的，并匹配具有给定 trait 的 `System`。但是，`HarnessBinder` 还会传递对 `TestHarness` 的引用（`th：HasHarnessSignalReferences`）以及相应 `IOBinder` 生成的端口列表（`ports: Seq[Data]`）。

例如，`WithUARTAdapter` 会将 UART SW 显示适配器连接到前面描述的 `WithUARTIOCells` 生成的端口（如果这些端口存在）。

```Scala
class WithUARTAdapter extends HarnessBinder({
  case (th: HasHarnessInstantiators, port: UARTPort, chipId: Int) => {
    val div = (th.getHarnessBinderClockFreqMHz.toDouble * 1000000 / port.io.c.initBaudRate.toDouble).toInt
    UARTAdapter.connect(Seq(port.io), div, false)
  }
})
```

`IOBinder` 和 `HarnessBinder` 系统旨在实现目标设计和仿真系统之间的关注点解耦。

对于给定的一组芯片 IO，可能不仅有多个仿真平台（可以说是“harnesses”），而且还可能有多种仿真策略。例如，将支持 AXI4 内存端口连接到精确 DRAM 模型 (`SimDRAM`) 还是简单模拟内存模型 (`SimAXIMem`) 的选择在 `HarnessBinders` 中被隔离，并且不会影响目标 RTL 生成。

类似地，对于给定的仿真平台和策略，可能有多种生成芯片IO的策略。该目标设计配置在 `IOBinder` 中被隔离。