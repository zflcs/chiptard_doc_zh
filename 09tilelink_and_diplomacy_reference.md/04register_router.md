# Register Router

内存映射设备通常遵循通用模式。它们向 CPU 公开一组寄存器。通过写入寄存器，CPU 可以更改设备的设置或发送命令。通过读取寄存器，CPU 可以查询设备的状态或检索结果。

虽然设计人员可以手动实例化 manager 节点并编写用于公开寄存器本身的逻辑，但使用 RocketChip 的 `regmap` 接口要容易得多，它可以生成大部分粘合逻辑。

对于 TileLink 设备，您可以通过扩展 `TLRegisterRouter` 类来使用 `regmap` 接口，如 [MMIO Peripherals](https://chipyard.readthedocs.io/en/stable/Customization/MMIO-Peripherals.html#mmio-accelerators) 中所示，或者您可以创建常规 LazyModule 并实例化 `TLRegisterNode`。本节将重点介绍第二种方法。

## Basic Usage

```Scala
import chisel3._
import chisel3.util._
import org.chipsalliance.cde.config.Parameters
import freechips.rocketchip.diplomacy._
import freechips.rocketchip.regmapper._
import freechips.rocketchip.tilelink.TLRegisterNode

class MyDeviceController(implicit p: Parameters) extends LazyModule {
  val device = new SimpleDevice("my-device", Seq("tutorial,my-device0"))
  val node = TLRegisterNode(
    address = Seq(AddressSet(0x10028000, 0xfff)),
    device = device,
    beatBytes = 8,
    concurrency = 1)

  lazy val module = new LazyModuleImp(this) {
    val bigReg = RegInit(0.U(64.W))
    val mediumReg = RegInit(0.U(32.W))
    val smallReg = RegInit(0.U(16.W))

    val tinyReg0 = RegInit(0.U(4.W))
    val tinyReg1 = RegInit(0.U(4.W))

    node.regmap(
      0x00 -> Seq(RegField(64, bigReg)),
      0x08 -> Seq(RegField(32, mediumReg)),
      0x0C -> Seq(RegField(16, smallReg)),
      0x0E -> Seq(
        RegField(4, tinyReg0),
        RegField(4, tinyReg1)))
  }
}
```

上面的代码示例显示了一个简单的 lazy module，它使用 `TLRegisterNode` 来内存映射不同大小的硬件寄存器。构造函数有两个必需参数：`address`（寄存器的基地址）和 `device`（设备树条目）。还有两个可选参数。 `beatBytes` 参数是以字节为单位的接口宽度。默认值为 4 字节。`concurrency` 参数是 TileLink 请求的内部队列的大小。默认情况下，该值为0，这意味着不会有队列。如果您希望解耦寄存器访问的请求和响应，则该值必须大于 0。这在 [Using Functions](https://chipyard.readthedocs.io/en/stable/TileLink-Diplomacy-Reference/Register-Router.html#using-functions) 中讨论。

与节点交互的主要方式是调用 `regmap` 方法，该方法采用一系列对。该对的第一个元素是距基地址的偏移量。第二个是 `RegField` 对象序列，每个对象映射一个不同的寄存器。 `RegField` 构造函数有两个参数。第一个参数是以位为单位的寄存器宽度。第二个是寄存器本身。

由于参数是一个序列，因此您可以将多个 RegField 对象与偏移量关联起来。如果这样做，则在访问偏移量时会并行读取或写入寄存器。寄存器采用小端顺序，因此列表中的第一个寄存器对应于写入值中的最低有效位。在此示例中，如果 CPU 将值 0xAB 写入偏移量 0x0E，则 `tinyReg0` 将获得值 0xB，`tinyReg1` 将获得值 0xA。

## Decoupled Interfaces

有时您可能想要执行除从硬件寄存器读取和写入之外的其他操作。 `RegField` 接口还提供对 `DeCoupledIO` 接口读写的支持。例如，您可以像这样实现硬件 FIFO。

```Scala
    // 4-entry 64-bit queue
    val queue = Module(new Queue(UInt(64.W), 4))

    node.regmap(
      0x00 -> Seq(RegField(64, queue.io.deq, queue.io.enq)))
```

`RegField` 构造函数的这个变体采用三个参数而不是两个。第一个参数仍然是位宽度。第二个是读取的解耦接口。第三个是要写入的解耦接口。在此示例中，写入“寄存器”会将数据推入队列，读取“寄存器”会将数据从队列中弹出。

您无需同时指定寄存器的读取和写入。您还可以创建只读或只写寄存器。因此，对于前面的示例，如果您希望入队和出队使用不同的地址，则可以编写以下内容。

```Scala
    node.regmap(
      0x00 -> Seq(RegField.r(64, queue.io.deq)),
      0x08 -> Seq(RegField.w(64, queue.io.enq)))
```

只读寄存器功能也可用于读取非寄存器信号。

```Scala
val constant = 0xf00d.U

node.regmap(
  0x00 -> Seq(RegField.r(8, constant)))
```

## Using Functions

您还可以使用函数创建寄存器。例如，假设您要创建一个计数器，该计数器在写入时递增并在读取时递减。

```Scala
    val counter = RegInit(0.U(64.W))

    def readCounter(ready: Bool): (Bool, UInt) = {
      when (ready) { counter := counter - 1.U }
      // (ready, bits)
      (true.B, counter)
    }

    def writeCounter(valid: Bool, bits: UInt): Bool = {
      when (valid) { counter := counter + 1.U }
      // Ignore bits
      // Return ready
      true.B
    }

    node.regmap(
      0x00 -> Seq(RegField.r(64, readCounter(_))),
      0x08 -> Seq(RegField.w(64, writeCounter(_, _))))
```

这里的功能本质上与解耦接口相同。读取函数传递 `ready` 信号并返回 `valid` 和 `bits` 信号。写入函数传递 `valid` 和 `bits` 并返回 `ready`。

您还可以传递解耦读/写请求和响应的函数。请求将显示为解耦输入，响应将显示为解耦输出。举例来说，如果我们想对前面的示例执行此操作。

```Scala
    val counter = RegInit(0.U(64.W))

    def readCounter(ivalid: Bool, oready: Bool): (Bool, Bool, UInt) = {
      val responding = RegInit(false.B)

      when (ivalid && !responding) { responding := true.B }

      when (responding && oready) {
        counter := counter - 1.U
        responding := false.B
      }

      // (iready, ovalid, obits)
      (!responding, responding, counter)
    }

    def writeCounter(ivalid: Bool, oready: Bool, ibits: UInt): (Bool, Bool) = {
      val responding = RegInit(false.B)

      when (ivalid && !responding) { responding := true.B }

      when (responding && oready) {
        counter := counter + 1.U
        responding := false.B
      }

      // (iready, ovalid)
      (!responding, responding)
    }

    node.regmap(
      0x00 -> Seq(RegField.r(64, readCounter(_, _))),
      0x08 -> Seq(RegField.w(64, writeCounter(_, _, _))))
```

在每个函数中，我们设置一个状态变量 `responding`。当该值为 false 时，该函数已准备好接受请求；当该值为 true 时，该函数正在发送响应。

在此变体中，读取和写入都接受 valid 输入并返回 ready 输出。唯一的区别是 bits 是读取的输入和写入的输出。

为了使用此变体，您需要将 `concurrency` 设置为大于 0 的值。

## Register Routers for Other Protocols

register router 接口的一项有用功能是您可以轻松更改正在使用的协议。例如，在 [Basic Usage](https://chipyard.readthedocs.io/en/stable/TileLink-Diplomacy-Reference/Register-Router.html#basic-usage) 的第一个示例中，您可以简单地将 `TLRegisterNode` 更改为 `AXI4RegisterNode`。

```Scala
import freechips.rocketchip.amba.axi4.AXI4RegisterNode

class MyAXI4DeviceController(implicit p: Parameters) extends LazyModule {
  val node = AXI4RegisterNode(
    address = AddressSet(0x10029000, 0xfff),
    beatBytes = 8,
    concurrency = 1)

  lazy val module = new LazyModuleImp(this) {
    val bigReg = RegInit(0.U(64.W))
    val mediumReg = RegInit(0.U(32.W))
    val smallReg = RegInit(0.U(16.W))

    val tinyReg0 = RegInit(0.U(4.W))
    val tinyReg1 = RegInit(0.U(4.W))

    node.regmap(
      0x00 -> Seq(RegField(64, bigReg)),
      0x08 -> Seq(RegField(32, mediumReg)),
      0x0C -> Seq(RegField(16, smallReg)),
      0x0E -> Seq(
        RegField(4, tinyReg0),
        RegField(4, tinyReg1)))
  }
}
```

除了 AXI4 节点不接受 `device` 参数，并且只能有一个地址集而不是多个之外，其他一切都没有改变。

