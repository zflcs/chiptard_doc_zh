# SoCs with NoC-based Interconnects

将片上网络集成到 Chipyard SoC 的主要方法是将标准 TileLink 基于交叉开关的总线之一（系统总线、内存总线、控制总线等）映射到 Constellation 生成的 NoC。

互连可以映射为 TileLink 总线的“专用”互连，在这种情况下，将生成承载总线流量的专用互连。或者，互连可以映射到共享全局互连，在这种情况下，可以通过单个共享互连传输多个 TileLink 总线。

## Private Interconnects

集成系统总线、内存总线和控制总线专用专用互连的示例可以在 [generators/chipyard/src/main/scala/config/NoCConfigs.scala](https://github.com/ucb-bar/chipyard/blob/main/generators/chipyard/src/main/scala/config/NoCConfigs.scala) 的 `MultiNoCConfig` 中看到。

```Scala
class MultiNoCConfig extends Config(
  new constellation.soc.WithCbusNoC(constellation.protocol.SimpleTLNoCParams(
    constellation.protocol.DiplomaticNetworkNodeMapping(
      inNodeMapping = ListMap(
        "serial_tl" -> 0),
      outNodeMapping = ListMap(
        "error" -> 1, "ctrls[0]" -> 2, "pbus" -> 3, "plic" -> 4,
        "clint" -> 5, "dmInner" -> 6, "bootrom" -> 7, "clock" -> 8)),
    NoCParams(
      topology = TerminalRouter(BidirectionalLine(9)),
      channelParamGen = (a, b) => UserChannelParams(Seq.fill(5) { UserVirtualChannelParams(4) }),
      routingRelation = NonblockingVirtualSubnetworksRouting(TerminalRouterRouting(BidirectionalLineRouting()), 5, 1))
  )) ++
  new constellation.soc.WithMbusNoC(constellation.protocol.SimpleTLNoCParams(
    constellation.protocol.DiplomaticNetworkNodeMapping(
      inNodeMapping = ListMap(
        "L2 InclusiveCache[0]" -> 1, "L2 InclusiveCache[1]" -> 2,
        "L2 InclusiveCache[2]" -> 5, "L2 InclusiveCache[3]" -> 6),
      outNodeMapping = ListMap(
        "system[0]" -> 0, "system[1]" -> 3,  "system[2]" -> 4 , "system[3]" -> 7,
        "serial_tl_0" -> 0)),
    NoCParams(
      topology        = TerminalRouter(BidirectionalTorus1D(8)),
      channelParamGen = (a, b) => UserChannelParams(Seq.fill(10) { UserVirtualChannelParams(4) }),
      routingRelation = BlockingVirtualSubnetworksRouting(TerminalRouterRouting(BidirectionalTorus1DShortestRouting()), 5, 2))
  )) ++
  new constellation.soc.WithSbusNoC(constellation.protocol.SimpleTLNoCParams(
    constellation.protocol.DiplomaticNetworkNodeMapping(
      inNodeMapping = ListMap(
        "Core 0" -> 1, "Core 1" -> 2,  "Core 2" -> 4 , "Core 3" -> 7,
        "Core 4" -> 8, "Core 5" -> 11, "Core 6" -> 13, "Core 7" -> 14,
        "serial_tl" -> 0),
      outNodeMapping = ListMap(
        "system[0]" -> 5, "system[1]" -> 6, "system[2]" -> 9, "system[3]" -> 10,
        "pbus" -> 3)),
    NoCParams(
      topology        = TerminalRouter(Mesh2D(4, 4)),
      channelParamGen = (a, b) => UserChannelParams(Seq.fill(8) { UserVirtualChannelParams(4) }),
      routingRelation = BlockingVirtualSubnetworksRouting(TerminalRouterRouting(Mesh2DEscapeRouting()), 5, 1))
  )) ++
  new freechips.rocketchip.subsystem.WithNBigCores(8) ++
  new freechips.rocketchip.subsystem.WithNBanks(4) ++
  new freechips.rocketchip.subsystem.WithNMemoryChannels(4) ++
  new chipyard.config.AbstractConfig
)
```
请注意，对于每个总线（`Sbus` / `Mbus` / `Cbus`），配置片段提供私有 NoC 的参数化，以及 TileLink 代理和物理 NoC 节点之间的映射。

有关如何构建 NoC 参数的更多信息，请参阅 [Constellation documentation](http://constellation.readthedocs.io/)。

## Shared Global Interconnect

集成支持传输多个 TileLink 总线的单个全局互连的示例可以在 [generators/chipyard/src/main/scala/config/NoCConfigs.scala](https://github.com/ucb-bar/chipyard/blob/main/generators/chipyard/src/main/scala/config/NoCConfigs.scala) 的 `SharedNoCConfig` 中看到。

```Scala
class SharedNoCConfig extends Config(
  new constellation.soc.WithGlobalNoC(GlobalNoCParams(
    NoCParams(
      topology        = TerminalRouter(HierarchicalTopology(
        base     = UnidirectionalTorus1D(10),
        children = Seq(HierarchicalSubTopology(1, 2, BidirectionalLine(5)),
                       HierarchicalSubTopology(7, 2, BidirectionalLine(5))))),
      channelParamGen = (a, b) => UserChannelParams(Seq.fill(22) { UserVirtualChannelParams(4) }),
      routingRelation = NonblockingVirtualSubnetworksRouting(TerminalRouterRouting(HierarchicalRouting(
        baseRouting = UnidirectionalTorus1DDatelineRouting(),
        childRouting = Seq(BidirectionalLineRouting(),
                           BidirectionalLineRouting()))), 10, 2)
    )
  )) ++
  new constellation.soc.WithMbusNoC(constellation.protocol.GlobalTLNoCParams(
    constellation.protocol.DiplomaticNetworkNodeMapping(
      inNodeMapping = ListMap(
        "Cache[0]" -> 0, "Cache[1]" -> 2, "Cache[2]" -> 8, "Cache[3]" -> 6),
      outNodeMapping = ListMap(
        "system[0]" -> 3, "system[1]" -> 5,
        "ram[0]" -> 9))
  )) ++
  new constellation.soc.WithSbusNoC(constellation.protocol.GlobalTLNoCParams(
    constellation.protocol.DiplomaticNetworkNodeMapping(
      inNodeMapping = ListMap(
        "serial_tl" -> 9, "Core 0" -> 2,
        "Core 1" -> 10, "Core 2" -> 11, "Core 3" -> 13, "Core 4" -> 14,
        "Core 5" -> 15, "Core 6" -> 16, "Core 7" -> 18, "Core 8" -> 19),
      outNodeMapping = ListMap(
        "system[0]" -> 0, "system[1]" -> 2, "system[2]" -> 8, "system[3]" -> 6,
        "pbus" -> 4))
  )) ++
  new freechips.rocketchip.subsystem.WithNBigCores(8) ++
  new freechips.rocketchip.subsystem.WithNBanks(4) ++
  new freechips.rocketchip.subsystem.WithNMemoryChannels(2) ++
  new chipyard.config.AbstractConfig
)
```

请注意，对于每个总线，配置片段仅提供 TileLink 代理和物理 NoC 节点之间的映射，而单独的片段提供全局互连的配置。

有关如何构建 NoC 参数的更多信息，请参阅 [Constellation documentation](http://constellation.readthedocs.io/)。