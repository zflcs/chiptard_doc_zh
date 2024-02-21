# FPGA-Accelerated Simulation

## FireSim

[FireSim](https://fires.im/) 是一款开源、周期精确的 FPGA 加速全系统硬件仿真平台，在 FPGA（Amazon EC2 F1 FPGA 和本地 FPGA）上运行。FireSim 允许以比软件 RTL 仿真器快几个数量级的速度进行 RTL 级仿真。FireSim 还提供额外的设备模型以允许全系统仿真，包括内存模型和网络模型。

FireSim 支持在支持 FPGA 的 Amazon EC2 F1 云实例以及连接了 FPGA 的本地管理的 Linux 计算机上运行。本文档的其余部分假设您正在支持 Amazon EC2 F1 FPGA 的虚拟实例上运行。为了使用 FireSim 模拟您的 Chipyard 设计，请确保遵循 [Initial Repository Setup](https://chipyard.readthedocs.io/en/stable/Chipyard-Basics/Initial-Repo-Setup.html#initial-repository-setup) 中所述的存储库设置（如果您尚未这样做）。此设置应该通过运行 `./scripts/firesim-setup.sh` 脚本来设置包含 FireSim 的 Chipyard 存储库。`firesim-setup.sh` 初始化其他子模块，然后调用 FireSim 的 `build-setup.sh` 脚本添加 `--library` 以将 FireSim 正确初始化为 Chipyard 中的库子模块。您可以运行 `./sims/firesim/build-setup.sh --help` 查看更多选项。

最后，在 FireSim 目录的根目录中激活以下环境：

```shell
cd sims/firesim
# (Recommended) The default manager environment (includes env.sh)
source sourceme-manager.sh
```

> **注意**：每次你想使用新的 shell 来使用 FireSim 时，你必须使用 `sourceme-manager.sh`

此时，您已准备好将 FireSim 与 Chipyard 结合使用。如果您还不熟悉 FireSim，请返回 [FireSim Docs](https://docs.fires.im/en/stable/Initial-Setup/Setting-up-your-Manager-Instance.html#completing-setup-using-the-manager)，并继续学习本教程的其余部分。

## Running your Design in FireSim

转换 Chipyard 配置（`chipyard/src/main/scala` 中的一个）以在 FireSim 中运行非常简单，可以通过传统配置系统或通过 FireSim 的构建配方方案来完成。

FireSim 仿真需要 3 个额外的配置片段：

1. `WithFireSimConfigTweaks` 修改您的设计以更好地适应 FireSim 使用模型。它由多个较小的配置片段组成。例如，删除时钟门控（使用 `WithoutClockGating` 配置片段），这是编译器正确运行所必需的。此配置片段还包括其他配置片段，例如在设计中包含 UART，尽管技术上可能是可选的，但强烈建议这样做。
2. `WithDefaultMemModel` 为 FireSim 模拟中的 FASED 内存模型提供默认配置。有关详细信息，请参阅 FireSim 文档。此配置片段目前默认包含在 `WithFireSimConfigTweaks` 中，因此无需单独添加，但如果您选择不使用 `WithFireSimConfigTweaks`，则需要添加。
3. `WithDefaultFireSimBridges` 设置 `IOBinders` 密钥以使用 FireSim 的 Bridge 系统，该系统可以通过在模拟主机上运行的软件桥模型来驱动目标 IO。有关详细信息，请参阅 FireSim 文档。

将此配置片段添加到自定义 Chipyard 配置的最简单方法是通过 FireSim 的构建配方方案。设置 FireSim 环境后，您将在 `sims/firesim/deploy/deploy/config_build_recipes.ini` 中定义自定义构建配方。通过将 FireSim 配置片段（用 _ 分隔）添加到您的 Chipyard 配置中，这些配置片段将被添加到您的自定义配置中，就像它们列在自定义 Chisel 配置类定义中一样。例如，如果您想将 Chipyard `LargeBoomConfig` 转换为具有 DDR3 内存模型的 FireSim 仿真，则相应的 FireSim `TARGET_CONFIG` 将为 `DDR3FRFCFSLLC4MB_WithDefaultFireSimBridges_WithFireSimConfigTweaks_chipyard.LargeBoomConfig`。请注意，FireSim 配置片段是 `firesim.firesim` scala 包的一部分，因此不需要使用完整的包名称作为前缀，而 Chipyard 配置片段需要使用 chipyard 包名称作为前缀。

在 FireSim 构建配方中添加 FireSim 配置片段的另一种方法是创建一个新的“永久” FireChip 自定义配置，其中包括 FireSim 配置片段。我们使用相同的目标（顶部）RTL，并且只需要为该模块的 IO 指定一组新的连接行为。只需在 `generators/firechip/src/main/scala/TargetConfigs` 中创建一个匹配的配置，该配置继承 chipyard 中定义的配置。

```Scala
class FireSimRocketConfig extends Config(
  new WithDefaultFireSimBridges ++
  new WithDefaultMemModel ++
  new WithFireSimConfigTweaks ++
  new chipyard.RocketConfig)
```

虽然此选项似乎需要维护额外的配置代码，但它的好处是允许包含更复杂的配置片段，这些片段也接受自定义参数（例如，`WithDefaultMemModel` 可以采用可选参数``）。