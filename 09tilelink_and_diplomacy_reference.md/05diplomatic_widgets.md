# Diplomatic Widgets

RocketChip 提供了一个 diplomatic TileLink 和 AXI4 小部件库。最常用的小部件都记录在此处。 TileLink 小部件可从 `freechips.rocketchip.tilelink` 获取，AXI4 小部件可从 `freechips.rocketchip.amba.axi4` 获取。

## TLBuffer

用于缓冲 TileLink 事务的小部件。它只是为 2 个（对于 TL-C 为 5 个）解耦通道中的每一个实例化队列。要为每个通道配置队列，请向构造函数传递一个 `freechips.rocketchip.diplomacy.BufferParams` 对象。该case 类的参数是：

1. `depth: Int` - 队列中的条目数。
2. `flow: Boolean` - 如果为 true，则组合耦合 valid 信号，以便可以在排队的同一周期消耗输入。
3. `pipe: Boolean` - 如果为 true，则组合耦合 ready 信号，以便单条目队列可以全速运行。

可以从 `Int` 进行隐式转换。如果传递一个整数而不是 `BufferParams` 对象，则队列将是整数中给定的深度，并且 `flow` 和 `pipeline` 都将为 false。

您还可以使用以下预定义的 BufferParams 对象之一。

1. `BufferParams.default` = `BufferParams(2, false, false)`
2. `BufferParams.none` = `BufferParams(0, false, false)`
3. `BufferParams.flow` = `BufferParams(1, true, false)`
4. `BufferParams.pipe` = `BufferParams(1, false, true)`

参数：

有四个可用的构造函数，带有零个、一个、两个或五个参数。

零参数构造函数对所有通道使用 `BufferParams.default`。

单参数构造函数采用 `BufferParams` 对象来用于所有通道。

双参数构造函数的参数是：
    1. `ace: BufferParams` - 用于 A、C 和 E 通道的参数。
    2. `bd：BufferParams` - 用于 B 和 D 通道的参数。

五参数构造函数的参数是：
    1. `a: BufferParams` - A 通道的缓冲区参数。 
    2. `b: BufferParams` - B 通道的缓冲区参数。
    3. `c: BufferParams` - C 通道的缓冲区参数。
    4. `d: BufferParams` - D 通道的缓冲区参数。
    5. `e: BufferParams` - E 通道的缓冲区参数。

用法示例：

```Scala
// Default settings
manager0.node := TLBuffer() := client0.node

// Using implicit conversion to make buffer with 8 queue entries per channel
manager1.node := TLBuffer(8) := client1.node

// Use default on A channel but pipe on D channel
manager2.node := TLBuffer(BufferParams.default, BufferParams.pipe) := client2.node

// Only add queues for the A and D channel
manager3.node := TLBuffer(
  BufferParams.default,
  BufferParams.none,
  BufferParams.none,
  BufferParams.default,
  BufferParams.none) := client3.node
```

## AXI4Buffer

