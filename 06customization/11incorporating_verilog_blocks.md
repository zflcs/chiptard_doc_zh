# Incorporating Verilog Blocks

使用现有的 Verilog IP 是许多芯片设计流程中不可或缺的一部分。幸运的是，Chisel 和 Chipyard 都为 Verilog 集成提供了广泛的支持。

在这里，我们将研究整合使用最大公分母 (GCD) 算法的 Verilog 实现的 MMIO 外设的过程。添加 Verilog 外设有几个步骤：

1. 将 Verilog 资源文件添加到项目中。
2. 定义代表 Verilog 模块的 Chisel `BlackBox`。
3. 实例化 `BlackBox` 并连接 `RegField` 条目。
4. 设置使用外设的芯片 `Top` 和 `Config`。

## Adding a Verilog Blackbox Resource File

和之前一样，可以将外围设备合并为您自己的生成器项目的一部分。但是，Verilog 资源文件必须位于与 Chisel (Scala) 源不同的目录中。

```
generators/yourproject/
    build.sbt
    src/main/
        scala/
        resources/
            vsrc/
                YourFile.v
```

除了上一节中概述的在顶层将项目添加到 `build.sbt` 中的步骤之外，还需要将包含 Verilog IP 作为依赖项的任何项目添加到 `tapeout` 项目中。这确保了 Verilog 源对下游 FIRRTL 通道可见，这些通道提供了将 Verilog 文件集成到构建过程中的实用程序，这些实用程序是 `Barstools/tapeout` 中的 `tapeout` 包的一部分。

```Scala
lazy val tapeout = conditionalDependsOn(project in file("./tools/barstools/tapeout/"))
  .dependsOn(chisel_testers, example, yourproject)
  .settings(commonSettings)
```

对于这个具体的 GCD 示例，我们将使用在 `chipyard` 项目中定义的`GCDMMIOBlackBox` Verilog 模块。 Scala 和 Verilog 源代码遵循规定的目录布局。

```
generators/chipyard/
    build.sbt
    src/main/
        scala/
            example/
                GCD.scala
        resources/
            vsrc/
                GCDMMIOBlackBox.v
```

## Defining a Chisel BlackBox

Chisel `BlackBox` 模块提供了一种实例化由外部 Verilog 源定义的模块的方法。黑盒的定义包括几个方面，允许将其转换为 Verilog 模块的实例：

1. `io` 字段：包含与 Verilog 模块的端口列表相对应的字段的捆绑包。
2. 构造函数参数，采用从 Verilog 参数名称到描述值的映射。
3. 添加了一项或多项资源以指示 Verilog 源依赖性

特别令人感兴趣的是，参数化的 Verilog 模块可以传递可能参数值的全部空间。这些值可能取决于 Chisel 生成器中的描述时间值，正如本例中 GCD 计算的位宽一样。

Verilog GCD 端口列表和参数：

```Verilog
module GCDMMIOBlackBox
  #(parameter WIDTH)
   (
    input                  clock,
    input                  reset,
    output                 input_ready,
    input                  input_valid,
    input [WIDTH-1:0]      x,
    input [WIDTH-1:0]      y,
    input                  output_ready,
    output                 output_valid,
    output reg [WIDTH-1:0] gcd,
    output                 busy
    );
```

Chisel 黑盒定义：

```Scala
class GCDMMIOBlackBox(val w: Int) extends BlackBox(Map("WIDTH" -> IntParam(w))) with HasBlackBoxResource
  with HasGCDIO
{
  addResource("/vsrc/GCDMMIOBlackBox.v")
}
```

## Instantiating the BlackBox and Defining MMIO

接下来，我们必须实例化黑盒。为了利用系统总线上的 diplomatic 内存映射，我们仍然必须通过将特定于外设的 traits 混合到 `TLRegisterRouter` 中来在 Chisel 级别集成外设。 `params` 成员和 `HasRegMap` 基本 trait 应该与前面的内存映射 GCD 设备示例相似。

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

## Defining a Chip with a BlackBox

