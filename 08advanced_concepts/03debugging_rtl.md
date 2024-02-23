# Debugging RTL

虽然封装的 Chipyard 配置和 RTL 已经过测试可以工作，但用户通常希望通过添加自己的 IP 或修改现有的 Chisel 生成器来构建定制芯片。此类更改可能会引入错误。本节旨在使用 Chipyard 运行典型的调试流程。我们假设用户具有自定义 SoC 配置，并尝试通过运行一些软件测试来验证功能。我们还假设该软件已经在功能模拟器（例如 Spike 或 QEMU）上进行了验证。本节将重点讨论硬件调试。

## Waveforms

默认软件 RTL 仿真器在执行期间不会转储波形。要构建具有波形转储功能的仿真器，必须使用 `debug` make 目标。例如：

```shell
make CONFIG=CustomConfig debug
```

`run-binary-debug` 规则还将自动构建仿真器，在自定义二进制文件上运行它，并生成波形。例如，要在 helloworld.riscv 上运行测试，请使用：

```shell
make CONFIG=CustomConfig run-binary-debug BINARY=helloworld.riscv
```

VCS 和 Verilator 还支持许多附加标志。例如，在 VCS 中指定 `+vpdfilesize` 标志会将输出文件视为循环缓冲区，从而为长时间运行的模拟节省磁盘空间。有关详细信息，请参阅 VCS 和 Verilator 手册 您可以使用 `SIM_FLAGS` make 变量来设置其他仿真器标志：

```shell
make CONFIG=CustomConfig run-binary-debug BINARY=linux.riscv SIM_FLAGS=+vpdfilesize=1024
```

> **注意**：在某些有多个仿真器标志的情况下，您可以像下面这样编写 `SIM_FLAGS`：`SIM_FLAGS="+vpdfilesize=XYZ +some_other_flag=ABC"`。

## Print Output

Rocket 和 BOOM 都可以配置不同级别的打印输出。有关信息，请参阅 Rocket 核心源代码或 BOOM [文档](https://docs.boom-core.org/en/latest/)网站。此外，开发人员可以在 Chisel 生成器中的任意条件下插入任意 printf。有关这方面的信息，请参阅 Chisel 文档。

一旦核配置了所需的打印语句，`+verbose` 标志将导致仿真器打印这些语句。以下命令都会生成所需的打印语句：

```shell
make CONFIG=CustomConfig run-binary-debug BINARY=helloworld.riscv

# The below command does the same thing
./simv-CustomConfig-debug +verbose helloworld.riscv
```

两个核心都可以配置为打印提交日志，然后可以将其与 Spike 提交日志进行比较以验证正确性。

## Basic tests

`riscv-tests` 包括基本 ISA 级测试和基本基准测试。这些用于 Chipyard CI，并且应该是验证芯片功能的第一步。make 规则是：

```shell
make CONFIG=CustomConfig run-asm-tests run-bmark-tests
```

## Torture tests

RISC-V torture 实用程序生成随机 RISC-V 汇编流，对其进行编译，在 Spike 功能模型和软件仿真器上运行它们，并验证相同的程序行为。torture 实用程序还可以配置为连续运行以进行压力测试。torture 实用程序存在于 `tools` 目录中。要运行 torture 测试，请在 simulation 目录中运行 `make`：

```shell
make CONFIG=CustomConfig torture
```

要运行 overnight 测试（重复随机测试），请运行：

```shell
make CONFIG=CustomConfig TORTURE_ONIGHT_OPTIONS=<overnight options> torture-overnight
```

您可以在 torture 仓库中的 `night/src/main/scala/main.scala` 中找到 overnight 选项。

## Firesim Debugging

FireSim FPGA 加速仿真中还提供 Chisel printfs、断言、Cospike 联合仿真和波形生成。有关更多详细信息，请参阅 FireSim [documentation](https://docs.fires.im/en/latest/)。




