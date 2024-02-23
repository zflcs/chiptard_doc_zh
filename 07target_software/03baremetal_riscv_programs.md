# Baremetal RISC-V Programs

为了构建在模拟中运行的裸机 RISC-V 程序，我们使用 riscv64-unknown-elf 交叉编译器和 libgloss 板级支持包的分支。要自己构建这样的程序，只需使用标志“-fno-common -fno-builtin-printf -specs=htif_nano.specs”和带有参数“-static -specs=htif_nano.specs”的链接来调用交叉编译器。例如，如果我们想在裸机中运行“Hello, World”程序，我们可以执行以下操作。

```C
#include <stdio.h>

int main(void)
{
    printf("Hello, World!\n");
    return 0;
}
```

```shell
$ riscv64-unknown-elf-gcc -fno-common -fno-builtin-printf -specs=htif_nano.specs -c hello.c
$ riscv64-unknown-elf-gcc -static -specs=htif_nano.specs hello.o -o hello.riscv
$ spike hello.riscv
Hello, World!
```

有关更多示例，请查看 chipyard 存储库中的 [tests/directory](https://github.com/ucb-bar/chipyard/tree/master/tests)。

有关 libgloss 的更多信息，请查看其 [README](https://github.com/ucb-bar/libgloss-htif/blob/master/README.md)。