# Mempress

Mempress 是一个 RoCC 加速器，通过 TileLink 生成内存请求。它尽可能发出请求，对基于 Chipyard/Rocketchip 的 SoC 的内存层次结构进行压力测试。

Mempress 可以生成多个内存请求流。每个流都可以设置为生成读取或写入请求，并配置为生成跨步或随机访问模式。此外，每个流的内存占用也是可配置的。

要将 Mempress 单元添加到 SoC 中，您应该将 `mempress.WithMemPress` 配置片段添加到 SoC 配置中。