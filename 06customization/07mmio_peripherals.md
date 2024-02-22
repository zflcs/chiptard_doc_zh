# MMIO Peripherals

创建 MMIO 外设的最简单方法是使用 `TLRegisterRouter` 或 `AXI4RegisterRouter` 小部件，它抽象了处理互连协议的细节，并提供了一个用于指定内存映射寄存器的便捷接口。由于 Chipyard 和 Rocket Chip SoC 主要使用 Tilelink 作为片上互连协议，因此本节将主要关注设计基于 Tilelink 的外设。但是，请参阅 `generators/chipyard/src/main/scala/example/GCD.scala`，了解如何定义基于 AXI4 的示例外设并通过转换器将其连接到 Tilelink 图。

要创建基于 RegisterRouter 的外设，您需要为配置设置指定参数 case 类、具有额外顶级端口的 bundle trait 以及包含实际 RTL 的模块实现。

在本示例中，我们将展示如何连接计算 GCD 的 MMIO 外设。完整代码可以在 `generators/chipyard/src/main/scala/example/GCD.scala` 中找到。

在本例中，我们使用子模块 `GCDMMIOChiselModule` 来实际执行 GCD。`GCDModule` 类仅创建寄存器并使用 `regmap` 将它们连接起来。

```Scala
class GCDMMIOChiselModule(val w: Int) extends Module
  with HasGCDIO
{
  val s_idle :: s_run :: s_done :: Nil = Enum(3)

  val state = RegInit(s_idle)
  val tmp   = Reg(UInt(w.W))
  val gcd   = Reg(UInt(w.W))

  io.input_ready := state === s_idle
  io.output_valid := state === s_done
  io.gcd := gcd

  when (state === s_idle && io.input_valid) {
    state := s_run
  } .elsewhen (state === s_run && tmp === 0.U) {
    state := s_done
  } .elsewhen (state === s_done && io.output_ready) {
    state := s_idle
  }

  when (state === s_idle && io.input_valid) {
    gcd := io.x
    tmp := io.y
  } .elsewhen (state === s_run) {
    when (gcd > tmp) {
      gcd := gcd - tmp
    } .otherwise {
      tmp := tmp - gcd
    }
  }

  io.busy := state =/= s_idle
}
```

```Scala
trait GCDModule extends HasRegMap {
  val io: GCDTopIO

  implicit val p: Parameters
  def params: GCDParams
  val clock: Clock
  val reset: Reset


  // How many clock cycles in a PWM cycle?
  val x = Reg(UInt(params.width.W))
  val y = Wire(new DecoupledIO(UInt(params.width.W)))
  val gcd = Wire(new DecoupledIO(UInt(params.width.W)))
  val status = Wire(UInt(2.W))

  val impl = if (params.useBlackBox) {
    Module(new GCDMMIOBlackBox(params.width))
  } else {
    Module(new GCDMMIOChiselModule(params.width))
  }

  impl.io.clock := clock
  impl.io.reset := reset.asBool

  impl.io.x := x
  impl.io.y := y.bits
  impl.io.input_valid := y.valid
  y.ready := impl.io.input_ready

  gcd.bits := impl.io.gcd
  gcd.valid := impl.io.output_valid
  impl.io.output_ready := gcd.ready

  status := Cat(impl.io.input_ready, impl.io.output_valid)
  io.gcd_busy := impl.io.busy

  regmap(
    0x00 -> Seq(
      RegField.r(2, status)), // a read-only register capturing current status
    0x04 -> Seq(
      RegField.w(params.width, x)), // a plain, write-only register
    0x08 -> Seq(
      RegField.w(params.width, y)), // write-only, y.valid is set on write
    0x0C -> Seq(
      RegField.r(params.width, gcd))) // read-only, gcd.ready is set on read
}
```

## Advanced Features of RegField Entries

`RegField` 公开了多态 `r` 和 `w` 方法，允许只读和只写内存映射寄存器以多种方式连接到硬件。

1. `RegField.r(2, status)` 用于创建一个 2 位只读寄存器，该寄存器在读取时捕获 `status` 信号的当前值。
2. `RegField.r(params.width, gcd)` 将解耦握手接口 gcd “连接”到只读内存映射寄存器。当通过 MMIO 读取该寄存器时，`ready` 信号被置位。这又通过胶合逻辑连接到 GCD 模块上的 `output_ready`。
3. `RegField.w(params.width, x)` 通过 MMIO 公开一个普通寄存器，但使其只读。
4. `RegField.w(params.width, y)` 将解耦接口信号 `y` 与只写内存映射寄存器相关联，从而导致在写入寄存器时断言 `y.valid`。

由于 `y` 的 ready/valid 信号分别连接到 GCD 模块的 `input_ready` 和 `input_valid` 信号，因此该寄存器映射和胶合逻辑具有在写入 `y` 时触发 GCD 算法的效果。因此，该算法是通过首先写入 `x` 然后对 `y` 执行触发写入来设置的。轮询可用于状态检查。

## Connecting by TileLink

