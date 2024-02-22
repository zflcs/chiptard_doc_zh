# Prefetchers

BAR-fetchers 库是 Chisel 实现的预取器的集合，旨在与 Chipyard 和 Rocket-Chip SoC 兼容。该包实现了通用预取器 API，以及 NextLine、Strided 和 AMPM 预取器的示例实现。

预取器可以在 L1D HellaCache 前面实例化，或者在某些 TileLink 总线前面实例化为 TileLink 节点。

使用预取器的示例配置可以在 `PrefetchingRocketConfig` 中找到。