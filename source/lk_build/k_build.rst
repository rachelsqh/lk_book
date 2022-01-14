Kconfig配置
-------------

输出文件
^^^^^^
	1. modules.order:
	2. modules.builtin:
	3. modules.builtin.modinfo

环境变量
^^^^^^^
	1. KCPPFLAGS：
	2. KAFLAGS：
	3. AFLAGS_MODULE:
	4. AFLAGS_KERNEL:
	5. KCFLAGS:
	6. CFLAGS_KERNEL:
	7. CFLAGS_MODULE:
	8. LDFLAGS_MODULE:
	9. HOSTCFLAGS:
	10. HOSTCXXFLAGS:
	11. HOSTLDFLAGS:
	12. HOSTLDLIBS:
	13. KBUILD_KCONFIG:
	14. KBUILD_VERBOSE:
	15. KBUILD_EXTMOD:
	16. KBUILD_OUTPUT:
	17. KBUILD_EXTRA_WARN:
	18. KBUILD_DEBARCH:
	19. ARCH:
	20. CROSS_COMPILE:
	21. CF
	22. INSTALL_PATH:
	23. INSTALLKERNEL:
	24. MODLIB
	25. INSTALL_MOD_PATH
	26. INSTALL_MOD_STRIP
	27. INSTALL_HDR_PATH
	28. KBUILD_ABS_SRCTREE
	29. KBUILD_SIGN_PIN
	30. KBUILD_MODPOST_WARN
	31. KBUILD_MODPOST_NOFINAL
	32. KBUILD_EXTRA_SYMBOLS
	33. ALLSOURCE_ARCHS
	34. KBUILD_BUILD_TIMESTAMP
	35. KBUILD_BUILD_USER,KBUILD_BUILD_HOST
	36. LLVM


Kconfig语法
^^^^^^^^^^^^
Kconfig作为Kbuild的配置数据库分散在需要编译的内核文件路径中，是配置选项以树形结构进行组织的集合。

.. code-block:: c
   :caption: Kconfig配置选项
   :emphasize-lines: 4,5
   :linenos:
   
   +- Code maturity level options
   |  +- Prompt for development and/or incomplete code/drivers
   +- General setup
   |  +- Networking support
   |  +- System V IPC
   |  +- BSD Process Accounting
   |  +- Sysctl support
   +- Loadable module support
   |  +- Enable loadable module support
   |     +- Set version information on all module symbols
   |     +- Kernel module loader
   +- ...
   
对Kconfig的理解其实很简单，不要往复杂了想，Kconfig文件只是根据定义的格式来存储配置。

其具体语法参考：https://docs.kernel.org/kbuild/kconfig-language.html

Kconfig宏
^^^^^^^^^
类似与C语言中的预处理阶段，将Kconfig宏经过处理后产生最终构建系统需要的数据库文件数据：

.. code-block:: c
   :caption: Kconfig宏
   :emphasize-lines: 4,5
   :linenos:
   
   CC := gcc
   
   config CC_HAS_FOO
        def_bool $(shell, $(srctree)/scripts/gcc-check-foo.sh $(CC))
        
预处理后：

.. code-block:: c
   :caption: Kconfig 宏扩展后
   :emphasize-lines: 4,5
   :linenos:
   
   config CC_HAS_FOO
        def_bool y

此时Kbuild可以直接用于解决依赖关系。

变量
^^^^^
包含在$()中，分为简单扩展变量和递归扩展变量。
	1. := 赋值运算符定义一个简单扩展变量，从Kconfig读取后，立即进行扩展；
	2. =  定义递归扩展变量。不立即展开，只简单存储，使用变量时进行展开；
	3. += 将文本附加到变量。由第一次赋值方式决定是简单变量还是递归扩展变量。
	
变量引用可以采用以下形式的参数：

.. code-block:: c
   :caption: Kconfig 宏扩展后
   :emphasize-lines: 4,5
   :linenos:
   
   $(name,arg1,arg2,arg3)
   
可以将参数化引用视为一个函数。（更准确地说，“用户定义的函数”与下面列出的“内置函数”形成对比）。

