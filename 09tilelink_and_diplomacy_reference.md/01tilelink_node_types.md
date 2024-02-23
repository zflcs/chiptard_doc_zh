# TileLink Node Types

Diplomacy 将 SoC 的不同组件表示为有向无环图的节点。 TileLink 节点可以有多种不同类型。

## Client Node

TileLink clients 是通过在 A 通道上发送请求并在 D 通道上接收响应来发起 TileLink 事务的模块。如果 client 实现 TL-C，它将在 B 通道上接收探测，在 C 通道上发送释放，并在 E 通道上发送授权确认。

RocketChip/Chipyard 中的 L1 缓存和 DMA 设备具有 client 节点。

您可以将 TileLink 客户端节点添加到 LazyModule，如下所示：

```Scala
class MyClient(implicit p: Parameters) extends LazyModule {
  val node = TLClientNode(Seq(TLMasterPortParameters.v1(Seq(TLClientParameters(
    name = "my-client",
    sourceId = IdRange(0, 4),
    requestFifo = true,
    visibility = Seq(AddressSet(0x10000, 0xffff)))))))

  lazy val module = new LazyModuleImp(this) {
    val (tl, edge) = node.out(0)

    // Rest of code here
  }
}
```

`name` 参数标识 Diplomacy 图中的节点。它是 `TLClientParameters` 唯一必需的参数。

`sourceId` 参数指定此 client 将使用的源标识符的范围。由于我们在这里将范围设置为 [0, 4)，因此该客户端一次最多可以发送四个请求。每个请求在其源字段中都会有一个不同的值。该字段的默认值为 IdRange(0, 1)，这意味着它只能发送单个请求。

`requestFifo` 参数是一个布尔选项，默认为 false。如果设置为 true，client 将请求支持它的下游管理器以 FIFO 顺序发送响应（即按照发送相应请求的顺序）。

`visibility` 参数指定 client 将访问的地址范围。默认情况下，它设置为包括所有地址。在本例中，我们将其设置为包含单个地址范围 `AddressSet(0x10000, 0xffff)`，这意味着 client 只能访问从 0x10000 到 `0x1ffff` 的地址。通常不指定这一点，但它可以通过不将 client 仲裁给地址范围与其可见性不重叠的管理器来帮助下游交叉开关生成器优化硬件。

在 lazy module implementation 中，您可以调用 `node.out` 来获取束/边对的列表。

`tl` bundle 是连接到该模块的 IO 的 Chisel 硬件 bundle。它包含两个（在 TL-UL 和 TL-UH 的情况下）或五个（在 TL-C 的情况下）对应于 TileLink 通道的解耦 bundle。这是您应该将硬件逻辑连接到的地方，以便实际发送/接收 TileLink 消息。

