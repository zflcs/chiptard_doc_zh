# Managing Published Scala Dependencies

为了准备 Chisel 3.5，在 Chipyard 1.5 Chisel 中，FIRRTL、FIRRTL 解释器和 Treadle 从源代码构建转变为作为已发布依赖项进行管理。它们的子模块已被删除。发布版本之间的切换可以通过更改 Chipyard 的 `build.sbt` 中指定的版本来实现。

可用工件的列表可以使用 search.maven.org 或 mvnrepository.org：

1. [Chisel3](https://mvnrepository.com/artifact/edu.berkeley.cs/chisel3)
2. [FIRRTL](https://mvnrepository.com/artifact/edu.berkeley.cs/firrtl)
3. [FIRRTL Interpreter](https://mvnrepository.com/artifact/edu.berkeley.cs/firrtl-interpreter)
4. [Treadle](https://mvnrepository.com/artifact/edu.berkeley.cs/treadle)

## Publishing Local Changes

在新系统下，对上述包进行自定义源修改的最简单方法是从每个存储库的本地修​​改克隆中运行 `sbt +publishLocal`。这会将您的自定义变体发布到本地 ivy2 存储库，通常可以在 `~/.ivy2` 中找到。有关更多详细信息，请参阅 [SBT documentation](https://www.scala-sbt.org/1.x/docs/Publishing.html#Publishing+locally)。

实际上，这需要执行以下步骤：

1. 检查并修改所需的项目。
2. 记下或修改每个项目的版本（在其 `build.sbt` 中）。如果您从 `master` 克隆，通常这些将具有 `1.X-SNAPSHOT` 的默认版本，其中 `X` 尚未发布下一个主要版本。您可以修改版本字符串，例如 `1.X-<MYSUFFIX>`，以唯一标识您的更改。
3. 在每个子项目中调用 `sbt +publishLocal`。您可能需要重建其他已发布的依赖项。 SBT 将明确其发布的内容以及发布的位置。`+` 通常是必需的，可确保发布包的所有交叉版本。
4. 更新 Chipyard 的 `build.sbt` 中的 Chisel 或 FIRRTL 版本，以匹配本地发布的软件包的版本。
5. 像平常一样使用 Chipyard。现在，当您在 Chipyard 中调用 make 时，您应该会看到 SBT 解决了对本地 ivy2 存储库中本地发布的实例的依赖关系。
6. 完成后，请考虑删除本地发布的包（通过删除 ivy2 存储库中的相应目录），以防止将来意外地重复使用它们。

最后要注意的是：您发布到本地 ivy 存储库的包将对您可能在系统上构建的其他项目可见。例如，如果您在本地发布 Chisel 3.5.0，则依赖 Chisel 3.5.0 的其他项目将优先使用本地发布的变体，而不是 Maven 上可用的版本（“真正的”3.5.0）。请注意记下您正在发布的版本，并在完成后删除本地发布的版本。