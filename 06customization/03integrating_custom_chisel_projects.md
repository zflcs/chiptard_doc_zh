# Integrating Custom Chisel Projects into the Generator Build System

> **警告**：本节假设通过 git 子模块集成自定义 Chisel。虽然可以直接将自定义 Chisel 提交到 Chipyard 框架中，但我们强烈建议通过 git 子模块管理自定义代码。使用子模块将自定义功能的开发与 Chipyard 框架上的开发分离。

在开发时，您希望将 Chisel 代码包含在子模块中，以便不同项目可以共享它。要将子模块添加到 Chipyard 框架，请确保您的项目按如下方式组织。

```
yourproject/
    build.sbt
    src/main/scala/
        YourFile.scala
```

将其放入 git 存储库并使其可访问。然后将其作为子模块添加到以下目录层次结构下：`generators/yourproject`。

`build.sbt` 是一个最小文件，用于描述 Chisel 项目的元数据。对于一个简单的项目，`build.sbt` 甚至可以为空，但下面我们提供了一个示例build.sbt。

```Scala
organization := "edu.berkeley.cs"

version := "1.0"

name := "yourproject"

scalaVersion := "2.12.4"
```

```shell
cd generators/
git submodule add https://git-repository.com/yourproject.git
```

然后将 `yourproject` 添加到 Chipyard 顶级 build.sbt 文件中。

```Scala
lazy val yourproject = (project in file("generators/yourproject")).settings(commonSettings).dependsOn(rocketchip)
```

如果将子模块添加为依赖项，则可以将子模块中定义的类导入到新项目中。例如，如果您想在 chipyard 项目中使用此代码，请将您的项目添加到 lazy val chipyard 的 `.dependsOn()` 中的子项目列表中。原始代码可能会随着时间的推移而改变，但它应该看起来像这样：

```Scala
lazy val chipyard = (project in file("generators/chipyard"))
    .dependsOn(testchipip, rocketchip, boom, hwacha, rocketchip_blocks, rocketchip_inclusive_cache, iocell,
        sha3, dsptools, `rocket-dsp-utils`,
        gemmini, icenet, tracegen, cva6, nvdla, sodor, ibex, fft_generator,
        yourproject, // <- added to the middle of the list for simplicity
        constellation, mempress)
    .settings(libraryDependencies ++= rocketLibDeps.value)
    .settings(
        libraryDependencies ++= Seq(
        "org.reflections" % "reflections" % "0.10.2"
        )
    )
    .settings(commonSettings)
```