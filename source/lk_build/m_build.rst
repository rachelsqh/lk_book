内核模块编译
-----------
本文档描述了如何构建树外内核模块。

简介
^^^^^
“kbuild”是 Linux 内核使用的构建系统。模块必须使用 kbuild 来与构建基础设施的变化保持兼容，并为“gcc”选择正确的标志。提供了在树内和树外构建模块的功能。构建两者的方法类似，所有模块最初都是在树外开发和构建的。

本文档中涵盖的信息面向对构建树外（或“外部”）模块感兴趣的开发人员。外部模块的作者应该提供一个隐藏大部分复杂性的 makefile，因此只需键入“make”即可构建模块。这很容易实现，第 3 节将介绍一个完整的示例。

如何构建外部模块
^^^^^^^^^^^^^
要构建外部模块，您必须有一个可用的预构建内核，其中包含构建中使用的配置和头文件。此外，内核必须是在启用模块的情况下构建的。如果您使用的是发行版内核，您的发行版将为您运行的内核提供一个包。

另一种方法是使用“make”目标“modules_prepare”。这将确保内核包含所需的信息。该目标仅作为一种简单的方法而存在，可以为构建外部模块准备内核源代码树。

注意：即使设置了 CONFIG_MODVERSIONS，“modules_prepare”也不会构建 Module.symvers；因此，需要执行完整的内核构建以使模块版本控制工作。

命令语法
""""""""
构建外部模块的命令是：

.. code-block:: c
	:caption: make 调用自定义函数
	:emphasize-lines: 4,5
	:linenos:
	
	$ make -C <path_to_kernel_src> M=$PWD

由于命令中给出的“M=<dir>”选项，kbuild 系统知道正在构建外部模块。

要针对正在运行的内核进行构建，请使用：

.. code-block:: c
	:caption: make 调用自定义函数
	:emphasize-lines: 4,5
	:linenos:

	$ make -C /lib/modules/`uname -r`/build M=$PWD


然后安装刚刚构建的模块，将目标“modules_install”添加到命令中：

.. code-block:: c
	:caption: make 调用自定义函数
	:emphasize-lines: 4,5
	:linenos:
	
	$ make -C /lib/modules/`uname -r`/build M=$PWD modules_install


选项说明
"""""""

（$KDIR 是指内核源目录的路径。）

使 -C $KDIR M=$PWD

- -C $KDIR:内核源代码所在的目录。“make”在执行时实际上会改变到指定的目录，完成后会变回来。
- M=$PWD:通知 kbuild 正在构建外部模块。赋予“M”的值是外部模块（kbuild 文件）所在目录的绝对路径。


目标
"""""""
在构建外部模块时，只有一部分“make”目标可用。

.. code-block:: c
	:caption: make 调用自定义函数
	:emphasize-lines: 4,5
	:linenos:
	
	make -C $KDIR M=$PWD [目标]
	
默认将构建位于当前目录中的模块，因此不需要指定目标。所有输出文件也将在此目录中生成。不尝试更新内核源代码，前提是内核已成功执行“make”。

- modules:外部模块的默认目标。它具有与未指定目标相同的功能。见上面的描述。
- modules_install: 安装外部模块。默认位置是 /lib/modules/<kernel_release>/extra/，但可以使用 INSTALL_MOD_PATH 添加前缀（在第 5 节中讨论）。
- clean: 仅删除模块目录中所有生成的文件。
- help: 列出外部模块的可用目标。

构建单独的文件
""""""""""""
可以构建作为模块一部分的单个文件。这同样适用于内核、模块，甚至外部模块。

示例（模块 foo.ko，由 bar.o 和 baz.o 组成）：


.. code-block:: c
	:caption: make 调用自定义函数
	:emphasize-lines: 4,5
	:linenos:
	
	make -C $KDIR M=$PWD bar.lst
	make -C $KDIR M=$PWD baz.o
	make -C $KDIR M=$PWD foo.ko
	make -C $KDIR M=$PWD ./

为外部模块创建 Kbuild 文件
^^^^^^^^^^^^^^^^^^^^^^^

在上一节中，我们看到了为正在运行的内核构建模块的命令。但是，该模块实际上并未构建，因为需要构建文件。此文件中包含正在构建的模块的名称，以及必需的源文件列表。该文件可能像一行一样简单：

.. code-block:: c
	:caption: make 调用自定义函数
	:emphasize-lines: 4,5
	:linenos:
	
	obj-m := <module_name>.o
	
kbuild 系统将从 <module_name>.c 构建 <module_name>.o，并在链接后生成内核模块 <module_name>.ko。上面的行可以放在“Kbuild”文件或“Makefile”中。当模块是从多个源构建的时，需要额外的一行来列出文件：

