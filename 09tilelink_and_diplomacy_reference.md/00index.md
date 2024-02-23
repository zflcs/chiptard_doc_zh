# TileLink and Diplomacy Reference

TileLink 是 RocketChip 和其他 Chipyard 生成器使用的缓存一致性和内存协议。这是缓存、内存、外设和 DMA 设备等不同模块相互通信的方式。

RocketChip 的 TileLink 实现建立在 Diplomacy 之上，这是一个用于在两阶段描述方案中在 Chisel 生成器之间交换配置信息的框架。有关 Diplomacy 的详细解释，请参阅 [the paper by Cook, Terpstra, and Lee](https://carrv.github.io/2017/papers/cook-diplomacy-carrv2017.pdf)。

有关如何连接简单 TileLink 小部件的简要概述可以在 [MMIO Peripherals](https://chipyard.readthedocs.io/en/stable/Customization/MMIO-Peripherals.html#mmio-accelerators) 部分找到。本节将详细介绍 RocketChip 提供的 TileLink 和 Diplomacy 功能。

TileLink 1.7 协议的详细规范可以在 [SiFive website](https://sifive.cdn.prismic.io/sifive%2F57f93ecf-2c42-46f7-9818-bcdd7d39400a_tilelink-spec-1.7.1.pdf) 上找到。