拥有这些类后，您可以通过扩展 `TLRegisterRouter` 并传递适当的参数来构造最终的外设。第一组参数确定寄存器路由器将放置在全局地址映射中的位置以及哪些信息将放置在其设备树条目中。第二组参数是 IO bundle 构造函数，我们通过使用 bundle trait 扩展 `TLRegBundle` 来创建它。最后一组参数是模块构造函数，我们通过使用模块特征扩展 `TLRegModule` 创建它。请注意我们如何创建外设的类似 AXI4 版本。

```Scala
class GCDTL(params: GCDParams, beatBytes: Int)(implicit p: Parameters)
  extends TLRegisterRouter(
    params.address, "gcd", Seq("ucbbar,gcd"),
    beatBytes = beatBytes)(
      new TLRegBundle(params, _) with GCDTopIO)(
      new TLRegModule(params, _, _) with GCDModule)

class GCDAXI4(params: GCDParams, beatBytes: Int)(implicit p: Parameters)
  extends AXI4RegisterRouter(
    params.address,
    beatBytes=beatBytes)(
      new AXI4RegBundle(params, _) with GCDTopIO)(
      new AXI4RegModule(params, _, _) with GCDModule)
```

## Top-level Traits

创建模块后，我们需要将其连接到我们的 SoC。Rocket Chip 使用 cake pattern 来实现这一点。这基本上涉及将代码放置在 traits 中。在 Rocket Chip cake 中，有两种特征：`LazyModule` trait 和 a module implementation trait。

`LazyModule` traits 运行设置代码，该代码必须在所有硬件被详细设计之前执行。对于简单的内存映射外设，这只需要将外设的 TileLink 节点连接到 MMIO crossbar。

```Scala
trait CanHavePeripheryGCD { this: BaseSubsystem =>
  private val portName = "gcd"

  // Only build if we are using the TL (nonAXI4) version
  val gcd_busy = p(GCDKey) match {
    case Some(params) => {
      val gcd = if (params.useAXI4) {
        val gcd = pbus { LazyModule(new GCDAXI4(params, pbus.beatBytes)(p)) }
        pbus.coupleTo(portName) {
          gcd.node :=
          AXI4Buffer () :=
          TLToAXI4 () :=
          // toVariableWidthSlave doesn't use holdFirstDeny, which TLToAXI4() needsx
          TLFragmenter(pbus.beatBytes, pbus.blockBytes, holdFirstDeny = true) := _
        }
        gcd
      } else {
        val gcd = pbus { LazyModule(new GCDTL(params, pbus.beatBytes)(p)) }
        pbus.coupleTo(portName) { gcd.node := TLFragmenter(pbus.beatBytes, pbus.blockBytes) := _ }
        gcd
      }
      val pbus_io = pbus { InModuleBody {
        val busy = IO(Output(Bool()))
        busy := gcd.module.io.gcd_busy
        busy
      }}
      val gcd_busy = InModuleBody {
        val busy = IO(Output(Bool())).suggestName("gcd_busy")
        busy := pbus_io
        busy
      }
      Some(gcd_busy)
    }
    case None => None
  }
}
```

请注意，我们从 register router 创建的 `GCDTL` 类本身就是一个 `LazyModule`。Register routers 有一个名为“node”的 TileLink 节点，我们可以将其连接到 Rocket Chip 总线。这将自动添加外设的地址映射和设备树条目。另请注意我们如何为此外设的 AXI4 版本放置额外的 AXI4 缓冲区和转换器。

使用 InModuleBody 可以将外设的 I/O 传递到 DigitalTop 模块。

## Constructing the DigitalTop and Config

现在我们想将我们的 traits 混合到整个系统中。此代码来自 `generators/chipyard/src/main/scala/DigitalTop.scala`。

