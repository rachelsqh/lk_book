linux内核构建系统
----------------
我们以内核代码的Makefile文件来理解整个构建系统。

概述
^^^^
Makefile组成：

.. code-block:: c
   :caption: make 调用自定义函数
   :emphasize-lines: 4,5
   :linenos:
   
   Makefile                    the top Makefile.
   .config                     the kernel configuration file.
   arch/$(SRCARCH)/Makefile    the arch Makefile.
   scripts/Makefile.*          common rules etc. for all kbuild Makefiles.
   kbuild Makefiles            exist in every subdirectory
   
   
顶层 Makefile 读取来自内核配置过程的 .config 文件。






顶层 Makefile 负责构建两个主要产品：vmlinux（驻留内核映像）和模块（任何模块文件）。它通过递归下降到内核源代码树的子目录来构建这些目标。访问的子目录列表取决于内核配置。最上面的 Makefile 包含一个名为 arch/$(SRCARCH)/Makefile 的 arch Makefile。拱 Makefile 向顶层 Makefile 提供特定于体系结构的信息。

每个子目录都有一个 kbuild Makefile，它执行从上面传递下来的命令。kbuild Makefile 使用 .config 文件中的信息来构建 kbuild 用来构建任何内置或模块化目标的各种文件列表。

scripts/Makefile.* 包含用于基于 kbuild makefile 构建内核的所有定义/规则等。


Kbuild文件
^^^^^^^^^^
内核中的大多数 Makefile 都是使用 kbuild 基础设施的 kbuild Makefile。本章介绍 kbuild makefile 中使用的语法。kbuild 文件的首选名称是“Makefile”，但可以使用“Kbuild”，如果“Makefile”和“Kbuild”文件都存在，则将使用“Kbuild”文件。

目标定义
""""""""
目标定义是 kbuild Makefile 的主要部分（核心）。这些行定义了要构建的文件、任何特殊的编译选项以及要递归输入的任何子目录。

最简单的 kbuild makefile 包含一行：


.. code-block:: c
   :caption: make 调用自定义函数
   :emphasize-lines: 4,5
   :linenos:
   
   obj-y += foo.o


这告诉 kbuild 该目录中有一个名为 foo.o 的对象。foo.o 将从 foo.c 或 foo.S 构建。

如果将 foo.o 构建为模块，则使用变量 obj-m。因此，经常使用以下模式：

.. code-block:: c
   :caption: make 调用自定义函数
   :emphasize-lines: 4,5
   :linenos:
   
   obj-$(CONFIG_FOO) += foo.o

$(CONFIG_FOO) 计算结果为 y（对于内置）或 m（对于模块）。如果 CONFIG_FOO 既不是 y 也不是 m，那么文件将不会被编译或链接。   
   
内置对象：obj-y
"""""""""""""
kbuild Makefile 在 $(obj-y) 列表中为 vmlinux 指定目标文件。这些列表取决于内核配置。

Kbuild 编译所有的 $(obj-y) 文件。然后它调用“$(AR) rcSTP”将这些文件合并到一个内置的.a 文件中。这是一个没有符号表的精简存档。稍后会通过 scripts/link-vmlinux.sh 链接到 vmlinux

$(obj-y) 中的文件顺序很重要。允许列表中的重复：第一个实例将链接到 built-in.a 中，后续实例将被忽略。

链接顺序很重要，因为某些函数（module_init()/__initcall）将在引导期间按照它们出现的顺序被调用。因此请记住，更改链接顺序可能会更改检测到 SCSI 控制器的顺序，因此您的磁盘会重新编号。

例子：

.. code-block:: c
   :caption: make 调用自定义函数
   :emphasize-lines: 4,5
   :linenos:
   
   #drivers/isdn/i4l/Makefile
   # Makefile for the kernel ISDN subsystem and device drivers.
   # Each configuration option enables a list of files.
   obj-$(CONFIG_ISDN_I4L)         += isdn.o
   obj-$(CONFIG_ISDN_PPP_BSDCOMP) += isdn_bsdcomp.o

可见在模块目标：obj-m
"""""""""""""""""""
$(obj-m) 指定构建为可加载内核模块的目标文件。

一个模块可以由一个源文件或多个源文件构建。对于一个源文件，kbuild makefile 只是简单地将文件添加到 $(obj-m)。

.. code-block:: c
   :caption: make 调用自定义函数
   :emphasize-lines: 4,5
   :linenos:
   
   #drivers/isdn/i4l/Makefile
   obj-$(CONFIG_ISDN_PPP_BSDCOMP) += isdn_bsdcomp.o
   
注意：在此示例中 $(CONFIG_ISDN_PPP_BSDCOMP) 计算结果为 'm'

如果一个内核模块是从多个源文件构建的，您指定要以与上述相同的方式构建一个模块；然而，kbuild 需要知道你想从哪个目标文件构建你的模块，所以你必须通过设置一个 $(<module_name>-y) 变量来告诉它。

.. code-block:: c
   :caption: make 调用自定义函数
   :emphasize-lines: 4,5
   :linenos:
   
   #drivers/isdn/i4l/Makefile
   obj-$(CONFIG_ISDN_I4L) += isdn.o
   isdn-y := isdn_net_lib.o isdn_v110.o isdn_common.o

在本例中，模块名称为 isdn.o。Kbuild 将编译 $(isdn-y) 中列出的对象，然后在这些文件列表上运行“$(LD) -r”以生成 isdn.o。

由于 kbuild 识别复合对象的 $(<module_name>-y)，您可以使用CONFIG_符号的值来选择将目标文件包含为复合对象的一部分。

.. code-block:: c
   :caption: make 调用自定义函数
   :emphasize-lines: 4,5
   :linenos:
   
   #fs/ext2/Makefile
   obj-$(CONFIG_EXT2_FS) += ext2.o
   ext2-y := balloc.o dir.o file.o ialloc.o inode.o ioctl.o \
          namei.o super.o symlink.o
   ext2-$(CONFIG_EXT2_FS_XATTR) += xattr.o xattr_user.o \
                                xattr_trusted.o
   
 
在此示例中，如果 $(CONFIG_EXT2_FS_XATTR) 的计算结果为“y”，则 xattr.o、xattr_user.o 和 xattr_trusted.o 只是复合对象 ext2.o 的一部分。

注意：当然，当您将对象构建到内核中时，上面的语法也可以使用。因此，如果您有 CONFIG_EXT2_FS=y，kbuild 将按照您的预期从各个部分为您构建一个 ext2.o 文件，然后将其链接到 built-in.a。

