# Chisel

[Chisel](https://chisel-lang.org/) 是一种嵌入 Scala 的开源硬件描述语言。它支持使用高度参数化生成器的高级硬件设计，并支持 Rocket Chip 和 BOOM 等。

编写完 Chisel 后，Chisel 源代码“变成” Verilog 之前还有多个步骤。首先是编译步骤。如果 Chisel 被认为是 Scala 中的一个库，那么正在构建的这些类只是调用 Chisel 函数的 Scala 类。因此，您在编译 Scala/Chisel 文件时遇到的任何错误都是由于您违反了类型系统、语法混乱等原因造成的。编译完成后，就开始描述（elaboration）。 Chisel 生成器使用传递给它的模块和配置类开始描述。这是使用给定参数调用 Chisel “库函数”的地方，并且 Chisel 尝试基于 Chisel 代码构建电路。如果此处发生运行时错误，Chisel 表示由于代码与 Chisel“库”之间的“冲突”，它无法“构建”您的电路。但是，如果通过，生成器的输出将为您提供 FIRRTL 文件和其他杂项抵押品！有关如何将 FIRRTL 文件获取到 Verilog 的更多信息，请参阅 [FIRRTL](https://chipyard.readthedocs.io/en/stable/Tools/FIRRTL.html#firrtl)。

有关如何使用 Chisel 和入门的交互式教程，请访问 [Chisel Bootcamp](https://github.com/freechipsproject/chisel-bootcamp)。否则，对于 Chisel 相关的所有内容，包括 API 文档、新闻等，请访问他们的 [webiste](https://chisel-lang.org/)。