.. code-block:: c
	:caption: make 调用自定义函数
	:emphasize-lines: 4,5
	:linenos:
	
	<module_name>-y := <src1>.o <src2>.o ...
	
注意：描述 kbuild 使用的语法的更多文档位于Linux Kernel Makefiles中。

下面的示例演示如何为模块 8123.ko 创建构建文件，该构建文件由以下文件构建：

.. code-block:: c
	:caption: make 调用自定义函数
	:emphasize-lines: 4,5
	:linenos:
	
	8123_if.c
	8123_if.h
	8123_pci.c
	8123_bin.o_shipped      <= Binary blob
	
共享Makefile
""""""""""""

外部模块总是包含一个包装器生成文件，该文件支持使用不带参数的“make”构建模块。kbuild 不使用这个目标；这只是为了方便。可以包含其他功能，例如测试目标，但由于可能的名称冲突，应从 kbuild 中过滤掉。

示例 1：

.. code-block:: c
	:caption: make 调用自定义函数
	:emphasize-lines: 4,5
	:linenos:
	
	--> filename: Makefile
	ifneq ($(KERNELRELEASE),)
	# kbuild part of makefile
	obj-m  := 8123.o
	8123-y := 8123_if.o 8123_pci.o 8123_bin.o
	
	else
	# normal makefile
	KDIR ?= /lib/modules/`uname -r`/build
	
	default:
        	$(MAKE) -C $(KDIR) M=$$PWD
        	
        # Module specific targets
        genbin:
        	echo "X" > 8123_bin.o_shipped

	endif

检查 KERNELRELEASE 用于分隔生成文件的两个部分。在这个例子中，kbuild 只会看到两个赋值，而“make”会看到除了这两个赋值之外的所有内容。这是由于对文件进行了两次传递：第一次传递是由命令行上运行的“make”实例；第二遍由 kbuild 系统执行，由默认目标中的参数化“make”启动。

独立的Kbuild文件和Makefile
"""""""""""""""""""""""
在较新版本的内核中，kbuild 将首先查找名为“Kbuild”的文件，只有在未找到时，它才会查找 makefile。利用“Kbuild”文件，我们可以将示例 1 中的 makefile 拆分为两个文件：

示例 2：

.. code-block:: c
	:caption: make 调用自定义函数
	:emphasize-lines: 4,5
	:linenos:
	
	--> filename: Kbuild
	obj-m  := 8123.o
	8123-y := 8123_if.o 8123_pci.o 8123_bin.o

	--> filename: Makefile
	KDIR ?= /lib/modules/`uname -r`/build

	default:
       		$(MAKE) -C $(KDIR) M=$$PWD

	# Module specific targets
	genbin:
        	echo "X" > 8123_bin.o_shipped

由于每个文件的简单性，示例 2 中的拆分是有问题的；但是，一些外部模块使用由数百行组成的 makefile，在这里将 kbuild 部分与其他部分分开确实是值得的。

下一个示例显示了向后兼容的版本。

示例 3：


.. code-block:: c
	:caption: make 调用自定义函数
	:emphasize-lines: 4,5
	:linenos:
	
	--> filename: Kbuild
	obj-m  := 8123.o
	8123-y := 8123_if.o 8123_pci.o 8123_bin.o

	--> filename: Makefile
	ifneq ($(KERNELRELEASE),)
	# kbuild part of makefile
	include Kbuild

	else
	# normal makefile
	KDIR ?= /lib/modules/`uname -r`/build

	default:
 	       $(MAKE) -C $(KDIR) M=$$PWD

	# Module specific targets
	genbin:
	        echo "X" > 8123_bin.o_shipped

	endif

这里的“Kbuild”文件包含在 makefile 中。当“make”和 kbuild 部分被拆分为单独的文件时，这允许使用只知道 makefile 的旧版本的 kbuild。

二进制 Blob
"""""""""""

一些外部模块需要包含一个对象文件作为 blob。kbuild 对此提供支持，但需要将 blob 文件命名为 <filename>_shipped。当 kbuild 规则启动时，<filename>_shipped 的副本被创建，而 _shipped 被剥离，给我们 <filename>。这个缩短的文件名可以用于分配给模块。

在本节中，8123_bin.o_shipped 一直用于构建内核模块 8123.ko；它已作为 8123_bin.o 包含在内：

.. code-block:: c
	:caption: make 调用自定义函数
	:emphasize-lines: 4,5
	:linenos:
	
	8123-y := 8123_if.o 8123_pci.o 8123_bin.o
	
虽然普通源文件和二进制文件没有区别，但是在为模块创建目标文件时，kbuild 会选择不同的规则。