由于我们已经参数化了 GCD 实例化以在 Chisel 和 Verilog 模块之间进行选择，因此创建配置很容易。

```Scala
class GCDAXI4BlackBoxRocketConfig extends Config(
  new chipyard.example.WithGCD(useAXI4=true, useBlackBox=true) ++            // Use GCD blackboxed verilog, connect by AXI4->Tilelink
  new freechips.rocketchip.subsystem.WithNBigCores(1) ++
  new chipyard.config.AbstractConfig)
```

您可以使用 mixin 的参数化来选择 GCD 的 TL/AXI4、BlackBox/Chisel 版本。

## Software Testing

GCD 模块的接口比较复杂，因此在每次触发读或写之前使用轮询来检查设备的状态。

```Scala
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
```

## Support for Verilog Within Chipyard Tool Flows

Chipyard 框架内的各种流程处理 Verilog 黑盒的方式存在重要差异。 Chipyard 内的一些流程依赖 FIRRTL 来提供强大的、非侵入性的源代码转换。由于 Verilog 黑盒仍然是 FIRRTL 中的黑盒，因此它们被 FIRRTL 变换处理的能力是有限的，并且 Chipyard 的一些高级功能可能对黑盒提供较弱的支持。请注意，设计的其余部分（设计的“非 Verilog”部分）通常仍可以通过任何 Chipyard FIRRTL 转换进行转换或增强。

1. 完全支持 Verilog 黑盒生成可流片的 RTL。
2. HAMMER 工作流程为集成 Verilog 黑盒提供强大支持。
3. FireSim 依靠 FIRRTL 转换来生成解耦的 FPGA 模拟器。因此，FireSim 对 Verilog 黑盒的支持目前有限，但正在迅速发展。敬请关注！
4. 自定义 FIRRTL 转换和分析有时可能能够处理黑盒 Verilog，具体取决于特定转换的机制。

正如本节前面提到的，`BlackBox` 资源文件必须集成到构建过程中，因此任何提供 `BlackBox` 资源的项目都必须对 `build.sbt` 中的 `tapeout` 项目可见。

## Differences between `HasBlackBoxPath` and `HasBlackBoxResource`

Chisel 提供了两种将黑盒文件集成到 Chisel 项目中的机制，这两种机制在 Chipyard 中的工作方式略有不同：`HasBlackBoxPath` 和 `HasBlackBoxResource`。

`HasBlackBoxResource` 通过在项目的 `src/main/resources` 区域中查找文件的相对路径来合并额外的文件。这要求 `addResource` 添加的文件存在于 `src/main/resources` 区域中并且不是自动生成的（该文件在生成 RTL 的整个生命周期中是静态的）。这是因为，当编译 Chisel 源代码时，它们会与 `src/main/resources` 区域一起放入一个 `jar` 文件中，并且该 `jar` 用于运行 Chisel 生成器。在 Chisel `描述过程中，addResource` 引用的文件必须位于此 `jar` 文件中。因此，如果在 Chisel 生成期间生成文件，则直到下次编译 Chisel 源代码时，该文件才会出现在 `jar` 文件中。

`HasBlackBoxPath` 的不同之处在于它通过使用额外文件的绝对路径来合并这些文件。稍后在构建过程中，FIRRTL 编译器会将文件从该位置复制到生成的源目录。因此，该文件必须在 FIRRTL 编译器运行之前存在（即该文件不需要位于 `src/main/resources` 中，或者可以在 Chisel 描述期间自动生成）。

此外，这两种机制都不强制添加文件的顺序。例如：

```Scala
addResource("fileA")
addResource("fileB")
```

在这种情况下，当传递给下游工具时，不保证 `fileA` 位于 `fileB` 之前。为了绕过这个问题，建议通过连接文件并使用 `HasBlackBoxPath` 给出的 `addPath` 来自动生成一个具有所需顺序的单个文件。一个例子是 https://github.com/ucb-bar/ibex-wrapper/blob/main/src/main/scala/IbexCoreBlackbox.scala。
