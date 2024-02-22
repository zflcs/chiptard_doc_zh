# Keys, Traits, and Configs

此时您可能已经看到 Chisel 引用 keys、traits 和 configs 的片段。本节旨在阐明这些 Chisel/Scala 组件之间的交互，并提供如何使用这些组件创建参数化设计并对其进行配置的最佳实践。

我们将继续使用 GCD 示例。

## Keys

Keys 指定一些控制某些自定义小部件的参数。Keys 通常应实现为 Option 类型，默认值为 `None`，这意味着系统不会发生任何变化。换句话说，当用户没有显式设置 key 时的默认行为应该是无操作。

Keys 应该在子项目中定义和记录，因为它们通常处理一些特定的块，而不是系统级集成。（我们对示例 GCD 小部件进行了例外处理）。

```Scala
case object GCDKey extends Field[Option[GCDParams]](None)
```

key 中的对象通常是一个 `case class XXXParams`，它定义了某些块接受的一组参数。例如，GCD 小部件的 `GCDParams` 参数化其地址、操作数宽度、小部件是否应通过 Tilelink 还是 AXI4 连接，以及小部件是否应使用 blackbox-Verilog 实现或 Chisel 实现。

```Scala
case class GCDParams(
  address: BigInt = 0x4000,
  width: Int = 32,
  useAXI4: Boolean = false,
  useBlackBox: Boolean = true)
```

在 Chisel 中访问存储在 key 中的值很容易，只要 `implicit p: Parameters` 对象被传递到相关模块即可。例如，`p(GCDKey).get.address` 返回 `GCDParams` 的地址字段。请注意，这仅在 `GCDKey` 未设置为 `None` 的情况下才有效，因此您的 Chisel 应检查这种情况！

## Traits

通常，大多数自定义块需要修改某些预先存在的块的行为。例如，GCD 小部件需要 `DigitalTop` 模块来实例化并通过 Tilelink 连接小部件，生成顶级 `gcd_busy` 端口，并将其连接到模块。 Traits 让我们无需修改 `DigitalTop` 的现有代码即可完成此操作，并且可以对不同自定义块的代码进行划分。

顶级 traits 指定 `DigitalTop` 已被参数化以读取某些自定义键，并可选择实例化和连接由该键定义的小部件。Traits 不应强制实例化自定义逻辑。换句话说，traits 应该用 `CanHave` 语义编写，其中未设置键时的默认行为是无操作。

应在子项目中定义和记录顶级 traits 及其相应的 keys。然后应将这些特征添加到 Chipyard 使用的 `DigitalTop` 中。

下面我们看到 GCD 示例的 traits。Lazy trait 将 GCD 模块连接到 Diplomacy 图。

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

这些特征被添加到 Chipyard 中的默认 `DigitalTop` 中。

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

## Config Fragments

配置片段将键设置为非默认值。定义配置的配置片段的集合一起生成生成器使用的所有 key 的值。

例如，`WithGCD` 配置片段由您要实例化的 GCD 小部件的类型参数化。当将此配置片段添加到配置中时，`GCDKey` 将设置为 `GCDParams` 的实例，通知前面提到的 traits 以适当地实例化和连接 GCD 小部件。

```Scala
class WithGCD(useAXI4: Boolean = false, useBlackBox: Boolean = false) extends Config((site, here, up) => {
  case GCDKey => Some(GCDParams(useAXI4 = useAXI4, useBlackBox = useBlackBox))
})
```

我们可以在编写配置时使用此配置片段。

```Scala
class GCDTLRocketConfig extends Config(
  new chipyard.example.WithGCD(useAXI4=false, useBlackBox=false) ++          // Use GCD Chisel, connect Tilelink
  new freechips.rocketchip.subsystem.WithNBigCores(1) ++
  new chipyard.config.AbstractConfig)
```

> **注意**：想要了解有关配置系统的更多信息的读者可能有兴趣阅读 [Context-Dependent-Environments](https://chipyard.readthedocs.io/en/stable/Advanced-Concepts/CDEs.html#cdes)。

## Chipyard Config Fragments

为了可发现性，用户可以运行 `make find-config-fragments` 来查看配置列表。



