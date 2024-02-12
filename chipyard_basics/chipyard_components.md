# Chipyard Components

## Generators

Chipyard 框架目前由一下 RTL 生成器组成。

### Processor Cores

#### Rocket Core

一个顺序 RISC-V 处理器核。更多信息请阅读 [Rocket Core]()。

#### BOOM（Berkeley Out-of-Order Machine）

一个乱序 RISC-V 处理器核。更多信息请阅读 [Berkeley Out-of-Order Machine(BOOM)]()。

#### CVA6 Core

一个用 System Verilog 写的顺序 RISC-V 处理器核（之前被称为 Ariane）。更多信息请阅读 [CVA6Core]()。

#### Ibex Core

一个用 System Verilog 写的 32 位顺序 RISC-V 处理器核。更多信息请阅读 [Ibex Core]()。

### Accelerators

#### Hwacha

一个解耦的向量架构协处理器。Hwacha 目前使用向量架构编程模型实现了一个非标准的 RISC-V 扩展，它用 RoCC（Rocket Custom Co-processor） 接口集成了 Rocket or BOOM。更多信息请阅读 [Hwacha]()。

#### Gemmini

一个针对神经网络矩阵乘法的加速器。

#### SHA3

一个用于 SHA3 哈希函数的加速器。旨在演示 Chipyard 如何使用 RoCC 接口。

### System Components

#### constellation

一个用于片上网络互连的生成器。

#### icenet

一个能够达到 200 Gbps 的网卡。

#### rocket-chip-blocks

原本由 SiFive 实现并使用的系统组建，目前被集成到 Rocket Chip 生成器中。目前由 Chips Alliance 维护。这些系统和外围组建包括 UART、SPI、JTAG、I2C、PWM 和其他的接口及设备。

#### AWL（Analog Widget Library）

高速串行链路集成所需的数字组件。

#### testchipip

一组用于在大规模测试环境中测试芯片和接口的工具。


## Tools

### Chisel

嵌入在 Scala 上的硬件描述库。Chisel 旨在使用元编程（使用 Scala 编程语言提供的硬件生成原语）写 RTL 生成器。
Chisel 将生成器输出为 FIRRTL。更多信息请阅读 [Chisel]()。

### FIRRTL

用于数字设计 RTL 描述的中间表示库。FIRRTL 是介于 CHisel 和 Verilog 之间的形式化数字电路表示。FIRRTL 支持 Chisel 细化和 Verilog 生成之间的数字电路操作。更多信息请阅读 [FIRRTL]()。

### Barstools

一个常见的 FIRRTL 变换集合，用于在不更改生成器源 RTL 的情况下操作数字电路。更多信息请阅读 [Barstools]()。

### Dsptools

一个 Chisel 库，用于编写自定义信号处理硬件，以及将自定义信号处理硬件集成到 SoC （尤其是基于 Rocket 的 SoC）中。

## Toolchains

### riscv-tools

一组用于在 RISC-V 指令集上开发和执行软件的软件工具链，包括编译其和汇编器，函数级指令集模拟器，Berkeley 启动加载器（BBL）和代理内核（proxy kernel）。riscv-tools 仓库之前被要求能运行任意的 RISC-V 软件，但许多 riscv-tools 组件已经上游到各自的开源项目（Linux、GNU 等）。尽管如此，为了保证版本一致性以及定制硬件的软件设计灵活性，我们在 Chipyard 框架中包含了 riscv-tools 仓库以及安装。

### esp-tools

riscv-tools 的一个分支，用于 Hwacha 非标准化 RISC-V 扩展的设计。这个分支同样可用于演示如何添加一个额外的 RoCC 加速器到指令集级别的模拟器（spike）上以及高级软件工具链（GNU binutils，riscv-opcodes 等）。

## Software

### FireMarshal

FireMarshal 是 Chipyard 用于创建在它的平台上运行的软件的默认工作负载生成工具。更多信息请阅读 [FireMarshal]()。

### Baremetal-IDE

一个裸机级别的 C/C++ 程序开发的一体化工具。更多信息请阅读 [Tutorial]()。

## Sims

### Verilator

Verilator 是一个开源的 Verilog 模拟器。verilator 目录提供了从相关生成的 RTL 构建基于 Verilator 的模拟器的封装器，允许在模拟器上执行 RISC-V 测试程序（包括 vcd 波形文件）。更多信息请阅读 [Verilator（Open-Source）]()。

### VCS

一个专有的 Verilog 模拟器。用户需要合法的 VCS 许可证和安装。vcs 目录提供了从相关生成的 RTL 构建基于 VCS 的模拟器的封装器，允许在模拟器上执行 RISC-V 测试程序（包括 vcd/vpd 波形文件）。更多信息请阅读 [Synopsys VCS（License Required）]()。

### FireSim

一个开源的 FPGA 加速模拟平台，使用公共云上的 Amazon Web Services（AWS）EC2 F1 实例。FireSim 将开放硬件设计自动地转换为快速（10s-100s MHz）的基于 FPGA 的模拟器，实现高效的流片前验证和性能验证。为了对 I/O 进行模拟，FireSim 包括了适用于 DRAM、以太网、UART 等标准接口的可综合且定时精确的模型。使用弹性公共云，FireSim 能够将模拟扩展至数千个节点。为了使用 FireSime，仓库必须克隆下来，且在 AWS 实例上运行。更多信息请阅读 [FireSim]()。

## Prototyping

### FPGA Prototyping

Chipyard 通过使用 SiFive 的 fpga-shells 对 FPGA 原型设计j进行支持。支持的 FPGA 示例包括 Xilinx Arty 35T 和 VCU118 板。若要使用大量调试工具进行快速、确定性仿真，请考虑使用 [FireSim]() 平台。更多信息请阅读 [Prototyping Flow]()。

## VLSI

### Hammer

Hammer 是一种 VLSI 流程，旨在在通用物理设计概念与特定于供应商的 EDA 工具命令之间提供一个抽象层。HAMMER 流程提供了自动化脚本，根据物理设计约束的更高级别描述生成相关的工具指令。它还允许通过构建特定于工艺技术的插件来复用工艺技术知识，这些插件描述了与该工艺技术相关的特定约束（过时的标准单元、金属层布线约束等）。Hammer 流程要求访问专用 EDA 工具和工艺技术库。更多信息请阅读 [Core Hammer]()。



