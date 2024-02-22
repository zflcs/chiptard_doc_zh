# NVDLA

[NVDLA](http://nvdla.org/) 是 NVIDIA 开发的开源深度学习加速器。 NVDLA 作为 TileLink 外设连接，因此可以用作 Rocket Chip SoC 生成器内的组件。加速器本身公开一个 AXI 内存接口（如果使用“大”配置则为两个）、一个控制接口和一条中断线。在 Chipyard 中使用加速器的主要方法是使用 [NVDLA SW repository](https://github.com/ucb-bar/nvdla-sw)，该存储库已移植到 FireSim Linux 上。但是，您也可以在裸机模拟中使用加速器（请参阅 `tests/nvdla.c`）。

有关硬件架构和软件的更多信息，请访问他们的 [website](http://nvdla.org/)。

## NVDLA Software with FireMarshal

位于 `software/nvdla-workload` 的是基于 FireMarshal 的工作负载，用于使用正确的 NVDLA 驱动程序启动 Linux。有关如何运行模拟的更多信息，请参阅 `README.md`。