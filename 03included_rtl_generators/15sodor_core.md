# Sodor Core

[Sodor](https://github.com/ucb-bar/riscv-sodor) 是 5 个简单的 RV32MI 核的集合，专为教育目的而设计。 Sodor 核心在生成过程中被包裹在瓦片中，因此它可以用作 Rocket Chip SoC 生成器中的组件。内核包含一个小型暂存存储器，程序通过 TileLink 从端口加载到其中，并且内核**不支持**外部存储器。

五个可用的内核及其相应的生成器配置是：

1. 1-stage（本质上是一个 ISA 模拟器）- `Sodor1StageConfig`。
2. 2-stage（在 Chisel 中演示i流水线）- `Sodor2StageConfig`。
3. 3 级（使用顺序内存；支持 Harvard (`Sodor3StageConfig`) 和Princeton (`Sodor3StageSinglePortConfig`) 版本）。
4. 5 级（可以在完全旁路或完全互锁之间切换）- `Sodor5StageConfig`。
5. 基于“总线”的微编码实现 - `SodorUCodeConfig`。

更多信息请阅读 [GitHub repository](https://github.com/ucb-bar/riscv-sodor)。