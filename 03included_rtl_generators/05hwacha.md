# Hwacha

Hwacha 项目正在为电力和能源消耗受限的未来计算机系统开发一种新的矢量架构。 Hwacha 项目的灵感来自 70 年代和 80 年代的传统向量机，以及从我们之前的向量线程架构（例如 Scale 和 Maven）中汲取的经验教训。 Hwacha 项目包括 Hwacha 微架构生成器，以及 `XHwacha` 非标准 RISC-V 扩大。 Hwacha 没有实现 RISC-V 标准向量扩展提案。

有关 Hwacha 项目的更多信息，请访问 [Hwacha website](http://hwacha.org/)。

要将 Hwacha 矢量单元添加到 SoC，您应该将 `hwacha.DefaultHwachaConfig` 配置片段添加到 SoC 配置。 Hwacha 矢量单元使用 Rocket 或 BOOM 块的 RoCC 端口，默认情况下通过系统总线连接到内存系统（即直接连接到 L2 缓存）。

要更改 Hwacha 矢量单元的配置，您可以编写自定义配置来替换 `DefaultHwachaConfig`。您可以查看 [generators/hwacha/src/main/scala/configs.scala](https://github.com/ucb-bar/hwacha/blob/master/src/main/scala/configs.scala) 下的 `DefaultHwachaConfig` 来查看可能的配置参数。

由于 Hwacha 实现了非标准 RISC-V 扩展，因此需要独特的软件工具链才能编译和汇编其向量指令。要安装 Hwacha 工具链，请在 Chipyard 根目录中运行 `./scripts/build-toolchains.sh esp-tools` 命令。这可能需要一段时间，并且它将在您的 Chipyard 根目录中安装 `esp-tools-install` 目录。 `esp-tools` 是 `riscv-tools`（以前是相关软件 RISC-V 工具的集合）的一个分支，通过附加非标准向量指令进行了增强。但是，由于等效 RISC-V 工具链的上游，`esp-tools` 可能无法与其中包含的工具的最新主线版本保持同步。