与 [TLBuffer](https://chipyard.readthedocs.io/en/stable/TileLink-Diplomacy-Reference/Widgets.html#tlbuffer) 类似，但适用于 AXI4。它还将 `BufferParams` 对象作为参数。

参数：

与 TLBuffer 一样，AXI4Buffer 具有零、一、二和五个参数的构造函数。

零参数构造函数对所有通道使用默认的 `BufferParams`。

单参数构造函数将提供的 `BufferParams` 用于所有通道。

双参数构造函数具有以下参数：
    1. `aw：BufferParams` - “ar”、“aw”和“w”通道的缓冲区参数。
    2. `br：BufferParams` - “b”和“r”通道的缓冲区参数。

五参数构造函数具有以下参数：
    1. `aw: BufferParams` - “ar”通道的缓冲区参数。
    2. `w：BufferParams` - “w”通道的缓冲区参数。
    3. `b：BufferParams` - “b”通道的缓冲区参数。
    4. `ar: BufferParams` - “ar”通道的缓冲区参数。 
    5. `r：BufferParams` - “r”通道的缓冲区参数。

用法示例：

```Scala
// Default settings
slave0.node := AXI4Buffer() := master0.node

// Using implicit conversion to make buffer with 8 queue entries per channel
slave1.node := AXI4Buffer(8) := master1.node

// Use default on aw/w/ar channel but pipe on b/r channel
slave2.node := AXI4Buffer(BufferParams.default, BufferParams.pipe) := master2.node

// Single-entry queues for aw, b, and ar but two-entry queues for w and r
slave3.node := AXI4Buffer(1, 2, 1, 1, 2) := master3.node
```

## AXI4UserYanker

该小部件采用具有用户字段的 AXI4 端口，并将其转换为不带用户字段的端口。输入 AR 和 AW 请求中的用户字段值保存在与 ARID/AWID 关联的内部队列中，然后使用该队列将正确的用户字段与响应关联起来。

参数：
    1. `capMaxFlight：Option[Int]` - （可选）可以保存每个 ID 可以进行的请求数量的选项。如果为 `None`（默认值），则 UserYanker 将支持最大数量的正在进行的请求。

用法示例：

```Scala
nouser.node := AXI4UserYanker(Some(1)) := hasuser.node
```

## AXI4Deinterleaver

不同 ID 的多节拍 AXI4 读取响应可能会交错。该小部件重新排序来自从属设备的读取响应，以便单个事务的所有节拍都是连续的。

参数：
    1. `maxReadBytes：Int` - 单个事务中可以读取的最大字节数。

用法示例：

```Scala
interleaved.node := AXI4Deinterleaver() := consecutive.node
```

## TLFragmenter

TLFragmenter 小部件通过将较大的事务分解为多个较小的事务来缩小 TileLink 接口的最大逻辑传输大小。

参数：
    1. `minSize：Int` - 所有向外 managers 支持的最小传输大小。
    2. `maxSize：Int` - 应用 Fragmenter 后支持的最大传输大小。
    3. `alwaysMin: Boolean` - （可选）将所有请求分段到 minSize（否则分段到管理器支持的最大大小）。（默认值：false）
    4. `EarlyAck：EarlyAck.T` -（可选）应该在第一个节拍还是最后一个节拍上确认多节拍 Put？可能的值（默认值：`EarlyAck.None`）：
          1. `EarlyAck.AllPuts` - 始终在第一拍时确认。
          2. `EarlyAck.PutFulls` - 如果 PutFull，则在第一个节拍上确认，否则在最后一个节拍上确认。
          3. `EarlyAck.None` - 始终在最后一个节拍上确认。
    5. `holdFirstDeny: Boolean` -（可选）允许 Fragmenter 通过获取整个突发的第一个拒绝来不安全地组合多节拍获取。（默认值：false）

用法示例：

```Scala
val beatBytes = 8
val blockBytes = 64

single.node := TLFragmenter(beatBytes, blockBytes) := multi.node

axi4lite.node := AXI4Fragmenter() := axi4full.node
```

补充：
    1. TLFragmenter 修改：PutFull、PutPartial、LogicalData、Get、Hint
    2. TLFragmenter 传递：ArithmeticData（如果 alwaysMin 则截断为 minSize）
    3. TLFragmenter 无法修改 acquire（可能是活锁）；因此两侧都放置缓存是不安全的

## AXI4Fragmenter

AXI4Fragmenter 与 [TLFragmenter](https://chipyard.readthedocs.io/en/stable/TileLink-Diplomacy-Reference/Widgets.html#tlfragmenter) 类似。 AXI4Fragmenter 将所有 AXI 访问分割为 manager 支持的最大大小的简单的二次幂大小和对齐传输。这使得它适合作为在 AXI4=>TL 桥之前应用的第一阶段转换。它还使其适合放置在驱动 AXI-lite slave 的 TL=>AXI4 桥之后。

用法示例：

```Scala
axi4lite.node := AXI4Fragmenter() := axi4full.node
```

## TLSourceShrinker

manager 看到的 source ID 的数量通常是根据连接到它的 client 来计算的。在某些情况下，您可能希望固定 source ID 的数量。例如，如果您希望将 TileLink 端口导出到 Verilog 黑匣子，则可以这样做。然而，如果 client 需要更多的源 ID，这就会产生问题。在这种情况下，您将需要使用 TLSourceShrinker。

参数：
    1. `maxInFlight：Int` - 将从 TLSourceShrinker 发送到 manager 的 source ID 的最大数量。

用法示例：

```Scala
// client.node may have >16 source IDs
// manager.node will only see 16
manager.node := TLSourceShrinker(16) := client.node
```

## AXI4IdIndexer

AXI4 相当于 [TLSourceShrinker](https://chipyard.readthedocs.io/en/stable/TileLink-Diplomacy-Reference/Widgets.html#tlsourceshrinker)。这限制了从 AXI4 接口中的 AWID/ARID 位的数量。用于连接到外部或黑盒 AXI4 端口。

参数：
    1. `idBits：Int` - slave 接口上的 ID 位数。

用法示例：

```Scala
// master.node may have >16 unique IDs
// slave.node will only see 4 ID bits
slave.node := AXI4IdIndexer(4) := master.node
```

注意：

AXI4IdIndexer 将在 slave 接口上创建一个 `user` 字段，因为它在此字段中存储 master 请求的 ID。如果连接到没有 `user` 字段的 AXI4 接口，您需要使用 [AXI4UserYanker](https://chipyard.readthedocs.io/en/stable/TileLink-Diplomacy-Reference/Widgets.html#axi4useryanker)。

## TLWidthWidget

该小部件更改 TileLink 接口的物理宽度。TileLink 接口的宽度由 manager 配置，但有时您希望 client 看到特定的宽度。

参数：
    1. `innerBeatBytes: Int` - tclien 看到的物理宽度（以字节为单位）

用法示例：

```Scala
// Assume the manager node sets beatBytes to 8
// With WidthWidget, client sees beatBytes of 4
manager.node := TLWidthWidget(4) := client.node
```

## TLFIFOFixer

声明 FIFO 域的 TileLink manager 必须确保从已请求 FIFO 排序的 client 向该域发出的所有请求都按顺序看到响应。然而，它们只能控制自己响应的顺序，而无法控制这些响应如何与同一 FIFO 域中其他 manager 的响应交织。 TLFIFOFixer 负责确保 manager 之间的 FIFO 顺序。

参数：
    1. `policy: TLFIFOFixer.Policy` -（可选）TLFIFOFixer 将在哪些 manager 上强制执行排序？ （默认：`TLFIFOFixer.all`）

`policy` 的可能值是：
    1. `TLFIFOFixer.all` - 所有 manager（包括那些没有 F​​IFO 域的 manager）都将保证排序。
    2. `TLFIFOFixer.allFIFO` - 所有定义 FIFO 域的 manager 都将保证排序。
    3. `TLFIFOFixer.allVolatile` - RegionType 为 `VOLATILE`、`PUT_EFFECTS` 或 `GET_EFFECTS` 的所有 amnager 都将保证排序（有关区域类型的说明，请参阅 [Manager Node](https://chipyard.readthedocs.io/en/stable/TileLink-Diplomacy-Reference/NodeTypes.html#manager-node)）。

## TLXbar and AXI4Xbar

这些是 TileLink 和 AXI4 的 crossbar 生成器，它将根据 manager/slave 设备中定义的地址将请求从 TL client/AXI4 master 节点路由到 TL manager/AXI4 slave 节点。通常，这些是在没有参数的情况下构造的。但是，您可以更改仲裁策略，该策略决定哪些 client 端口在仲裁器中具有优先权。默认策略是 `TLArbiter.roundRobin`，但如果您想要固定的仲裁优先级，可以将其更改为 `TLArbiter.lowestIndexFirst`。

参数：所有参数都是可选的。

   1. `arbitrationPolicy: TLArbiter.Policy` - 要使用的仲裁策略。
   2. `maxFlightPerId：Int` -（仅限 AXI4）一次可以进行的具有相同 ID 的事务数。 （默认值：7）。
   3. `awQueueDepth：Int` -（仅限 AXI4）写入地址队列的深度。 （默认值：2）。

用法示例：

```Scala
// Instantiate the crossbar lazy module
val tlBus = LazyModule(new TLXbar)

// Connect a single input edge
tlBus.node := tlClient0.node
// Connect multiple input edges
tlBus.node :=* tlClient1.node

// Connect a single output edge
tlManager0.node := tlBus.node
// Connect multiple output edges
tlManager1.node :*= tlBus.node

// Instantiate a crossbar with lowestIndexFirst arbitration policy
// Yes, we still use the TLArbiter singleton even though this is AXI4
val axiBus = LazyModule(new AXI4Xbar(TLArbiter.lowestIndexFirst))

// The connections work the same as TL
axiBus.node := axiClient0.node
axiBus.node :=* axiClient1.node
axiManager0.node := axiBus.node
axiManager1.node :*= axiBus.node
```

## TLToAXI4 and AXI4ToTL

这些是 TileLink 和 AXI4 协议之间的转换器。TLToAXI4 采用 TileLink client 并连接到 AXI4 slave 设备。 AXI4ToTL 采用 AXI4 master 设备并连接到 TileLink manager。通常，您不想覆盖这些小部件的构造函数的默认参数。

用法示例：

```Scala
axi4slave.node :=
    AXI4UserYanker() :=
    AXI4Deinterleaver(64) :=
    TLToAXI4() :=
    tlclient.node

tlmanager.node :=
    AXI4ToTL() :=
    AXI4UserYanker() :=
    AXI4Fragmenter() :=
    axi4master.node
```

您需要在 TLToAXI4 转换器之后添加 [AXI4Deinterleaver](https://chipyard.readthedocs.io/en/stable/TileLink-Diplomacy-Reference/Widgets.html#axi4deinterleaver)，因为它无法处理交错读取响应。 TLToAXI4 转换器还使用 AXI4 用户字段来存储一些信息，因此如果您想连接到没有用户字段的 AXI4 端口，则需要 [AXI4UserYanker](https://chipyard.readthedocs.io/en/stable/TileLink-Diplomacy-Reference/Widgets.html#axi4useryanker)。

在将 AXI4 端口连接到 AXI4ToTL 小部件之前，您需要添加 [AXI4Fragmenter](https://chipyard.readthedocs.io/en/stable/TileLink-Diplomacy-Reference/Widgets.html#axi4fragmenter) 和 [AXI4UserYanker](https://chipyard.readthedocs.io/en/stable/TileLink-Diplomacy-Reference/Widgets.html#axi4useryanker)，因为转换器无法处理多节拍事务或用户字段。

## TLROM

TLROM 小部件提供了可以使用 TileLink 访问的只读存储器。注意：这个小部件位于 `freechips.rocketchip.devices.tilelink` 包中，而不是像其他小部件一样位于 `freechips.rocketchip.tilelink` 包中。

参数：
    1. `base：BigInt` - 内存的基地址。
    2. `size: Int` - 内存大小（以字节为单位）。
    3. `contentDelayed: => Seq[Byte]` - 调用时生成 ROM 字节内容的函数。
    4. `executable: Boolean` - （可选）指定 CPU 是否可以从 ROM 获取指令（默认值：true）。
    5. `beatBytes：Int` - （可选）接口的宽度（以字节为单位）。 （默认值：4）。
    6. `resources: Seq[Resource]` - （可选）添加到设备树的资源序列。

用法示例：

```Scala
val rom = LazyModule(new TLROM(
  base = 0x100A0000,
  size = 64,
  contentsDelayed = Seq.tabulate(64) { i => i.toByte },
  beatBytes = 8))
rom.node := TLFragmenter(8, 64) := client.node
```

支持的操作：

TLROM 仅支持单拍读取。如果要执行多节拍读取，则应在 ROM 前面附加一个 TLFragmenter。

## TLRAM and AXI4RAM

TLRAM 和 AXI4RAM 小部件提供作为 SRAM 实现的读写存储器。

参数：
    1. `address: AddressSet` - 该 RAM 将覆盖的地址范围。
    2. `cacheable: Boolean` -（可选）此 RAM 的内容是否可以缓存（默认值：true）。
    3. `executable: Boolean` - （可选）该 RAM 的内容是否可以作为指令获取（默认值：true）。
    4. `beatBytes：Int` -（可选）TL/AXI4 接口的宽度（以字节为单位）（默认值：4）。
    5. `atomics: Boolean` - （可选，仅限 TileLink）RAM 是否支持原子操作？（默认值：假）

用法示例：

```Scala
val xbar = LazyModule(new TLXbar)

val tlram = LazyModule(new TLRAM(
  address = AddressSet(0x1000, 0xfff)))

val axiram = LazyModule(new AXI4RAM(
  address = AddressSet(0x2000, 0xfff)))

tlram.node := xbar.node
axiram := TLToAXI4() := xbar.node
```
支持的操作：

TLRAM 仅支持单拍 TL-UL 请求。如果将 `atomics` 设置为true，它还将支持逻辑和算术运算。如果您想要多节拍读/写，请使用 `TLFragmenter`。

AXI4RAM 仅支持 AXI4-Lite 操作，因此不支持多节拍读/写和小于全角的读/写。如果您想使用完整的 AXI4 协议，请使用 `AXI4Fragmenter`。