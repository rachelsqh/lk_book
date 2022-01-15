其他
----

导出头文件供用户空间使用
^^^^^^^^^^^^^^^^^^^^^^

“make headers_install”命令以适合用户空间程序使用的形式导出内核的头文件。

linux 内核导出的头文件描述了试图使用内核服务的用户空间程序的 API。系统的 C 库（例如 glibc 或 uClibc）使用这些内核头文件来定义可用的系统调用，以及与这些系统调用一起使用的常量和结构。C 库的头文件包括来自“linux”子目录的内核头文件。系统的 libc 头文件通常安装在默认位置 /usr/include 中，而内核头文件则安装在该位置下的子目录中（最值得注意的是 /usr/include/linux 和 /usr/include/asm）。

内核标头向后兼容，但不向前兼容。这意味着使用旧内核头文件针对 C 库构建的程序应该在新内核上运行（尽管它可能无法访问新功能），但针对新内核头文件构建的程序可能无法在旧内核上运行。

“make headers_install”命令可以在内核源代码的顶级目录中运行（或使用标准的树外构建）。它需要两个可选参数：

.. code-block:: c
	:caption: make 调用自定义函数
	:emphasize-lines: 4,5
	:linenos:
	make headers_install ARCH=i386 INSTALL_HDR_PATH=/usr
	
ARCH 指示要为哪个体系结构生成标头，并且默认为当前体系结构。导出的内核头文件的 linux/asm 目录是特定于平台的，要查看支持的架构的完整列表，请使用以下命令：

.. code-block:: c
	:caption: make 调用自定义函数
	:emphasize-lines: 4,5
	:linenos:
	
	ls -d include/asm-* | sed 's/.*-//'
	
INSTALL_HDR_PATH 指示安装头文件的位置。它默认为“./usr”。

在 INSTALL_HDR_PATH 中会自动创建一个“include”目录，并且在“INSTALL_HDR_PATH/include”中安装头文件。

内核标头导出基础结构由 David Woodhouse< dwmw2 @ infradead 维护。组织>。









递归问题
^^^^^^^^^
https://www.kernel.org/doc/html/latest/kbuild/issues.html

可重现的构建
^^^^^^^^^^
通常希望使用相同的工具集构建相同的源代码是可重现的，即输出总是完全相同的。这使得验证二进制发行版或嵌入式系统的构建基础设施没有被破坏成为可能。这也可以更容易地验证源或工具更改不会对生成的二进制文件产生任何影响。

Reproducible Builds 项目有关于这个一般主题的更多信息。本文档涵盖了构建内核可能无法重现的各种原因，以及如何避免这些原因。


时间戳
""""""
内核在三个地方嵌入了时间戳：

- 公开uname()并包含在 其中的版本字符串/proc/version
- 嵌入式 initramfs 中的文件时间戳
- 如果启用 via CONFIG_IKHEADERS，则嵌入在内核或相应模块中的内核头文件的文件时间戳，通过/sys/kernel/kheaders.tar.xz,默认情况下，时间戳是当前时间，在 kheaders各种文件的修改时间的情况下。这必须使用KBUILD_BUILD_TIMESTAMP变量覆盖。如果您是从 git 提交构建的，则可以使用其提交日期。

内核并没有使用__DATE__和__TIME__宏，并启用警告如果使用它们。如果您合并使用这些的外部代码，则必须通过设置SOURCE_DATE_EPOCH环境变量来覆盖它们对应的时间戳。

User, host
""""""""""

内核将构建用户名和主机名嵌入到 /proc/version.这些必须使用KBUILD_BUILD_USER 和 KBUILD_BUILD_HOST变量覆盖 。如果你是从 git commit 构建的，你可以使用它的提交者地址。

绝对文件名
"""""""""
当内核在树外构建时，调试信息可能包括源文件的绝对文件名。这必须通过-fdebug-prefix-map在KCFLAGS变量中包含选项来覆盖。

根据所使用的编译器，__FILE__宏还可以在树外构建中扩展为绝对文件名。-fmacro-prefix-map如果支持，Kbuild 会自动使用该选项来防止这种情况发生。

Reproducible Builds 网站有更多关于这些 前缀映射选项的信息。

源包中生成的文件
"""""""""""""
子目录下某些程序的构建过程tools/ 不完全支持树外构建。这可能会导致稍后使用例如包含生成的文件的源包构建。您应该通过运行或在构建源代码包之前确保源代码树是原始的。make rpm-pkgmake mrpropergit clean -d -f -x

模块签名
"""""""

