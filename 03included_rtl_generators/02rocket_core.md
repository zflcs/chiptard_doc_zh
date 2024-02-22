# Rocket Core

[Rocket](https://github.com/chipsalliance/rocket-chip) 是一个 5 级顺序标量处理器核心生成器，最初由加州大学伯克利分校和 SiFive 开发，现在由 Chips Alliance 维护。 Rocket 核心用作 Rocket Chip SoC 生成器内的组件。 Rocket 核心与 L1 缓存（数据和指令缓存）相结合形成了 Rocket 区块。 Rocket 块是 Rocket Chip SoC 生成器的可复制组件。

Rocket core 支持开源 RV64GC RISC-V 指令集，并采用 Chisel 硬件构建语言编写。它有一个支持基于页面的虚拟内存的 MMU、一个非阻塞数据缓存和一个具有分支预测的前端。分支预测是可配置的，并由分支目标缓冲区 (BTB)、分支历史表 (BHT) 和返回地址堆栈 (RAS) 提供。对于浮点，Rocket 使用 Berkeley 的 Chisel 浮点单元实现。 Rocket 还支持 RISC-V 机器、管理员和用户权限级别。公开了许多参数，包括某些 ISA 扩展（M、A、F、D）的可选支持、浮点管道级数以及缓存和 TLB 大小。

有关更多信息，请参阅 [GitHub 仓库](https://github.com/freechipsproject/rocket-chip)、[技术报告](https://www2.eecs.berkeley.edu/Pubs/TechRpts/2016/EECS-2016-17.html)或此 [Chisel 社区会议视频](https://youtu.be/Eko86PGEoDY)。