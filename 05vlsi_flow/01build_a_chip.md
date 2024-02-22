# Building A Chip

在本节中，我们将讨论 Chipyard 内的许多特定于 ASIC 的转换和方法。有关如何使用 VLSI 工具流程的完整文档，请参阅 [Hammer Documentation](https://hammer-vlsi.readthedocs.io/)。

## Transforming the RTL

构建芯片需要专门化 FIRRTL 发出的通用 verilog，以遵守制造技术所施加的限制。这包括将 Chisel 内存映射到可用的技术宏（例如 SRAM），映射芯片的输入和输出以连接到技术 IO 单元，请参阅 [Barstools](https://chipyard.readthedocs.io/en/stable/Tools/Barstools.html#barstools)。除了这些必需的转换之外，转换 RTL 使其更容易分层物理设计也可能是有益的。这通常包括通过将组件分组在一起或将组件扁平化为单个更大的模块来修改逻辑层次结构以匹配物理层次结构。

### Modifying the logical hierarchy

构建大型或复杂的芯片通常需要使用分层设计来单独放置和路由芯片的各个部分。此外，Chipyard 中编写的设计可能没有与布局布线工具中最适合的物理层次结构相匹配的层次结构。为了重新组织设计以使其逻辑层次结构与其物理层次结构相匹配，可以运行多个 FIRRTL 转换。其中包括分组（将多个模块拉成一个更大的模块）和展平（消除模块边界，将其组件保留在其包含的模块中）。这些转换可以重复应用于设计的不同部分，以按照物理设计师认为合适的方式进行排列。有关如何使用这些转换来重新组织设计层次结构的更多详细信息即将发布。

## Creating a floorplan

ASIC 布局规划是布局布线工具在设计中放置实例时将遵循的规范。这包括顶层芯片尺寸、SRAM 宏的布局、定制（模拟）电路的布局、IO 单元布局、凸块或焊线焊盘布局、阻塞、分层边界和引脚布局。

构建芯片的大部分设计工作都涉及为正在制造的设计实例开发最佳布局规划。通常，这是一个高度手动和迭代的过程，消耗了物理设计师的大量时间。当使用 Chisel 等工具时，随着参数化空间快速增长，这种成本变得越来越明显，而循环时间受到对设计的每个实例进行布局规划所需的人力的阻碍。 Hammer 团队正在积极开发方法，以提高基于生成器的设计（例如使用 Chisel 的设计）的布局规划敏捷性。我们正在开发的库将发出 Hammer IR，可以直接传递到 Hammer 工具，无需人工干预。敬请期待更多的信息。

同时，请参阅 [Hammer Documentation](https://hammer-vlsi.readthedocs.io/)，了解有关 Hammer IR 布局 API 的信息。可以直接编写此 IR，也可以使用简单的 python 脚本生成它。虽然我们当然期待拥有一个功能更丰富的工具包，但迄今为止我们已经以这种方式构建了许多芯片。

## Running the VLSI tool flow

有关如何使用 VLSI 工具流程的完整文档，请参阅 [Hammer Documentation](https://hammer-vlsi.readthedocs.io/)。有关 Chipyard 上下文中 VLSI 工具流程的设置和说明，请参阅 [Using Hammer To Place and Route a Custom Block](https://chipyard.readthedocs.io/en/stable/VLSI/Basic-Flow.html#hammer-basic-flow)。具体示例请参见 [ASAP7 Tutorial](https://chipyard.readthedocs.io/en/stable/VLSI/ASAP7-Tutorial.html#tutorial)、[Sky130 Commercial Tutorial](https://chipyard.readthedocs.io/en/stable/VLSI/Sky130-Commercial-Tutorial.html#sky130-commercial-tutorial)、[Sky130 + OpenROAD Tutorial](https://chipyard.readthedocs.io/en/stable/VLSI/Sky130-OpenROAD-Tutorial.html#sky130-openroad-tutorial)。