# Core Hammer

[Hammer](https://github.com/ucb-bar/hammer) 是一种物理设计流程，它通过将物理设计规范划分为三个不同的关注点来鼓励可重用性：设计、CAD 工具和工艺技术。 Hammer 围绕供应商特定的技术和工具提供单一 API 来解决 ASIC 设计问题。 Hammer 允许 ASIC 设计中的可重用性，同时仍然为设计人员提供自行修改的余地。

有关更多信息，请阅读 [Hammer paper](https://dl.acm.org/doi/abs/10.1145/3489517.3530672) 并查看 [GitHub 仓库](https://github.com/ucb-bar/hammer) 和相关文档。

Hammer 使用以下高级结构实现 VLSI 流程：

## Actions

Actions 是 Hammer 能够执行的顶级任务（例如综合、布局布线等）。

## Steps

Steps 是可在 Hammer 中单独寻址的操作的子组件（例如布局布线操作中的放置）。

## Hooks

Hooks 是对 Hammer 配置中以编程方式定义的步骤或操作的修改。

## VLSI Flow Control

有时我们希望对 VLSI 流进行比操作级别更细粒度的控制。 Hammer 流支持能够在特定操作中的任何步骤之前/之后启动/停止。有关选项的完整列表和说明，请参阅 [Hammer documentation on Flow Control](https://hammer-vlsi.readthedocs.io/en/latest/Hammer-Use/Flow-Control.html)。 `vlsi` 目录中的 `Makefile` 通过 `HAMMER_EXTRA_ARGS` 变量传递此额外信息。此变量还可用于指定初始构建中可能已更改或省略的其他 YAML 配置。

## Configuration (Hammer IR)

要配置 Hammer 流，请提供一组 `yaml` 或 `json` 配置文件，用于选择工具和技术插件和版本以及任何设计特定的配置选项。总的来说，此配置 API 称为 Hammer IR，可以从更高级别的抽象生成。

当前所有可用的 Hammer API 集均已编入[此处](https://github.com/ucb-bar/hammer/blob/master/hammer/config/defaults.yml)。

## Tool Plugins

Hammer 支持不同 CAD 工具供应商单独管理的插件。在获得相应 CAD 工具供应商的许可后，您也许能够访问所包含的 Mentor 插件子模块。目前支持的工具类型（按 Hammer 名称）包括：

1. synthesis
2. par
3. drc
4. lvs
5. sram_generator
6. sim
7. power
8. pcb

需要几个配置变量来配置您选择的工具插件。

首先，通过将 `vlsi.core.<tool_type>_tool` 设置为工具的包名称来选择每个操作要使用的工具，例如 `vlsi.core.par_tool："hammer.par.innovus"`。

该包目录应包含一个以该工具名称命名的文件夹，该文件夹本身包含一个 python 文件 `__init__.py` 和一个 yaml 文件 `defaults.yml`。通过将 `<tool_type>.<tool_name>.version` 设置为工具特定字符串来自定义工具的版本。

`__init__.py` 文件应包含一个变量 `tool`，它指向实现该工具的类。该类应该是 `Hammer<tool_type>Tool` 的子类，而 Tool 也将是 `HammerTool` 的子类。该类应该实现该工具的所有步骤的方法。

`defaults.yml` 文件包含特定于工具的配置变量。必要时可以覆盖默认值。

## Technology Plugins

Hammer 支持单独管理的技术插件以满足 NDA 的要求。经技术供应商许可，您也许能够访问某些预构建的技术插件。或者，要构建您自己的技术插件，您至少需要一个 `<tech_name>.tech.json` 和 `defaults.yml`。如果有任何特定于技术的方法或挂钩要运行，则 `__init__.py` 是可选的。

[ASAP7 插件](https://github.com/ucb-bar/hammer/blob/master/hammer/technology/asap7) 是设置技术插件的一个很好的起点，因为它是一个开源示例，不适合流片芯片。有关架构和详细设置说明，请参阅 Hammer 的文档。

需要几个配置变量来配置您选择的技术。

首先，选择技术包，例如 `vlsi.core.technology：hammer.technology.asap7`，然后指向带有 `technology.<tech_name>.tarball_dir` 的 PDK tarball 的位置或带有 `technology.<tech_name>.install_dir` 的预安装目录。

特定于技术的选项（例如电源、MMMC 角等）在其各自的 `vlsi.inputs...` 配置中定义。最常见用例的选项已在技术的 `defaults.yml` 中定义，并且可以由用户覆盖。