如果启用CONFIG_MODULE_SIG_ALL，则默认行为是为每个构建生成不同的临时密钥，从而导致模块无法重现。但是，在您的源代码中包含签名密钥可能会破坏签名模块的目的。

一种方法是划分构建过程，以便将不可重现的部分视为源：

1. 生成持久签名密钥。将密钥的证书添加到内核源。
#. 将CONFIG_SYSTEM_TRUSTED_KEYS符号设置为包含签名密钥的证书，设置CONFIG_MODULE_SIG_KEY为空字符串，然后禁用CONFIG_MODULE_SIG_ALL。构建内核和模块。
#. 为模块创建分离的签名，并将它们作为源发布。
#. 执行附加模块签名的第二次构建。它可以重建模块或使用步骤 2 的输出

结构随机化
"""""""""

如果启用CONFIG_GCC_PLUGIN_RANDSTRUCT，则需要预先生成随机种子， scripts/gcc-plugins/randomize_layout_seed.h以便在重建中使用相同的值。

调试信息冲突
"""""""""""

这不是不可重现的问题，而是生成的文件过于重现的问题。

一旦为可重现的构建设置了所有必要的变量，即使对于不同的内核版本，vDSO 的调试信息也可能相同。这可能导致不同内核版本的调试信息包之间的文件冲突。

为避免这种情况，您可以通过在其中包含任意字符串“salt”来使不同内核版本的 vDSO 不同。这是由 Kconfig 符号指定的CONFIG_BUILD_SALT。



GCC插件基础
^^^^^^^^^^

GCC 插件是为编译器提供额外功能的可加载模块1。它们对于运行时检测和静态分析很有用。我们可以在编译期间通过回调2、GIMPLE 3、IPA 4和 RTL pass 5分析、更改和添加更多代码。

内核的 GCC 插件基础结构支持构建树外模块、交叉编译和在单独的目录中构建。插件源文件必须可由 C++ 编译器编译。

目前 GCC 插件基础架构仅支持某些架构。Grep “select HAVE_GCC_PLUGINS” 找出哪些架构支持 GCC 插件。

该基础架构是从 grsecurity 6和 PaX 7移植而来的。

目的
""""""
GCC 插件旨在提供一个试验潜在编译器功能的地方，这些功能既不在 GCC 也不在 Clang 上游。一旦它们的实用性得到证明，目标就是将该特性上游化到 GCC（和 Clang）中，然后一旦该特性在所有受支持的 GCC 版本中可用，最终将它们从内核中删除。

具体来说，新插件应该只实现不支持上游编译器的功能（在 GCC 或 Clang 中）。

当 Clang 中存在某个功能但 GCC 不存在时，应努力将该功能带到上游 GCC（而不仅仅是作为特定于内核的 GCC 插件），以便整个生态系统都可以从中受益。

同样，即使 GCC 插件提供的功能在 Clang中不存在，但该功能被证明是有用的，也应该努力将功能上游到 GCC（和 Clang）。

在上游 GCC 中提供某个功能后，该插件将无法用于相应的 GCC 版本（及更高版本）。一旦所有内核支持的 GCC 版本都提供了该功能，该插件将从内核中删除。

文件
""""""
- $(src)/scripts/gcc-plugins:这是 GCC 插件的目录。
- $(src)/scripts/gcc-plugins/gcc-common.h:这是 GCC 插件的兼容性标头。它应该始终包含在其中，而不是单独的 gcc 标头。
- $(src)/scripts/gcc-plugins/gcc-generate-gimple-pass.h, $(src)/scripts/gcc-plugins/gcc-generate-ipa-pass.h, $(src)/scripts/gcc -plugins/gcc-generate-simple_ipa-pass.h, $(src)/scripts/gcc-plugins/gcc-generate-rtl-pass.h:这些标头自动生成 GIMPLE、SIMPLE_IPA、IPA 和 RTL 通道的注册结构。他们应该更喜欢手工创建结构。


用法
""""""
您必须为您的 gcc 版本安装 gcc 插件头文件，例如，在 Ubuntu 上安装 gcc-10：

.. code-block:: c
	:caption: make 调用自定义函数
	:emphasize-lines: 4,5
	:linenos:
	
	apt-get install gcc-10-plugin-dev

或者在 Fedora 上：


.. code-block:: c
	:caption: make 调用自定义函数
	:emphasize-lines: 4,5
	:linenos:	
	
	dnf install gcc-plugin-devel

启用 GCC 插件基础结构和您要在内核配置中使用的一些插件：

