# Included RTL Generators

生成器可以被认为是一种通用的 RTL 设计，使用元编程和标准 RTL 的组合来编写。这种类型的元编程由 Chisel 硬件描述语言启用（请参阅 [Chisel](https://chipyard.readthedocs.io/en/stable/Tools/Chisel.html#chisel)）。标准 RTL 设计本质上只是来自生成器的设计的单个实例。然而，通过使用元编程和参数系统，生成器可以允许以自动化方式集成复杂的硬件设计。以下几页介绍了与 Chipyard 框架集成的生成器。

Chipyard 将生成器的源代码捆绑在 `generators/` 目录下。它每次都从源代码构建它们（尽管构建系统会缓存未更改的结果），因此在使用 Chipyard 构建时将自动使用对生成器本身的更改，并传播到软件仿真、FPGA 加速仿真和 VLSI 流程。