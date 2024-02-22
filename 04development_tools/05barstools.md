# Barstools

Barstools 是有用的 FIRRTL 转换和编译器的集合，可帮助构建过程。这些工具中包括 MacroCompiler（用于将 Chisel 内存构造映射到供应商 SRAM）、FIRRTL 转换（用于分离测试和顶级 SoC 文件）等。

## Mapping technology SRAMs (MacroCompiler)

如果您计划构建真正的芯片，您可能会计划使用一定量的静态随机存取存储器 (SRAM)。 SRAM 宏提供优于触发器阵列的存储密度，但代价是限制一个周期内可能发生的读取或写入事务的数量。与 Verilog 不同，这些类型的顺序存储器元素是 Chisel 和 FIRRTL（`SeqMem` 元素）中的一流原语。这使得 Chisel 设计能够包含顺序内存元素的抽象实例，而无需了解底层实现或处理技术。

现代 CAD 工具通常无法从高级 RTL 描述合成 SRAM。遗憾的是，这要求设计人员将 SRAM 实例化包含在源 RTL 中，从而消除了其工艺可移植性。在 Verilog 入口设计中，可以创建一个抽象层，允许新的处理技术在包装器模块中实现特定的顺序内存块。然而，这种方法可能脆弱且费力。

FIRRTL 编译器包含一个转换，用于替换称为 `ReplSeqMem` 的 `SeqMem` 原语。这只是将所有超过大小阈值的 `SeqMem` 实例转换为外部模块引用。外部模块引用是一种 FIRRTL 构造，它使设计能够引用模块，而无需描述其内容，仅描述其输入和输出。 FIRRTL 将唯一 SRAM 配置的列表输出到 `.conf` 文件，该文件用于映射技术 SRAM。如果没有这种转换，FIRRTL 会将所有 `SeqMem` 映射到具有等效行为的触发器阵列，这可能会导致难以布线的设计。

