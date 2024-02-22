# Adding a DMA Device

DMA 设备是充当主设备的 Tilelink 小部件。换句话说，DMA 设备可以向芯片的内存系统发送自己的读写请求。

对于 IO 设备或加速器（如磁盘或网络驱动程序），我们可能希望设备直接写入相干内存系统，而不是让 CPU 从设备轮询数据。例如，这是一个将零写入内存中配置地址的设备。

```Scala
package chipyard.example

import chisel3._
import chisel3.util._
import freechips.rocketchip.subsystem.{BaseSubsystem, CacheBlockBytes}
import org.chipsalliance.cde.config.{Parameters, Field, Config}
import freechips.rocketchip.diplomacy.{LazyModule, LazyModuleImp, IdRange}
import freechips.rocketchip.tilelink._

case class InitZeroConfig(base: BigInt, size: BigInt)
case object InitZeroKey extends Field[Option[InitZeroConfig]](None)

class InitZero(implicit p: Parameters) extends LazyModule {
  val node = TLClientNode(Seq(TLMasterPortParameters.v1(Seq(TLClientParameters(
    name = "init-zero", sourceId = IdRange(0, 1))))))

  lazy val module = new InitZeroModuleImp(this)
}

class InitZeroModuleImp(outer: InitZero) extends LazyModuleImp(outer) {
  val config = p(InitZeroKey).get

  val (mem, edge) = outer.node.out(0)
  val addrBits = edge.bundle.addressBits
  val blockBytes = p(CacheBlockBytes)

  require(config.size % blockBytes == 0)

  val s_init :: s_write :: s_resp :: s_done :: Nil = Enum(4)
  val state = RegInit(s_init)

  val addr = Reg(UInt(addrBits.W))
  val bytesLeft = Reg(UInt(log2Ceil(config.size+1).W))

  mem.a.valid := state === s_write
  mem.a.bits := edge.Put(
    fromSource = 0.U,
    toAddress = addr,
    lgSize = log2Ceil(blockBytes).U,
    data = 0.U)._2
  mem.d.ready := state === s_resp

  when (state === s_init) {
    addr := config.base.U
    bytesLeft := config.size.U
    state := s_write
  }

  when (edge.done(mem.a)) {
    addr := addr + blockBytes.U
    bytesLeft := bytesLeft - blockBytes.U
    state := s_resp
  }

  when (mem.d.fire) {
    state := Mux(bytesLeft === 0.U, s_done, s_write)
  }
}

trait CanHavePeripheryInitZero { this: BaseSubsystem =>
  implicit val p: Parameters

  p(InitZeroKey) .map { k =>
    val initZero = LazyModule(new InitZero()(p))
    fbus.coupleFrom("init-zero") { _ := initZero.node }
  }
}


// DOC include start: WithInitZero
class WithInitZero(base: BigInt, size: BigInt) extends Config((site, here, up) => {
  case InitZeroKey => Some(InitZeroConfig(base, size))
})
// DOC include end: WithInitZero
```

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

我们使用 `TLClientNode` 为我们创建一个 TileLink 客户端节点。然后我们通过前端总线（fbus）将客户端节点连接到内存系统。有关创建 TileLink 客户端节点的更多信息，请查看 [Client Node](https://chipyard.readthedocs.io/en/stable/TileLink-Diplomacy-Reference/NodeTypes.html#client-node)。

一旦我们创建了包括 DMA 小部件的顶级模块，我们就可以像之前一样为其创建配置。

```Scala
class WithInitZero(base: BigInt, size: BigInt) extends Config((site, here, up) => {
  case InitZeroKey => Some(InitZeroConfig(base, size))
})
```

```Scala
class InitZeroRocketConfig extends Config(
  new chipyard.example.WithInitZero(0x88000000L, 0x1000L) ++   // add InitZero
  new freechips.rocketchip.subsystem.WithNBigCores(1) ++
  new chipyard.config.AbstractConfig)
```
