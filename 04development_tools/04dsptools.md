# Dsptools

[Dsptools](https://github.com/ucb-bar/dsptools/) 是一个用于编写自定义信号处理硬件的 Chisel 库。此外，dsptools 对于将自定义信号处理硬件集成到 SoC（尤其是基于 Rocket 的 SoC）非常有用。

特征：

1. 复数类型。
2. 用于编写多态硬件生成器的类型类 * 例如，编写一个适用于实数或复数输入的 FIR 滤波器生成器。
3. 针对定点和浮点类型的 Chisel 测试仪的扩展。
4. AXI4-Stream 的 diplomatic 实现。
5. 使用 chisel-testers 验证 APB、AXI-4 和 TileLink 接口的模型。
6. DSP 构建块。