构建多个模块
""""""""""

kbuild 支持使用单个构建文件构建多个模块。例如，如果您想构建两个模块 foo.ko 和 bar.ko，则 kbuild 行将是：

.. code-block:: c
	:caption: make 调用自定义函数
	:emphasize-lines: 4,5
	:linenos:
	
	obj-m := foo.o bar.o
	foo-y := <foo_srcs>
	bar-y := <bar_srcs>

包含文件
^^^^^^^^^^^

在内核中，头文件根据以下规则保存在标准位置：

- 如果头文件只描述一个模块的内部接口，那么该文件与源文件放在同一目录下。
- 如果头文件描述了位于不同目录的内核其他部分使用的接口，则该文件位于 include/linux/ 中。

注意：此规则有两个值得注意的例外：较大的子系统在 include/ 下有自己的目录，例如 include/scsi；架构特定的头文件位于 arch/$(SRCARCH)/include/ 下。


内核包含
"""""""
要包含位于 include/linux/ 下的头文件，只需使用：

.. code-block:: c
	:caption: make 调用自定义函数
	:emphasize-lines: 4,5
	:linenos:
	
	#include <linux/module.h>
	
kbuild 将向“gcc”添加选项，以便搜索相关目录。

单个子目录
"""""""""
外部模块倾向于将头文件放在其源代码所在的单独的 include/ 目录中，尽管这不是通常的内核样式。要通知 kbuild 目录，请使用 ccflags-y 或 CFLAGS_<filename>.o。

使用第 3 节中的示例，如果我们将 8123_if.h 移动到名为 include 的子目录，生成的 kbuild 文件将如下所示：

.. code-block:: c
	:caption: make 调用自定义函数
	:emphasize-lines: 4,5
	:linenos:
	
	--> filename: Kbuild
	obj-m := 8123.o

	ccflags-y := -Iinclude
	8123-y := 8123_if.o 8123_pci.o 8123_bin.o
	
请注意，在分配中，-I 和路径之间没有空格。这是 kbuild 的一个限制：必须没有空间存在。

几个子目录
""""""""""
kbuild 可以处理分布在多个目录中的文件。考虑以下示例：


.. code-block:: c
	:caption: make 调用自定义函数
	:emphasize-lines: 4,5
	:linenos:
	
	.
	|__ src
	|   |__ complex_main.c
	|   |__ hal
	|       |__ hardwareif.c
	|       |__ include
	|           |__ hardwareif.h
	|__ include
	|__ complex.h
	
要构建模块 complex.ko，我们需要以下 kbuild 文件：


.. code-block:: c
	:caption: make 调用自定义函数
	:emphasize-lines: 4,5
	:linenos:
	
	--> filename: Kbuild
	obj-m := complex.o
	complex-y := src/complex_main.o
	complex-y += src/hal/hardwareif.o

	ccflags-y := -I$(src)/include
	ccflags-y += -I$(src)/src/hal/include
	

如您所见，kbuild 知道如何处理位于其他目录中的目标文件。诀窍是指定相对于 kbuild 文件位置的目录。话虽如此，这不是推荐的做法。

对于头文件，必须明确告知 kbuild 在哪里查找。当 kbuild 执行时，当前目录始终是内核树的根目录（“-C”的参数），因此需要一个绝对路径。$(src) 通过指向当前执行的 kbuild 文件所在的目录来提供绝对路径。

模块安装
^^^^^^^^
内核中包含的模块安装在目录中：


.. code-block:: c
	:caption: make 调用自定义函数
	:emphasize-lines: 4,5
	:linenos:
	
	/lib/modules/$(KERNELRELEASE)/内核/


外部模块安装在：


.. code-block:: c
	:caption: make 调用自定义函数
	:emphasize-lines: 4,5
	:linenos:
	
	/lib/modules/$(KERNELRELEASE)/extra/
	
INSTALL_MOD_PATH
"""""""""""""""""

以上是默认目录，但始终可以进行某种程度的自定义。可以使用变量 INSTALL_MOD_PATH 将前缀添加到安装路径：

.. code-block:: c
	:caption: make 调用自定义函数
	:emphasize-lines: 4,5
	:linenos:
	
	$ make INSTALL_MOD_PATH=/frodo modules_install
	=> Install dir: /frodo/lib/modules/$(KERNELRELEASE)/kernel/
	
INSTALL_MOD_PATH 可以设置为普通的 shell 变量，或者如上所示，可以在调用“make”时在命令行中指定。这在安装树内和树外模块时会起作用。


