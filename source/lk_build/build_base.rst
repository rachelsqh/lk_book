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
^^^^^^^^^^^^^^^^^^^^^^
always-y 指定在 Kbuild 访问 Makefile 时总是构建的目标。
例如：
.. code-block:: c
	:caption: make 调用自定义函数
	:emphasize-lines: 4,5
	:linenos:
	
	# ./Kbuild offsets-file := include/generated/asm-offsets.h always-y += $(offsets-file)

编译标志
"""""""


依赖跟踪
"""""""
Kbuild跟踪对以下内容的依赖关系

	1. 所有必备文件（*.c和*.h）
	2. 所有必备文件中使用的CONFIG_选项
	3. 用于编译目标的命令行

因此，如果您将选项更改为 $(CC) 所有受影响的文件将被重新编译。

自定义规则
""""""""


命令变化检测
""""""""""

$(CC) 支持函数
"""""""""""""


$(LD) 支持函数
"""""""""""""


脚本调用
"""""""


Host软件支持
^^^^^^^^^^^
hostprogs

简单host程序
"""""""""""


复合host程序
""""""""""

host程序中使用C++
"""""""""""""""
	
host程序的编译器选项
"""""""""""""""""



hostprogs 构建示例
"""""""""""""""""


用户空间空虚支持
^^^^^^^^^^^^^
userprogs



Kbuild clean 框架
^^^^^^^^^^^^^^^^^

架构 Makefile
^^^^^^^^^^^^^

导出头文件的 Kbuild 语法
^^^^^^^^^^^^^^^^^^^^^


Kbuild 变量
^^^^^^^^^^


Makefile 语言
^^^^^^^^^^^^^


构建流程总结：
^^^^^^^^^^















   
