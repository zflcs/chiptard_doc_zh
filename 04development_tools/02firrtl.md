# FIRRTL

[FIRRTL](https://github.com/freechipsproject/firrtl) 是电路的中间表示。它由 Chisel 编译器发出，用于将 Chisel 源文件转换为另一种表示形式，例如 Verilog。无需赘述，FIRRTL 由 FIRRTL 编译器使用，FIRRTL 编译器通过一系列电路级转换传递电路。 FIRRTL 传递（转换）的一个示例是优化未使用的信号的传递。转换完成后，将发出 Verilog 文件并完成构建过程。

要了解如何在 Chipyard 中将 FIRRTL 转换为 Verilog，请访问 [Adding a Firrtl Transform](https://chipyard.readthedocs.io/en/stable/Customization/Firrtl-Transforms.html#firrtl-transforms)。

有关 FIRRTL 的更多信息，请访问他们的 [website](https://chisel-lang.org/firrtl/)。

