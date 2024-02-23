# Architectural Checkpoints

Chipyard 支持使用 Spike 生成架构检查点。这些检查点包含程序执行过程中某个时刻 RISC-V SoC 架构状态的快照。检查点包括可缓存内存、核心架构寄存器和核心 CSR 的内容。 SoC 的 RTL 仿真可以在恢复架构状态后从检查点恢复执行。

> **注意**：目前仅支持单核系统的检查点。

## Generating Checkpoints

`scripts/generate-ckpt.sh` 是一个使用正确命令运行 spike 的脚本，以生成架构检查点 `scripting/generate-ckpt.sh -h` 列出了检查点生成的选项。

示例：在生成检查点之前运行 `hello.riscv` 二进制文件 1000 条指令。这应该会生成一个名为 `hello.riscv.0x80000000.1000.loadarch` 的目录。

```shell
scripts/generate-ckpt.sh -b tests/hello.riscv -i 1000
```

## Loading Checkpoints in RTL Simulation

可以使用 `LOADARCH` 标志在 RTL 仿真中加载检查点。目标配置必须使用基于 dmi 的 Bringup（而不是默认的基于 TSI 的 Bringup），并支持快速 `LOADMEM`。目标配置还应该与生成检查点时配置的 spike 的架构配置相匹配。

```shell
cd sims/vcs
make CONFIG=dmiRocketConfig run-binary LOADARCH=../../hello.riscv.0x80000000.1000.loadarch
```