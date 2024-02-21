# Initial Repository Setup

## Prerequisites

Chipyard 在基于 linux 的系统上开发和测试。

> **警告**：在 macOS 或其他基于 BSD 的系统上也可能，但需要安装 GNU 工具。推荐从 `brew` 安装 RISC-V 工具链。

> **警告**：如果使用 windows，推荐您使用 Windows Subsystem for Linux <https://learn.microsoft.com/en-us/windows/wsl/> (WSL)。

### Running on AWS EC2 with FireSim

如果您计划在 AWS EC2 实例上使用 Chipyard 以及 FireSim，您应该阅读 [FireSim documentation](https://docs.fires.im/en/latest/)。具体的，您应该遵循文档中的 [Initial Setup/Installation](https://docs.fires.im/en/latest/Initial-Setup/index.html) 章节直到 [Setting up the FireSim Repo](https://docs.fires.im/en/latest/Initial-Setup/Setting-up-your-Manager-Instance.html#setting-up-the-firesim-repo)。在这之后，您可以遵循 [Setting up the Chipyard Repo](https://chipyard.readthedocs.io/en/latest/Chipyard-Basics/Initial-Repo-Setup.html#setting-up-the-chipyard-repo) 。

### Default Requirements Installation

在 Chipyard 中，我们将使用 [Conda](https://docs.conda.io/en/latest/) 包管理工具帮助管理系统依赖。Conda 允许用户创建环境来管理例如 `make` 和 `gcc` 等系统依赖。

> **注意**：Chipyard 也能在没有安装 Conda 的系统上运行。然而，在这些系统上必须手动安装工具链和依赖。

首先，Chipyard 要求在系统上安装最新的 Conda。使用 Miniforge 安装器安装最新的 Conda 请阅读 [Conda installation instructions](https://github.com/conda-forge/miniforge/#download)。

在安装完 Conda 且添加到环境变量后，我们需要安装 git 来初始化签出仓库。您可以使用包管理工具 `yum` 或 `apt` 安装 `git`。这个 `git` 只会在第一次签出仓库时使用，我们后续将通过 Conda 使用最新的 `git`。

接下来，当初始化设置仓库时，我们将安装 [libmamba](https://www.anaconda.com/blog/a-faster-conda-for-a-growing-community) 以便快速解决依赖关系。

```shell
conda install -n base conda-libmamba-solver
conda config --set solver libmamba
```

最后，我们需要安装 `conda-lock` 到基础的 conda 环境中，由以下指令完成：

```shell
conda install -n base conda-lock==1.4.0
conda activate base
```

## Setting up the Chipyard Repo

从签出合适的 Chipyard 版本开始，运行以下指令：

```shell
git clone https://github.com/ucb-bar/chipyard.git
cd chipyard
# checkout latest official chipyard release
# note: this may not be the latest release if the documentation version != "stable"
git checkout v?.?.?
```

接下来，运行下列脚本使用具体的工具链来设置 Chipyard。存在两个工具链，一个为了普通的 RISC-V 编程，叫做 `riscv-tools`，被用于大多数 Chipyard 使用测例中，另一个用于 Hwacha 的叫做 `esp-tools`。基于您乐意使用的编译器来运行以下脚本。

> **注意**：先前版本的 Chipyard 推荐使用 `esp-tools` 用于 Gemmini 开发。目前的 Gemmini 使用标准的 `riscv-tools`。

> **警告**：以下脚本将会完成 Chipyard 的所有安装，将会花费很长时间，具体取决于系统。在继续之前确保此脚本全部完成（无中断）。用户可以使用 `--skip` 或 `-s` 标记跳过步骤：
>
> `-s 1` 跳过初始化 Conda 环境
>
> `-s 2` 跳过初始化 Chipyard 子模块
>
> `-s 3` 跳过初始化附属工具链（Spike、PK、tests、libgloss）
>
> `-s 4` 跳过 ctags 初始化
>
> `-s 5` 跳过预编译 Chipyard Scala 源码
>
> `-s 6` 跳过 FireSim 初始化
>
> `-s 7` 跳过预编译 FireSim 源码
>
> `-s 8` 跳过初始化 FireMarshal
>
> `-s 9` 跳过预编译 FireMarshal 默认构建 Linux 源码
>
> `-s 10` 跳过仓库清空

```shell
./build-setup.sh riscv-tools # or esp-tools
```

这个脚本封装了 conda 环境初始化过程、所有子模块初始化过程（使用 `init-submodules-no-riscv-tools.sh` 脚本）、安装工具链以及运行其他设置。更多的细节请阅读 `./build-setup.sh --help`。

> **警告**：直接使用 `git` 将会试图初始化所有子模块，不推荐这样做，除非您极度需要。

> **注意**：`build-setup.sh` 脚本敬爱唔会默认安装额外的工具链（RISC-V tests、PK、Spike 等）到 `$CONDA_PREFIX/<toolchain-type>`。因此，如果您使用 `conda remove` 卸载编译器，这些工具/测例将会需要重新安装/编译。

> **注意**：如果您已经由一个正在工作的 conda 环境设置，通过组合运行先前提到的脚本（`init-submodules...`、`build-toolchain...` 等），您可以在分离的 Chipyard 克隆可以使用先前使用的环境。

> **注意**：如果您是高级用户且想编译您自己的编译器/工具链，您可以阅读 https://github.com/ucb-bar/riscv-tools-feedstock 和 https://github.com/ucb-bar/esp-tools-feedstock 仓库（在 `toolchains/` 目录下的子模块）来编译您自己的编译器。

通过运行以下指令，您将会看到在 `$CHIPYARD_DIRECTORY/.conda-env` 路径下的环境列表。

```shell
conda env list
```

> **注意**：请阅读 [FireSim 的 Conda 文档](https://docs.fires.im/en/latest/Advanced-Usage/Conda.html)，了解更多有关如何使用 Conda 以及一些优点的信息。

## Sourcing `env.sh`

一旦设置完成，在仓库的顶层目录将会出现 `env.sh` 文件。该文件激活在 `build-setup.sh` 中创建的 `conda` 环境，并设置未来 Chipyard 步骤所必需的必要环境变量（为保证工作正常，需要 `make` 系统）。一旦脚本运行，将针对请求的工具链正确的设置 `PATH`、`RISCV` 以及 `LD_LIBRARY_PATH` 等环境变量。您可以在 `.bashrc` 文件中激活这个文件或者等价的环境设置文件来获取正确的变量，或者直接将其包含到您目前的环境中：

> **注意**：如果您在 Mac 或基于 RHEL/CentOS 的 Linux 发行版上工作，您在开始之前必须使用 `conda deactivate` 停用基础的 conda 环境。您可能还需要通过 `conda config --set auto_activate_base false` 使其默认情况下处于停用的状态。更多的细节请阅读 [issue](https://github.com/conda/conda/issues/9392)。

```shell
source ./env.sh
```

> **警告**：在运行任何 `make` 指令之前，需要激活 `env.sh` 文件。

> **注意**：您可以通过运行 `source $CONDA_PREFIX/etc/conda/deactivate.d/deactivate-${PKG_NAME}.sh or $CONDA_PREFIX/etc/conda/activate.d/activate-${PKG_NAME}.sh` 来激活或停用编译器/工具链（需要保证安装；`PKG_NAME` 可以是 `ucb-bar-riscv-tools`）。这将会修改前面提到的 3 个环境变量。

> **警告**：`env.sh` 文件由每个 Chipyard 仓库生成，在多个 Chipyard 仓库设置中，可能需要激活多个 `env.sh` 文件（任何顺序）。然而，推荐最后激活的 `env.sh` 文件位于是您期望运行 `make` 指令的 Chipyard 仓库下。

## DEPRECATED: Pre-built Docker Image

一种本地的候选的设置 Chipyard 仓库的方式是从 Docker Hub 拉取预编译好的 Docker 镜像。这个镜像安装了所有的依赖，克隆了 Chipyard 且安装了工具链。这个镜像只包括基础的 Chipyard（不包括 FireMarshal、FireSim 和 Hammer 初始化）。每个镜像都带有一个与该镜像中的 Chipyard 版本对应的标签。在拉取过程中不包含标签则会使用最新版本的 Chipyard 镜像。首先，通过运行以下指令拉取 Docker 镜像：

```shell
sudo docker pull ucbbar/chipyard-image:<TAG>
```

通过以下指令在交互式 shell 中运行 Docker 容器：

```shell
sudo docker run -it ucbbar/chipyard-image bash
```

## What’s Next?

这取决于您计划使用 Chipyard 做的事情。

1. 如果您打算运行普通 Chipyard 示例之一的仿真，请转至 [Software RTL Simulation](https://chipyard.readthedocs.io/en/stable/Simulation/Software-RTL-Simulation.html#sw-rtl-sim-intro) 并按照说明进行操作。
2. 如果您打算运行自定义 Chipyard SoC 配置的仿真，请转至 [Simulating A Custom Project](https://chipyard.readthedocs.io/en/stable/Simulation/Software-RTL-Simulation.html#simulating-a-custom-project) 并按照说明进行操作。
3. 如果您打算运行全系统 FireSim 仿真，请转至 [FPGA-Accelerated Simulation](https://chipyard.readthedocs.io/en/stable/Simulation/FPGA-Accelerated-Simulation.html#firesim-sim-intro) 并按照说明进行操作。
4. 如果您打算添加新的加速器，请转到 [Basic customization](https://docs.python.org/3/reference/datamodel.html#customization) 并按照说明进行操作。
5. 如果您想了解Chipyard的结构，请访问 [Chipyard Components](https://chipyard.readthedocs.io/en/stable/Chipyard-Basics/Chipyard-Components.html#chipyard-components)。
6. 如果您打算更改生成器（BOOM、Rocket 等）本身，请参阅 [Included RTL Generators](https://chipyard.readthedocs.io/en/stable/Generators/index.html#generator-index)。
7. 如果您打算使用 Chipyard 示例之一运行教程 VLSI 流程，请转至 [ASAP7 Tutorial](https://chipyard.readthedocs.io/en/stable/VLSI/ASAP7-Tutorial.html#tutorial) 并按照说明进行操作。
8. 如果您打算使用普通 Chipyard 示例之一构建芯片，请转至 [Building A Chip](https://chipyard.readthedocs.io/en/stable/VLSI/Building-A-Chip.html#build-a-chip) 并按照说明进行操作。

## Upgrading Chipyard Release Versions

为了在 Chipyard 版本之间进行升级，我们建议使用存储库的全新克隆（或您的分支，将新版本合并到其中）。

Chipyard 是一个复杂的框架，依赖于构建系统和脚本的组合。具体来说，它依赖于 git 子模块、sbt 构建文件以及自定义编写的 bash 脚本和生成的文件。因此，在 Chipyard 版本之间进行升级并不像运行 `git submodule update --recursive` 那么简单。这将导致递归克隆大型子模块，而这些子模块不一定在您的特定 Chipyard 环境中使用。此外，它不会解决过时状态生成的文件的状态，这些文件在发行版本之间可能不兼容。

如果您是高级 git 用户，克隆新存储库的另一种方法可能是运行 git clean -dfx，然后运行标准 Chipyard 设置序列。这种方法很危险，不建议不太熟悉 git 的用户使用，因为它会“炸毁”存储库状态，并在没有警告的情况下删除所有未跟踪和修改的文件。因此，如果您正在处理自定义的未提交的更改，您将会丢失它们。

如果您仍想尝试执行就地手动版本升级（不推荐），我们建议至少尝试解决以下领域的过时状态：

1. 删除 sbt 生成的陈旧目标目录。
2. 重新生成生成的脚本和源文件（例如 `env.sh`）。
3. 在 FireMarshal 中重新生成/删除目标软件状态（Linux 内核二进制文件、Linux 映像）。

这绝不是 Chipyard 内潜在陈旧状态的完整列表。因此，如前所述，Chipyard 版本升级的推荐方法是全新克隆（或合并，然后是全新克隆）。