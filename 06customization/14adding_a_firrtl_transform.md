# Adding a Firrtl Transform

与 LLVM IR 通道在软件上执行转换和优化的方式类似，FIRRTL 转换可以修改 Chisel 精心设计的 RTL。如 [FIRRTL](https://chipyard.readthedocs.io/en/stable/Tools/FIRRTL.html#firrtl) 部分所述，变换是发生在 FIRRTL IR 上的修改，可以修改电路。换是一种强大的工具，可以接收 Chisel 发出的 FIRRTL IR 并运行分析或将电路转换为新形式。

## The Scala FIRRTL Compiler and the MLIR FIRRTL Compiler

在 Chipyard 中，两个 FIRRTL 编译器协同工作，将 Chisel 编译为 Verilog。 Scala FIRRTL 编译器 (SFC) 和 MLIR FIRRTL 编译器 (MFC)。它们基本上做同样的事情，除了 MFC 是用 C++ 编写的，这使得编译速度更快（生成的 Verilog 会有所不同）。在默认设置下，SFC 会将 Chisel 编译成 CHIRRTL，MFC 会将 CHIRRTL 编译成 Verilog（到目前为止，我们使用 SFC 作为 MFC 不起作用的情况的备份，例如，当设计使用固定类型时）。通过将 `ENABLE_CUSTOM_FIRRTL_PASS` 环境变量设置为非零值，我们可以使 SFC 将 Chisel 编译为 LowFIRRTL，以便应用我们的自定义 FIRRTL 传递。

有关 MLIR FIRRTL 编译器的更多信息，请访问 https://mlir.llvm.org/ 和 https://circt.llvm.org/。

## Where to add transforms

在 Chipyard 中，多次调用 FIRRTL 编译器来创建包含 DUT 的“Top”文件和包含测试工具的“模型”文件，该测试工具实例化 DUT。“模型”文件不包含 DUT 的模块定义或其任何子模块。这是通过调用 `GenerateModelStageMain`（包装多个FIRRTL编译器调用和额外转换的函数）的 `tapeout` SBT 项目（位于 `tools/barstools/tapeout` 中）完成的。

```Makefile
# There are two possible cases for this step. In the first case, SFC
# compiles Chisel to CHIRRTL, and MFC compiles CHIRRTL to Verilog. Otherwise,
# when custom FIRRTL transforms are included or if a Fixed type is used within
# the dut, SFC compiles Chisel to LowFIRRTL and MFC compiles it to Verilog.
# Users can indicate to the Makefile of custom FIRRTL transforms by setting the
# "ENABLE_CUSTOM_FIRRTL_PASS" variable.
#
# hack: lower to low firrtl if Fixed types are found
# hack: when using dontTouch, io.cpu annotations are not removed by SFC,
# hence we remove them manually by using jq before passing them to firtool

$(SFC_LEVEL) $(EXTRA_FIRRTL_OPTIONS) &: $(FIRRTL_FILE)
ifeq (,$(ENABLE_CUSTOM_FIRRTL_PASS))
	echo $(if $(shell grep "Fixed<" $(FIRRTL_FILE)), low, none) > $(SFC_LEVEL)
	echo "$(EXTRA_BASE_FIRRTL_OPTIONS)" $(if $(shell grep "Fixed<" $(FIRRTL_FILE)), "$(SFC_REPL_SEQ_MEM)",) > $(EXTRA_FIRRTL_OPTIONS)
else
	echo low > $(SFC_LEVEL)
	echo "$(EXTRA_BASE_FIRRTL_OPTIONS)" "$(SFC_REPL_SEQ_MEM)" > $(EXTRA_FIRRTL_OPTIONS)
endif

$(MFC_LOWERING_OPTIONS):
	mkdir -p $(dir $@)
ifeq (,$(ENABLE_YOSYS_FLOW))
	echo "$(MFC_BASE_LOWERING_OPTIONS)" > $@
else
	echo "$(MFC_BASE_LOWERING_OPTIONS),disallowPackedArrays" > $@
endif

$(FINAL_ANNO_FILE): $(EXTRA_ANNO_FILE) $(SFC_EXTRA_ANNO_FILE) $(SFC_LEVEL)
	if [ $(shell cat $(SFC_LEVEL)) = low ]; then jq -s '[.[][]]' $(EXTRA_ANNO_FILE) $(SFC_EXTRA_ANNO_FILE) > $@; fi
	if [ $(shell cat $(SFC_LEVEL)) = none ]; then cat $(EXTRA_ANNO_FILE) > $@; fi
	touch $@

$(SFC_MFC_TARGETS) &: private TMP_DIR := $(shell mktemp -d -t cy-XXXXXXXX)
$(SFC_MFC_TARGETS) &: $(TAPEOUT_CLASSPATH_TARGETS) $(FIRRTL_FILE) $(FINAL_ANNO_FILE) $(SFC_LEVEL) $(EXTRA_FIRRTL_OPTIONS) $(MFC_LOWERING_OPTIONS)
	rm -rf $(GEN_COLLATERAL_DIR)
	$(call run_jar_scala_main,$(TAPEOUT_CLASSPATH),barstools.tapeout.transforms.GenerateModelStageMain,\
		--no-dedup \
		--output-file $(SFC_FIRRTL_BASENAME) \
		--output-annotation-file $(SFC_ANNO_FILE) \
		--target-dir $(GEN_COLLATERAL_DIR) \
		--input-file $(FIRRTL_FILE) \
		--annotation-file $(FINAL_ANNO_FILE) \
		--log-level $(FIRRTL_LOGLEVEL) \
		--allow-unrecognized-annotations \
		-X $(shell cat $(SFC_LEVEL)) \
		$(shell cat $(EXTRA_FIRRTL_OPTIONS)))
	-mv $(SFC_FIRRTL_BASENAME).lo.fir $(SFC_FIRRTL_FILE) 2> /dev/null # Optionally change file type when SFC generates LowFIRRTL
	@if [ $(shell cat $(SFC_LEVEL)) = low ]; then cat $(SFC_ANNO_FILE) | jq 'del(.[] | select(.target | test("io.cpu"))?)' > $(TMP_DIR)/unnec-anno-deleted.sfc.anno.json; fi
	@if [ $(shell cat $(SFC_LEVEL)) = low ]; then cat $(TMP_DIR)/unnec-anno-deleted.sfc.anno.json | jq 'del(.[] | select(.class | test("SRAMAnnotation"))?)' > $(TMP_DIR)/unnec-anno-deleted2.sfc.anno.json; fi
	@if [ $(shell cat $(SFC_LEVEL)) = low ]; then cat $(TMP_DIR)/unnec-anno-deleted2.sfc.anno.json > $(SFC_ANNO_FILE) && rm $(TMP_DIR)/unnec-anno-deleted.sfc.anno.json && rm $(TMP_DIR)/unnec-anno-deleted2.sfc.anno.json; fi
	firtool \
		--format=fir \
		--export-module-hierarchy \
		--verify-each=true \
		--warn-on-unprocessed-annotations \
		--disable-annotation-classless \
		--disable-annotation-unknown \
		--mlir-timing \
		--lowering-options=$(shell cat $(MFC_LOWERING_OPTIONS)) \
		--repl-seq-mem \
		--repl-seq-mem-file=$(MFC_SMEMS_CONF) \
		--annotation-file=$(SFC_ANNO_FILE) \
		--split-verilog \
		-o $(GEN_COLLATERAL_DIR) \
		$(SFC_FIRRTL_FILE)
	-mv $(SFC_SMEMS_CONF) $(MFC_SMEMS_CONF) 2> /dev/null
	$(SED) -i 's/.*/& /' $(MFC_SMEMS_CONF) # need trailing space for SFC macrocompiler touch $(MFC_BB_MODS_FILELIST) # if there are no BB's then the file might not be generated, instead always generate it
```

如果您查看 [tools/barstools/tapeout/src/main/scala/transforms/GenerateModelStageMain.scala](https://github.com/ucb-bar/barstools/blob/master/tapeout/src/main/scala/transforms/GenerateModelStageMain.scala) 文件内部，您可以看到为“Model”调用了 FIRRTL。目前，FIRRTL 编译器对 `TOP` 和 `MODEL` 差异不可知，用户负责提供注释，通知编译器在何处（`TOP` 与 `MODEL`）执行自定义 FIRRTL 转换。

有关 Barstools 的更多信息，请访问 [Barstools](https://chipyard.readthedocs.io/en/stable/Tools/Barstools.html#barstools) 部分。

## Examples of transforms

您可以应用多个转换示例，这些示例遍布 FIRRTL 生态系统。在 FIRRTL 中，有一组默认支持的转换，位于 https://github.com/freechipsproject/firrtl/tree/master/src/main/scala/firrtl/transforms。这包括可以展平模块(`Flatten`)、将模块分组在一起 (`GroupAndDedup`) 等的转换。

转换可以是独立的，也可以将注释作为输入。注释用于在 FIRRTL 变换之间传递信息。这包括有关要展平、分组哪些模块等的信息。可以通过将注释添加到 Chisel 源代码中或通过创建序列化注释 `json` 文件并将其添加到 FIRRTL 编译器来将注释添加到代码中（注意：注释 Chisel 源代码会自动将注释序列化为 `json` 片段到构建系统中）。**推荐的注释方法是在 Chisel 源代码中进行注释，但并非所有注释类型都有 Chisel API。**

下面的示例显示了使用 `DontTouchAnnotation` 注释信号的两种方法（确保 FIRRTL 中的“死代码消除”过程不会删除特定信号）：

1. 使用名为 `dontTouch` 的 Chisel API/包装函数，它会自动为您执行此操作（更多 [dontTouch](https://www.chisel-lang.org/api/SNAPSHOT/chisel3/dontTouch$.html) 信息）。
2. 如果没有 Chisel API，则直接使用 `annotate` 函数和 `DontTouchAnnotation` 类来注释信号（注意：大多数 FIRRTL 注释都有 Chisel API）。

```Scala
class TopModule extends Module {
    ...
    val submod = Module(new Submodule)
    ...
}

class Submodule extends Module {
    ...
    val some_signal := ...

    // MAIN WAY TO USE `dontTouch`
    // how to annotate if there is a Chisel API/wrapper
    chisel3.dontTouch(some_signal)

    // how to annotate WITHOUT a Chisel API/wrapper
    annotate(new ChiselAnnotation {
        def toFirrtl = DontTouchAnnotation(some_signal.toNamed)
    })

    ...
}
```

以下是序列化 DontTouchAnnotation 的示例：

```Json
[
    {
        "class": "firrtl.transforms.DontTouchAnnotation",
        "target": "~TopModule|Submodule>some_signal"
    }
]
```

在这种情况下，具体语法取决于注释的类型及其字段。弄清楚序列化语法的更简单方法之一是首先尝试找到要添加到代码中的 Chisel 注释。然后，您可以查看从构建系统生成的抵押品，找到 `*.anno.json`，并找到注释的正确语法。

创建 `yourAnnoFile.json` 后，您可以将 `-faf yourAnnoFile.json` 添加到 `common.mk` 中的 FIRRTL 编译器调用中。

```Makefile
# There are two possible cases for this step. In the first case, SFC
# compiles Chisel to CHIRRTL, and MFC compiles CHIRRTL to Verilog. Otherwise,
# when custom FIRRTL transforms are included or if a Fixed type is used within
# the dut, SFC compiles Chisel to LowFIRRTL and MFC compiles it to Verilog.
# Users can indicate to the Makefile of custom FIRRTL transforms by setting the
# "ENABLE_CUSTOM_FIRRTL_PASS" variable.
#
# hack: lower to low firrtl if Fixed types are found
# hack: when using dontTouch, io.cpu annotations are not removed by SFC,
# hence we remove them manually by using jq before passing them to firtool

$(SFC_LEVEL) $(EXTRA_FIRRTL_OPTIONS) &: $(FIRRTL_FILE)
ifeq (,$(ENABLE_CUSTOM_FIRRTL_PASS))
	echo $(if $(shell grep "Fixed<" $(FIRRTL_FILE)), low, none) > $(SFC_LEVEL)
	echo "$(EXTRA_BASE_FIRRTL_OPTIONS)" $(if $(shell grep "Fixed<" $(FIRRTL_FILE)), "$(SFC_REPL_SEQ_MEM)",) > $(EXTRA_FIRRTL_OPTIONS)
else
	echo low > $(SFC_LEVEL)
	echo "$(EXTRA_BASE_FIRRTL_OPTIONS)" "$(SFC_REPL_SEQ_MEM)" > $(EXTRA_FIRRTL_OPTIONS)
endif

$(MFC_LOWERING_OPTIONS):
	mkdir -p $(dir $@)
ifeq (,$(ENABLE_YOSYS_FLOW))
	echo "$(MFC_BASE_LOWERING_OPTIONS)" > $@
else
	echo "$(MFC_BASE_LOWERING_OPTIONS),disallowPackedArrays" > $@
endif

$(FINAL_ANNO_FILE): $(EXTRA_ANNO_FILE) $(SFC_EXTRA_ANNO_FILE) $(SFC_LEVEL)
	if [ $(shell cat $(SFC_LEVEL)) = low ]; then jq -s '[.[][]]' $(EXTRA_ANNO_FILE) $(SFC_EXTRA_ANNO_FILE) > $@; fi
	if [ $(shell cat $(SFC_LEVEL)) = none ]; then cat $(EXTRA_ANNO_FILE) > $@; fi
	touch $@

$(SFC_MFC_TARGETS) &: private TMP_DIR := $(shell mktemp -d -t cy-XXXXXXXX)
$(SFC_MFC_TARGETS) &: $(TAPEOUT_CLASSPATH_TARGETS) $(FIRRTL_FILE) $(FINAL_ANNO_FILE) $(SFC_LEVEL) $(EXTRA_FIRRTL_OPTIONS) $(MFC_LOWERING_OPTIONS)
	rm -rf $(GEN_COLLATERAL_DIR)
	$(call run_jar_scala_main,$(TAPEOUT_CLASSPATH),barstools.tapeout.transforms.GenerateModelStageMain,\
		--no-dedup \
		--output-file $(SFC_FIRRTL_BASENAME) \
		--output-annotation-file $(SFC_ANNO_FILE) \
		--target-dir $(GEN_COLLATERAL_DIR) \
		--input-file $(FIRRTL_FILE) \
		--annotation-file $(FINAL_ANNO_FILE) \
		--log-level $(FIRRTL_LOGLEVEL) \
		--allow-unrecognized-annotations \
		-X $(shell cat $(SFC_LEVEL)) \
		$(shell cat $(EXTRA_FIRRTL_OPTIONS)))
	-mv $(SFC_FIRRTL_BASENAME).lo.fir $(SFC_FIRRTL_FILE) 2> /dev/null # Optionally change file type when SFC generates LowFIRRTL
	@if [ $(shell cat $(SFC_LEVEL)) = low ]; then cat $(SFC_ANNO_FILE) | jq 'del(.[] | select(.target | test("io.cpu"))?)' > $(TMP_DIR)/unnec-anno-deleted.sfc.anno.json; fi
	@if [ $(shell cat $(SFC_LEVEL)) = low ]; then cat $(TMP_DIR)/unnec-anno-deleted.sfc.anno.json | jq 'del(.[] | select(.class | test("SRAMAnnotation"))?)' > $(TMP_DIR)/unnec-anno-deleted2.sfc.anno.json; fi
	@if [ $(shell cat $(SFC_LEVEL)) = low ]; then cat $(TMP_DIR)/unnec-anno-deleted2.sfc.anno.json > $(SFC_ANNO_FILE) && rm $(TMP_DIR)/unnec-anno-deleted.sfc.anno.json && rm $(TMP_DIR)/unnec-anno-deleted2.sfc.anno.json; fi
	firtool \
		--format=fir \
		--export-module-hierarchy \
		--verify-each=true \
		--warn-on-unprocessed-annotations \
		--disable-annotation-classless \
		--disable-annotation-unknown \
		--mlir-timing \
		--lowering-options=$(shell cat $(MFC_LOWERING_OPTIONS)) \
		--repl-seq-mem \
		--repl-seq-mem-file=$(MFC_SMEMS_CONF) \
		--annotation-file=$(SFC_ANNO_FILE) \
		--split-verilog \
		-o $(GEN_COLLATERAL_DIR) \
		$(SFC_FIRRTL_FILE)
	-mv $(SFC_SMEMS_CONF) $(MFC_SMEMS_CONF) 2> /dev/null
	$(SED) -i 's/.*/& /' $(MFC_SMEMS_CONF) # need trailing space for SFC macrocompiler
	touch $(MFC_BB_MODS_FILELIST) # if there are no BB's then the file might not be generated, instead always generate it
```

如果您有兴趣编写 FIRRTL 转换，请参阅此处的 FIRRTL 文档：https://github.com/freechipsproject/firrtl/wiki。