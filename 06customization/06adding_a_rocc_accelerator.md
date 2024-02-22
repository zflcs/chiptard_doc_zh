# Adding a RoCC Accelerator

RoCC 加速器是一个可以添加到特定 Rocket 或 BooM 块中的组件。它接收与某个操作码匹配的指令，与核心或 SoC 的其他部分（L1、L2、PTW、FPU）交流，然后选择性地将一个值写回到与指令的 rd 字段对应的寄存器中。RoCC 加速器通过扩展 `LazyRoCC` 类的模块进行实例化。这些模块延迟实例化另一个扩展 `LazyRoCCModule` 类的模块。使用这个额外的间接层，以便 Diplomacy 可以弄清楚如何将 RoCC 模块连接到芯片，而无需提前实例化该模块。lazy module 在 [Cake Pattern / Mixin](https://chipyard.readthedocs.io/en/stable/Chipyard-Basics/Configs-Parameters-Mixins.html#cake-pattern-mixin) 部分中有进一步解释。下面是 RoCC 加速器的最小实例。

```Scala
class CustomAccelerator(opcodes: OpcodeSet)
    (implicit p: Parameters) extends LazyRoCC(opcodes) {
  override lazy val module = new CustomAcceleratorModule(this)
}

class CustomAcceleratorModule(outer: CustomAccelerator)
    extends LazyRoCCModuleImp(outer) {
  val cmd = Queue(io.cmd)
  // The parts of the command are as follows
  // inst - the parts of the instruction itself
  //   opcode
  //   rd - destination register number
  //   rs1 - first source register number
  //   rs2 - second source register number
  //   funct
  //   xd - is the destination register being used?
  //   xs1 - is the first source register being used?
  //   xs2 - is the second source register being used?
  // rs1 - the value of source register 1
  // rs2 - the value of source register 2
  ...
}
```

LazyRoCC 的 `opcodes` 参数是映射到此加速器的一组自定义操作码。下一小节将详细介绍这一点。

`LazyRoCC` 类包含两个 TLOutputNode 实例：`atlNode` 和 `tlNode`。前者与 L1 指令缓存的 backside 一起连接到 tile 本地仲裁器。后者直接连接到 L1-L2 crossbar。模块实现的 IO 包中对应的 Tilelink 端口分别是 `atl` 和 `tl`。

加速器可用的其他接口是 `mem`，它提供对 L1 缓存的访问； `ptw` 提供对页表遍历器的访问；`busy` 信​​号，指示加速器何时仍在处理指令；以及中断信号，可用于中断CPU。

有关不同 IO 的详细信息，请参阅 `generators/rocket-chip/src/main/scala/tile/LazyRoCC.scala` 中的示例。[the RoCC Documentation written by UCSD](https://docs.google.com/document/d/1CH2ep4YcL_ojsa3BVHEW-uwcKh1FlFTjH_kg5v8bxVw/edit) 中还提供了有关每个信号的更多信息，尽管该文档已从树中更新并且可能已过时。

## Accessing Memory via L1 Cache

RoCC 加速器可以通过其所连接的内核的 L1 高速缓存访​​问内存。对于加速器架构来说，这是一个更简单的接口，但可实现的吞吐量通常低于专用 TileLink 端口。

在 `LazyRoCCModuleImp` 中，信号 `io.mem` 是一个 `HellaCacheIO`，它在 `generators/rocket-chip/src/main/scala/rocket/HellaCache.scala` 中定义。

```Scala
class HellaCacheIO(implicit p: Parameters) extends CoreBundle()(p) {
    val req = Decoupled(new HellaCacheReq)
    val s1_kill = Output(Bool()) // kill previous cycle's req
    val s1_data = Output(new HellaCacheWriteData()) // data for previous cycle's req
    val s2_nack = Input(Bool()) // req from two cycles ago is rejected
    val s2_nack_cause_raw = Input(Bool()) // reason for nack is store-load RAW hazard (performance hint)
    val s2_kill = Output(Bool()) // kill req from two cycles ago
    val s2_uncached = Input(Bool()) // advisory signal that the access is MMIO
    val s2_paddr = Input(UInt(paddrBits.W)) // translated address

    val resp = Flipped(Valid(new HellaCacheResp))
    val replay_next = Input(Bool())
    val s2_xcpt = Input(new HellaCacheExceptions)
    val s2_gpa = Input(UInt(vaddrBitsExtended.W))
    val s2_gpa_is_pte = Input(Bool())
    val uncached_resp = tileParams.dcache.get.separateUncachedResp.option(Flipped(Decoupled(new HellaCacheResp)))
    val ordered = Input(Bool())
    val perf = Input(new HellaCachePerfEvents())

    val keep_clock_enabled = Output(Bool()) // should D$ avoid clock-gating itself?
    val clock_enabled = Input(Bool()) // is D$ currently being clocked?
}
```

在较高级别上，您必须使用 `io.mem.req.tag` 来标记通过此接口发送的请求，并且当数据准备就绪时，该标记将返回给您。如果您发出多个请求，响应可能会无序返回，因此您可以使用这些标签来告诉返回的数据。请注意，标记位数由 `dcacheReqTagBits` 控制，通常设置为 6。使用超过 6 位将导致错误或挂起。

## Adding RoCC accelerator to Config

可以通过覆盖配置中的 `BuildRoCC` 参数将 RoCC 加速器添加到内核中。这需要一系列函数来生成 `LazyRoCC` 对象，每个函数对应您要添加的每个加速器。

例如，如果我们想添加之前定义的加速器并向其路由 custom0 和 custom1 指令，我们可以执行以下操作。

```Scala
class WithCustomAccelerator extends Config((site, here, up) => {
  case BuildRoCC => Seq((p: Parameters) => LazyModule(
    new CustomAccelerator(OpcodeSet.custom0 | OpcodeSet.custom1)(p)))
})

class CustomAcceleratorConfig extends Config(
  new WithCustomAccelerator ++
  new RocketConfig)
```

要在程序中添加 RoCC 指令，请使用 `tests/rocc.h` 中提供的 RoCC C 宏。您可以在文件 `tests/accum.c` 和 `charcount.c` 中找到示例。
