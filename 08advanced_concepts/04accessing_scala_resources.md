# Accessing Scala Resources

将源文件复制到构建目录以用于模拟编译或 VLSI 流程的一种简单方法是使用 FIRRTL 提供的 `addResource` 函数。其使用示例可以在 [generators/testchipip/src/main/scala/SimTSI.scala](https://github.com/ucb-bar/testchipip/blob/master/src/main/scala/SimTSI.scala) 中看到。这是内联的示例：

```Scala
class SimTSI extends BlackBox with HasBlackBoxResource {
  val io = IO(new Bundle {
    val clock = Input(Clock())
    val reset = Input(Bool())
    val tsi = Flipped(new TSIIO)
    val exit = Output(Bool())
  })

  addResource("/testchipip/vsrc/SimTSI.v")
  addResource("/testchipip/csrc/SimTSI.cc")
}
```

在此示例中，`SimTSI` 文件将从特定文件夹（在本例中为路径 `/to/testchipip/src/main/resources/testchipip/...`）复制到构建文件夹。`addResource` 路径从 `src/main/resources` 目录检索资源。因此，要获取 `src/main/resources/fileA.v` 中的项目，您可以使用 `addResource("/fileA.v")`。但是，此方法的一个警告是，要在 FIRRTL 编译期间检索文件，您必须在 FIRRTL 编译器的类路径中包含该项目。因此，您需要将 SBT 项目添加为 Chipyard `build.sbt` 中 FIRRTL 编译器的依赖项，在 Chipyards 中，该项目是 `tapeout` 项目。例如，您在 Chipyard `build.sbt` 中添加了一个名为 `myAwesomeAccel` 的新项目。然后您可以将其作为 `dependsOn` 依赖项添加到 `tapeout` 项目中。例如：

```Scala
lazy val myAwesomeAccel = (project in file("generators/myAwesomeAccelFolder"))
  .dependsOn(rocketchip)
  .settings(commonSettings)

lazy val tapeout = conditionalDependsOn(project in file("./tools/barstools/tapeout/"))
  .dependsOn(myAwesomeAccel)
  .settings(commonSettings)
```

