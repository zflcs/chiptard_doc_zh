# Adding a custom core

您可能希望将自定义 RISC-V 内核集成到 Chipyard 框架中。本文档页面提供了有关如何实现此目标的分步说明。

> **注意**：目前除 Rocket 和 BOOM 之外的内核不支持 RoCC。如果需要使用RoCC，请使用Rocket或BOOM作为RoCC基础核心。

> **注意**：此页面包含指向包含 Rocket chip 存储库中重要定义的文件的链接，该存储库与 Chipyard 分开维护。如果您发现本页代码与源文件中的代码有任何差异，请通过 GitHub issues 报告！

## Wrap Verilog Module with Blackbox (Optional)

由于 Chipyard 使用 Scala 和 Chisel，如果你的核心的顶层模块不在 Chisel 中，你首先需要为其创建一个 Verilog 黑盒，以便 Chipyard 可以处理它。有关说明，请参阅 [Incorporating Verilog Blocks](https://chipyard.readthedocs.io/en/stable/Customization/Incorporating-Verilog-Blocks.html#incorporating-verilog-blocks)。

## Create Parameter Case Classes

Chipyard 将为它在 `TilesLocated(InSubsystem)` key 中发现的每个 `InstantiableTileParams` 对象生成一个核心。该对象派生自“TileParams”，这是一个包含创建 tile 所需信息的 trait。所有核心都必须有自己的 `InstantiableTileParams` 实现，以及作为 `TileParams` 中的字段传递的 `CoreParams`。

`TileParams` 保存 tile 的参数，其中包括 tile 中所有组件的参数（例如核心、缓存、MMU 等），而 `CoreParams` 包含 tile 上特定于核心的参数。它们必须作为 case 类实现，其中的字段可以被其他配置片段作为构造函数参数覆盖。有关要实现的变量列表，请参阅页面底部的附录。您还可以向其中添加自定义字段，但应始终首选标准字段。

`InstantiableTileParams[TileType]` 将 `TileType` 的构造函数保存在 `TileParams` 字段之上，其中 `TileType` 是 tile 类（请参阅下一节）。所有自定义核心还需要在其 tile case 类中实现 `instantiate()` 以返回 tile 类 `TileType` 的新实例。

`TileParams`（在文件 [BaseTile.scala](https://github.com/chipsalliance/rocket-chip/blob/master/src/main/scala/tile/BaseTile.scala) 中）、`InstantiableTileParams`（在文件 [BaseTile.scala ](https://github.com/chipsalliance/rocket-chip/blob/master/src/main/scala/tile/BaseTile.scala) 中）、`CoreParams`（在文件 [Core.scala](https://github.com/chipsalliance/rocket-chip/blob/master/src/main/scala/tile/Core.scala) 中）和 `FPUParams`（在文件 [FPU.scala](https://github.com/chipsalliance/rocket-chip/blob/master/src/main/scala/tile/FPU.scala) 中）包含以下字段：

```Scala
trait TileParams {
  val core: CoreParams                  // Core parameters (see below)
  val icache: Option[ICacheParams]      // Rocket specific: I1 cache option
  val dcache: Option[DCacheParams]      // Rocket specific: D1 cache option
  val btb: Option[BTBParams]            // Rocket specific: BTB / branch predictor option
  val hartId: Int                       // Hart ID: Must be unique within a design config (This MUST be a case class parameter)
  val beuAddr: Option[BigInt]           // Rocket specific: Bus Error Unit for Rocket Core
  val blockerCtrlAddr: Option[BigInt]   // Rocket specific: Bus Blocker for Rocket Core
  val name: Option[String]              // Name of the core
}

abstract class InstantiableTileParams[TileType <: BaseTile] extends TileParams {
  def instantiate(crossing: TileCrossingParamsLike, lookup: LookupByHartIdImpl)
                (implicit p: Parameters): TileType
}

trait CoreParams {
  val bootFreqHz: BigInt              // Frequency
  val useVM: Boolean                  // Support virtual memory
  val useUser: Boolean                // Support user mode
  val useSupervisor: Boolean          // Support supervisor mode
  val useDebug: Boolean               // Support RISC-V debug specs
  val useAtomics: Boolean             // Support A extension
  val useAtomicsOnlyForIO: Boolean    // Support A extension for memory-mapped IO (may be true even if useAtomics is false)
  val useCompressed: Boolean          // Support C extension
  val useVector: Boolean = false      // Support V extension
  val useSCIE: Boolean                // Support custom instructions (in custom-0 and custom-1)
  val useRVE: Boolean                 // Use E base ISA
  val mulDiv: Option[MulDivParams]    // *Rocket specific: M extension related setting (Use Some(MulDivParams()) to indicate M extension supported)
  val fpu: Option[FPUParams]          // F and D extensions and related setting (see below)
  val fetchWidth: Int                 // Max # of insts fetched every cycle
  val decodeWidth: Int                // Max # of insts decoded every cycle
  val retireWidth: Int                // Max # of insts retired every cycle
  val instBits: Int                   // Instruction bits (if 32 bit and 64 bit are both supported, use 64)
  val nLocalInterrupts: Int           // # of local interrupts (see SiFive interrupt cookbook)
  val nPMPs: Int                      // # of Physical Memory Protection units
  val pmpGranularity: Int             // Size of the smallest unit of region for PMP unit (must be power of 2)
  val nBreakpoints: Int               // # of hardware breakpoints supported (in RISC-V debug specs)
  val useBPWatch: Boolean             // Support hardware breakpoints
  val nPerfCounters: Int              // # of supported performance counters
  val haveBasicCounters: Boolean      // Support basic counters defined in the RISC-V counter extension
  val haveFSDirty: Boolean            // If true, the core will set FS field in mstatus CSR to dirty when appropriate
  val misaWritable: Boolean           // Support writable misa CSR (like variable instruction bits)
  val haveCFlush: Boolean             // Rocket specific: enables Rocket's custom instruction extension to flush the cache
  val nL2TLBEntries: Int              // # of L2 TLB entries
  val mtvecInit: Option[BigInt]       // mtvec CSR (of V extension) initial value
  val mtvecWritable: Boolean          // If mtvec CSR is writable

  // Normally, you don't need to change these values (except lrscCycles)
  def customCSRs(implicit p: Parameters): CustomCSRs = new CustomCSRs

  def hasSupervisorMode: Boolean = useSupervisor || useVM
  def instBytes: Int = instBits / 8
  def fetchBytes: Int = fetchWidth * instBytes
  // Rocket specific: Longest possible latency of Rocket core D1 cache. Simply set it to the default value 80 if you don't use it.
  def lrscCycles: Int

  def dcacheReqTagBits: Int = 6

  def minFLen: Int = 32
  def vLen: Int = 0
  def sLen: Int = 0
  def eLen(xLen: Int, fLen: Int): Int = xLen max fLen
  def vMemDataBits: Int = 0
}

case class FPUParams(
  minFLen: Int = 32,          // Minimum floating point length (no need to change)
  fLen: Int = 64,             // Maximum floating point length, use 32 if only single precision is supported
  divSqrt: Boolean = true,    // Div/Sqrt operation supported
  sfmaLatency: Int = 3,       // Rocket specific: Fused multiply-add pipeline latency (single precision)
  dfmaLatency: Int = 4        // Rocket specific: Fused multiply-add pipeline latency (double precision)
)
```

这里的大多数字段（标记为“Rocket specific”）最初是为 Rocket core 设计的，因此包含一些特定于实现的细节，但其中许多字段足够通用，足以对其他核心有用。您可以忽略任何标记为“Rocket specific”的字段并使用它们的默认值；但是，如果您需要存储与这些“Rocket specific”字段类似的含义或用法的附加信息，建议使用这些字段，而不是创建自己的自定义字段。

您还需要一个 `CanAttachTile` 类来将 tile 配置添加到配置系统中，格式如下：

```Scala
case class MyTileAttachParams(
  tileParams: MyTileParams,
  crossingParams: RocketCrossingParams
) extends CanAttachTile {
  type TileType = MyTile
  val lookup = PriorityMuxHartIdFromSeq(Seq(tileParams))
}
```

在详细阐述过程中，Chipyard 将在配置系统中查找 `CanAttachTile` 的子类，并根据该类中的参数为找到的每个此类类实例化一个 tile。

> **注意**：实现可以选择忽略此处的某些字段或以非标准方式使用它们，但使用不准确的值可能会破坏依赖它们的 Chipyard 组件（例如，支持的 ISA 扩展的不准确指示将导致生成不正确的测试套件）以及使用它们的任何自定义模块。始终记录您在核心实现中忽略或更改用法的任何字段，并且如果您正在实现将查找这些配置值的其他设备，也记录它们。 “Rocket specific”值通常可以安全地忽略，但如果使用它们，则应该记录它们。

## Create Tile Class

在 Chipyard 中，所有 Tiles 都被 diplomatically 实例化。在第一阶段，评估指定 tile 到系统互连的 diplomatic 节点，而在第二个“模块实现”阶段，详细阐述硬件。有关更多详细信息，请参阅 [TileLink and Diplomacy Reference](https://chipyard.readthedocs.io/en/stable/TileLink-Diplomacy-Reference/index.html#tilelink-and-diplomacy)。在此步骤中，您需要为核心实现一个 tile 类，该类指定核心参数的约束以及与其他 diplomatic 节点的连接。该类通常仅包含 Diplomacy/TileLink 代码，Chisel RTL 代码不应放在这里。

所有 tile 类都实现 `BaseTile``，并且通常会实现SinksExternalInterrupts` 和 `SourcesExternalNotifications`，这允许 tile 接受外部中断。典型的 tile 具有以下形式：

```Scala
class MyTile(
  val myParams: MyTileParams,
  crossing: ClockCrossingType,
  lookup: LookupByHartIdImpl,
  q: Parameters)
  extends BaseTile(myParams, crossing, lookup, q)
  with SinksExternalInterrupts
  with SourcesExternalNotifications
{

  // Private constructor ensures altered LazyModule.p is used implicitly
  def this(params: MyTileParams, crossing: HierarchicalElementCrossingParamsLike, lookup: LookupByHartIdImpl)(implicit p: Parameters) =
    this(params, crossing.crossingType, lookup, p)

  // Require TileLink nodes
  val intOutwardNode = None
  val masterNode = visibilityNode
  val slaveNode = TLIdentityNode()

  // Implementation class (See below)
  override lazy val module = new MyTileModuleImp(this)

  // Required entry of CPU device in the device tree for interrupt purpose
  val cpuDevice: SimpleDevice = new SimpleDevice("cpu", Seq("my-organization,my-cpu", "riscv")) {
    override def parent = Some(ResourceAnchors.cpus)
    override def describe(resources: ResourceBindings): Description = {
      val Description(name, mapping) = super.describe(resources)
      Description(name, mapping ++
                        cpuProperties ++
                        nextLevelCacheProperty ++
                        tileProperties)
    }
  }

  ResourceBinding {
    Resource(cpuDevice, "reg").bind(ResourceAddress(tileId))
  }

  // TODO: Create TileLink nodes and connections here.
```

## Connect TileLink Buses

Chipyard 使用 TileLink 作为其板载总线协议。如果您的内核不使用 TileLink，则需要在 Tile 模块内的内核内存协议和 TileLink 之间插入转换器。在 tile 类中。下面是如何使用 Rocket chip 提供的转换器将使用 AXI4 的内核连接到 TileLink 总线的示例：

```Scala
 (tlMasterXbar.node  // tlMasterXbar is the bus crossbar to be used when this core / tile is acting as a master; otherwise, use tlSlaveXBar
    := memoryTap
    := TLBuffer()
    := TLFIFOFixer(TLFIFOFixer.all) // fix FIFO ordering
    := TLWidthWidget(masterPortBeatBytes) // reduce size of TL
    := AXI4ToTL() // convert to TL
    := AXI4UserYanker(Some(2)) // remove user field on AXI interface. need but in reality user intf. not needed
    := AXI4Fragmenter() // deal with multi-beat xacts
    := memAXI4Node) // The custom node, see below
```

请记住，您可能不需要所有这些中间小部件。有关每个中间小部件的含义，请参阅 [Diplomatic Widgets](https://chipyard.readthedocs.io/en/stable/TileLink-Diplomacy-Reference/Widgets.html#diplomatic-widgets)。如果您使用的是 TileLink，那么您只需要组件使用的 tap 节点和 TileLink 节点。Chipyard 还提供 AHB、APB 和 AXIS 的转换器，并且大多数 AXI4 小部件都有针对这些总线协议的等效小部件；有关更多信息，请参阅 `generators/rocket-chip/src/main/scala/amba` 中的源文件。

如果您使用其他总线协议，您可以使用 `generators/rocket-chip/src/main/scala/amba` 中的文件作为模板来实现自己的转换器，但不建议这样做，除非您熟悉 TileLink。

`memAXI4Node` 是 AXI4 主节点，在我们的示例中定义如下：

```Scala
  // # of bits used in TileLink ID for master node. 4 bits can support 16 master nodes, but you can have a longer ID if you need more.
  val idBits = 4
  val memAXI4Node = AXI4MasterNode(
    Seq(AXI4MasterPortParameters(
      masters = Seq(AXI4MasterParameters(
        name = "myPortName",
        id = IdRange(0, 1 << idBits))))))
  val memoryTap = TLIdentityNode() // Every bus connection should have their own tap node
```

其中 `portName` 和 `idBits`（表示端口 ID 的位数）是该 tile 提供的参数。请务必阅读 [TileLink Node Types](https://chipyard.readthedocs.io/en/stable/TileLink-Diplomacy-Reference/NodeTypes.html#node-types) 以查看 Chipyard 支持的节点类型及其参数！

另外，默认情况下，当主从连接离开 tile 时，总线上的主从连接都有边界缓冲区，您可以覆盖以下两个函数来控制如何缓冲总线请求/响应：（您可以找到定义文件 [BaseTile.scala](https://github.com/chipsalliance/rocket-chip/blob/master/src/main/scala/tile/BaseTile.scala) 中的 `BaseTile` 类中的这两个函数）

```Scala
// By default, their value is "TLBuffer(BufferParams.none)".
protected def makeMasterBoundaryBuffers(implicit p: Parameters): TLBuffer
protected def makeSlaveBoundaryBuffers(implicit p: Parameters): TLBuffer
```

您可以在 [Diplomatic Widgets](https://chipyard.readthedocs.io/en/stable/TileLink-Diplomacy-Reference/Widgets.html#diplomatic-widgets) 中找到有关 TLBuffer 的更多信息。

## Create Implementation Class

Implementation 类包含参数化的实际硬件，该硬件取决于 Diplomacy 框架根据 Tile 类中提供的信息解析的值。该类通常包含 Chisel RTL 代码。如果您的核心采用 Verilog，则需要实例化包装 Verilog 实现的黑盒类，并将其与总线和其他组件连接。此类中不应包含 Diplomacy/TileLink 代码；您应该只连接 TileLink 接口或其他外交定义的组件中的 IO 信号，这些组件位于 tile 类中。

您的核心的实现类具有以下形式：

```Scala
class MyTileModuleImp(outer: MyTile) extends BaseTileModuleImp(outer){
  // annotate the parameters
  Annotated.params(this, outer.myParams)

  // TODO: Create the top module of the core and connect it with the ports in "outer"

  // If your core is in Verilog (assume your blackbox is called "MyCoreBlackbox"), instantiate it here like
  //   val core = Module(new MyCoreBlackbox(params...))
  // (as described in the blackbox tutorial) and connect appropriate signals. See the blackbox tutorial
  // (link on the top of the page) for more info.
  // You can look at https://github.com/ucb-bar/cva6-wrapper/blob/master/src/main/scala/CVA6Tile.scala
  // for a Verilog example.

  // If your core is in Chisel, you can simply instantiate the top module here like other Chisel module
  // and connect appropriate signal. You can even implement this class as your top module.
  // See https://github.com/riscv-boom/riscv-boom/blob/master/src/main/scala/common/tile.scala and
  // https://github.com/chipsalliance/rocket-chip/blob/master/src/main/scala/tile/RocketTile.scala for
  // Chisel example.
```

如果您创建 AXI4 节点（或等效节点），则需要将它们连接到您的核心。您可以像这样连接端口：

```Scala
  outer.memAXI4Node.out foreach { case (out, edgeOut) =>
    // Connect your module IO port to "out"
    // The type of "out" here is AXI4Bundle, which is defined in generators/rocket-chip/src/main/scala/amba/axi4/Bundles.scala
    // Please refer to this file for the definition of the ports.
    // If you are using APB, check APBBundle in generators/rocket-chip/src/main/scala/amba/apb/Bundles.scala
    // If you are using AHB, check AHBSlaveBundle or AHBMasterBundle in generators/rocket-chip/src/main/scala/amba/ahb/Bundles.scala
    // (choose one depends on the type of AHB node you create)
    // If you are using AXIS, check AXISBundle and AXISBundleBits in generators/rocket-chip/src/main/scala/amba/axis/Bundles.scala
  }
```

## Connect Interrupt

Chipyard 允许 tile 接收来自其他设备的中断或发起中断以通知其他内核/设备。在继承 `SinksExternalInterrupts` 的 tile 中，可以创建一个 `TileInterrupts` 对象（Chisel 包）并以该对象作为参数调用 `decodeCoreInterrupts()`。请注意，您应该在实现类中调用此函数，因为它返回 RTL 代码使用的 Chisel 包。然后，您可以从我们上面创建的 `TileInterrupts` 包中读取中断位。`TileInterrupts` 的定义（在文件 [Interrupts.scala](https://github.com/chipsalliance/rocket-chip/blob/master/src/main/scala/tile/Interrupts.scala) 中）是：

```Scala
class TileInterrupts(implicit p: Parameters) extends CoreBundle()(p) {
  val debug = Bool() // debug interrupt
  val mtip = Bool() // Machine level timer interrupt
  val msip = Bool() // Machine level software interrupt
  val meip = Bool() // Machine level external interrupt
  val seip = usingSupervisor.option(Bool()) // Valid only if supervisor mode is supported
  val lip = Vec(coreParams.nLocalInterrupts, Bool())  // Local interrupts
}
```

以下是如何在实现类中连接这些信号的示例：

```Scala
  // For example, our core support debug interrupt and machine-level interrupt, and suppose the following two signals
  // are the interrupt inputs to the core. (DO NOT COPY this code - if your core treat each type of interrupt differently,
  // you need to connect them to different interrupt ports of your core)
  val debug_i = Wire(Bool())
  val mtip_i = Wire(Bool())
  // We create a bundle here and decode the interrupt.
  val int_bundle = new TileInterrupts()
  outer.decodeCoreInterrupts(int_bundle)
  debug_i := int_bundle.debug
  mtip_i := int_bundle.meip & int_bundle.msip & int_bundle.mtip
```

此外，该 tile 还可以通过从实现类调用 `SourcesExternalNotifications` 中的以下函数来通知其他内核或设备发生某些事件：（这些函数可以在文件 [Interrupts.scala](https://github.com/chipsalliance/rocket-chip/blob/master/src/main/scala/tile/Interrupts.scala) 的特征 `SourcesExternalNotifications` 中找到）

```Scala
def reportHalt(could_halt: Option[Bool]) // Triggered when there is an unrecoverable hardware error (halt the machine)
def reportHalt(errors: Seq[CanHaveErrors]) // Varient for standard error bundle (Rocket specific: used only by cache when there's an ECC error)
def reportCease(could_cease: Option[Bool], quiescenceCycles: Int = 8) // Triggered when the core stop retiring instructions (like clock gating)
def reportWFI(could_wfi: Option[Bool]) // Triggered when a WFI instruction is executed
```

这是有关如何使用这些函数引发中断的示例。

```Scala
  // This is a demo. You should call these function according to your core
  // Suppose that the following signal is from the decoder indicating a WFI instruction is received.
  val wfi_o = Wire(Bool())
  outer.reportWFI(Some(wfi_o))
  // Suppose that the following signal indicate an unreconverable hardware error.
  val halt_o = Wire(Bool())
  outer.reportHalt(Some(halt_o))
  // Suppose that our core never stall for a long time / stop retiring. Use None to indicate that this interrupt never fires.
  outer.reportCease(None)
```

## Create Config Fragments to Integrate the Core

要在 Chipyard 配置中使用您的核心，您将需要一个配置片段，它将在当前配置中创建核心的 `TileParams` 对象。此类配置的示例如下：

```Scala
class WithNMyCores(n: Int = 1) extends Config((site, here, up) => {
  case TilesLocated(InSubsystem) => {
    // Calculate the next available hart ID (since hart ID cannot be duplicated)
    val prev = up(TilesLocated(InSubsystem), site)
    val idOffset = up(NumTiles)
    // Create TileAttachParams for every core to be instantiated
    (0 until n).map { i =>
      MyTileAttachParams(
        tileParams = MyTileParams(tileId = i + idOffset),
        crossingParams = RocketCrossingParams()
      )
    } ++ prev
  }
  // Configurate # of bytes in one memory / IO transaction. For RV64, one load/store instruction can transfer 8 bytes at most.
  case SystemBusKey => up(SystemBusKey, site).copy(beatBytes = 8)
  // The # of instruction bits. Use maximum # of bits if your core supports both 32 and 64 bits.
  case XLen => 64
  case NumTiles => up(NumTiles) + n
})
```

Chipyard 在字段 `TilesLocated(InSubsystem)` 中查找 tile 参数，其类型是 `InstantiableTileParams` 列表。此配置片段只是将新的 tile 参数附加到此列表的末尾。

现在您已经完成了为 Chipyard 准备核心的所有步骤！要生成自定义内核，只需按照 [Integrating Custom Chisel Projects into the Generator Build System](https://chipyard.readthedocs.io/en/stable/Customization/Custom-Chisel.html#custom-chisel) 中的说明进行操作，将项目添加到构建系统中，然后按照 [Heterogeneous SoCs](https://chipyard.readthedocs.io/en/stable/Customization/Heterogeneous-SoCs.html#hetero-socs) 中的步骤创建配置。您现在可以为新配置运行最需要的工作流程，就像为内置核心运行一样（取决于您的核心支持的功能）。

如果您想查看集成到 Chipyard 中的完整第三方 Verilog 内核的示例，`generators/ariane/src/main/scala/CVA6Tile.scala` 提供了 CVA6 核的具体示例。请注意，此特定示例包含 AXI 接口与内存一致性系统交互方面的其他细微差别。