.. code-block:: c
	:caption: make 调用自定义函数
	:emphasize-lines: 4,5
	:linenos:
	
	CONFIG_GCC_PLUGINS=y
	CONFIG_GCC_PLUGIN_LATENT_ENTROPY=y
	...

要编译包含插件的最小工具集：

.. code-block:: c
	:caption: make 调用自定义函数
	:emphasize-lines: 4,5
	:linenos:
	
	make scripts
	
或者只是运行内核 make 并使用圈复杂度 GCC 插件编译整个内核。


如何添加新的GCC插件
"""""""""""""""""

GCC 插件位于 scripts/gcc-plugins/ 中。您需要将插件源文件放在 scripts/gcc-plugins/ 下。不支持创建子目录。它必须添加到 scripts/gcc-plugins/Makefile、scripts/Makefile.gcc-plugins 和相关的 Kconfig 文件中。

Clang/LLVM构建
^^^^^^^^^^^^^^

本文档介绍如何使用 Clang 和 LLVM 实用程序构建 Linux 内核。

概述
""""""

Linux 内核传统上一直使用 GNU 工具链（例如 GCC 和 binutils）进行编译。正在进行的工作允许将Clang和LLVM实用程序用作可行的替代品。Android、ChromeOS和OpenMandriva等发行版使用 Clang 构建的内核。 LLVM 是根据 C++ 对象实现的工具链组件的集合。Clang 是 LLVM 的前端，支持内核所需的 C 和 GNU C 扩展，发音为“klang”，而不是“see-lang”。

Clang
""""""

使用的编译器可以通过CC=命令行参数换出到make. CC=应在选择配置和构建期间设置。

.. code-block:: c
	:caption: make 调用自定义函数
	:emphasize-lines: 4,5
	:linenos:
	
	make CC=clang defconfig
	make CC=clang

交叉编译
"""""""

单个 Clang 编译器二进制文件通常包含所有支持的后端，这有助于简化交叉编译。


.. code-block:: c
	:caption: make 调用自定义函数
	:emphasize-lines: 4,5
	:linenos:
	
	make ARCH=arm64 CC=clang CROSS_COMPILE=aarch64-linux-gnu-
	
CROSS_COMPILE不用于 Clang 编译器二进制文件的前缀，而是 CROSS_COMPILE用于设置命令行标志：--target=<triple>. 例如：

.. code-block:: c
	:caption: make 调用自定义函数
	:emphasize-lines: 4,5
	:linenos:
	
	clang --target=aarch64-linux-gnu foo.c
	

LLVM 实用程序
""""""""""""
LLVM 可以替代 GNU binutils 实用程序。Kbuild 支持LLVM=1 启用它们。


.. code-block:: c
	:caption: make 调用自定义函数
	:emphasize-lines: 4,5
	:linenos:
	
	make LLVM=1
	
它们可以单独启用。完整的参数列表：

.. code-block:: c
	:caption: make 调用自定义函数
	:emphasize-lines: 4,5
	:linenos:
	
	make CC=clang LD=ld.lld AR=llvm-ar NM=llvm-nm STRIP=llvm-strip \
	OBJCOPY=llvm-objcopy OBJDUMP=llvm-objdump READELF=llvm-readelf \
	HOSTCC=clang HOSTCXX=clang++ HOSTAR=llvm-ar HOSTLD=ld.lld
	
默认情况下启用集成汇编器。你可以通过LLVM_IAS=0禁用它。

省略 CROSS_COMPILE
""""""""""""""""""

如上所述，CROSS_COMPILE用于设置--target=<triple>.

如果CROSS_COMPILE未指定，--target=<triple>则从 推断ARCH。

这意味着如果你只使用 LLVM 工具，CROSS_COMPILE就没有必要了。

例如，交叉编译 arm64 内核：

.. code-block:: c
	:caption: make 调用自定义函数
	:emphasize-lines: 4,5
	:linenos:
	
	make ARCH=arm64 LLVM=1
	
如果LLVM_IAS=0指定，CROSS_COMPILE也用于派生 --prefix=<path>搜索 GNU 汇编器和链接器。

.. code-block:: c
	:caption: make 调用自定义函数
	:emphasize-lines: 4,5
	:linenos:
	
	make ARCH=arm64 LLVM=1 LLVM_IAS=0 CROSS_COMPILE=aarch64-linux-gnu-
	
支持的架构
"""""""""
可通过产寻LLVM相关版本文档进行确认。

参考
""""""
https://www.kernel.org/doc/html/latest/kbuild/llvm.html

	
	
	
	
	
	
	
	
	
	
	
	





