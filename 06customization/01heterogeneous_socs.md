# Heterogeneous SoCs

Chipyard 框架涉及可以任意方式组成的多个内核和加速器。本讨论将重点讨论如何以特定方式组合 Rocket、BOOM 和 Hwacha 来创建独特的 SoC。

## Creating a Rocket and BOOM System

使用 Rocket 和 BOOM 内核实例化 SoC 都是通过配置系统和两个特定的配置片段完成的。BOOM 和 Rocket 都有标记为 `WithN{Small|Medium|Large|etc.}BoomCores(X)` 和 `WithNBigCores(X)` 的配置片段，它们会自动创建核心/图块 的 `X` 个副本。一起使用时，您可以创建异构系统。

以下示例显示了带有单核 Rocket 的双核 BOOM。

```Scala
class DualLargeBoomAndSingleRocketConfig extends Config(
  new boom.common.WithNLargeBooms(2) ++                   // add 2 boom cores
  new freechips.rocketchip.subsystem.WithNBigCores(1) ++  // add 1 rocket core
  new chipyard.config.WithSystemBusWidth(128) ++
  new chipyard.config.AbstractConfig)
```

## Adding Hwachas

添加 Hwacha 加速器就像添加 `DefaultHwachaConfig` 一样简单，这样它就可以设置 Hwacha 参数并将其自身添加到 BuildRoCC 参数中。下面是向系统中所有图块添加 Hwacha 的示例。

```Scala
class HwachaLargeBoomAndHwachaRocketConfig extends Config(
  new chipyard.config.WithHwachaTest ++
  new hwacha.DefaultHwachaConfig ++                          // add hwacha to all harts
  new boom.common.WithNLargeBooms(1) ++                      // add 1 boom core
  new freechips.rocketchip.subsystem.WithNBigCores(1) ++     // add 1 rocket core
  new chipyard.config.WithSystemBusWidth(128) ++
  new chipyard.config.AbstractConfig)
```

在此示例中，Hwachas 被添加到 BOOM 块和 Rocket 块中。全部具有相同的 Hwacha 参数。

## Assigning Accelerators to Specific Tiles with MultiRoCC

位于 `generators/chipyard/src/main/scala/config/fragments/RoCCFragments.scala` 中的是一个配置片段，它支持将 RoCC 加速器添加到 SoC 中的特定块。`MultiRoCCKey` 允许您根据块的 hartId 连接 RoCC 加速器。例如，使用此功能可以让您创建一个 8 个切片系统，并且仅在一部分切片上使用 RoCC 加速器。下面显示了一个示例，其中包含两个 BOOM 核心和一个附加了 RoCC 加速器 (Hwacha) 的 Rocket 块。

```Scala
class DualLargeBoomAndHwachaRocketConfig extends Config(
  new chipyard.config.WithMultiRoCC ++                                  // support heterogeneous rocc
  new chipyard.config.WithMultiRoCCHwacha(0) ++                         // put hwacha on hart-0 (rocket)
  new hwacha.DefaultHwachaConfig ++                                     // set default hwacha config keys
  new boom.common.WithNLargeBooms(2) ++                                 // add 2 boom cores
  new freechips.rocketchip.subsystem.WithNBigCores(1) ++                // add 1 rocket core
  new chipyard.config.WithSystemBusWidth(128) ++
  new chipyard.config.AbstractConfig)
```

`WithMultiRoCCHwacha` 配置片段将 Hwacha 加速器分配给特定的 `hartId`（在本例中，`hartId 0` 对应于 Rocket 核心）。最后，调用 `WithMultiRoCC` 配置片段。此配置片段将 `BuildRoCC` 密钥设置为使用 `MultiRoCCKey` 而不是默认值。必须在设置所有 RoCC 参数后使用它，因为它需要覆盖 `BuildRoCC` 参数。如果在配置序列中较早使用它，则 MultiRoCC 将不起作用。

可以更改此配置片段，通过更改参数以覆盖更多 `hartId`（即 `WithMultiRoCCHwacha(0,1,3,6,...)`），在更多内核上放置更多加速器。

由于配置片段是从右到左应用的（或者从下到上，这里以这种形式出现），指定核心的最右边的配置片段（在上面的示例中是 `freechips.rocketchip.subsystem.WithNBigCores` ）得到第一个 hart ID。考虑这个配置：

```Scala
class RocketThenBoomHartIdTestConfig extends Config(
  new boom.common.WithNLargeBooms(2) ++
  new freechips.rocketchip.subsystem.WithNBigCores(3) ++
  new chipyard.config.AbstractConfig)
```

这指定了具有三个 Rocket 内核和两个 BOOM 内核的 SoC。 Rocket 核心将具有 hart ID 0、1 和 2，而 BOOM 核心将具有 hart ID 3 和 4。另一方面，请考虑此配置，它颠倒了这两个片段的顺序：

```Scala
class BoomThenRocketHartIdTestConfig extends Config(
  new freechips.rocketchip.subsystem.WithNBigCores(3) ++
  new boom.common.WithNLargeBooms(2) ++
  new chipyard.config.AbstractConfig)
```

这还指定了具有三个 Rocket 内核和两个 BOOM 内核的 SoC，但由于 BOOM 配置片段是在 Rocket 配置片段之前评估的，因此 hart ID 是相反的。 BOOM 核心的 hart ID 为 0 和 1，而 Rocket 核心的 hart ID 为 2、3 和 4。