```Scala
class DigitalTop(implicit p: Parameters) extends ChipyardSystem
  with testchipip.tsi.CanHavePeripheryUARTTSI // Enables optional UART-based TSI transport
  with testchipip.boot.CanHavePeripheryCustomBootPin // Enables optional custom boot pin
  with testchipip.boot.CanHavePeripheryBootAddrReg // Use programmable boot address register
  with testchipip.cosim.CanHaveTraceIO // Enables optionally adding trace IO
  with testchipip.soc.CanHaveBankedScratchpad // Enables optionally adding a banked scratchpad
  with testchipip.iceblk.CanHavePeripheryBlockDevice // Enables optionally adding the block device
  with testchipip.serdes.CanHavePeripheryTLSerial // Enables optionally adding the backing memory and serial adapter
  with testchipip.soc.CanHavePeripheryChipIdPin // Enables optional pin to set chip id for multi-chip configs
  with sifive.blocks.devices.i2c.HasPeripheryI2C // Enables optionally adding the sifive I2C
  with sifive.blocks.devices.pwm.HasPeripheryPWM // Enables optionally adding the sifive PWM
  with sifive.blocks.devices.uart.HasPeripheryUART // Enables optionally adding the sifive UART
  with sifive.blocks.devices.gpio.HasPeripheryGPIO // Enables optionally adding the sifive GPIOs
  with sifive.blocks.devices.spi.HasPeripherySPIFlash // Enables optionally adding the sifive SPI flash controller
  with sifive.blocks.devices.spi.HasPeripherySPI // Enables optionally adding the sifive SPI port
  with icenet.CanHavePeripheryIceNIC // Enables optionally adding the IceNIC for FireSim
  with chipyard.example.CanHavePeripheryInitZero // Enables optionally adding the initzero example widget
  with chipyard.example.CanHavePeripheryGCD // Enables optionally adding the GCD example widget
  with chipyard.example.CanHavePeripheryStreamingFIR // Enables optionally adding the DSPTools FIR example widget
  with chipyard.example.CanHavePeripheryStreamingPassthrough // Enables optionally adding the DSPTools streaming-passthrough example widget
  with nvidia.blocks.dla.CanHavePeripheryNVDLA // Enables optionally having an NVDLA
  with chipyard.clocking.HasChipyardPRCI // Use Chipyard reset/clock distribution
  with chipyard.clocking.CanHaveClockTap // Enables optionally adding a clock tap output port
  with fftgenerator.CanHavePeripheryFFT // Enables optionally having an MMIO-based FFT block
  with constellation.soc.CanHaveGlobalNoC // Support instantiating a global NoC interconnect
{
  override lazy val module = new DigitalTopModule(this)
}

class DigitalTopModule[+L <: DigitalTop](l: L) extends ChipyardSystemModule(l)
  with sifive.blocks.devices.i2c.HasPeripheryI2CModuleImp
  with sifive.blocks.devices.pwm.HasPeripheryPWMModuleImp
  with sifive.blocks.devices.uart.HasPeripheryUARTModuleImp
  with sifive.blocks.devices.gpio.HasPeripheryGPIOModuleImp
  with sifive.blocks.devices.spi.HasPeripherySPIFlashModuleImp
  with sifive.blocks.devices.spi.HasPeripherySPIModuleImp
  with freechips.rocketchip.util.DontTouch
```

正如我们需要 `LazyModule` 和 module implementation 的单独特征一样，我们需要两个类来构建系统。`DigitalTop` 类包含参数化和定义 `DigitalTop` 的一组特征。通常，这些特征将选择性地向 `DigitalTop` 添加 IO 或外设。DigitalTop 类包含预描述代码以及用于生成 module implementation 的 `lazy val`（因此称为 LazyModule）。`DigitalTopModule` 类是被综合的实际 RTL。

最后，我们在 `generators/chipyard/src/main/scala/config/MMIOAcceleratorConfigs.scala` 中创建一个配置类，它使用前面定义的 `WithGCD` 配置片段。

```Scala
class WithGCD(useAXI4: Boolean = false, useBlackBox: Boolean = false) extends Config((site, here, up) => {
  case GCDKey => Some(GCDParams(useAXI4 = useAXI4, useBlackBox = useBlackBox))
})
```

```Scala
class GCDTLRocketConfig extends Config(
  new chipyard.example.WithGCD(useAXI4=false, useBlackBox=false) ++          // Use GCD Chisel, connect Tilelink
  new freechips.rocketchip.subsystem.WithNBigCores(1) ++
  new chipyard.config.AbstractConfig)
```

## Testing

现在我们可以测试 GCD 是否正常工作。测试程序位于 `tests/gcd.c` 中。

```C
#include "mmio.h"

#define GCD_STATUS 0x4000
#define GCD_X 0x4004
#define GCD_Y 0x4008
#define GCD_GCD 0x400C

unsigned int gcd_ref(unsigned int x, unsigned int y) {
  while (y != 0) {
    if (x > y)
      x = x - y;
    else
      y = y - x;
  }
  return x;
}

// DOC include start: GCD test
int main(void)
{
  uint32_t result, ref, x = 20, y = 15;

  // wait for peripheral to be ready
  while ((reg_read8(GCD_STATUS) & 0x2) == 0) ;

  reg_write32(GCD_X, x);
  reg_write32(GCD_Y, y);


  // wait for peripheral to complete
  while ((reg_read8(GCD_STATUS) & 0x1) == 0) ;

  result = reg_read32(GCD_GCD);
  ref = gcd_ref(x, y);

  if (result != ref) {
    printf("Hardware result %d does not match reference value %d\n", result, ref);
    return 1;
  }
  printf("Hardware result %d is correct for GCD\n", result);
  return 0;
}
// DOC include end: GCD test
```

这只是写入我们之前定义的寄存器。默认情况下，模块的 MMIO 区域的基址位于 0x2000。当您生成 Verilog 代码时，这将在地址映射部分中打印出来。您还可以看到这如何更改 `generated-src` 中发出的 `.json` 地址映射文件。

使用 `make` 编译该程序会生成 `gcd.riscv` 可执行文件。

现在所有这些都完成了，我们可以继续运行我们的仿真。

```shell
cd sims/verilator
make CONFIG=GCDTLRocketConfig BINARY=../../tests/gcd.riscv run-binary
```

