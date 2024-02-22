# FFT Generator

FFT 生成器是一个可参数化的 FFT 加速器。

## Configuration

以下配置创建 8 点 FFT：

```Scala
class FFTRocketConfig extends Config(
  new chipyard.harness.WithDontTouchChipTopPorts(false) ++              // TODO: hack around dontTouch not working in SFC
  new fftgenerator.WithFFTGenerator(numPoints=8, width=16, decPt=8) ++ // add 8-point mmio fft at the default addr (0x2400) with 16bit fixed-point numbers.
  new freechips.rocketchip.subsystem.WithNBigCores(1) ++
  new chipyard.config.AbstractConfig)
```

`baseAddress` 指定 FFT 读取和写入通道的起始地址。 FFT 写入通道始终位于 `baseAddress`。每个输出点有1个读取通道；由于此配置指定 8 点 FFT，因此将有 8 个读取通道。读取通道 `i`（可以从中加载以检索输出点`i`）将位于 `baseAddr + 64bits(assuming 64bit system) + (i * 8)`。 `baseAddress` 应该是 64 位对齐的 `width` 是二进制输入点的大小。宽度为 `w` 意味着每个点将具有 `w` 位实部和 `w` 位虚部，每个点总共有 2w 位。`decPt` 是每个点的实部和虚部值的固定精度表示中小数点的位置。在上面的 Config 中，每个点都是 32 位宽，其中 16 位用于表示实部，16 位用于表示虚部。在每个分量的 16 位中，8 个 LSB 用于表示值的小数部分，其余 (8) 个 MSB 用于表示整数部分。实部和虚部都使用固定精度表示。

要构建此示例 Chipyard 配置的模拟，请运行以下命令：

```shell
cd sims/verilator # or "cd sims/vcs"
make CONFIG=FFTRocketConfig
```

## Usage and Testing

点通过单写入通道传递到 FFT。在 C 伪代码中，这可能如下所示：

```C
for (int i = 0; i < num_points; i++) {
    // FFT_WRITE_LANE = baseAddress
    uint32_t write_val = points[i];
    volatile uint32_t* ptr = (volatile uint32_t*) FFT_WRITE_LANE;
    *ptr = write_val;
}
```

一旦传入正确数量的输入（在上面的配置中，将传入 8 个值），就可以读取读取通道（同样是 C 伪代码）：

```C
for (int i = 0; i < num_points; i++) {
    // FFT_RD_LANE_BASE = baseAddress + 64bits (for write lane)
    volatile uint32_t* ptr_0 = (volatile uint32_t*) (FFT_RD_LANE_BASE + (i * 8));
    uint32_t read_val = *ptr_0;
}
```

`test/` 目录中的 `fft.c` 测试文件可用于验证使用 `FFTRocketConfig` 构建的 SoC 上的 fft 功能。

## Acknowledgements

FFT 生成器的代码改编自加州大学伯克利分校 [Hydra Spine](https://adept.eecs.berkeley.edu/projects/hydra-spine/) 项目的 ADEPT 实验室。

原始项目的作者（排名不分先后）：

1. James Dunn, UC Berkeley (dunn [at] eecs [dot] berkeley [dot] edu)
   1. `Deserialize.scala`
   2. `Tail.scala`
   3. `Unscramble.scala`
2. Stevo Bailey (stevo.bailey [at] berkeley [dot] edu)
   1. `FFT.scala`