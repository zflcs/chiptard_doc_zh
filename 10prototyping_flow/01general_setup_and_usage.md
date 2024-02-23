# General Setup and Usage

## Sources

所有与 FPGA 原型设计相关的资料和来源都位于 `fpga` 顶级 Chipyard 目录中。这包括 `fpga-shells` 子模块和包含 Scala、TCL 和其他 collateral 的 `src` 目录。

## Generating a Bitstream

使用 Vivado 为任何 FPGA 目标生成比特流类似于为软件 RTL 仿真构建 RTL。与软件 RTL 仿真（[Simulating A Custom Project](https://chipyard.readthedocs.io/en/stable/Simulation/Software-RTL-Simulation.html#simulating-a-custom-project)）类似，您可以在 `fpga` 目录中运行以下命令来使用 Vivado 构建比特流：

```shell
make SBT_PROJECT=... MODEL=... VLOG_MODEL=... MODEL_PACKAGE=... CONFIG=... CONFIG_PACKAGE=... GENERATOR_PACKAGE=... TB=... TOP=... BOARD=... FPGA_BRAND=... bitstream

# or

make SUB_PROJECT=<sub_project> bitstream
```

`SUB_PROJECT` make 变量是元 make 变量的一种方式，它将所有其他 make 变量设置为特定的默认值。例如：

```shell
make SUB_PROJECT=vcu118 bitstream

# converts to

make SBT_PROJECT=fpga_platforms MODEL=VCU118FPGATestHarness VLOG_MODEL=VCU118FPGATestHarness MODEL_PACKAGE=chipyard.fpga.vcu118 CONFIG=RocketVCU118Config CONFIG_PACKAGE=chipyard.fpga.vcu118 GENERATOR_PACKAGE=chipyard TB=none TOP=ChipTop BOARD=vcu118 FPGA_BRAND=... bitstream
```

一些 `SUB_PROJECT` 默认值已定义可供使用，包括 `vcu118` 和 `arty`。这些默认的 `SUB_PROJECT` 为 Chipyard make 系统设置了必要的测试工具、包等。与软件 RTL 仿真 make 调用一样，所有 make 变量都可以用用户特定值覆盖（例如，包括具有 `CONFIG` 和 `CONFIG_PACKAGE` 覆盖的 `SUB_PROJECT`）。在大多数情况下，您只需要运行带有 `SUB_PROJECT` 和覆盖的 `CONFIG` 指向的命令。例如，在 VCU118 上构建 BOOM 配置：

```shell
make SUB_PROJECT=vcu118 CONFIG=BoomVCU118Config bitstream
```

该命令将构建 RTL 并使用 Vivado 生成比特流。生成的比特流将位于您的设计特定的构建文件夹中（`generated-src/<LONG_NAME>/obj`）。但是，与软件 RTL 仿真一样，您也可以运行中间 make 步骤来生成 Verilog 或 FIRRTL。

## Debugging with ILAs on Supported FPGAs

ILA（集成逻辑分析仪）可以添加到某些设计中以调试相关信号。首先，在 Vivado 中打开位于构建目录中的综合后检查点（应标记为 `post_synth.dcp`）。然后使用 Vivado，为您的设计添加 ILA（和其他调试工具）（在线搜索有关如何添加 ILA 的更多信息）。这可以通过修改综合后检查点、保存它并运行 `make ... debug-bitstream` 来完成。这将在名为 `generated-src/<LONG_NAME>/debug_obj/` 的文件夹中创建一个名为 `top.bit` 的新比特流。例如，为 BOOM 配置运行添加的 ILA 的比特流构建：

```shell
make SUB_PROJECT=vcu118 CONFIG=BoomVCU118Config debug-bitstream
```

> **重要**：有关 FPGA 仿真的更广泛的调试工具，包括 printf 综合、断言综合、指令跟踪、ILA、带外分析、协同仿真等，请参阅 [FireSim](https://chipyard.readthedocs.io/en/stable/Simulation/FPGA-Accelerated-Simulation.html#firesim) 平台。