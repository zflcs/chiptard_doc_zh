# TileLink Edge Object Methods

与 TileLink 节点关联的 `edge` 对象有几个有用的方法来构造 TileLink 消息并从中检索数据。

## Get

TLBundleA 的构造函数，编码 Get 消息，从内存请求数据。D 通道对该消息的响应将是一个 `AccessAckData`，它可能有多个节拍。

参数：
    1. `romSource: UInt` - 事务的 source ID。
    2. `toAddress: UInt` - 要读取的地址。
    3. `lgSize：UInt` - 要读取的字节数的以 2 为底的对数。
返回值：
    (Bool, TLBundleA) 元组。该对中的第一项是一个布尔值，指示该操作对于该边是否合法。第二个是 A 通道 bundle。

## Put

TLBundleA 的构造函数，编码 `PutFull` 或 `PutPartial` 消息，将数据写入内存。如果指定了掩码，则它将是 `PutPartial`；如果省略，则它将是 `PutFull`。put 可能需要多次节拍。如果是这种情况，则每个节拍仅应更改 `data` 和 `mask`。对于事务中的所有节拍，所有其他字段必须相同，包括地址。manager 将使用单个 `AccessAck` 响应此消息。

参数：
    1. `romSource: UInt` - 事务的 source ID。
    2. `toAddress: UInt` - 要读取的地址。
    3. `lgSize：UInt` - 要读取的字节数的以 2 为底的对数。
    4. `data: UInt` - 要在此节拍上写入的数据。
    5. `mask: UInt` - （可选）此节拍的写入掩码。
返回值：
    (Bool, TLBundleA) 元组。该对中的第一项是一个布尔值，指示该操作对于该边是否合法。第二个是 A 通道 bundle。

## Arithmetic

TLBundleA 的构造函数，对 `Arithmetic` 消息进行编码，这是一个原子操作。原子字段的可能值在 `TLAtomics` 对象中定义。它可以是 `MIN`、`MAX`、`MINU`、`MAXU` 或 `ADD`，分别对应于原子最小值、最大值、无符号最小值、无符号最大值或加法运算。内存位置的先前值将在响应中返回，该值将以 `AccessAckData` 的形式出现。

参数：
    1. `romSource: UInt` - 事务的 source ID。
    2. `toAddress: UInt` - 要读取的地址。
    3. `lgSize：UInt` - 要读取的字节数的以 2 为底的对数。
    4. `data: UInt` - 算术运算的右侧操作数。
    5. `atomic: UInt` - 算术运算类型（来自 `TLAtomics`）
返回值：
    (Bool, TLBundleA) 元组。该对中的第一项是一个布尔值，指示该操作对于该边是否合法。第二个是 A 通道 bundle。

## Logical

TLBundleA 的构造函数，编码 `Logical` 消息，原子操作。原子字段的可能值为 `XOR`、`OR`、`AND` 和 `SWAP`，分别对应于原子按位异或、按位或、按位与和交换操作。内存位置的先前值将在 `AccessAckData` 响应中返回。

参数：
    1. `romSource: UInt` - 事务的 source ID。
    2. `toAddress: UInt` - 要读取的地址。
    3. `lgSize：UInt` - 要读取的字节数的以 2 为底的对数。
    4. `data: UInt` - 算术运算的右侧操作数。
    5. `atomic: UInt` - 算术运算类型（来自 `TLAtomics`）
返回值：
    (Bool, TLBundleA) 元组。该对中的第一项是一个布尔值，指示该操作对于该边是否合法。第二个是 A 通道 bundle。

## Hint

TLBundleA 的构造函数，编码 `Hint` 消息，用于将预取 hint 发送到缓存。 `param` 参数决定了它是什么类型的 hint。可能的值来自 `TLHints` 对象，包括 `PREFETCH_READ` 和 `PREFETCH_WRITE`。第一个告诉缓存获取共享状态的数据。第二个告诉缓存以独占状态获取数据。如果该消息到达的缓存是末级缓存，则没有任何区别。如果该消息到达的 manager 不是缓存，则它将被忽略。无论如何，都会发送一条 `HintAck` 消息作为响应。

参数：
    1. `romSource: UInt` - 事务的 source ID。
    2. `toAddress: UInt` - 要读取的地址。
    3. `lgSize：UInt` - 要读取的字节数的以 2 为底的对数。
    4. `param: UInt` - hint 类型（来自 TLHints）
返回值：
    (Bool, TLBundleA) 元组。该对中的第一项是一个布尔值，指示该操作对于该边是否合法。第二个是 A 通道 bundle。

## AccessAck

编码 `AccessAck` 或 `AccessAckData` 消息的 TLBundleD 的构造函数。如果提供了可选 `data` 字段，它将是 `AccessAckData`。否则，它将是一个 `AccessAck`。

参数：
    1. `a: TLBundleA` - 要确认的 A 通道消息。
    2. `data: UInt` - （可选）要发回的数据。
返回值：
    D 通道消息的 `TLBundleD`。

## HintAck

编码 `HintAck` 消息的 TLBundleD 的构造函数。

参数：
    1. `a: TLBundleA` - 要确认的 A 通道消息。
返回值：
    D 通道消息的 `TLBundleD`。

## first

此方法采用解耦通道（A 通道或 D 通道）并确定当前节拍是否是事务中的第一个节拍。

参数：
    1. `x: De CoupledIO[TLChannel]` - 要监听的解耦通道。
返回值：
    一个布尔值，如果当前节拍是第一个，则为 true，否则为 false。

## last

此方法采用解耦通道（A 通道或 D 通道）并确定当前节拍是否是事务中的最后一个节拍。

参数：
    1. `x: De CoupledIO[TLChannel]` - 要监听的解耦通道。
返回值：
    一个布尔值，如果当前节拍是最后一个，则为 true，否则为 false。

## done

等价于 `x.fire() && last(x)`。

参数：
    1. `x: De CoupledIO[TLChannel]` - 要监听的解耦通道。
返回值：
    如果当前节拍是最后一个并且在此周期发送节拍，则为 true。否则为 false。

## count

此方法采用解耦通道（A 通道或 D 通道）并确定事务中当前节拍的计数（从 0 开始）。

参数：
    1. `x: De CoupledIO[TLChannel]` - 要监听的解耦通道。
返回值：
    指示当前节拍计数的 UInt。

## numBeats

此方法接受 TileLink bundle 并给出事务的节拍数。

参数：
    1. `x：TLChannel` - 用于获取节拍数的 TileLink bundle。
返回值：
    一个 UInt，表示当前事务中的节拍数。

## numBeats1

与 numBeats 类似，只不过它给出的节拍数减一。如果这是您所需要的，您应该使用它而不是执行 `numBeats - 1.U`，因为这样更有效。

此方法接受 TileLink bundle 并给出事务的节拍数。

参数：
    1. `x：TLChannel` - 用于获取节拍数的 TileLink bundle。
返回值：
    一个 UInt，表示当前事务中的节拍数减一。

## hasData

确定 TileLink 消息是否包含数据。如果消息是 PutFull、PutPartial、Arithmetic、Logical 或 AccessAckData，则为 true。

参数：
    1. `x：TLChannel` - 要检查的 TileLink bundle。
返回值：
    如果当前消息有数据则为 true 的布尔值，否则为 false。

