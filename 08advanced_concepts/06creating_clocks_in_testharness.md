# Creating Clocks in the Test Harness

Chipyard 目前允许SoC设计（`ChipTop` 旗下的所有产品）通过 diplomacy 拥有独立的时钟域。 `ChipTop` 时钟端口由`harnessClockInstantiator.requestClock(freq)` 驱动。 `ChipTop` 复位端口由 `referenceReset()` 函数驱动，旨在提供异步复位。

`ChipTop` 中的 `HarnessBinder` 由 `HarnessBinderClockFrequencyKey` 值提供时钟。复位作为同步复位提供，与时钟同步。

对 harness 时钟的请求由 `generators/chipyard/src/main/scala/harness/HarnessClocks.scala` 中的 `HarnessClockInstantiator` 类完成。然后，您可以通过调用 `requestClock` 函数来请求时钟并以特定频率同步复位。举个例子：

```Scala
    port.io := th.harnessClockInstantiator.requestClockMHz(s"clock_${port.freqMHz}MHz", port.freqMHz)
```

在这里您可以看到 `th.harnessClockInstantiator` 用于请求时钟并以 `memFreq` 频率重置。