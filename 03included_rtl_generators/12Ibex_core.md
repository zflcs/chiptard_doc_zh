# Ibex Core

[Ibex](https://github.com/lowRISC/ibex) 是一个用SystemVerilog 编写的可参数化的 RV32IMC 嵌入式核，目前由 [lowRISC](https://lowrisc.org/) 维护。 Ibex 核心包裹在 Ibex 瓦片中，因此可以与 Rocket Chip SoC 生成器一起使用。核心公开了自定义内存接口、中断端口和其他杂项。从图块内连接到 TileLink 总线和其他参数化信号的端口。

> **警告**：Ibex mtvec 寄存器是 256 字节对齐的。编写/运行测试时，请确保陷阱向量也是 256 字节对齐的。

> **警告**：Ibex 复位向量位于 BOOT_ADDR + 0x80。

虽然核心本身不是生成器，但我们公开了 Ibex 核心提供的相同参数化，以便所有支持的 Ibex 配置可用。

有关更多信息，请参阅 [GitHub repository for Ibex](https://github.com/lowRISC/ibex)。