有用的函数在使用时必须扩展，因为如果传递不同的参数，相同的函数会以不同的方式扩展。因此，用户定义的函数是使用 = 赋值运算符定义的。在主体定义中使用 $(1)、$(2) 等来引用参数。

事实上，递归扩展的变量和用户定义的函数在内部是一样的。（换句话说，“变量”是“零参数的函数”。）当我们说广义的“变量”时，它包括“用户定义的函数”。

内置函数
^^^^^^^
Kconfig 提供了几个内置函数。每个函数都接受特定数量的参数。允许内置函数使用零参数，例如 $(filename)、$(lineno)。您可以将它们视为“内置变量”，支持的内置函数有：
	
	1. $(shell,command):command作为子shell进行执行。然后读取命令的标准输出并将其作为函数的值返回。输出中的每个换行符都替换为一个空格。任何尾随的换行符都将被删除。不返回标准错误，也不返回任何程序退出状态。
	2. $(info,text)：“info”函数接受一个参数并将其打印到标准输出。它评估为一个空字符串。
	3. $(warning-if,condition,text)：“warning-if”函数有两个参数。如果条件部分是“y”，则将文本部分发送到 stderr。该文本以当前 Kconfig 文件的名称和当前行号为前缀。
	4. $(error-if,condition,text)：“error-if”函数类似于“warning-if”，但如果条件部分是“y”，它会立即终止解析。
	5. $(filename)：'filename' 不带参数，$(filename) 被扩展为被解析的文件名。
	6. $(lineno)：'lineno' 不带参数，并且 $(lineno) 被扩展为被解析的行号。
	
make 与 Kconfig 函数调用方式比较
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
make中函数调用方式如下：

.. code-block:: c
   :caption: Kconfig 宏扩展后
   :emphasize-lines: 4,5
   :linenos:
   
   $(func-name arg1,arg2,arg3)
   
函数名和第一个参数至少用一个空格分隔。然后，从第一个参数中删除前导空格，而保留其他参数中的空格。您需要使用一种技巧来以空格开头第一个参数。例如，如果要让“info”函数打印“hello”，可以这样写：

.. code-block:: c
   :caption: Kconfig 宏扩展后
   :emphasize-lines: 4,5
   :linenos:
   
   empty :=
   space := $(empty) $(empty)
   $(info $(space)$(space)hello)
   
Kconfig 仅使用逗号作为分隔符，并在函数调用中保留所有空格。有些人喜欢在每个逗号分隔符后放置一个空格：

.. code-block:: c
   :caption: Kconfig 宏扩展后
   :emphasize-lines: 4,5
   :linenos:
   
   $(func-name, arg1, arg2, arg3)

在这种情况下，“func-name”将收到“arg1”、“arg2”、“arg3”。前导空格的存在可能很重要，具体取决于功能。

make中,使用内置函数"call"来引用用户定义的函数：

.. code-block:: c
   :caption: make 调用自定义函数
   :emphasize-lines: 4,5
   :linenos:
   
   $(call my-func,arg1,arg2,arg3)
   
Kconfig以相同的方式调用用户定义的函数和内置函数，区别只是省略了‘call'。

在 Make 中，一些函数逐字处理逗号而不是参数分隔符。例如，$(shell echo hello, world) 运行命令“echo hello, world”。同样，$(info hello, world) 将“hello, world”打印到标准输出。你可以说这是_有用_不一致。

在 Kconfig 中，为了更简单的实现和语法一致性，出现在 $( ) 上下文中的逗号始终是分隔符。它的意思是：

.. code-block:: c
   :caption: make 调用自定义函数
   :emphasize-lines: 4,5
   :linenos:
   
   $(shell, echo hello, world)

并不能正确解析。Kconfig中固定把“,”作为分割符。所以无法输出“,"，要想输出逗号，可以以如下方式处理：

.. code-block:: c
   :caption: make 调用自定义函数
   :emphasize-lines: 4,5
   :linenos:
   
   comma := ,
   $(shell, echo hello$(comma) world)
   
注意：（这个注意选项，需要很多地方确认）
^^^^
变量（函数）不能跨标记扩展。不能扩展为Kconfig中的任何关键字。


make kconfig:配置内核
^^^^^^^^^^^^^^^^^^^^^^
xconfig,menuconfig,nconfig:






   
