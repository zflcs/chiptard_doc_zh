# CVA6 Core

[CVA6](https://github.com/openhwgroup/cva6)（以前称为 Ariane）是一个 6 级顺序标量处理器内核，最初由 F.Zaruba 和 L.Benini 在苏黎世联邦理工学院开发。 CVA6 核包裹在 CVA6 块中，因此可以用作 Rocket Chip SoC 生成器中的组件。核心本身公开了 AXI 接口、中断端口和其他杂项。从图块内连接到 TileLink 总线和其他参数化信号的端口。

> **警告**：由于内核使用 AXI 接口连接到内存，因此强烈建议在单核设置中使用该内核（因为 AXI 是非一致性内存接口）。

虽然核心本身不是生成器，但我们公开了 CVA6 核心提供的相同参数化（即更改分支预测参数）。

> **警告**：该目标目前不支持 Verilator 仿真。请使用VCS。

有关更多信息，请参阅 [GitHub 仓库](https://github.com/openhwgroup/cva6)。 