INSTALL_MOD_DIR
"""""""""""""""""

外部模块默认安装在 /lib/modules/$(KERNELRELEASE)/extra/ 下的目录中，但您可能希望在单独的目录中找到特定功能的模块。为此，请使用 INSTALL_MOD_DIR 指定“extra”的替代名称。


.. code-block:: c
	:caption: make 调用自定义函数
	:emphasize-lines: 4,5
	:linenos:
	
	$ make INSTALL_MOD_DIR=gandalf -C $KDIR \
       		M=$PWD modules_install
	=> Install dir: /lib/modules/$(KERNELRELEASE)/gandalf/
	
模块版本控制
^^^^^^^^^^^^^

模块版本控制由 CONFIG_MODVERSIONS 标签启用，并用作简单的 ABI 一致性检查。创建导出符号的完整原型的 CRC 值。当一个模块被加载/使用时，内核中包含的 CRC 值与模块中的相似值进行比较；如果它们不相等，内核将拒绝加载模块。

Module.symvers 包含从内核构建中导出的所有符号的列表。

来自内核的符号（vmlinux + 模块）
"""""""""""""""""""""""""""
在内核构建期间，将生成一个名为 Module.symvers 的文件。Module.symvers 包含从内核和编译模块导出的所有符号。对于每个符号，还存储相应的 CRC 值。

Module.symvers 文件的语法是：

.. code-block:: c
	:caption: make 调用自定义函数
	:emphasize-lines: 4,5
	:linenos:
	
	<CRC>       <Symbol>         <Module>                         <Export Type>     <Namespace>

	0xe1cc2a05  usb_stor_suspend drivers/usb/storage/usb-storage  EXPORT_SYMBOL_GPL USB_STORAGE
	
字段由制表符分隔，值可以为空（例如，如果没有为导出的符号定义命名空间）。

对于未启用 CONFIG_MODVERSIONS 的内核构建，CRC 将读取 0x00000000。

Module.symvers 有两个目的：
- 它列出了从 vmlinux 和所有模块导出的所有符号。
- 如果启用了 CONFIG_MODVERSIONS，它会列出 CRC。

 符号和外部模块
 """"""""""""""
 
 构建外部模块时，构建系统需要访问内核中的符号以检查是否定义了所有外部符号。这是在 MODPOST 步骤中完成的。modpost 通过从内核源代码树中读取 Module.symvers 来获取符号。在 MODPOST 步骤中，将写入一个新的 Module.symvers 文件，其中包含从该外部模块导出的所有符号。


来自另一个外部模块的符号
""""""""""""""""""""

有时，外部模块使用来自另一个外部模块的导出符号。Kbuild 需要完全了解所有符号，以避免发出有关未定义符号的警告。针对这种情况有两种解决方案。

注意：建议使用顶级 kbuild 文件的方法，但在某些情况下可能不切实际。

- 使用顶级 kbuild 文件
  如果你有两个模块，foo.ko 和 bar.ko，其中 foo.ko 需要来自 bar.ko 的符号，你可以使用一个通用的顶级 kbuild 文件，这样两个模块都在同一个构建中编译。考虑以下目录布局：
  
  .. code-block:: c
	:caption: make 调用自定义函数
	:emphasize-lines: 4,5
	:linenos:
	
	./foo/ <= contains foo.ko
	./bar/ <= contains bar.ko	
	
   顶级 kbuild 文件将如下所示：
   
   .. code-block:: c
	:caption: make 调用自定义函数
	:emphasize-lines: 4,5
	:linenos:
	
	#./Kbuild (or ./Makefile):
        obj-m := foo/ bar/
   并执行：
    .. code-block:: c
	:caption: make 调用自定义函数
	:emphasize-lines: 4,5
	:linenos: 
	
	$ make -C $KDIR M=$PWD
	
    然后将执行预期并编译两个模块，并充分了解任一模块的符号。
    
- 使用“make”变量 KBUILD_EXTRA_SYMBOLS:如果添加顶级 kbuild 文件不切实际，您可以在构建文件中将空格分隔的文件列表分配给 KBUILD_EXTRA_SYMBOLS。这些文件将在 modpost 的符号表初始化期间加载。

其他
^^^^^
测试 CONFIG_FOO_BAR
"""""""""""""""""""
模块通常需要检查某些CONFIG_选项以确定模块中是否包含特定功能。在 kbuild 中，这是通过直接引用CONFIG_变量来完成的：

.. code-block:: c
	:caption: make 调用自定义函数
	:emphasize-lines: 4,5
	:linenos:
	
	#fs/ext2/Makefile
	obj-$(CONFIG_EXT2_FS) += ext2.o

	ext2-y := balloc.o bitmap.o dir.o
	ext2-$(CONFIG_EXT2_FS_XATTR) += xattr.o
	
外部模块传统上使用“grep”直接在 .config中检查特定的CONFIG_设置。这种用法被打破了。如前所述，外部模块应该使用 kbuild 进行构建，因此在测试CONFIG_定义时可以使用与树内模块相同的方法。

内核模块原理分析
"""""""""""""
参考分析：https://











