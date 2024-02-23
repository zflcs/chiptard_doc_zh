# Target Software

Chipyard 包含用于开发目标软件工作负载的工具。主要工具是 FireMarshal，它管理工作负载描述并生成二进制文件和磁盘映像以在目标设计上运行。工作负载可以是裸机，也可以基于标准 Linux 发行版。用户可以自定义构建过程的每个部分，包括提供自定义内核（如果硬件需要）。

FireMarshal 还可以在 Spike 和 Qemu 等高性能功能模拟器上运行您的工作负载。 Spike 易于定制，可作为官方 RISC-V ISA 参考实现。 Qemu 是一种高性能功能模拟器，其运行速度几乎与本机代码一样快，但修改起来可能具有挑战性。

要初始化其他软件存储库，例如 Coremark、SPEC2017 的包装器和 NVDLA 的工作负载，请运行以下脚本。子模块位于 `software` 目录中。

```shell
./scripts/init-software.sh
```