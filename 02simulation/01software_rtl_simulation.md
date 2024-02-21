# Software RTL Simulation

## Verilator (Open-Source)

[Verilator](https://www.veripool.org/wiki/verilator) 是由 Veripool 维护的开源 LGPL 许可仿真器。 Chipyard 框架可以使用 Verilator 下载、构建和执行e仿真。

## Synopsys VCS (License Required)

VCS 是 Synopsys 开发的商业 RTL 仿真器。它需要商业许可证。 Chipyard 框架可以使用 VCS 编译和执行仿真。 VCS 模拟通常比 Verilator 模拟编译得更快。

要运行 VCS 模拟，请确保 VCS 仿真器位于您的 `PATH` 中。

## Choice of Simulator

首先，我们首先进入 Verilator 或 VCS 目录：

对于开源 Verilator 模拟，请进入 `sims/verilator` 目录。

```shell
# Enter Verilator directory
cd sims/verilator
```

对于专有的 VCS 模拟，请输入 `sims/vcs` 目录。

```shell
# Enter Verilator directory
cd sims/vcs
```

## Simulating The Default Example

要编译示例设计，请在选定的 verilator 或 VCS 目录中运行 `make`。这里将详细阐述示例项目中的 `RocketConfig。`

> **注意**：`RocketConfig` 的描述需要大约 6.5 GB 的主存。否则，该过程将失败，并显示 make: *** [firrtl_temp] 错误 137，这很可能与资源有限有关。其他配置可能需要更多的主内存。

将生成一个名为 `Simulator-chipyard-RocketConfig` 的可执行文件。该可执行文件是一个根据所构建的设计编译的仿真器。然后，您可以使用此可执行文件来运行任何兼容的 RV64 代码。例如，运行 riscv-tools 组装测试之一。

```shell
./simulator-chipyard-RocketConfig $RISCV/riscv64-unknown-elf/share/riscv-tests/isa/rv64ui-p-simple
```

> **注意**：在 VCS 模拟器中，模拟器名称将为 `simv-chipyard-RocketConfig`，而不是 `Simulator-chipyard-RocketConfig`。

makefile 有一个 `run-binary` 规则，可以简化仿真可执行文件的运行。它为您添加了许多常见的命令行选项并将输出重定向到文件。

```shell
make run-binary BINARY=$RISCV/riscv64-unknown-elf/share/riscv-tests/isa/rv64ui-p-simple
```

或者，我们可以通过添加 make 目标 `run-asm-tests` 或 `run-bmark-tests` 来运行预打包的 RISC-V 汇编或基准测试套件。例如：

```shell
make run-asm-tests
make run-bmark-tests
```

> **注意**：在运行预打包的套件之前，您必须运行普通的 `make` 命令，因为精化命令会生成一个 `Makefile` 片段，其中包含预打包的测试套件的目标。否则，您可能会遇到 Makefile 目标错误。

## Custom Benchmarks/Tests

要编译您自己的裸机代码以在 Verilator/VCS 仿真中运行，请将其添加到 Chipyard 的 `tests` 目录中，然后将其名称添加到 `Makefile` 内的 `PROGRAMS` 列表中。这些二进制文件是使用 libgloss-htif 库编译的，该库为裸机二进制文件实现了一组最小的有用系统调用。然后，当您运行 `make` 时，测试中的所有程序都将被编译为 `.riscv` ELF 二进制文件，可以如上所述与仿真器一起使用。

```shell
# Enter Tests directory
cd tests
make

# Enter Verilator or VCS directory
cd ../sims/verilator
make run-binary BINARY=../../tests/hello.riscv
```

> **注意**：在多核配置上，只有 hart（硬件线程）0 执行 `main()` 函数。所有其他 hart 都执行 secondary `__main()` 函数，该函数默认为繁忙循环。要在 Verilator/VCS 仿真上运行多线程工作负载，请使用您自己的代码覆盖 `__main()`。更多详情在 [这里](https://github.com/ucb-bar/libgloss-htif)。

## Makefile Variables and Commands

您可以从 Verilator 或 VCS 目录获取有用的 Makefile 变量和命令的列表。只需运行 `make help`：

```shell
# Enter Verilator directory
cd sims/verilator
make help

# Enter VCS directory
cd sims/vcs
make help
```

## Simulating A Custom Project

如果您稍后创建自己的项目，则可以使用环境变量来构建备用配置。

为了使用我们的自定义设计构建仿真器，我们在仿真器目录中运行以下命令：

```shell
make SBT_PROJECT=... MODEL=... VLOG_MODEL=... MODEL_PACKAGE=... CONFIG=... CONFIG_PACKAGE=... GENERATOR_PACKAGE=... TB=... TOP=...
```

这些 make 变量中的每一个都对应于设计/代码库的特定部分，并且是 make 系统可以正确构建和进行 RTL 仿真所必需的。

`SBT_PROJECT` 是包含所有源文件的 `build.sbt` 项目，并将在 RTL 构建期间运行。

`MODEL` 和 `VLOG_MODEL` 是设计的顶级类名称。通常，它们是相同的，但在某些情况下它们可能不同（如果 Chisel 类与 Verilog 中发出的类不同）。

`MODEL_PACKAGE` 是包含 `MODEL` 类的 Scala 包（在 Scala 代码中表示 package ...）。

`CONFIG` 是用于参数配置的类的名称，而 `CONFIG_PACKAGE` 是它所在的 Scala 包。

`GENERATOR_PACKAGE` 是 Scala 包，其中包含详细说明设计的生成器类。

`TB` 是将 `TestHarness` 连接到 VCS/Verilator 进行仿真的 Verilog 包装器的名称。

最后，`TOP` 变量用于区分设计的顶层和我们系统中的 `TestHarness`。例如，在正常情况下，`MODEL` 变量将 `TestHarness` 指定为设计的顶层。然而，真正的顶层设计，即正在仿真的 SoC，是由 `TOP` 变量指向的。这种分离允许基础设施根据测试或 SoC 顶层来分离文件。

所有这些变量的通用配置都使用 `SUB_PROJECT` make 变量进行打包。因此，为了仿真一个简单的基于 Rocket 的示例系统，我们可以使用：

```shell
make SUB_PROJECT=yourproject
./simulator-<yourproject>-<yourconfig> ...
```

所有可应用于默认示例的 `make` 目标也可以使用自定义环境变量应用于自定义项目。例如，以下代码示例将在 Hwacha 子项目上运行 RISC-V 汇编基准测试套件：

```shell
make SUB_PROJECT=hwacha run-asm-tests
```

最后，在 `generated-src/<...>-<package>-<config>/` 目录中保留所有附属资料，而生成的 Verilog 源文件驻留在 `generated-src/<...>-<package>- 中<config>/gen-collaborative` 用于构建/仿真。

## Fast Memory Loading

仿真器通过仿真串行线路加载程序的二进制文件。如果有大量静态数据，这可能会很慢，因此仿真器还允许将数据从文件直接加载到 DRAM 模型中。Loadmem 文件应该是 ELF 文件。在最常见的用例中，这可以是二进制文件。

```shell
make run-binary BINARY=test.riscv LOADMEM=test.riscv
```

通常 `LOADMEM` ELF 与 `BINARY` ELF相同，因此可以使用 `LOADMEM=1` 作为快捷方式。

```shell
make run-binary BINARY=test.riscv LOADMEM=1
```

## Generating Waveforms

如果您想从仿真中提取波形，请运行命令 `make debug` 而不仅仅是 `make`。

还可以使用自动生成特定测试波形文件的特殊目标。

```shell
make run-binary-debug BINARY=test.riscv
```

对于 Verilator 仿真，这将生成一个可以加载到任何常见波形查看器的 vcd 文件（vcd 是标准波形表示文件格式）。 [GTKWave](http://gtkwave.sourceforge.net/) 是一款支持 vcd 的开源波形查看器。

对于 VCS 仿真，这将生成一个 fsdb 文件，该文件可以加载到支持 fsdb 的波形查看器中。如果您拥有 Synopsys 许可证，我们建议使用 Verdi 波形查看器。

## Visualizing Chipyard SoCs

在 verilog 创建过程中，会生成一个 graphml 文件，使您可以将 Chipyard SoC 可视化为 diplomacy 图。

要查看图表，请首先下载 [yEd](https://www.yworks.com/products/yed/) 等查看器。

`*.graphml` 文件将位于 `generated-src/<...>/` 中。在图形查看器中打开文件。要更清晰地了解 SoC，请切换到“分层”视图。对于 yEd，这可以通过选择 `layout` -> `hierarchical`，然后选择“OK”而不更改任何设置来完成。

## Additional Verilator Options

构建 verilator 仿真器时，还有一些附加选项：

```shell
make VERILATOR_THREADS=8 NUMACTL=1
```

`VERILATOR_THREADS=<num>` 选项使编译后的 Verilator 仿真器能够使用 `<num>` 个并行线程。在多插槽计算机上，您需要使用 `NUMACTL=1` 来启用 `numactl`，以确保所有线程都位于同一插槽上。通过启用此功能，您将使用 Chipyard 的 `numa_prefix` 包装器，它是 `numactl` 的一个简单包装器，可运行经过验证的仿真器，如下所示： `$(numa_prefix) ./simulator-<name> <simulator-args>`。请注意，这两个标志是互斥的，您可以独立使用其中一个（尽管在 Verilator 模拟期间仅使用 `NUMACTL` 和 `VERILATOR_THREADS=8` 是有意义的）。

## Speeding up your RTL Simulation by 2x!

在很多情况下，您的自定义模块与 Tilelink 交互（例如，当您编写自定义加速器时）。Tilelink 的错误接口可能会导致 SoC 挂起，并且调试起来可能会很棘手。为了帮助处理这些情况，您可以将称为 Tilelink 监视器的硬件模块添加到 SoC 中，该模块将在发送错误的 Tilelink 消息时触发断言。然而，这些模块会显着降低 RTL 仿真的速度。

这些模块默认添加到 SoC，用户必须通过将以下行添加到配置中来手动删除这些模块。

```Scala
new freechips.rocketchip.subsystem.WithoutTLMonitors ++
```

例如：

```Scala
class FastRTLSimRocketConfig extends Config(
  new freechips.rocketchip.subsystem.WithoutTLMonitors ++
  new chipyard.RocketConfig)
```

