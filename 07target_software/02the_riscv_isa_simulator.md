# The RISC-V ISA Simulator (Spike)

Spike 是标准（golden）参考功能 RISC-V ISA C++ 软件模拟器。它通过 [HTIF/FESVR](https://github.com/riscv/riscv-isa-sim/tree/master/fesvr) 提供完整的系统仿真或代理仿真。它是在 RISC-V 目标上运行软件的起点。以下是 Spikes 的一些主要功能的亮点：

1. 多个 ISA：RV32IMAFDQCV 扩展。
2. 多种内存模型：弱内存顺序 (WMO) 和全存储顺序 (TSO)。
3. 特权级：Machine、Supervisor、User 模式 ​​(v1.11)。
4. 调试规范。
5. 单步调试，支持查看内存/寄存器内容。
6. 多CPU支持。
7. JTAG支持。
8. 高度可扩展（添加和测试新指令）。

在大多数情况下，Chipyard 目标的软件开发将从使用 Spike 的功能仿真开始（通常为定制加速器功能添加定制 Spike 模型），然后才使用软件 RTL 仿真器或 FireSim 进行全周期精确仿真。

Spike 预先打包在 RISC-V 工具链中，并可在路径上作为 Spike 使用。更多信息可以在 [Spike repository](https://github.com/riscv/riscv-isa-sim) 中找到。

## Spike-as-a-Tile

Chipyard 包含对使用非核心模拟 Spike 处理器模型的实验支持，类似于虚拟平台。在此配置中，Spike 是缓存一致的，并通过 C++ TileLink 私有缓存模型与非核心进行通信。

```shell
make CONFIG=SpikeConfig run-binary BINARY=hello.riscv
```

Spike-as-a-Tile 还支持 SpikeTile 的紧耦合内存 (TCM)，其中主系统内存完全在 Spike Tile 内建模，从而实现非常快速的模拟性能。

```shell
make CONFIG=SpikeUltraFastConfig run-binary BINARY=hello.riscv
```

Spike-as-a-Tile 可以配置自定义 IPC、提交日志记录和其他行为。尖峰特定标志可以作为 plusargs 添加到 `EXTRA_SIM_FLAGS`。

```shell
make CONFIG=SpikeUltraFastConfig run-binary BINARY=hello.riscv EXTRA_SPIKE_FLAGS="+spike-ipc=10000 +spike-fast-clint +spike-debug" LOADMEM=1
```

1. `+spike-ipc=`：设置 Spike 在单个“tick”或非核心模拟周期中可以退出的最大指令数。
2. `+spike-fast-clint`：通过生成假定时器中断来启用 WFI 停顿快进。
3. `+spike-debug`：启用调试 Spike 日志记录。
4. `+spike-verbose`：启用 Spike 提交日志生成。

## Adding a new spike device model

Spike 附带了一些功能设备模型，例如 UART、CLINT 和 PLIC。但是，您可能希望将自定义设备模型添加到 Spike 中，例如块设备。示例设备位于 `toolchains/riscv-tools/riscv-spike-devices` 目录中。这些设备被编译为可以动态链接到 Spike 的共享库。

要编译这些插件，请在 `toolchains/riscv-tools/riscv-spike-devices` 内运行 `make`。这将生成一个 `libspikedevices.so`。

要将块设备连接到 spike 并提供默认映像来初始化块设备，请运行

```shell
spike --extlib=libspikedevices.so --device="iceblk,img=<path to Linux image>" <path to kernel binary>
```

`--device` 选项由设备名称和参数组成。在上面的示例中，`iceblk` 是设备名称，`img=<Linux 映像的路径>` 是传递给插件设备的参数。