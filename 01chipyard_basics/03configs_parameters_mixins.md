# Configs, Parameters, Mixins, and Everything In Between

在 Chipyard 项目中，大部分生成器都被用于 Rocket Chip 参数化系统中。该参数化系统可实现 SoC 的灵活配置，而无需进行侵入性的 RTL 修改。为了正确使用参数化系统，我们将使用一下几个术语和约定：

## Parameters

Rocket 参数化系统中最大的挑战是能够识别要使用的正确参数，以及参数对于整个系统的影响。我们仍在研究促进参数探索和发现的方法。

## Configs

一个配置是多个被设置为具体数值的生成器参数的集合。配置是可叠加的，能够相互覆盖，并且可以由其他配置（有时成为配置片段）组成。可叠加配置或配置片段的命名约定为 `With<YourConfigName>`，不可叠加配置的命名约定为 `<YourConfig>`。配置中可以采用参数，这些参数将依次设置设计中的参数或引用设计中的其他参数（请阅读 [Parameters]()）。

这个示例展示了一个基础配置片段类型，它接收 0 个参数，并且使用硬编码设置 RTL 设计参数。在这个示例中，`MyAcceleratorConfig` 是一个 Scala case 类型，它定义了生成器在设计中引用 `MyAcceleratorKey` 时可以使用的一组变量。

```Scala
class WithMyAcceleratorParams extends Config((site, here, up) => {
  case BusWidthBits => 128
  case MyAcceleratorKey =>
    MyAcceleratorConfig(
      rows = 2,
      rowBits = 64,
      columns = 16,
      hartId = 1,
      someLength = 256)
})
```

下一个示例展示了高级的可叠加片段配置，它使用先前设置的参数来导出其他参数。

```Scala
class WithMyMoreComplexAcceleratorConfig extends Config((site, here, up) => {
  case BusWidthBits => 128
  case MyAcceleratorKey =>
    MyAcceleratorConfig(
      Rows = 2,
      rowBits = site(SystemBusKey).beatBits,
      hartId = up(RocketTilesKey, site).length)
})
```

接下来的示例展示了使用 `++` 对先前的两个配置片段进行组合或 “组装”构成的不可叠加片段。可叠加配置片段在列表中的从右到左应用（在本示例中从下到上）。因此，参数的顺序首先是 `DefaultExampleConfig`，然后是 `WithMyAcceleratorParams`，接下来是 `WithMyMoreComplexAcceleratorConfig`。

```Scala
class SomeAdditiveConfig extends Config(
  new WithMyMoreComplexAcceleratorConfig ++
  new WithMyAcceleratorParams ++
  new DefaultExampleConfig
)
```

在 `WithMyMoreComplexAcceleratorConfig` 中的 `site`，`here` 以及 `up` 对象是从 configuration keys 到其定义的映射。`site` 映射能够让您能够从配置层次结构的根部（在本示例中为 `SomeAdditiveConfig`）看到定义。`here` 映射能够让您从层次结构中的当前层次中（在本示例中为 `WithMyMoreComplexAcceleratorConfig`）看到定义。`up` 映射能够让您从当前的下一层中（在本示例中为 `WithMyAcceleratorParams`）看到定义。

## Cake Pattern / Mixin

Cake Pattern 或 mixin 是 Scala 中的一种编程模式，允许混合多个 traits 或接口定义（有时被称为依赖注入）。它在 Rocket Chip SoC 库和 Chipyard 框架中用于将多种系统组件和 IO 接口混合成一个大的系统组件。

这个示例展示了 Chipyard 默认的顶层，它将多个 traits 组合成一个具有许多可选组件的全功能 SoC。

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

有两个 cakes 或 mixins。一个是 lazy module（例如 `CanHavePeripheryTLSerial`），另一个是 lazy module implementation（例如 `CanHavePeripheryTLSerialModuleImp`，`Imp` 表示 `implementation`）。lazy module 定义了生成器之间的所有逻辑连接并在它们之间的交换配置信息，而 lazy module implementation 则执行实际的 Chisel RTL 描述。

在 `DigitalTop` 示例类中，外部的 `DigitalTop` 实例化了内部的 `DigitalTopModule` 作为一个 lazy module implementation。它延迟了 module 的立即实现，直到所有的逻辑连接确定以及所有的配置信息交换完成。外部基类 `System` 以及外部 traits `CanHavePeriphery<X>` 包含了执行高级逻辑连接的代码。例如，外部 trait `CanHavePeripheryTLSerial` 包含了可选惰性实例化 `TLSerdesser` 的代码，且将 `TLSerdesser` 的 TileLink 节点连接到前端总线上。

`ModuleImp` 类型和 trait 执行实际的 RTL 描述。

在测试工具中，SoC 使用 `val dut = p(BuildTop)(p)` 进行描述。

在描述之后， 系统子模块 `ChiTop` 将会成为一个 `DigitalTop` 模块，如果为要实例化的块指定了配置，则其中将包含 `TLSerdesser` 模块（以及其他模块）。

从更高层次来看，继承了 `Lazyodule` 的类必须通过 `lazy val module` 引用自己的 module implementation，且它们可选地引用其他 lazy modules（在模块层次结构中被描述为子模块）。内部模块包含了模块的 implementation，且可能实例化其他正常模块或 lazy modules（例如嵌套 Diplomacy 图）。

对于可叠加 mixin 或 trait 的命名约定为 `CanHave<YourMixin>`。在 `Top` 类中，诸如 `CanHavePeripheryTLSerial` 这类的东西将 RTL 组件连接到总线上且向顶层暴露信号。

## Additional References

其他的关于 traits/mixin 和配置片段的描述在 [Keys, Traits, and Configs](https://chipyard.readthedocs.io/en/latest/Customization/Keys-Traits-Configs.html#keys-traits-and-configs)。此外，关于这些主题的简介描述在一下视频中：https://www.youtube.com/watch?v=Eko86PGEoDY。

> **注意**：Chipyard 使用 “配置片段” 而不是 “配置 mixins” 是为了避免应用于配置与系统顶层之间发生混淆（即使两者都是 Scala mixins）。