`edge` 对象代表 Diplomacy 图的边缘。它包含一些有用的辅助函数，这些函数将记录在 [TileLink Edge Object Methods](https://chipyard.readthedocs.io/en/stable/TileLink-Diplomacy-Reference/EdgeFunctions.html#tilelink-edge-object-methods) 中。

## Manager Node

TileLink managers 在 A 通道上接收来自 client 的请求，并在 D 通道上发送回响应。您可以像这样创建一个 manager：

```Scala
class MyManager(implicit p: Parameters) extends LazyModule {
  val device = new SimpleDevice("my-device", Seq("tutorial,my-device0"))
  val beatBytes = 8
  val node = TLManagerNode(Seq(TLSlavePortParameters.v1(Seq(TLManagerParameters(
    address = Seq(AddressSet(0x20000, 0xfff)),
    resources = device.reg,
    regionType = RegionType.UNCACHED,
    executable = true,
    supportsArithmetic = TransferSizes(1, beatBytes),
    supportsLogical = TransferSizes(1, beatBytes),
    supportsGet = TransferSizes(1, beatBytes),
    supportsPutFull = TransferSizes(1, beatBytes),
    supportsPutPartial = TransferSizes(1, beatBytes),
    supportsHint = TransferSizes(1, beatBytes),
    fifoId = Some(0))), beatBytes)))

  lazy val module = new LazyModuleImp(this) {
    val (tl, edge) = node.in(0)
  }
}
```

`makeManagerNode` 方法有两个参数。第一个是 `beatBytes`，它是TileLink 接口的物理宽度（以字节为单位）。第二个是 `TLManagerParameters` 对象。

`TLManagerParameters` 唯一必需的参数是 `address`，它是该 manager 将服务的地址范围集。此信息用于路由来自 client 的请求。在此示例中，manager 将仅接受地址从 0x20000 到 0x20fff 的请求。 `AddressSet` 中的第二个参数是掩码，而不是大小。通常应将其设置为 2 的幂小一。否则，寻址行为可能不是您所期望的。

第二个参数是 `resources`，通常从 `Device` 对象中检索。在本例中，我们使用 `SimpleDevice` 对象。如果要向 BootROM 中的 DeviceTree 添加条目以便 Linux 驱动程序可以读取该条目，则此参数是必需的。 `SimpleDevice` 的两个参数是设备树条目的名称和兼容性列表。对于这个管理器来说，设备树条目将如下所示:

```
L12: my-device@20000 {
    compatible = "tutorial,my-device0";
    reg = <0x20000 0x1000>;
};
```

下一个参数是 `regionType`，它提供了有关 manager 缓存行为的一些信息。有七种区域类型，如下所示：

1. `CACHED` - 中间代理可能已为您缓存了该区域的副本。
2. `TRACKED` - 该区域可能已被另一个主设备缓存，但正在提供一致性。
3. `UNCACHED` - 该区域尚未缓存，但应尽可能缓存。
4. `IDEMPOTENT` - 获取返回最近放置的内容，但内容不应被缓存。
5. `VOLATILE` - 内容可能会在没有 put 的情况下发生变化，但 put 和 gets 没有副作用。
6. `PUT_EFFECTS` - put 会产生副作用，因此不能 combined/delayed。
7. `GET_EFFECTS` - get 会产生副作用，因此不能 be issued speculatively。

接下来是 `executable` 参数，它确定是否允许 CPU 从此 manager 获取指令。默认情况下为 false，这是大多数 MMIO 外设应将其设置为的值。

接下来的六个参数从 `support` 开始，并确定 manager 可以接受的不同 A 通道消息类型。消息类型的定义在 [TileLink Edge Object Methods](https://chipyard.readthedocs.io/en/stable/TileLink-Diplomacy-Reference/EdgeFunctions.html#tilelink-edge-object-methods) 中进行了解释。 `TransferSizes` case 类指定 manager 可以接受的特定消息类型的逻辑大小范围（以字节为单位）。这是一个包含范围，所有逻辑大小都必须是 2 的幂。因此在这种情况下，manager 可以接受大小为 1、2、4 或 8 字节的请求。

此处显示的最后一个参数是 `fifoId` 设置，它确定 manager 所在的 FIFO 域（如果有）。如果此参数设置为 `None`（默认值），manager 将不保证响应的任何顺序。如果设置了 `fifoId`，它将与指定相同 `fifoId` 的所有其他管理器共享 FIFO 域。这意味着发送到该 FIFO 域的 client 请求将以相同的顺序看到响应。

## Register Node

虽然您可以直接指定 manager 节点并编写所有逻辑来处理 TileLink 请求，但使用 register 节点通常要容易得多。这种类型的节点提供了 `regmap` 方法，允许您指定 control/status 寄存器并自动生成处理 TileLink 协议的逻辑。有关如何使用注册节点的更多信息可以在 [Register Router](https://chipyard.readthedocs.io/en/stable/TileLink-Diplomacy-Reference/Register-Router.html#register-router) 中找到。

## Identity Node

与之前只有输入或只有输出的节点类型不同，Identity 节点两者都有。顾名思义，它只是将输入连接到输出而不改变。该节点主要用于将多个节点组合成具有多条边的单个节点。例如，假设我们有两个 client lazy module，每个模块都有自己的 client 节点。

```Scala
class MyClient1(implicit p: Parameters) extends LazyModule {
  val node = TLClientNode(Seq(TLMasterPortParameters.v1(Seq(TLClientParameters(
    "my-client1", IdRange(0, 1))))))

  lazy val module = new LazyModuleImp(this) {
    // ...
  }
}

class MyClient2(implicit p: Parameters) extends LazyModule {
  val node = TLClientNode(Seq(TLMasterPortParameters.v1(Seq(TLClientParameters(
    "my-client2", IdRange(0, 1))))))

  lazy val module = new LazyModuleImp(this) {
    // ...
  }
}
```

现在，我们在另一个 lazy module 中实例化这两个 client，并将它们的节点公开为单个节点。

```Scala
class MyClientGroup(implicit p: Parameters) extends LazyModule {
  val client1 = LazyModule(new MyClient1)
  val client2 = LazyModule(new MyClient2)
  val node = TLIdentityNode()

  node := client1.node
  node := client2.node

  lazy val module = new LazyModuleImp(this) {
    // Nothing to do here
  }
}
```

我们也可以为 manager 做同样的事情。

```Scala
class MyManager1(beatBytes: Int)(implicit p: Parameters) extends LazyModule {
  val node = TLManagerNode(Seq(TLSlavePortParameters.v1(Seq(TLManagerParameters(
    address = Seq(AddressSet(0x0, 0xfff)))), beatBytes)))

  lazy val module = new LazyModuleImp(this) {
    // ...
  }
}

class MyManager2(beatBytes: Int)(implicit p: Parameters) extends LazyModule {
  val node = TLManagerNode(Seq(TLSlavePortParameters.v1(Seq(TLManagerParameters(
    address = Seq(AddressSet(0x1000, 0xfff)))), beatBytes)))

  lazy val module = new LazyModuleImp(this) {
    // ...
  }
}

class MyManagerGroup(beatBytes: Int)(implicit p: Parameters) extends LazyModule {
  val man1 = LazyModule(new MyManager1(beatBytes))
  val man2 = LazyModule(new MyManager2(beatBytes))
  val node = TLIdentityNode()

  man1.node := node
  man2.node := node

  lazy val module = new LazyModuleImp(this) {
    // Nothing to do here
  }
}
```

如果我们想将 client 和 manager 连接在一起，我们现在就可以这样做。

```Scala
class MyClientManagerComplex(implicit p: Parameters) extends LazyModule {
  val client = LazyModule(new MyClientGroup)
  val manager = LazyModule(new MyManagerGroup(8))

  manager.node :=* client.node

  lazy val module = new LazyModuleImp(this) {
    // Nothing to do here
  }
}
```

`:=*` 运算符的含义在 [Diplomacy Connectors](https://chipyard.readthedocs.io/en/stable/TileLink-Diplomacy-Reference/Diplomacy-Connectors.html#diplomacy-connectors) 部分中有更详细的解释。总之，它使用多个边将两个节点连接在一起。 Identity 节点中的边是按顺序分配的，因此在这种情况下，`client1.node` 最终将连接到 `manager1.node`，`client2.node` 将连接到 `manager2.node`。

Identity 节点的输入数量应与输出数量匹配。不匹配将导致描述错误。

## Adapter Node

与 Identity 节点一样，Adapter 节点接受一定数量的输入并产生相同数量的输出。然而，与 Identity 节点不同，Adapter 节点并不简单地让连接保持不变。它可以改变输入和输出之间的逻辑和物理接口，并重写所经过的消息。 RocketChip 提供了一个适配器库，这些适配器的条目在 [Diplomatic Widgets](https://chipyard.readthedocs.io/en/stable/TileLink-Diplomacy-Reference/Widgets.html#diplomatic-widgets) 下。

您很少需要自己创建适配器节点，但调用如下。

```Scala
val node = TLAdapterNode(
  clientFn = { cp =>
    // ..
  },
  managerFn = { mp =>
    // ..
  })
```

`clientFn` 是一个函数，它将输入的 `TLClientPortParameters` 作为参数并返回输出的相应参数。 `managerFn` 将输出的 `TLManagerPortParameters` 作为参数，并返回输入的相应参数。

## Nexus Node

Nexus 节点与 Adapter 节点类似，因为它具有与输入接口不同的输出接口。但它也可以具有与输出数量不同的输入。该节点类型主要由 `TLXbar` 小部件使用，它提供了 TileLink crossbar 生成器。您可能也不需要手动定义此节点类型，但其调用如下。

```Scala
val node = TLNexusNode(
  clientFn = { seq =>
    // ..
  },
  managerFn = { seq =>
    // ..
  })
```

它具有与 Adapter 节点的构造函数类似的参数，但函数不是采用单个参数对象作为参数并返回单个对象作为结果，而是采用并返回参数序列。正如您所期望的，返回序列的大小不需要与输入序列的大小相同。
