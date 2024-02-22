# Customization

这些指南将引导您完成片上系统的定制：

1. 使用现有的 Chipyard 生成器和配置系统构建异构片上系统。
2. 使用 Constellation 构建具有基于 NoC（片上网络）互连的 SoC。
3. 如何将自定义 Chisel 源包含在 Chipyard 构建系统中。
4. 添加自定义核心。
5. 将自定义 RoCC 加速器添加到现有 Chipyard 核心（BOOM 或 Rocket）。
6. 通过 Tilelink 或 AXI4 将自定义 MMIO 小部件添加到 Chipyard 内存系统，并具有自定义顶级 IO。
7. 添加基于自定义 Dsptools 的块作为 MMIO 小部件。
8. 使用 Keys、Traits 和 Configs 来参数化设计的标准实践。
9. 自定义内存层次结构。
10. 连接充当 TileLink master 的小部件。
11. 将定制黑盒 Verilog 添加到 Chipyard 设计中。

我们还提供以下信息：

1. Chipyard SoC 的启动过程。
2. Chipyard 中使用的 FIRRTL 转换示例及其指定位置。

我们建议按顺序阅读所有这些页面。