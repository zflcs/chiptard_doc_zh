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

一旦设置完成，在仓库的顶层目录将会出现 `env.sh` 文件。
