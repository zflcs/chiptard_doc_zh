# Memory Hierarchy

## The L1 Caches

每个 CPU 块都有一个 L1 指令缓存和 L1 数据缓存。这些缓存的大小和关联性是可以配置的。默认的 `RocketConfig` 使用 16 KiB、4 路组相联指令和数据缓存。但是，如果您使用 `WithNMedCores` 或 `WithNSmallCores` 配置，则可以为 L1I 和 L1D 配置 4 KiB 直接映射缓存。

如果您只想更改大小或关联性，也可以使用这些配置片段。请参阅 [Config Fragments](https://chipyard.readthedocs.io/en/stable/Customization/Keys-Traits-Configs.html#config-fragments) 了解如何将它们添加到自定义 `Config` 中。

```Scala
new freechips.rocketchip.subsystem.WithL1ICacheSets(128) ++  // change rocket I$
new freechips.rocketchip.subsystem.WithL1ICacheWays(2) ++    // change rocket I$
new freechips.rocketchip.subsystem.WithL1DCacheSets(128) ++  // change rocket D$
new freechips.rocketchip.subsystem.WithL1DCacheWays(2) ++    // change rocket D$
```

您还可以将 L1 数据缓存配置为数据暂存器。然而，这有一些限制。如果您使用数据暂存器，则只能使用单个内核，并且无法为设计提供外部 DRAM。请注意，这些配置完全删除了 L2 缓存和 mbus。

```Scala
class ScratchpadOnlyRocketConfig extends Config(
  new chipyard.config.WithL2TLBs(0) ++
  new testchipip.soc.WithNoScratchpads ++                      // remove subsystem scratchpads, confusingly named, does not remove the L1D$ scratchpads
  new freechips.rocketchip.subsystem.WithNBanks(0) ++
  new freechips.rocketchip.subsystem.WithNoMemPort ++          // remove offchip mem port
  new freechips.rocketchip.subsystem.WithScratchpadsOnly ++    // use rocket l1 DCache scratchpad as base phys mem
  new freechips.rocketchip.subsystem.WithNBigCores(1) ++
  new chipyard.config.AbstractConfig)
```

此配置通过将通道数和存储体数设置为 0 来完全删除 L2 缓存和内存总线。

## The System Bus

系统总线是位于 tile 与 L2 代理和 MMIO 外设之间的 TileLink 网络。通常，它是一个全连接的交叉开关，但可以使用 Constellation 生成基于片上网络的实现。有关更多信息，请参阅 [SoCs with NoC-based Interconnects](https://chipyard.readthedocs.io/en/stable/Customization/NoC-SoCs.html#socs-with-noc-based-interconnects)。

## The Inclusive Last-Level Cache

Chipyard 示例项目中提供的默认 `RocketConfig` 使用 Rocket-Chip InclusiveCache 生成器来生成共享 L2 缓存。在默认配置中，L2 使用具有 512 KiB 容量和 8 路组相联性的单个缓存组。但是，您可以更改这些参数以获得所需的缓存配置。主要限制是路数和组数必须是 2 的幂。

请参阅 `rocket-chip-inclusive-cache` 中定义的 `CacheParameters` 对象以获取自定义选项。

## The Broadcast Hub

如果您不想使用 L2 缓存（例如，对于资源有限的嵌入式设计），您可以创建一个没有它的配置。您将不使用 L2 缓存，而是使用 RocketChip 的 TileLink 广播中心。要进行这样的配置，您只需复制 `RocketConfig` 的定义，但从包含的 mixim 列表中省略 `WithInclusiveCache` 配置片段即可。

如果您想进一步减少使用的资源，您可以将广播集线器配置为使用无缓冲区设计。此配置片段是 `freechips.rocketchip.subsystem.WithBufferlessBroadcastHub`。

## The Outer Memory System

L2 一致性代理（L2 缓存或广播集线器）向由 AXI4 兼容 DRAM 控制器组成的外部内存系统发出请求。

默认配置使用单个内存通道，但您可以将系统配置为使用多个通道。与 L2 存储体的数量一样，DRAM 通道的数量也限制为 2 的幂。

```Scala
new freechips.rocketchip.subsystem.WithNMemoryChannels(2)
```

在 VCS 和 Verilator 仿真中，使用 `SimAXIMem` 模块对 DRAM 进行仿真，该模块只需将单周期 SRAM 连接到每个内存通道。

您可以连接暂存器并删除片外链接，而不是连接到片外 DRAM。这是通过将 `testchipip.soc.WithScratchpad` 之类的片段添加到您的配置中并使用 `freechips.rocketchip.subsystem.WithNoMemPort` 删除内存端口来完成的。

```Scala
class MbusScratchpadOnlyRocketConfig extends Config(
  new testchipip.soc.WithMbusScratchpad(banks=2, partitions=2) ++ // add 2 partitions of 2 banks mbus backing scratchpad
  new freechips.rocketchip.subsystem.WithNoMemPort ++         // remove offchip mem port
  new freechips.rocketchip.subsystem.WithNBigCores(1) ++
  new chipyard.config.AbstractConfig)
```

如果想要更真实的内存模拟，可以使用 FireSim，它可以模拟 DDR3 控制器的时序。有关 FireSim 内存模型的更多文档可在 [FireSim docs](https://docs.fires.im/en/latest/) 中找到。