库文件目标:lib-y
""""""""""""""
用 obj-* 列出的对象用于模块，或组合在该特定目录的 built-in.a 中。还可以列出将包含在库 lib.a 中的对象。使用 lib-y 列出的所有对象都组合在该目录的单个库中。在 obj-y 中列出并在 lib-y 中另外列出的对象将不会包含在库中，因为它们无论如何都可以访问。为了保持一致，lib-m 中列出的对象将包含在 lib.a 中。

请注意，相同的 kbuild makefile 可能会列出要内置并成为库一部分的文件。因此，同一目录可能同时包含 built-in.a 和 lib.a 文件。

.. code-block:: c
   :caption: make 调用自定义函数
   :emphasize-lines: 4,5
   :linenos:
   
   #arch/x86/lib/Makefile
   lib-y    := delay.o
   
这将创建一个基于 delay.o 的库 lib.a。为了让 kbuild 真正识别出正在构建的 lib.a，该目录应在 libs-y 中列出。

另见“7.4 列出下降时要访问的目录”。

lib-y 的使用通常仅限于lib/和arch/*/lib。


目录递归
"""""""
Makefile 只负责在自己的目录中构建对象。子目录中的文件应由这些子目录中的 Makefiles 处理。构建系统将自动在子目录中递归调用 make，前提是您让它知道它们。

为此，使用了 obj-y 和 obj-m。ext2 位于一个单独的目录中，并且 fs/ 中的 Makefile 告诉 kbuild 使用以下分配下降。

.. code-block:: c
   :caption: make 调用自定义函数
   :emphasize-lines: 4,5
   :linenos:
   
   #fs/Makefile
   obj-$(CONFIG_EXT2_FS) += ext2/
   
如果 CONFIG_EXT2_FS 设置为“y”（内置）或“m”（模块化），则将设置相应的 obj- 变量，并且 kbuild 将下降到 ext2 目录中。

Kbuild 使用这些信息不仅决定它需要访问该目录，还决定是否将目录中的对象链接到 vmlinux。

当 Kbuild 进入带有 'y' 的目录时，该目录中的所有内置对象都会合并到 built-in.a 中，最终将链接到 vmlinux。

相反，当 Kbuild 进入带有“m”的目录时，该目录中的任何内容都不会链接到 vmlinux。如果该目录中的 Makefile 指定 obj-y，则这些对象将成为孤立对象。这很可能是 Makefile 或 Kconfig 中的依赖项的错误。

Kbuild 还支持专用语法 subdir-y 和 subdir-m，用于下降到子目录。当您知道它们根本不包含内核空间对象时，这是一个很好的选择。一个典型的用法是让 Kbuild 进入子目录来构建工具。

.. code-block:: c
   :caption: make 调用自定义函数
   :emphasize-lines: 4,5
   :linenos:
   
   # scripts/Makefile
   subdir-$(CONFIG_GCC_PLUGINS) += gcc-plugins
   subdir-$(CONFIG_MODVERSIONS) += genksyms
   subdir-$(CONFIG_SECURITY_SELINUX) += selinux

与 obj-y/m 不同，subdir-y/m 不需要尾部斜杠，因为此语法始终用于目录。

分配目录名称时最好使用CONFIG_变量。如果相应的CONFIG_选项既不是“y”也不是“m”，这允许 kbuild 完全跳过目录。

非内置vmlinux目标：extra-y:理解起来有些模糊呢
"""""""""""""""""""""""""""""""""""""""
extra-y 指定构建 vmlinux 所需的目标，但未合并到 built-in.a 中。也就是说不通过built-in.a直接链接到vmlinux中。
例如：
	1. 头对象：有些对象必须放在 vmlinux 的头部。它们直接链接到 vmlinux 而无需通过buildin.a 一个典型的用例是一个包含入口点的对象。
	   arch/$(SRCARCH)/Makefile should specify such objects as head-y.
	   讨论：
	      鉴于我们可以控制链接描述文件中的节顺序，为什么我们需要 head-y？
	   答案：
	2. vmlinux 链接器脚本
		.. code-block:: c
   			:caption: make 调用自定义函数
   			:emphasize-lines: 4,5
   			:linenos:
   			
   			# arch/x86/kernel/Makefile
   			extra-y := head_$(BITS).o
   			extra-y += head$(BITS).o
   			extra-y += ebda.o
   			extra-y += platform-quirks.o
   			extra-y += vmlinux.lds

$(extra-y) 应该只包含 vmlinux 所需的目标。也就是说其中指定的目标直接面对vmlinux,没有其他中间目标。

当 vmlinux 显然不是最终目标时，Kbuild 会跳过 extra-y。（例如“制作模块”，或构建外部模块）

如果您打算无条件地构建目标，always-y（在下一节中解释）是正确使用的语法。

总是构建的目标：always-y
"""""""""""""""""""""""
always-y 指定在 Kbuild 访问 Makefile 时总是构建的目标。
例如：
.. code-block:: c
	:caption: make 调用自定义函数
	:emphasize-lines: 4,5
	:linenos:
	
	# ./Kbuild offsets-file := include/generated/asm-offsets.h always-y += $(offsets-file)

编译标志
"""""""
ccflags-y、asflags-y 和 ldflags-y
**********************************
这三个标志仅适用于分配它们的 kbuild makefile。它们用于递归构建期间发生的所有正常 cc、as 和 ld 调用。注意：具有相同行为的标志以前被命名为：EXTRA_CFLAGS、EXTRA_AFLAGS 和 EXTRA_LDFLAGS。它们仍受支持，但已弃用。

	1. ccflags-y 指定使用 $(CC) 编译的选项。
		
		.. code-block:: c
			:caption: make 调用自定义函数
			:emphasize-lines: 4,5
			:linenos:
			
			# drivers/acpi/acpica/Makefile
			ccflags-y                       := -Os -D_LINUX -DBUILDING_ACPICA
			ccflags-$(CONFIG_ACPI_DEBUG)    += -DACPI_DEBUG_OUTPUT
		
		这个变量是必要的，因为顶层 Makefile 拥有变量 $(KBUILD_CFLAGS) 并将它用于整个树的编译标志。
	
	2. asflags-y 指定汇编程序选项。
		
		.. code-block:: c
			:caption: make 调用自定义函数
			:emphasize-lines: 4,5
			:linenos:
			
			#arch/sparc/kernel/Makefile
			asflags-y := -ansi
			
	3. ldflags-y 指定与 $(LD) 链接的选项。
		
		.. code-block:: c
			:caption: make 调用自定义函数
			:emphasize-lines: 4,5
			:linenos:
			
			#arch/cris/boot/compressed/Makefile
			ldflags-y += -T $(srctree)/$(src)/decompress_$(arch-y).lds

subdir-ccflags-y, subdir-asflags-y
**********************************

上面列出的两个标志类似于 ccflags-y 和 asflags-y。不同之处在于子目录变体对它们所在的 kbuild 文件和所有子目录都有影响。使用 subdir-* 指定的选项被添加到命令行中，在使用 non-subdir 变体指定的选项之前。

.. code-block:: c
	:caption: make 调用自定义函数
	:emphasize-lines: 4,5
	:linenos:
	
	subdir-ccflags-y := -Werror

ccflags-remove-y, asflags-remove-y
**********************************
这些标志用于删除编译器、汇编器调用的特定标志。

.. code-block:: c
	:caption: make 调用自定义函数
	:emphasize-lines: 4,5
	:linenos:
	
	ccflags-remove-$(CONFIG_MCOUNT) += -pg


CFLAGS_$@, AFLAGS_$@
*********************

CFLAGS_$@ 和 AFLAGS_$@ 仅适用于当前 kbuild makefile 中的命令。

$(CFLAGS_$@) 为 $(CC) 指定每个文件的选项。$@ 部分有一个文字值，它指定它所针对的文件。

CFLAGS_$@ 的优先级高于 ccflags-remove-y；CFLAGS_$@ 可以重新添加被 ccflags-remove-y 删除的编译器标志。

.. code-block:: c
	:caption: make 调用自定义函数
	:emphasize-lines: 4,5
	:linenos:
	
	# drivers/scsi/Makefile
	CFLAGS_aha152x.o =   -DAHA152X_STAT -DAUTOCONF

此行指定 aha152x.o 的编译标志。

$(AFLAGS_$@) 是汇编语言源文件的类似功能。

AFLAGS_$@ 的优先级高于 asflags-remove-y；AFLAGS_$@ 可以重新添加被 asflags-remove-y 删除的汇编器标志。

.. code-block:: c
	:caption: make 调用自定义函数
	:emphasize-lines: 4,5
	:linenos:
	
	# arch/arm/kernel/Makefile
	AFLAGS_head.o        := -DTEXT_OFFSET=$(TEXT_OFFSET)
	AFLAGS_crunch-bits.o := -Wa,-mcpu=ep9312
	AFLAGS_iwmmxt.o      := -Wa,-mcpu=iwmmxt

依赖跟踪
"""""""
Kbuild跟踪对以下内容的依赖关系

	1. 所有必备文件（*.c和*.h）
	2. 所有必备文件中使用的CONFIG_选项
	3. 用于编译目标的命令行

因此，如果您将选项更改为 $(CC) 所有受影响的文件将被重新编译。

自定义规则
""""""""
当 kbuild 基础架构不提供所需的支持时，将使用自定义规则。一个典型的例子是在构建过程中生成的头文件。另一个例子是特定于架构的 Makefile，它需要自定义规则来准备启动映像等。自定义规则与普通 Make 规则一样编写。Kbuild 不在 Makefile 所在的目录中执行，因此所有自定义规则都应使用必备文件和目标文件的相对路径。

定义自定义规则时使用两个变量：
	1. $(src):指向 Makefile 所在目录的相对路径。引用位于 src 树中的文件时，始终使用 $(src)。
	2. $(obj):是一个相对路径，指向保存目标的目录。引用生成的文件时始终使用 $(obj)。
	

.. code-block:: c
	:caption: make 调用自定义函数
	:emphasize-lines: 4,5
	:linenos:
	
	#drivers/scsi/Makefile
	$(obj)/53c8xx_d.h: $(src)/53c7,8xx.scr $(src)/script_asm.pl
        $(CPP) -DCHIP=810 - < $< | ... $(src)/script_asm.pl
        
这是一个自定义规则，遵循 make 所需的正常语法。

目标文件取决于两个必备文件。对目标文件的引用以 $(obj) 为前缀，对先决条件的引用以 $(src) 引用（因为它们不是生成的文件）。

	3. $(kecho)在规则中向用户回显信息通常是一种好习惯，但是当执行“make -s”时，除了警告/错误之外，不会看到任何输出。为了支持这个 kbuild 定义了 $(kecho) ，它将把 $(kecho) 后面的文本回显到标准输出，除非使用了“make -s”。

.. code-block:: c
	:caption: make 调用自定义函数
	:emphasize-lines: 4,5
	:linenos:
	
	# arch/arm/Makefile
	$(BOOT_TARGETS): vmlinux
        $(Q)$(MAKE) $(build)=$(boot) MACHINE=$(MACHINE) $(boot)/$@
        @$(kecho) '  Kernel: $(boot)/$@ is ready'


当 kbuild 以 KBUILD_VERBOSE=0 执行时，通常只显示命令的简写。要为自定义命令启用此行为，kbuild 需要设置两个变量：

.. code-block:: c
	:caption: make 调用自定义函数
	:emphasize-lines: 4,5
	:linenos:
	
	quiet_cmd_<command>     - what shall be echoed
	cmd_<command>     - the command to execute
如

.. code-block:: c
	:caption: make 调用自定义函数
	:emphasize-lines: 4,5
	:linenos:
	
	# lib/Makefile
	quiet_cmd_crc32 = GEN     $@
	cmd_crc32 = $< > $@
	$(obj)/crc32table.h: $(obj)/gen_crc32table
        $(call cmd,crc32)


更新 $(obj)/crc32table.h 目标时，该行：

GEN lib/crc32table.h

将显示“make KBUILD_VERBOSE=0”。


命令变化检测
""""""""""
评估规则时，会比较目标文件与其先决条件文件之间的时间戳。当任何先决条件更新时，GNU Make 会更新目标。

自上次调用以来命令行已更改时，也应重建目标。Make 本身不支持这一点，因此 Kbuild 通过一种元编程来实现这一点。

if_changed 是用于此目的的宏，格式如下：

.. code-block:: c
	:caption: make 调用自定义函数
	:emphasize-lines: 4,5
	:linenos:
	
	quiet_cmd_<command> = ...
	cmd_<command> = ...
	<target>: <source(s)> FORCE
        	$(call if_changed,<command>)
        	

任何使用 if_changed 的​​目标都必须列在 $(targets) 中，否则命令行检查将失败，并且将始终构建目标。

如果目标已在可识别的语法中列出，例如 obj-y/m、lib-y/m、extra-y/m、always-y/m、hostprogs、userprogs，Kbuild 会自动将其添加到 $(targets)。否则，必须将目标显式添加到 $(targets)。

对 $(targets) 的分配没有 $(obj)/ 前缀。if_changed 可以与“3.11 自定义规则”中定义的自定义规则结合使用。

注意：忘记 FORCE 先决条件是一个典型的错误。另一个常见的陷阱是空白有时很重要。例如，以下将失败（注意逗号后的额外空格）：

.. code-block:: c
	:caption: make 调用自定义函数
	:emphasize-lines: 4,5
	:linenos:
	
	target: source(s) FORCE
	
	
错误的！ $(调用 if_changed, objcopy)

笔记：
if_changed 每个目标不应多次使用。它将执行的命令存储在相应的 .cmd 文件中，当目标是最新的并且只有对更改命令的测试触发命令执行时，多次调用将导致覆盖和不需要的结果。



$(CC) 支持函数
"""""""""""""
内核可以使用几个不同版本的 $(CC) 构建，每个版本都支持一组独特的功能和选项。kbuild 提供基本支持来检查 $(CC) 的有效选项。$(CC) 通常是 gcc 编译器，但也可以使用其他替代方法。

as-option 
**********


用于检查 $(CC) - 当用于编译汇编程序 ( *.S ) 文件时 - 是否支持给定选项。如果不支持第一个选项，则可以指定可选的第二个选项。

例子：

.. code-block:: c
	:caption: make 调用自定义函数
	:emphasize-lines: 4,5
	:linenos:
	
	#arch/sh/Makefile
	cflags-y += $(call as-option,-Wa$(comma)-isa=$(isa-y),)
	
在上面的例子中，如果 $(CC) 支持 cflags-y，它将被分配选项 -Wa$(comma)-isa=$(isa-y)。第二个参数是可选的，如果不支持第一个参数，将使用如果提供。

as-instr 
*********

检查汇编器是否报告特定指令，然后输出 option1 或 option2 在测试指令中支持 C 转义 注意：as-instr-option 使用 KBUILD_AFLAGS 表示汇编器选项

cc-option 
***********

用于检查 $(CC) 是否支持给定选项，如果不支持则使用可选的第二个选项。

例子：

.. code-block:: c
	:caption: make 调用自定义函数
	:emphasize-lines: 4,5
	:linenos:
	
	#arch/x86/Makefile
	cflags-y += $(call cc-option,-march=pentium-mmx,-march=i586)
	
在上面的例子中，如果 $(CC) 支持，cflags-y 将被分配选项 -march=pentium-mmx，否则 -march=i586。cc-option 的第二个参数是可选的，如果省略，如果不支持第一个选项，则 cflags-y 将不被赋值。注意：cc-option 将 KBUILD_CFLAGS 用于 $(CC) 选项

cc-option-yn
************

cc-option-yn 用于检查 gcc 是否支持给定的选项，如果支持则返回 'y'，否则返回 'n'。

.. code-block:: c
	:caption: make 调用自定义函数
	:emphasize-lines: 4,5
	:linenos:
	
	#arch/ppc/Makefile
	biarch := $(call cc-option-yn, -m32)
	aflags-$(biarch) += -a32
	cflags-$(biarch) += -m32

在上面的示例中，如果 $(CC) 支持 -m32 选项，则 $(biarch) 设置为 y。当 $(biarch) 等于 'y' 时，扩展变量 $(aflags-y) 和 $(cflags-y) 将分别分配值 -a32 和 -m32。注意： cc-option-yn 将 KBUILD_CFLAGS 用于 $(CC) 选项



cc-disable-warning
******************
cc-disable-warning 检查 gcc 是否支持给定的警告并返回命令行开关以禁用它。需要这个特殊功能，因为 gcc 4.4 及更高版本接受任何未知的 -Wno-* 选项，并且仅在源文件中有另一个警告时才发出警告。

.. code-block:: c
	:caption: make 调用自定义函数
	:emphasize-lines: 4,5
	:linenos:
	
	KBUILD_CFLAGS += $(call cc-disable-warning, unused-but-set-variable)
	
在上面的例子中，-Wno-unused-but-set-variable 只有在 gcc 真正接受的情况下才会被添加到 KBUILD_CFLAGS 中。





cc-ifversion
************

cc-ifversion 测试 $(CC) 的版本，如果版本表达式为真，则等于第四个参数，如果版本表达式为假，则等于第五个参数（如果给定）。


.. code-block:: c
	:caption: make 调用自定义函数
	:emphasize-lines: 4,5
	:linenos:
	
	#fs/reiserfs/Makefile
	ccflags-y := $(call cc-ifversion, -lt, 0402, -O1)
	
在此示例中，如果 $(CC) 版本小于 4.2，ccflags-y 将被分配值 -O1。cc-ifversion 接受所有的 shell 运算符：-eq、-ne、-lt、-le、-gt 和 -ge 第三个参数可以是本例中的文本，但也可以是扩展变量或宏.






cc-cross-prefix
****************

cc-cross-prefix 用于检查路径中是否存在具有所列前缀之一的 $(CC)。返回 PATH 中存在前缀 $(CC) 的第一个前缀 - 如果没有找到前缀 $(CC)，则不返回任何内容。在 cc-cross-prefix 的调用中，附加前缀由单个空格分隔。此功能对于尝试将 CROSS_COMPILE 设置为众所周知的值但可能有多个值可供选择的体系结构 Makefile 很有用。如果是交叉构建（主机架构与目标架构不同），建议仅尝试设置 CROSS_COMPILE。如果 CROSS_COMPILE 已设置，则将其保留为旧值。

.. code-block:: c
	:caption: make 调用自定义函数
	:emphasize-lines: 4,5
	:linenos:
	
	#arch/m68k/Makefile
	ifneq ($(SUBARCH),$(ARCH))
        	ifeq ($(CROSS_COMPILE),)
               		CROSS_COMPILE := $(call cc-cross-prefix, m68k-linux-gnu-)
        	endif
	endif



$(LD) 支持函数
"""""""""""""
ld-option
*********

.. code-block:: c
	:caption: make 调用自定义函数
	:emphasize-lines: 4,5
	:linenos:
	
	#Makefile
	LDFLAGS_vmlinux += $(call ld-option, -X)




脚本调用
"""""""
Make 规则可以调用脚本来构建内核。规则应始终提供适当的解释器来执行脚本。它们不应依赖于设置的执行位，也不应直接调用脚本。为了方便手动调用脚本，例如调用./scripts/checkpatch.pl，建议还是在脚本上设置执行位。

Kbuild 提供变量 $(CONFIG_SHELL)、$(AWK)、$(PERL) 和 $(PYTHON3) 来引用相应脚本的解释器。
.. code-block:: c
	:caption: make 调用自定义函数
	:emphasize-lines: 4,5
	:linenos:
	
	#Makefile
	cmd_depmod = $(CONFIG_SHELL) $(srctree)/scripts/depmod.sh $(DEPMOD) \
             $(KERNELRELEASE)


Host软件支持
^^^^^^^^^^^

Kbuild 支持在主机上构建可执行文件以供在编译阶段使用。为了使用主机可执行文件，需要两个步骤。

第一步是告诉 kbuild 存在一个宿主程序。这是使用变量“hostprogs”完成的。

第二步是向可执行文件添加显式依赖项。这可以通过两种方式完成。在规则中添加依赖项，或使用变量“always-y”。下面描述了这两种可能性。

简单host程序
"""""""""""

在某些情况下，需要在运行构建的计算机上编译和运行程序。以下行告诉 kbuild 程序 bin2hex 应在构建主机上构建。

.. code-block:: c
	:caption: make 调用自定义函数
	:emphasize-lines: 4,5
	:linenos:
	
	hostprogs := bin2hex
	
Kbuild 在上面的例子中假设 bin2hex 是由一个名为 bin2hex.c 的 c 源文件生成的，该文件与 Makefile 位于同一目录中。





复合host程序
""""""""""

宿主程序可以基于复合对象组成。用于为主机程序定义复合对象的语法类似于用于内核对象的语法。$(<executable>-objs) 列出用于链接最终可执行文件的所有对象。

.. code-block:: c
	:caption: make 调用自定义函数
	:emphasize-lines: 4,5
	:linenos:
	
	#scripts/lxdialog/Makefile
	hostprogs     := lxdialog
	lxdialog-objs := checklist.o lxdialog.o
	
扩展名为 .o 的对象是从相应的 .c 文件编译而来的。在上面的例子中，checklist.c 被编译为 checklist.o，lxdialog.c 被编译为 lxdialog.o。

最后，这两个 .o 文件链接到可执行文件 lxdialog。注意：主机程序不允许使用语法 <executable>-y。




host程序中使用C++
"""""""""""""""

kbuild 支持用 C++ 编写的主机程序。这只是为了支持 kconfig 而引入的，不推荐用于一般用途。

.. code-block:: c
	:caption: make 调用自定义函数
	:emphasize-lines: 4,5
	:linenos:
	
	#scripts/kconfig/Makefile
	hostprogs     := qconf
	qconf-cxxobjs := qconf.o
	
	
在上面的示例中，可执行文件由 C++ 文件 qconf.cc 组成 - 由 $(qconf-cxxobjs) 标识。

如果 qconf 由 .c 和 .cc 文件混合组成，则可以使用附加行来识别它。

.. code-block:: c
	:caption: make 调用自定义函数
	:emphasize-lines: 4,5
	:linenos:
	
	#scripts/kconfig/Makefile
	hostprogs     := qconf
	qconf-cxxobjs := qconf.o
	qconf-objs    := check.o
	
host程序的编译器选项
"""""""""""""""""
编译主机程序时，可以设置特定的标志。程序将始终使用 $(HOSTCC) 通过 $(KBUILD_HOSTCFLAGS) 中指定的选项进行编译。要设置对该 Makefile 中创建的所有主机程序生效的标志，请使用变量 HOST_EXTRACFLAGS。

.. code-block:: c
	:caption: make 调用自定义函数
	:emphasize-lines: 4,5
	:linenos:
	
	#scripts/lxdialog/Makefile
	HOST_EXTRACFLAGS += -I/usr/include/ncurses

要为单个文件设置特定标志，请使用以下构造：

.. code-block:: c
	:caption: make 调用自定义函数
	:emphasize-lines: 4,5
	:linenos:
	
	#arch/ppc64/boot/Makefile
	HOSTCFLAGS_piggyback.o := -DKERNELBASE=$(KERNELBASE)
	
也可以为链接器指定附加选项。

.. code-block:: c
	:caption: make 调用自定义函数
	:emphasize-lines: 4,5
	:linenos:
	
	#scripts/kconfig/Makefile
	HOSTLDLIBS_qconf := -L$(QTDIR)/lib

链接 qconf 时，将传递额外的选项“-L$(QTDIR)/lib”。


hostprogs 构建示例
"""""""""""""""""
Kbuild 仅在将它们作为先决条件引用时才会构建主机程序。这可以通过两种方式实现：


	1. 在自定义规则中明确列出先决条件。
	   
	   .. code-block:: c
	   	:caption: make 调用自定义函数
	   	:emphasize-lines: 4,5
	   	:linenos:
	   	
	   	#drivers/pci/Makefile
	   	hostprogs := gen-devlist
	   	$(obj)/devlist.h: $(src)/pci.ids $(obj)/gen-devlist
	   	( cd $(obj); ./gen-devlist ) < $<
	
	目标 $(obj)/devlist.h 在 $(obj)/gen-devlist 更新之前不会被构建。请注意，自定义规则中对主机程序的引用必须以 $(obj) 为前缀。
	   	
	2. 总是使用-y
	   如果没有合适的自定义规则，并且需要在输入 makefile 时构建主机程序，则应使用 always-y 变量。
	   
	   .. code-block:: c
	   	:caption: make 调用自定义函数
	   	:emphasize-lines: 4,5
	   	:linenos:
	   	
	   	#scripts/lxdialog/Makefile
	   	hostprogs     := lxdialog
	   	always-y      := $(hostprogs)
	   	
	   Kbuild 为此提供了以下简写：
	   
	   .. code-block:: c
	   	:caption: make 调用自定义函数
	   	:emphasize-lines: 4,5
	   	:linenos:
	   	
	   	hostprogs-always-y := lxdialog
	   	
	   即使没有在任何规则中引用，这也会告诉 kbuild 构建 lxdialog。

用户空间程序支持
^^^^^^^^^^^^^
就像宿主程序一样，Kbuild 也支持为目标体系结构（即与您构建内核的体系结构相同）构建用户空间可执行文件。语法非常相似。不同之处在于使用“userprogs”而不是“hostprogs”。

简单的用户空间程序
***************
以下行告诉 kbuild 程序 bpf-direct 应为目标架构构建。

.. code-block:: c
	:caption: make 调用自定义函数
	:emphasize-lines: 4,5
	:linenos:
	
	userprogs := bpf-direct
	
在上面的示例中，Kbuild 假设 bpf-direct 是由一个名为 bpf-direct.c 的 C 源文件创建的，该文件与 Makefile 位于同一目录中。


复合用户空间程序
*************

用户空间程序可以基于复合对象组成。用于为用户空间程序定义复合对象的语法类似于用于内核对象的语法。$(<executable>-objs) 列出用于链接最终可执行文件的所有对象。

.. code-block:: c
	:caption: make 调用自定义函数
	:emphasize-lines: 4,5
	:linenos:
	
	#samples/seccomp/Makefile
	userprogs      := bpf-fancy
	bpf-fancy-objs := bpf-fancy.o bpf-helper.o
	
扩展名为 .o 的对象是从相应的 .c 文件编译而来的。在上面的示例中，bpf-fancy.c 被编译为 bpf-fancy.o，而 bpf-helper.c 被编译为 bpf-helper.o。最后，这两个 .o 文件链接到可执行文件 bpf-fancy。注意：用户空间程序不允许使用语法 <executable>-y。

控制用户空间程序的编译器选项
************************

在编译用户空间程序时，可以设置特定的标志。程序将始终使用 $(CC) 通过 $(KBUILD_USERCFLAGS) 中指定的选项进行编译。要设置对该 Makefile 中创建的所有用户空间程序生效的标志，请使用变量 userccflags。

.. code-block:: c
	:caption: make 调用自定义函数
	:emphasize-lines: 4,5
	:linenos:
	
	# samples/seccomp/Makefile
	userccflags += -I usr/include
	
要为单个文件设置特定标志，请使用以下构造：

.. code-block:: c
	:caption: make 调用自定义函数
	:emphasize-lines: 4,5
	:linenos:
	
	bpf-helper-userccflags += -I user/include

也可以为链接器指定附加选项。

.. code-block:: c
	:caption: make 调用自定义函数
	:emphasize-lines: 4,5
	:linenos:
	
	# net/bpfilter/Makefile
	bpfilter_umh-userldflags += -static
	
链接 bpfilter_umh 时，将传递额外的选项 -static。

用户空间程序实际构建过程
********************

Kbuild 只有在被告知这样做时才会构建用户空间程序。有两种方法可以做到这一点。

	1. 将其添加为另一个文件的先决条件

	   .. code-block:: c
	   	:caption: make 调用自定义函数
	   	:emphasize-lines: 4,5
	   	:linenos:
	   	
	   	#net/bpfilter/Makefile
	   	userprogs := bpfilter_umh
	   	$(obj)/bpfilter_umh_blob.o: $(obj)/bpfilter_umh
	   $(obj)/bpfilter_umh 在 $(obj)/bpfilter_umh_blob.o 之前构建
	   
	2. 总是使用-y
	
	   .. code-block:: c
	   	:caption: make 调用自定义函数
	   	:emphasize-lines: 4,5
	   	:linenos:
	   	
	   	userprogs := binderfs_example
	   	always-y := $(userprogs) 
	   	
	   Kbuild 为此提供了以下简写：
	   
	   .. code-block:: c
	   	:caption: make 调用自定义函数
	   	:emphasize-lines: 4,5
	   	:linenos:
	   	userprogs-always-y := binderfs_example
	   	
	   这将告诉 Kbuild 在访问此 Makefile 时构建 binderfs_example。
	   
	    
	


Kbuild clean 框架
^^^^^^^^^^^^^^^^^
“make clean”删除编译内核的 obj 树中大多数生成的文件。这包括生成的文件，例如主机程序。Kbuild 知道 $(hostprogs)、$(always-y)、$(always-m)、$(always-)、$(extra-y)、$(extra-) 和 $(targets) 中列出的目标。它们都在“make clean”期间被删除。当执行“make clean”时，匹配模式“ .[oas]”、“.ko”的文件以及由 kbuild 生成的一些附加文件会在整个内核源代码树中被删除。

可以使用 $(clean-files) 在 kbuild makefile 中指定其他文件或目录。

.. code-block:: c
	:caption: make 调用自定义函数
	:emphasize-lines: 4,5
	:linenos:
	
	#lib/Makefile
	clean-files := crc32table.h

执行“make clean”时，文件“crc32table.h”将被删除。Kbuild 将假定文件与 Makefile 位于相同的相对目录中，除非以 $(objtree) 为前缀。

要从 make clean 中排除某些文件或目录，请使用 $(no-clean-files) 变量。

由于“obj-* := dir/”，kbuild 通常会在子目录中下降，但在 kbuild 基础设施不足的架构 makefile 中，有时需要明确说明。

.. code-block:: c
	:caption: make 调用自定义函数
	:emphasize-lines: 4,5
	:linenos:
	
	#arch/x86/boot/Makefile
	subdir- := compressed
	
上面的分配指示 kbuild 在执行“make clean”时下降到目录compressed/。

注意 1：arch/$(SRCARCH)/Makefile 不能使用“subdir-”，因为该文件包含在顶级 makefile 中。相反，arch/$(SRCARCH)/Kbuild 可以使用“subdir-”。

注意 2：在“make clean”期间将访问 core-y、libs-y、drivers-y 和 net-y 中列出的所有目录。




架构 Makefile
^^^^^^^^^^^^^

顶层 Makefile 设置环境并进行准备，然后开始下降到各个目录中。顶层 makefile 包含通用部分，而 arch/$(SRCARCH)/Makefile 包含为所述架构设置 kbuild 所需的内容。为此，arch/$(SRCARCH)/Makefile 设置了许多变量并定义了一些目标。

当 kbuild 执行时，（大致）遵循以下步骤：

	1. 内核配置 => 生成 .config
	2. 将内核版本存储在 include/linux/version.h 中
	3. 将所有其他先决条件更新到目标准备： - 其他先决条件在 arch/$(SRCARCH)/Makefile 中指定
	4. 在 init-* core* drivers-* net-* libs-* 中列出的所有目录中递归下降并构建所有目标。- 以上变量的值在arch/$(SRCARCH)/Makefile 中展开。
	5. 然后链接所有对象文件，生成的文件 vmlinux 位于 obj 树的根目录。第一个链接的对象列在 head-y 中，由 arch/$(SRCARCH)/Makefile 分配。
	6. 最后，特定于体系结构的部分执行任何所需的后处理并构建最终的引导映像。- 这包括构建引导记录 - 准备 initrd 映像等

设置变量以调整构建到架构
********************

	1. KBUILD_LDFLAGS: 通用 $(LD) 选项,用于链接器的所有调用的标志。通常指定仿真就足够了。
	   
	   .. code-block:: c
	   	:caption: make 调用自定义函数
	   	:emphasize-lines: 4,5
	   	:linenos:
	   	
	   	#arch/s390/Makefile
	   	KBUILD_LDFLAGS         := -m elf_s390
	   	
	   注意：ldflags-y 可用于进一步自定义使用的标志。
	  
	2. LDFLAGS_vmlinux:链接 vmlinux 时的 $(LD) 选项,LDFLAGS_vmlinux 用于在链接最终的 vmlinux 映像时指定要传递给链接器的附加标志。LDFLAGS_vmlinux 使用 LDFLAGS_$@ 支持。
	
	   .. code-block:: c
	   	:caption: make 调用自定义函数
	   	:emphasize-lines: 4,5
	   	:linenos:
	   	
	3. OBJCOPYFLAGS: 对象复制标志,当 $(call if_changed,objcopy) 用于翻译 .o 文件时，将使用 OBJCOPYFLAGS 中指定的标志。$(call if_changed,objcopy) 通常用于在 vmlinux 上生成原始二进制文件。

	   .. code-block:: c
	   	:caption: make 调用自定义函数
	   	:emphasize-lines: 4,5
	   	:linenos:
	   	#arch/s390/Makefile
	   	OBJCOPYFLAGS := -O binary
	   	#arch/s390/boot/Makefile
	   	$(obj)/image: vmlinux FORCE
	   		$(call if_changed,objcopy)
	   		
	   在此示例中，二进制 $(obj)/image 是 vmlinux 的二进制版本。$(call if_changed,xxx) 的用法将在后面介绍。
	4. KBUILD_AFLAGS: 汇编器标志,默认值 - 请参阅顶层 Makefile 根据架构的需要附加或修改。
	5. KBUILD_CFLAGS: $(CC) 编译器标志,默认值 - 请参阅顶层 Makefile 根据架构的需要附加或修改。通常，KBUILD_CFLAGS 变量取决于配置。
	6. KBUILD_AFLAGS_KERNEL: 特定于内置的汇编器选项,$(KBUILD_AFLAGS_KERNEL) 包含用于编译驻留内核代码的额外 C 编译器标志。
	7. KBUILD_AFLAGS_MODULE: 特定于模块的汇编器选项,$(KBUILD_AFLAGS_MODULE) 用于添加用于汇编程序的特定于架构的选项。从命令行 AFLAGS_MODULE 应使用（见Kbuild）。
	8. KBUILD_CFLAGS_KERNEL: $(CC) 特定于内置的选项,$(KBUILD_CFLAGS_KERNEL) 包含用于编译驻留内核代码的额外 C 编译器标志。
	9. KBUILD_CFLAGS_MODULE: 构建模块时的 $(CC) 选项,$(KBUILD_CFLAGS_MODULE) 用于添加用于 $(CC) 的特定于架构的选项。从命令行 CFLAGS_MODULE 应使用（见Kbuild）。
	11. KBUILD_LDFLAGS_MODULE: 链接模块时的 $(LD) 选项,$(KBUILD_LDFLAGS_MODULE) 用于添加链接模块时使用的特定于架构的选项。这通常是链接描述文件。从命令行 LDFLAGS_MODULE 应使用（见Kbuild）。
	12. KBUILD_LDS:具有完整路径的链接描述文件。由顶级 Makefile 分配。
	13. KBUILD_LDS_MODULE: 具有完整路径的模块链接描述文件。由顶级 Makefile 分配，另外由 arch Makefile 分配。
	14. KBUILD_VMLINUX_OBJS: vmlinux 的所有目标文件。它们以与 KBUILD_VMLINUX_OBJS 中列出的相同顺序链接到 vmlinux。
	15. KBUILD_VMLINUX_LIBS: vmlinux 的所有 .a “lib”文件。KBUILD_VMLINUX_OBJS 和 KBUILD_VMLINUX_LIBS 一起指定用于链接 vmlinux 的所有目标文件。

向archheaders 添加先决条件
************************

archheaders: 规则用于生成可以通过“make header_install”安装到用户空间的头文件。在架构本身上运行时，它会在“make archprepare”之前运行。

为 archprepare 添加先决条件
*************************

archprepare: 规则用于列出在开始下降到子目录之前需要构建的先决条件。这通常用于包含汇编程序常量的头文件。

.. code-block:: c
	:caption: make 调用自定义函数
	:emphasize-lines: 4,5
	:linenos:
	
	#arch/arm/Makefile
	archprepare: maketools
	
在此示例中，文件目标 maketools 将在下降到子目录之前进行处理。另见第 XXX-TODO 章，它描述了 kbuild 如何支持生成偏移头文件。


列出降序访问的目录
***************

Arch Makefile 与顶层 Makefile 合作定义变量，这些变量指定如何构建 vmlinux 文件。请注意，模块没有相应的特定于架构的部分；模块构建机制完全独立于架构。

	1. head-y、core-y、libs-y、drivers-y:
		a. $(head-y) 列出要在 vmlinux 中首先链接的对象。
		b. $(libs-y) 列出了 lib.a 存档所在的目录。
	   其余列出了可以找到 built-in.a 目标文件的目录。然后其余的按照这个顺序：$(core-y), $(libs-y), $(drivers-y)
	   顶层 Makefile 定义了所有通用目录的值，而 arch/$(SRCARCH)/Makefile 仅添加特定于体系结构的目录。
	   .. code-block:: c
	   	:caption: make 调用自定义函数
	   	:emphasize-lines: 4,5
	   	:linenos:
	   	
	   	# arch/sparc/Makefile
	   	core-y                 += arch/sparc/
	   	
	   	libs-y                 += arch/sparc/prom/
	   	libs-y                 += arch/sparc/lib/
	   	
	   	drivers-$(CONFIG_PM) += arch/sparc/power/	   
	   	
	   	
特定于架构的启动映像
*****************
Arch Makefile 指定了获取 vmlinux 文件的目标，将其压缩，将其包装在引导代码中，并将生成的文件复制到某处。这包括各种安装命令。实际目标没有跨架构标准化。通常在 arch/$(SRCARCH)/ 下的 boot/ 目录中定位任何附加处理。Kbuild 没有提供任何智能方式来支持构建 boot/.xml 中指定的目标。因此，arch/$(SRCARCH)/Makefile 应该手动调用 make 在 boot/ 中构建目标。推荐的方法是在 arch/$(SRCARCH)/Makefile 中包含快捷方式，并在调用 arch/$(SRCARCH)/boot/Makefile 时使用完整路径。

.. code-block:: c
	:caption: make 调用自定义函数
	:emphasize-lines: 4,5
	:linenos:
	
	#arch/x86/Makefile
	boot := arch/x86/boot
	bzImage: vmlinux
        $(Q)$(MAKE) $(build)=$(boot) $(boot)/$@

“$(Q)$(MAKE) $(build)=<dir>” 是在子目录中调用 make 的推荐方式。命名特定于架构的目标没有规则，但执行“make help”将列出所有相关目标。为了支持这一点，必须定义 $(archhelp)。

.. code-block:: c
	:caption: make 调用自定义函数
	:emphasize-lines: 4,5
	:linenos:
	
	#arch/x86/Makefile
	define archhelp
	  echo  '* bzImage      - Compressed kernel image (arch/x86/boot/bzImage)'
	endif
	
当不带参数执行 make 时，将构建遇到的第一个目标。在顶层 Makefile 中，第一个目标是 all:。默认情况下，架构应始终构建可引导映像。在“make help”中，默认目标用“*”突出显示。为所有添加一个新的先决条件：选择一个不同于 vmlinux 的默认目标。

.. code-block:: c
	:caption: make 调用自定义函数
	:emphasize-lines: 4,5
	:linenos:
	
	#arch/x86/Makefile
	all: bzImage
	
当不带参数执行“make”时，将构建 bzImage。

用于构建引导映像的命令
******************

Kbuild 提供了一些在构建启动映像时很有用的宏。
	
	1. ld: 链接目标。通常，LDFLAGS_$@ 用于将特定选项设置为 ld。
	
	2. objcopy: 
	
	3. gzip:
	
	4. dtc:
	
	
预处理链接描述文件
***************

构建 vmlinux 映像时，使用链接描述文件 arch/$(SRCARCH)/kernel/vmlinux.lds。该脚本是位于同一目录中的文件 vmlinux.lds.S 的预处理变体。kbuild 知道 .lds 文件并包含一个规则*lds.S -> *lds。

.. code-block:: c
	:caption: make 调用自定义函数
	:emphasize-lines: 4,5
	:linenos:
	
	#arch/x86/kernel/Makefile
	extra-y := vmlinux.lds

对 extra-y 的赋值用于告诉 kbuild 构建目标 vmlinux.lds。$(CPPFLAGS_vmlinux.lds) 的赋值告诉 kbuild 在构建目标 vmlinux.lds 时使用指定的选项。在构建*.lds目标时，kbuild 使用以下变量：

.. code-block:: c
	:caption: make 调用自定义函数
	:emphasize-lines: 4,5
	:linenos:
	
	KBUILD_CPPFLAGS : Set in top-level Makefile
	cppflags-y      : May be set in the kbuild makefile
	CPPFLAGS_$(@F)  : Target-specific flags.
                Note that the full filename is used in this
                assignment.

*lds文件的 kbuild 基础结构用于多个特定于体系结构的文件中。


通用头文件
*********
目录 include/asm-generic 包含可以在各个体系结构之间共享的头文件。如何使用通用头文件的推荐方法是在 Kbuild 文件中列出该文件。有关语法等的更多信息，请参阅“8.2 generic-y”。

Post-link pass
**************
如果文件 arch/xxx/Makefile.postlink 存在，则该 makefile 将为 post-link 对象（vmlinux 和 modules.ko）调用，以便体系结构运行 post-link 传递。还必须处理干净的目标。

此通行证在 kallsyms 生成之后运行。如果体系结构需要修改符号位置，而不是操纵 kallsyms，为 .tmp_vmlinux 添加另一个 postlink 目标可能更容易？从 link-vmlinux.sh 调用的目标。

例如，powerpc 使用它来检查链接的 vmlinux 文件的重定位健全性。


导出头文件的 Kbuild 语法
^^^^^^^^^^^^^^^^^^^^^^^
内核包含一组导出到用户空间的头文件。许多标头可以按原样导出，但其他标头在准备好用于用户空间之前需要进行最少的预处理。预处理会：

	1. 删除内核特定的注释
	2. 删除 compiler.h 的包含
	3. 删除内核内部的所有部分（由ifdef __KERNEL__ 保护）
	
include/uapi/、include/generated/uapi/、arch/<arch>/include/uapi/ 和 arch/<arch>/include/generated/uapi/ 下的所有头文件都被导出。可以在 arch/<arch>/include/uapi/asm/ 和 arch/<arch>/include/asm/ 下定义 Kbuild 文件以列出来自 asm-generic 的 asm 文件。有关 Kbuild 文件的语法，请参阅后续章节。

无出口标头
********

no-export-headers 本质上由 include/uapi/linux/Kbuild 使用，以避免在不支持它的架构上导出特定的头文件（例如 kvm.h）。应尽可能避免。

generic-y
*********

如果架构使用来自 include/asm-generic 的头文件的逐字副本，那么它会在文件 arch/$(SRCARCH)/include/asm/Kbuild 中列出，如下所示：

.. code-block:: c
	:caption: make 调用自定义函数
	:emphasize-lines: 4,5
	:linenos:
	
	#arch/x86/include/asm/Kbuild
	generic-y += termios.h
	generic-y += rtc.h
	
generated-y
************


mandatory-y
************


Kbuild 变量
^^^^^^^^^^
	1. VERSION, PATCHLEVEL, SUBLEVEL, EXTRAVERSION
	2. KERNELRELEASE
	3. ARCH
	4. SRCARCH
	5. INSTALL_PATH
	6. INSTALL_MOD_PATH, MODLIB
	7. INSTALL_MOD_STRIP




Makefile 语言
^^^^^^^^^^^^^
内核 Makefile 被设计为与 GNU Make 一起运行。Makefiles 仅使用 GNU Make 的文档化功能，但它们确实使用了许多 GNU 扩展。

GNU Make 支持基本的列表处理功能。内核 Makefile 使用了一种新颖的列表构建和操作方式，几乎没有“if”语句。

GNU Make 有两个赋值运算符，“:=”和“=”。“:=” 立即计算右侧并将实际字符串存储到左侧。“=”就像一个公式定义；它将右侧存储为未评估的形式，然后在每次使用左侧时评估此形式。

在某些情况下，“=”是合适的。但通常，“:=”是正确的选择。


其他
^^^^
描述 kbuild 如何使用 _shipped 支持交付的文件。
*****************************************
生成偏移头文件。
*************

更多变量
********















   
