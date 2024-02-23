# Chipyard Boot Process

本节将详细描述基于 Chipyard 的 SoC 启动 Linux 内核的过程以及您可以进行的更改来自定义此过程。

## BootROM and RISC-V Frontend Server

BootROM 包含 SoC 开机时运行的第一条指令以及详细说明系统组件的设备树二进制文件 (dtb)。BootROM 代码的程序集位于 [generators/testchipip/src/main/resources/testchipip/bootrom/bootrom.S](https://github.com/ucb-bar/testchipip/blob/master/src/main/resources/testchipip/bootrom/bootrom.S) 中。 BootROM 地址空间从 `0x10000`（由配置中的 `BootROMParams` key 确定）开始，执行从地址 `0x10000`（由 `BootROMParams` 中的链接器脚本和复位向量给出）开始，该地址由 BootROM 程序集中的 `_hang` 标签标记。

Chisel 生成器在描述时将汇编指令编码到 BootROM 硬件中，因此如果要更改 BootROM 代码，则需要在 bootrom 目录中运行 `make`，然后重新生成Verilog。如果您不想覆盖现有的 `bootrom.S`，您还可以通过覆盖配置中的 `BootROMParams` key 将生成器指向不同的 bootrom 映像。

```Scala
class WithMyBootROM extends Config((site, here, up) => {
  case BootROMParams =>
    BootROMParams(contentFileName = "/path/to/your/bootrom.img")
})
```

当 RISC-V 前端服务器 (FESVR) 加载实际程序时，默认引导加载程序只是循环等待中断 (WFI) 指令。FESVR 是一个在主机 CPU 上运行的程序，可以使用驻留串行接口 (TSI) 读取/写入目标系统内存的任意部分。

FESVR 使用 TSI 将裸机可执行文件或第二阶段引导加载程序加载到 SoC 内存中。在 [Software RTL Simulation](https://chipyard.readthedocs.io/en/stable/Simulation/Software-RTL-Simulation.html#software-rtl-simulation) 中，这将是您传递给仿真器的二进制文件。一旦完成加载程序，FESVR 将写入 CPU 0 的软件中断寄存器，这将使 CPU 0 脱离其 WFI 循环。一旦收到中断，CPU 0 将写入系统中其他 CPU 的软件中断寄存器，然后跳转到 DRAM 的开头以执行加载的可执行文件的第一条指令。其他CPU将被第一个CPU唤醒，并跳转到DRAM的开头。

FESVR 加载的可执行文件应具有指定为 tohost 和 fromhost 的内存位置。一旦可执行文件运行，FESVR 使用这些内存位置与可执行文件进行通信。该可执行文件使用 tohost 向 FESVR 发送命令，以执行打印到控制台、代理系统调用和关闭 SoC 等操作。fromhost 寄存器用于发回 tohost 命令的响应以及发送控制台输入。

## The Berkeley Boot Loader and RISC-V Linux

对于裸机程序来说，故事到这里就结束了。加载的可执行文件将在机器模式下运行，直到它通过 tohost 寄存器发送命令告诉 FESVR 关闭 SoC。

但是，为了引导 Linux 内核，您需要使用称为 Berkeley Boot Loader（BBL）的第二阶段引导加载程序。该程序读取 boot ROM 中编码的设备树，并将其转换为与 Linux 内核兼容的格式。然后，它设置虚拟内存和中断控制器，加载内核（作为有效负载嵌入到引导加载程序二进制文件中），并开始在管理程序模式下执行内核。引导加载程序还负责服务来自内核的机器模式陷阱并通过 FESVR 代理它们。

一旦 BBL 跳转到管理模式，Linux 内核就会接管并开始其进程。它最终加载 `init` 程序并在用户模式下运行它，从而开始用户空间执行。

构建启动 Linux 的 BBL 映像的最简单方法是使用 [firesim-software](https://github.com/firesim/firesim-software) 存储库中的 FireMarshal 工具。有关如何使用 FireMarshal 的说明可以在 [FireSim documentation](https://docs.fires.im/en/stable/Advanced-Usage/FireMarshal/index.html) 中找到。使用 FireMarshal，您可以将自定义内核配置和用户空间软件添加到您的工作负载中。