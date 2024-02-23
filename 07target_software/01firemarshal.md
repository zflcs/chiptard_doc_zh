# FireMarshal

FireMarshal 是一款适用于基于 RISC-V 的系统的工作负载生成工具。目前仅支持FireSim FPGA加速仿真平台。

FireMarshal 中的工作负载由一系列分配给目标系统中的逻辑节点的作业组成。如果未指定作业，则工作负载被认为是均匀的，并且只会为系统中的所有节点生成单个映像。工作负载由 `json` 文件和相应的工作负载目录描述，并且可以从现有工作负载继承其定义。通常，工作负载配置保存在 `workloads/` 中，尽管您可以使用您喜欢的任何目录。我们提供了一些基本的工作负载，包括 buildroot 或基于 Fedora 的 Linux 发行版和裸机。

定义工作负载后，`marshal` 命令将为工作负载中的每个作业生成相应的启动二进制文件和 rootfs。然后可以在 qemu 或 spike 上启动该二进制文件和 rootfs（用于功能模拟），或安装到在真实 RTL 上运行的平台（目前只有 FireSim 是自动化的）。

请查看完整的 [FireMarshal documentation](https://firemarshal.readthedocs.io/en/latest/index.html)。