`.conf` 文件由名为 MacroCompiler 的工具使用，该工具是 [Barstools](https://chipyard.readthedocs.io/en/stable/Tools/Barstools.html#barstools) scala 包的一部分。 MacroCompiler 还会传递一个 `.mdf` 文件，该文件描述可用的技术 SRAM 列表或 SRAM 编译器的功能（如果代工厂提供）。通常，代工厂 SRAM 编译器将能够根据尺寸、纵横比等的一些要求生成一组不同的 SRAM 附属品（请参阅 [SRAM MDF Fields](https://chipyard.readthedocs.io/en/stable/Tools/Barstools.html#sram-mdf-fields)）。使用用户可自定义的成本函数，MacroCompiler 将选择最适合 `.conf` 文件中每个维度的 SRAM。这可能包括过度配置（例如，如果请求的 60x1024 不可用，则使用 64x1024 SRAM）或阵列。阵列可以在宽度和深度上完成，也可以解决掩蔽约束。例如，一个 128x2048 阵列可以由四个 64x1024 阵列组成，并用两个宏并行创建两个 128x1024 虚拟 SRAM，将它们组合复用以增加深度。如果此宏需要字节粒度写入屏蔽，但没有技术 SRAM 支持屏蔽，则该工具可能会选择在类似配置中使用 32 个 8x1024 阵列。您可能希望手动或通过脚本创建可用 SRAM 宏的缓存。用于创建 SRAM 宏的 JSON 的参考脚本位于 [asap7 technology library folder](https://github.com/ucb-bar/hammer/blob/8fd1486499b875d56f09b060f03a62775f0a6aa7/src/hammer-vlsi/technology/asap7/sram-cache-gen.py) 中。有关编写 `.mdf` 文件的信息，请查看 [MDF on github](https://github.com/ucb-bar/plsi-mdf) 以及 [SRAM MDF Fields](https://chipyard.readthedocs.io/en/stable/Tools/Barstools.html#sram-mdf-fields) 部分中的简要说明。

MacroCompiler 的输出是一个 Verilog 文件，其中包含将技术 SRAM 封装到 `.conf` 中指定接口名称中的模块。如果该技术支持 SRAM 编译器，则 MacroCompiler 还将发出 HammerIR，它可以传递给 Hammer 以运行编译器本身并生成设计资料。 SRAM 编译器的文档即将发布。

### MacroCompiler Options

MacroCompiler 接受许多命令行参数，这些参数会影响它将 `SeqMem` 映射到技术特定宏的方式。这个最高级别的选项 `--mode` 一般指定 MacroCompiler 应如何将输入 `SeqMem` 映射到技术宏。`strict` 强制 MacroCompiler 将所有内存映射到技术宏，如果无法这样做，则会出错。 `synflops` 值强制 MacroCompiler 将所有内存映射到触发器。 `compileandsynflops` 值指示 MacroCompiler 使用技术编译器来确定所使用的技术宏的大小，然后使用触发器创建这些宏的模拟版本。 `Fallbacksynflops` 值使 MacroCompiler 将所有可能的存储器编译为技术宏，但当无法这样做时，使用触发器来实现剩余存储器。最终默认值 `compileavailable` 指示 MacroCompiler 将所有内存编译为技术宏，如果无法映射它们则不执行任何操作。

其余大部分选项用于控制预期和产生不同输入和输出的位置。选项 `--macro-conf` 是包含要映射为上述 `.conf` 格式的一组输入 `SeqMem` 配置的文件。选项 `--macro-mdf` 还描述了输入 `SeqMem`，但采用 `.mdf` 格式。选项 `--library` 是可映射到的可用技术宏的 `.mdf` 描述。该文件可以是固定大小存储器的列表，通常称为宏高速缓存，或者是通过某些技术特定过程（通常是 SRAM 编译器）可用的存储器大小的描述，或者两者的混合。选项 `--use-compiler` 指示 MacroCompiler 允许使用 `--library` 规范中列出的任何编译器。如果未设置此选项，MacroCompiler 将仅映射到 `--library` 规范中直接列出的宏。 `--verilog` 选项指定 MacroCompiler 将在何处写入包含新技术映射存储器的 verilog。 `--firrtl` 选项同样指定 MacroCompiler 将在何处写入将用于生成此 verilog 的 FIRRTL。该选项是可选的，如果未指定，则不会发出 FIRRTL。`--hammer-ir` 选项指定 MacroCompiler 将在何处写入需要从技术编译器生成的宏的详细信息。如果未指定 `--use-compiler`，则不需要此选项。然后可以将该文件传递给 HAMMER，使其运行技术编译器，生成相关的宏附属内容。`--cost-func` 选项允许用户为映射任务指定不同的成本函数。由于存储器的映射是跨越性能、功耗和面积的多维空间，因此 MacroCompiler 的成本函数设置允许用户根据自己的喜好调整映射。默认选项是一个合理的启发式方法，尝试最大限度地减少每个 `SeqMem` 实例化的技术宏的数量，而不浪费太多内存位。有两种方法可以添加额外的成本函数。首先，您可以简单地在 scala 中编写另一个并调用 registerCostMetric，然后您可以将其名称传递给此命令行标志。其次，有一个预定义的ExternalMetric，它将执行一个程序（作为路径传入），其中包含正在编译的内存的MDF 描述以及建议作为映射的内存。程序应该打印一个浮点数，它是此映射的成本，如果没有打印数字，MacroCompiler 将假定这是非法映射。 `--cost-param` 选项允许用户指定要传递给成本函数的参数（如果成本函数支持）。 `--force-synflops [mem]` 选项允许用户覆盖 MacroCompiler 中的任何启发式方法并强制其将给定内存映射到触发器。同样，`--force-compile [mem]` 选项允许用户强制 MacroCompiler 将给定的 `mem` 映射到技术宏。

### SRAM MDF Fields

MDF 中描述的技术 SRAM 宏可以在三个详细级别上进行定义。可以使用 SRAMMacro 格式定义单个实例。可以使用 SRAMGroup 格式定义一组共享端口数量和类型但宽度和深度不同的实例。可以使用 SRAMCompiler 格式定义一组可以从单个源（如编译器）一起生成的 SRAM 组。

在最具体的层面上，SRAMMAcro 定义了 SRAM 的特定实例。这包括其功能属性，例如宽度、深度和访问端口的数量。这些端口可以是读、写或读写端口，并且实例可以有任意数量。为了将这些功能端口正确映射到物理实例，每个端口都在父实例结构中的子结构列表中进行描述。每个端口只需要有一个地址和数据字段，但可以有许多其他可选字段。这些可选字段包括时钟、写使能、读使能、芯片使能、掩码及其粒度。掩码字段可以具有与数据字段不同的粒度，例如它可以是位掩码或字节掩码。每个字段还必须指定其极性，无论是高电平有效还是低电平有效。

上面介绍的具体 JSON 文件格式在[这里](https://github.com/ucb-bar/plsi-mdf/blob/4be9b173647c77f990a542f4eb5f69af01d77316/macro_format.json)。[此处](https://github.com/ucb-bar/hammer/blob/8fd1486499b875d56f09b060f03a62775f0a6aa7/src/hammer-vlsi/technology/nangate45/sram-cache.json)提供了 nangate45 技术库中的 SRAM 参考缓存。

除了 SRAM 的这些功能描述之外，还有其他字段指定物理/实现特性。其中包括阈值电压、多路复用器因子以及额外的非功能端口列表。

更详细地说，SRAMGroup 包括一系列深度和宽度，以及一组阈值电压。范围具有下限、上限和步长。最不具体的层面是，SRAMCompiler 只是一组 SRAMGroup。

## Separating the Top module from the TestHarness module

与 FireSim 和软件仿真流程不同，VLSI 流程需要将测试工具和芯片（也称为 DUT）分离到单独的文件中。这对于促进综合后和布局布线后仿真是必要的，因为 RTL 和门级 verilog 文件中的模块名称会发生​​冲突。在您的设计经过 VLSI 流程后，仿真将使用该流程生成的 verilog 网表，并且需要一个未受影响的测试工具来驱动它。将这些组件分离到单独的文件中使这变得简单。如果不进行分离，包含测试工具的文件也会重新定义 DUT，这在仿真工具中通常是不允许的。为此，[Barstools](https://chipyard.readthedocs.io/en/stable/Tools/Barstools.html#barstools) 中有一个名为 `GenerateTopAndHarness` 的 FIRRTL 应用程序，它运行适当的转换来分别详细说明模块。这还会重命名测试工具中的模块，以便在测试工具和芯片中实例化的任何模块都是唯一的。

> **注意**：对于 VLSI 项目，运行 `App` 而不是普通的 FIRRTL `App` 来详细说明 Verilog。

## Macro Description Format

SRAM 技术宏和 IO 单元以称为宏描述格式 (MDF) 的 json 格式进行描述。 MDF 专门针对它支持的每种类型的宏。专业化在各自的部分中定义。

## Mapping technology IO cells

与技术 SRAM 一样，IO 单元几乎总是包含在数字 ASIC 设计中，以实现引脚可配置性、提高 IO 信号的电压电平并提供 ESD 保护。与 SRAM 不同，Chisel 或 FIRRTL 中没有相应的原语。然而，这个问题可以通过利用这些基于 scala 的工具中可用的强类型来解决，类似于 `SeqMems`。我们正在积极开发 FIRRTL 转换，该转换将自动配置、映射和连接技术 IO 单元。敬请期待更多的信息！

同时，建议您在 Chisel 设计中实例化 IO 单元。不幸的是，这打破了与流程无关的 RTL 抽象，因此建议使用 `rocket-chip` 参数化系统来配置这些单元的包含。最简单的方法是使用一个配置片段，当包含更新时，该片段会实例化 IO 单元并将它们连接到测试工具中。仿真芯片特定设计时，包含 IO 单元非常重要。 IO 单元行为模型通常会断言它们是否连接不正确，这是一个有用的运行时检查。它们还在综合和布局布线后保持芯片和测试工具边界处的 IO 接口（请参阅 [Separating the Top module from the TestHarness module](https://chipyard.readthedocs.io/en/stable/Tools/Barstools.html#separating-the-top-module-from-the-testharness-module)）保持一致，从而允许重复使用 RTL 模拟测试工具。