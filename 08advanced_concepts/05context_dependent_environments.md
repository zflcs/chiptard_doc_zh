# Context-Dependent-Environments

读者可能会注意到参数化系统经常使用（`site`，`here`，`up`）。这种构造是“上下文相关环境”系统的产物，Chipyard 和 Rocket Chip 都利用该系统来实现强大的可组合硬件配置。

CDE 参数化系统提供单个全局参数化的不同“视图”。访问 `View` 中 `Field` 的语法是 `my_view(MyKey, site_view)`，其中 `site_view` 是一个“全局” view，它将递归传递到 `my_view(MyKey, site_view)` 调用堆栈中的各种函数和 key-lookups 中。

> **注意**：基于 Rocket Chip 的设计将经常使用 `val p:Parameters` 和 `p(SomeKey)` 来查找 key 的值。 `Parameters` 只是 `View` 抽象类的子类，`p(SomeKey)` 实际上扩展为 `p(SomeKey, p)`。这是因为我们认为调用 `p(SomeKey)` 是原始 key 查询的“site”或“source”，因此我们需要通过 `site` 参数将 `p` 提供的配置视图递归地传递给将来的调用。

考虑以下使用 CDE 的示例。

```Scala
case object SomeKeyX extends Field[Boolean](false) // default is false
case object SomeKeyY extends Field[Boolean](false) // default is false
case object SomeKeyZ extends Field[Boolean](false) // default is false

class WithX(b: Boolean) extends Config((site, here, up) => {
  case SomeKeyX => b
})

class WithY(b: Boolean) extends Config((site, here, up) => {
  case SomeKeyY => b
})
```

当基于 `Parameters` 对象（如 `p(SomeKeyX)`）形成查询时，配置系统会遍历配置片段的“链”，直到找到在 key 处定义的部分函数，​​然后返回该值。

```Scala
val params = new Config(new WithX(true) ++ new WithY(true)) // "chain" together config fragments
params(SomeKeyX) // evaluates to true
params(SomeKeyY) // evaluates to true
params(SomeKeyZ) // evaluates to false
```

在此示例中，`params(SomeKeyX)` 的计算将在 `WithX(true)` 中定义的部分函数中终止，而 `params(SomeKeyY)` 的计算将在 `WithY(true)` 中定义的部分函数中终止。请注意，当没有部分函数匹配时，计算将返回该参数的默认值。

配置片段从左到右优先，这意味着链开头的片段可以覆盖右侧片段的值。它有助于从右到左读取片段链。

```Scala
case object SomeKeyX extends Field[Int](0)

class WithX(n: Int) extends Config((site, here, up) => {
  case SomeKeyX => n
})

val params = new Config(new WithX(10) ++ new WithX(5))
println(params(SomeKeyX)) // evaluates to 10
```

CDE 的真正能力来自于 partial 函数的（site，here，up）参数，这些参数为全局参数化提供了有用的“视图”，partial 函数可以访问该视图来确定参数化。

> **注意**：关于 CDE 动机的更多信息可以在 [Henry Cook’s Thesis](https://www2.eecs.berkeley.edu/Pubs/TechRpts/2016/EECS-2016-89.pdf) 的第二章中找到。

## Site

`site` 提供了原始参数查询“源”的 `View`。

```Scala
class WithXEqualsYSite extends Config((site, here, up) => {
  case SomeKeyX => site(SomeKeyY) // expands to site(SomeKeyY, site)
})

val params_1 = new Config(new WithXEqualsYSite ++ new WithY(true))
val params_2 = new Config(new WithY(true) ++ new WithXEqualsYSite)
params_1(SomeKeyX) // evaluates to true
params_2(SomeKeyX) // evaluates to true
```

在此示例中，`WithXEqualsYSite` 中的 partial 函数将在原始 `params_N` 对象中查找 `SomeKeyY` 的值，该对象在递归遍历中的每次调用中都会成为 `site`。

## Here

`here` 提供了本地定义的配置的 `View`，通常只包含一些 partial 功能。

```Scala
class WithXEqualsYHere extends Config((site, here, up) => {
  case SomeKeyY => false
  case SomeKeyX => here(SomeKeyY, site)
})

val params_1 = new Config(new WithXEqualsYHere ++ new WithY(true))
val params_2 = new Config(new WithY(true) ++ new WithXEqualsYHere)

params_1(SomeKeyX) // evaluates to false
params_2(SomeKeyX) // evaluates to false
```

在此示例中，请注意，尽管 `params_2` 中的最终参数化将 `SomeKeyY` 设置为 `true`，但对 `here(SomeKeyY, site)` 的调用仅查找 `WithXEqualsYHere` 中定义的本地部分函数。请注意，我们将 `site` 传递到 `here`，因为 `site` 可能会在递归调用中使用。

`up` 提供了 partial 函数“链”中先前定义的 partial 函数集的 `View`。当我们想要查找某个 key 的先前设置值而不是该 key 的最终值时，这非常有用。

```Scala
class WithXEqualsYUp extends Config((site, here, up) => {
  case SomeKeyX => up(SomeKeyY, site)
})

val params_1 = new Config(new WithXEqualsYUp ++ new WithY(true))
val params_2 = new Config(new WithY(true) ++ new WithXEqualsYUp)

params_1(SomeKeyX) // evaluates to true
params_2(SomeKeyX) // evaluates to false
```

在此示例中，请注意 `WithXEqualsYUp` 中的 `up(SomeKeyY, site)` 将如何引用 `WithY(true)` 中定义 `SomeKeyY` 的 partial 函数或原始 `case object SomeKeyY` 定义中提供的 `SomeKeyY` 的默认值，具体取决于其中的顺序使用了配置片段。由于配置片段的顺序会影响 View 遍历的顺序，因此 `up` 在 `params_1` 和 `params_2` 中提供了不同的参数化 `View`。

还要再次注意，`site` 必须通过调用递归地传递到 `up`。


