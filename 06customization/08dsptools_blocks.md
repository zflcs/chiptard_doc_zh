Dsptools 是一个 Chisel 库，可帮助编写自定义信号处理加速器。它通过以下方式实现这一点： * 提供类型和帮助器，使您可以更直接地表达数学运算。 * 允许您编写多态生成器的类型类，例如适用于实数和复数滤波器的 FIR 滤波器生成器。 * 用于封装 DSP 块并将其集成到基于 rocketchip 的 SoC 中的结构。 * 用于测试 DSP 电路的测试工具，以及用于 DSP 模块的 VIP 型驱动器和监视器。

[Dsptools repository](https://github.com/ucb-bar/dsptools/) 有更多文档。

# Dsptools Blocks

`DspBlock` 是可集成到 SoC 中的信号处理功能的基本单元。它具有 AXI4 流接口和可选的内存接口。我们的想法是，这些 `DspBlock` 可以轻松设计、单元测试和乐高风格组装，以构建复杂的功能。 `DspChain` 是如何组装 `DspBlock` 的一个示例，在这种情况下，流接口串行连接到流水线中，并启动总线并通过内存接口连接到每个块。

Chipyard 提供了将 `DspBlock` 集成到基于 rocketchip 的 SoC 作为 MMIO 外设的示例设计。自定义 `DspBlock` 前面有一个 `ReadQueue`，后面有一个 `WriteQueue`，允许对流接口进行内存映射访问，以便 rocket core 可以与 `DspBlock` 交互。本节将主要关注设计基于 Tilelink 的外设。然而，通过 Dsptools 中提供的资源，人们还可以按照类似的步骤定义基于 AXI4 的外设。此外，这里的示例很简单，但可以扩展以实现更复杂的加速器，例如 [OFDM baseband](https://github.com/grebe/ofdm) 或 [spectrometer](https://github.com/ucb-art/craft2-chip)。

![Dspchain](../assets/dspfir.png)

在本示例中，我们将向您展示如何将使用 Dsptools 创建的简单 FIR 滤波器连接为 MMIO 外设，如上图所示。完整的代码可以在 `generators/chipyard/src/main/scala/example/dsptools/GenericFIR.scala` 中找到。话虽这么说，我们可以用任何具有 ready valid 接口的模块来代替 FIR，并获得相同的结果。只要模块的 read 和 valid 信号附加到相应 DSPBlock 包装器的信号，并且该包装器放置在具有 ReadQueue 和 WriteQueue 的链中，遵循这些步骤建立的总体轮廓将允许您与该块使用 MMIO 进行交互。

`GenericFIR` 模块是我们的 FIR 模块的整体包装器。该模块将数量可变的 `GenericFIRDirectCell` 子模块链接在一起，每个子模块都执行 FIR 直接形式架构中的一个系数的计算。值得注意的是，这两个模块都是类型通用的，这意味着它们可以为任何实现 `Ring` 运算（例如加法、乘法、恒等）的数据类型 `T` 进行实例化。

```Scala
class GenericFIR[T<:Data:Ring](genIn:T, genOut:T, coeffs: => Seq[T]) extends Module {
  val io = IO(GenericFIRIO(genIn, genOut))

  // Construct a vector of genericFIRDirectCells
  val directCells = Seq.fill(coeffs.length){ Module(new GenericFIRDirectCell(genIn, genOut)).io }

  // Construct the direct FIR chain
  for ((cell, coeff) <- directCells.zip(coeffs)) {
    cell.coeff := coeff
  }

  // Connect input to first cell
  directCells.head.in.bits.data := io.in.bits.data
  directCells.head.in.bits.carry := Ring[T].zero
  directCells.head.in.valid := io.in.valid
  io.in.ready := directCells.head.in.ready

  // Connect adjacent cells
  // Note that .tail() returns a collection that consists of all
  // elements in the inital collection minus the first one.
  // This means that we zip together directCells[0, n] and
  // directCells[1, n]. However, since zip ignores unmatched elements,
  // the resulting zip is (directCells[0], directCells[1]) ...
  // (directCells[n-1], directCells[n])
  for ((current, next) <- directCells.zip(directCells.tail)) {
    next.in.bits := current.out.bits
    next.in.valid := current.out.valid
    current.out.ready := next.in.ready
  }

  // Connect output to last cell
  io.out.bits.data := directCells.last.out.bits.carry
  directCells.last.out.ready := io.out.ready
  io.out.valid := directCells.last.out.valid

}
```

```Scala
class GenericFIRDirectCell[T<:Data:Ring](genIn: T, genOut: T) extends Module {
  val io = IO(GenericFIRCellIO(genIn, genOut))

  // Registers to delay the input and the valid to propagate with calculations
  val hasNewData = RegInit(0.U)
  val inputReg = Reg(genIn.cloneType)

  // Passthrough ready
  io.in.ready := io.out.ready

  // When a new transaction is ready on the input, we will have new data to output
  // next cycle. Take this data in
  when (io.in.fire) {
    hasNewData := 1.U
    inputReg := io.in.bits.data
  }

  // We should output data when our cell has new data to output and is ready to
  // recieve new data. This insures that every cell in the chain passes its data
  // on at the same time
  io.out.valid := hasNewData & io.in.fire
  io.out.bits.data := inputReg

  // Compute carry
  // This uses the ring implementation for + and *, i.e.
  // (a * b) maps to (Ring[T].prod(a, b)) for whicever T you use
  io.out.bits.carry := inputReg * io.coeff + io.in.bits.carry
}
```

## Creating a DspBlock

将 FIR 滤波器附加为 MMIO 外设的第一步是创建 `DspBlock` 的抽象子类，以包裹 `GenericFIR` 模块。流输出和输入被打包和解包到 UInt 中。如果有控制信号，它们就是从原始 IO 到内存映射的地方。该过程的主要步骤如下。

1. 在 `GenericFIRBlock` 中实例化 `GenericFIR`。
2. 连接来自输入和输出连接的 ready 和 valid 信号。
3. 将模块输入数据转换为 `GenericFIR` (`GenericFIRBundle`) 的输入类型并附加。
4. 将 `GenericFIR` 的输出转换为 `UInt` 并附加到模块输出。

```Scala
abstract class GenericFIRBlock[D, U, EO, EI, B<:Data, T<:Data:Ring]
(
  genIn: T,
  genOut: T,
  coeffs: => Seq[T]
)(implicit p: Parameters) extends DspBlock[D, U, EO, EI, B] {
  val streamNode = AXI4StreamIdentityNode()
  val mem = None

  lazy val module = new LazyModuleImp(this) {
    require(streamNode.in.length == 1)
    require(streamNode.out.length == 1)

    val in = streamNode.in.head._1
    val out = streamNode.out.head._1

    // instantiate generic fir
    val fir = Module(new GenericFIR(genIn, genOut, coeffs))

    // Attach ready and valid to outside interface
    in.ready := fir.io.in.ready
    fir.io.in.valid := in.valid

    fir.io.out.ready := out.ready
    out.valid := fir.io.out.valid

    // cast UInt to T
    fir.io.in.bits := in.bits.data.asTypeOf(GenericFIRBundle(genIn))

    // cast T to UInt
    out.bits.data := fir.io.out.bits.asUInt
  }
}
```

请注意，此时 `GenericFIRBlock` 尚未指定存储器接口类型。这个抽象类可用于创建使用 AXI-4、TileLink、AHB 或您喜欢的任何其他内存接口的不同风格。

## Connecting DspBlock by TileLink

实现这些类后，您可以通过扩展 `GenericFIRBlock` 开始构建链，同时通过 mixin 使用 `TLDspBlock` 特征。

```Scala
class TLGenericFIRBlock[T<:Data:Ring]
(
  val genIn: T,
  val genOut: T,
  coeffs: => Seq[T]
)(implicit p: Parameters) extends
GenericFIRBlock[TLClientPortParameters, TLManagerPortParameters, TLEdgeOut, TLEdgeIn, TLBundle, T](
    genIn, genOut, coeffs
) with TLDspBlock
```

然后，我们可以利用 `generators/chipyard/src/main/scala/example/dsptools/DspBlocks.scala` 中的 `TLWriteQueue` 和 `TLReadeQueue` 模块构建最终的链。该链是通过将工厂函数列表传递给 `TLChain` 的构造函数来创建的。然后，构造函数自动实例化这些 `DspBlock`，按顺序连接它们的流节点，创建总线，并将任何具有内存接口的 `DspBlock` 连接到总线。

```Scala
class TLGenericFIRChain[T<:Data:Ring] (genIn: T, genOut: T, coeffs: => Seq[T], params: GenericFIRParams)(implicit p: Parameters)
  extends TLChain(Seq(
    TLWriteQueue(params.depth, AddressSet(params.writeAddress, 0xff))(_),
    { implicit p: Parameters =>
      val fir = LazyModule(new TLGenericFIRBlock(genIn, genOut, coeffs))
      fir
    },
    TLReadQueue(params.depth, AddressSet(params.readAddress, 0xff))(_)
  ))
```

## Top Level Traits

与前面的 MMIO 示例一样，我们使用 cake pattern 将模块连接到 SoC。

```Scala
trait CanHavePeripheryStreamingFIR extends BaseSubsystem {
  val streamingFIR = p(GenericFIRKey) match {
    case Some(params) => {
      val domain = pbus.generateSynchronousDomain.suggestName("fir_domain")
      val streamingFIR = domain { LazyModule(new TLGenericFIRChain(
        genIn = FixedPoint(8.W, 3.BP),
        genOut = FixedPoint(8.W, 3.BP),
        coeffs = Seq(1.U.asFixedPoint(0.BP), 2.U.asFixedPoint(0.BP), 3.U.asFixedPoint(0.BP)),
        params = params)) }
      pbus.coupleTo("streamingFIR") { domain { streamingFIR.mem.get := TLFIFOFixer() := TLFragmenter(pbus.beatBytes, pbus.blockBytes) } := _ }
      Some(streamingFIR)
    }
    case None => None
  }
}
```

请注意，这是我们决定 FIR 数据类型的点。您可以创建使用不同类型的 FIR 的不同配置，例如实例化复数 FIR 滤波器的配置。

## Constructing the Top and Config

再次遵循前面的 MMIO 示例的路径，我们现在希望将我们的特征混合到整个系统中。代码来自 `generators/chipyard/src/main/scala/DigitalTop.scala`。

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

最后，我们在 `generators/chipyard/src/main/scala/config/MMIOAcceleratorConfigs.scala` 中创建配置类，它使用 `generators/chipyard/src/main/scala/example/dsptools/GenericFIR.scala` 中定义的 `WithFIR` mixin。

```Scala
class WithStreamingFIR extends Config((site, here, up) => {
  case GenericFIRKey => Some(GenericFIRParams(depth = 8))
})
```

```Scala
class StreamingFIRRocketConfig extends Config (
  new chipyard.example.WithStreamingFIR ++                  // use top with tilelink-controlled streaming FIR
  new freechips.rocketchip.subsystem.WithNBigCores(1) ++
  new chipyard.config.AbstractConfig)
```

## FIR Testing

我们现在可以测试 FIR 是否正常工作。测试程序位于 `tests/streaming-fir.c` 中。

```C
#define PASSTHROUGH_WRITE 0x2000
#define PASSTHROUGH_WRITE_COUNT 0x2008
#define PASSTHROUGH_READ 0x2100
#define PASSTHROUGH_READ_COUNT 0x2108

#define BP 3
#define BP_SCALE ((double)(1 << BP))

#include "mmio.h"

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <stdint.h>

uint64_t roundi(double x)
{
  if (x < 0.0) {
    return (uint64_t)(x - 0.5);
  } else {
    return (uint64_t)(x + 0.5);
  }
}

int main(void)
{
  double test_vector[15] = {1.0, 2.0, 3.0, 4.0, 5.0, 4.0, 3.0, 2.0, 1.0, 0.5, 0.25, 0.125, 0.125};
  uint32_t num_tests = sizeof(test_vector) / sizeof(double);
  printf("Starting writing %d inputs\n", num_tests);

  for (int i = 0; i < num_tests; i++) {
    reg_write64(PASSTHROUGH_WRITE, roundi(test_vector[i] * BP_SCALE));
  }

  printf("Done writing\n");
  uint32_t rcnt = reg_read32(PASSTHROUGH_READ_COUNT);
  printf("Write count: %d\n", reg_read32(PASSTHROUGH_WRITE_COUNT));
  printf("Read count: %d\n", rcnt);

  int failed = 0;
  if (rcnt != 0) {
    for (int i = 0; i < num_tests - 3; i++) {
      uint32_t res = reg_read32(PASSTHROUGH_READ);
      // double res = ((double)reg_read32(PASSTHROUGH_READ)) / BP_SCALE;
      double expected_double = 3*test_vector[i] + 2*test_vector[i+1] + test_vector[i+2];
      uint32_t expected = ((uint32_t)(expected_double * BP_SCALE + 0.5)) & 0xFF;
      if (res == expected) {
        printf("\n\nPass: Got %u Expected %u\n\n", res, expected);
      } else {
        failed = 1;
        printf("\n\nFail: Got %u Expected %u\n\n", res, expected);
      }
    }
  } else {
    failed = 1;
  }

  if (failed) {
    printf("\n\nSome tests failed\n\n");
  } else {
    printf("\n\nAll tests passed\n\n");
  }

  return 0;
}
```

该测试将一系列值输入到 fir 中，并将输出与 golden 计算模型进行比较。默认情况下，模块的 MMIO 写入区域基址为 0x2000，读区域基址为 0x2100。

使用 `make` 编译该程序会生成 `streaming-fir.riscv` 可执行文件。

现在我们可以运行我们的仿真了。

```shell
cd sims/verilator
make CONFIG=StreamingFIRRocketConfig BINARY=../../tests/streaming-fir.riscv run-binary
```