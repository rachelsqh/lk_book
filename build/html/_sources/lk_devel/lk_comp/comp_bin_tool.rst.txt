二进制工具
----------
nm
^^^^^^^

nm [-A|-o|--print-file-name]
   [-a|--debug-syms]
   [-B|--format=bsd]
   [-C|--demangle[=style]]
   [-D|--dynamic]
   [-fformat|--format=format]
   [-g|--extern-only]
   [-h|--help]
   [--ifunc-chars=CHARS]
   [-j|--format=just-symbols]
   [-l|--line-numbers] [--inlines]
   [-n|-v|--numeric-sort]
   [-P|--portability]
   [-p|--no-sort]
   [-r|--reverse-sort]
   [-S|--print-size]
   [-s|--print-armap]
   [-t radix|--radix=radix]
   [-u|--undefined-only]
   [-U method] [--unicode=method]
   [-V|--version]
   [-X 32_64]
   [--defined-only]
   [--no-demangle]
   [--no-recurse-limit|--recurse-limit]]
   [--plugin name]
   [--size-sort]
   [--special-syms]
   [--synthetic]
   [--target=bfdname]
   [--with-symbol-versions]
   [--without-symbol-versions]
   [objfile…]
   
GNU nm列出了来自目标文件objfile ... 的符号。如果没有目标文件被列为参数，nm则假定文件 a.out.

对于每个符号，nm显示：
符号值，在选项选择的基数中（见下文），或默认为十六进制。
符号类型。至少使用以下类型；其他的也取决于目标文件格式。如果是小写，符号通常是本地的；如果是大写，则符号是全局的（外部的）。但是，对于特殊的全局符号（和） u，显示了一些小写符号。vw
A
符号的值是绝对的，不会通过进一步的链接而改变。

B
b
该符号位于 BSS 数据部分中。此部分通常包含零初始化或未初始化的数据，尽管确切的行为取决于系统。

C
c
符号很常见。常用符号是未初始化的数据。链接时，可能会出现多个同名的常用符号。如果符号在任何地方定义，则公共符号将被视为未定义的引用。有关常用符号的更多详细信息，请参阅The GNU linker的链接器选项中的 –warn-common 讨论。当符号位于小公地的特殊部分时，使用 小写c字符。

D
d
该符号位于初始化数据部分中。

G
g
该符号位于小对象的初始化数据部分中。一些对象文件格式允许更有效地访问小型数据对象，例如全局 int 变量，而不是大型全局数组。

i
对于 PE 格式文件，这表明符号位于特定于 DLL 实现的部分中。

对于 ELF 格式文件，这表明该符号是一个间接函数。这是标准 ELF 符号类型集的 GNU 扩展。它表示一个符号，如果被重定位引用，则不会计算其地址，而是必须在运行时调用。然后运行时执行将返回要在重定位中使用的值。

注意 - GNU 间接符号的实际符号显示由--ifunc 字符命令行选项。如果已提供此选项，则字符串中的第一个字符将用于全局间接函数符号。如果字符串包含第二个字符，那么它将用于本地间接函数符号。

I
该符号是对另一个符号的间接引用。

N
该符号是一个调试符号。

n
该符号位于只读数据部分。

p
该符号位于堆栈展开部分中。

R
r
该符号位于只读数据段中。

S
s
该符号位于小对象的未初始化或零初始化数据部分中。

T
t
该符号位于文本（代码）部分。

U
符号未定义。

u
该符号是唯一的全局符号。这是标准 ELF 符号绑定集的 GNU 扩展。对于这样的符号，动态链接器将确保在整个过程中只有一个具有此名称和类型的符号在使用中。

V
v
符号是一个弱对象。当弱定义符号与正常定义符号链接时，使用正常定义符号不会出错。当链接一个弱未定义符号且该符号未定义时，该弱符号的值变为零且没有错误。在某些系统上，大写表示已指定默认值。

W
w
该符号是一个弱符号，尚未专门标记为弱对象符号。当弱定义符号与正常定义符号链接时，使用正常定义符号不会出错。当链接弱未定义符号且未定义符号时，符号的值以系统特定的方式确定而不会出错。在某些系统上，大写表示已指定默认值。

-
该符号是 a.out 目标文件中的 stabs 符号。在这种情况下，打印的下一个值是 stabs other 字段、stabs desc 字段和 stab 类型。Stabs 符号用于保存调试信息。

?
符号类型未知，或特定于目标文件格式。

符号名称。如果符号具有与之关联的版本信息，则也会显示版本信息。如果版本化符号未定义或对链接器隐藏，则版本字符串显示为符号名称的后缀，前面有一个 @ 字符。例如 'foo@VER_1'。如果版本是解析对符号的无版本引用时使用的默认版本，则它显示为一个后缀，前面有两个 @ 字符。例如 'foo@@VER_2'。
在这里作为备选方案显示的多头和空头形式的期权是等价的。

-A
-o
--print-file-name
在每个符号之前加上它所在的输入文件（或存档成员）的名称，而不是在所有符号之前仅标识输入文件一次。

-a
--debug-syms
显示所有符号，甚至是调试器专用符号；通常这些都没有列出。

-B
一样--格式=bsd（为了与 MIPS 兼容nm）。

-C
--demangle[=style]
将 ( demangle ) 低级符号名称解码为用户级名称。除了删除系统前面的任何初始下划线外，这使得 C++ 函数名称可读。不同的编译器有不同的修饰风格。可选的 demangling 样式参数可用于为您的编译器选择合适的 demangling 样式。参见c++filt，了解更多关于去重的信息。

--no-demangle
不要对低级符号名称进行拆解。这是默认设置。

--recurse-limit
--no-recurse-limit
--recursion-limit
--no-recursion-limit
启用或禁用在对字符串进行分解时执行的递归量的限制。由于名称修饰格式允许无限级别的递归，因此可以创建字符串，其解码将耗尽主机上可用的堆栈空间量，从而触发内存故障。该限制试图通过将递归限制为 2048 级嵌套来防止这种情况发生。

默认情况下启用此限制，但可能需要禁用它才能破解真正复杂的名称。但是请注意，如果禁用递归限制，则可能会耗尽堆栈，并且任何有关此类事件的错误报告都将被拒绝。

-D
--dynamic
显示动态符号而不是普通符号。这仅对动态对象有意义，例如某些类型的共享库。

-f format
--format=format
使用输出格式format，可以是bsd, sysv,posix或just-symbols. 默认值为bsd. 只有格式的第一个字符是有意义的；它可以是大写或小写。

-g
--extern-only
仅显示外部符号。

-h
--help
显示选项摘要nm并退出。

--ifunc-chars=CHARS
当显示 GNU 间接函数符号nm时，将默认使用i本地间接函数和全局间接函数的字符。这--ifunc 字符选项允许用户指定一个包含一个或两个字符的字符串。第一个字符将用于全局间接函数符号，第二个字符（如果存在）将用于局部间接函数符号。

j
一样--format=just-symbols.

-l
--line-numbers
对于每个符号，使用调试信息来尝试查找文件名和行号。对于已定义的符号，查找符号地址的行号。对于未定义的符号，查找引用该符号的重定位条目的行号。如果可以找到行号信息，则将其打印在其他符号信息之后。

--inlines
选项时-l处于活动状态，如果地址属于被内联的函数，则此选项会导致所有封闭范围的源信息也被打印回第一个非内联函数。例如，如果maininlines callee1which inlinescallee2和 address 是 from callee2，则 和 的源信息callee1也main 将被打印。

-n
-v
--numeric-sort
将符号按地址按数字排序，而不是按名称按字母顺序排序。

-p
--no-sort
不要费心按任何顺序对符号进行排序；按遇到的顺序打印它们。

-P
--portability
使用 POSIX.2 标准输出格式而不是默认格式。相当于 '-f posix'。

-r
--reverse-sort
颠倒排序的顺序（无论是数字还是字母）；让最后一个先来。

-S
--print-size
打印bsd输出样式的已定义符号的值和大小。此选项对不记录符号大小的对象格式无效，除非'--大小排序' 也用于在这种情况下显示计算的大小。

-s
--print-armap
ar当列出来自归档成员的符号时，包括索引：一个映射（由或存储在归档中ranlib）哪些模块包含哪些名称的定义。

-t radix
--radix=radix
使用radix作为打印符号值的基数。一定是 'd'对于十进制，'○' 表示八进制，或 'X' 为十六进制。

-u
--undefined-only
仅显示未定义的符号（每个目标文件外部的符号）。

-U [d|i|l|e|x|h]
--unicode=[default|invalid|locale|escape|hex|highlight]
控制字符串中 UTF-8 编码的多字节字符的显示。默认（--unicode=默认) 就是不给他们特殊待遇。这--unicode=语言环境选项显示当前语言环境中的序列，可能支持也可能不支持。选项 --unicode=十六进制和--unicode=无效将它们显示为用尖括号或花括号括起来的十六进制字节序列。

这--unicode=转义选项将它们显示为转义序列（\uxxxx）和--unicode=高亮选项将它们显示为以红色突出显示的转义序列（如果输出设备支持）。着色旨在引起人们对可能不期望出现的 unicode 序列的关注。

-V
--version
显示版本号nm并退出。

-X
为了与 AIX 版本的 nm. 它需要一个参数，该参数必须是字符串 32_64. AIX 的默认模式nm对应于-X 32, GNU nm不支持。

--defined-only
仅显示每个目标文件的已定义符号。

--plugin name
加载名为name的插件以添加对额外目标类型的支持。只有在构建工具链并启用插件支持时，此选项才可用。

如果- 插入未提供，但已启用插件支持，然后nm迭代中的文件 ${libdir}/bfd-plugins按字母顺序，并使用第一个声称有问题的对象的插件。

请注意，这个插件搜索目录不是ld's使用的目录-插入选项。为了 nm使用链接器插件，必须将其复制到 ${libdir}/bfd-plugins目录。对于基于 GCC 的编译，链接器插件称为liblto_plugin.so.0.0.0. 对于基于 Clang 的编译，它被称为LLVMgold.so. GCC 插件始终向后兼容早期版本，因此只需复制最新版本就足够了。

--size-sort
按大小对符号进行排序。对于 ELF 对象，符号大小从 ELF 中读取，对于其他对象类型，符号大小计算为符号值与具有下一个更高值的符号值之间的差。如果使用bsd输出格式，则打印符号的大小，而不是值，并且 '-S' 必须使用才能打印大小和值。

--special-syms
显示具有特定于目标的特殊含义的符号。这些符号通常由目标用于某些特殊处理，并且在包含在正常符号列表中时通常没有帮助。例如，对于 ARM 目标，此选项将跳过用于标记 ARM 代码、THUMB 代码和数据之间转换的映射符号。

--synthetic
在输出中包含合成符号。这些是链接器为各种目的创建的特殊符号。默认情况下不显示它们，因为它们不是二进制原始源代码的一部分。

--with-symbol-versions
--without-symbol-versions
启用或禁用符号版本信息的显示。版本字符串显示为符号名称的后缀，前面有一个 @ 字符。例如 'foo@VER_1'。如果版本是解析对符号的无版本引用时使用的默认版本，则它显示为一个后缀，前面有两个 @ 字符。例如 'foo@@VER_2'。默认显示符号版本信息。

--target=bfdname
指定系统默认格式以外的目标代码格式。有关详细信息，请参阅目标选择。

 objcopy
 ^^^^^^^^^^^^^
 objcopy [-F bfdname|--target=bfdname]
        [-I bfdname|--input-target=bfdname]
        [-O bfdname|--output-target=bfdname]
        [-B bfdarch|--binary-architecture=bfdarch]
        [-S|--strip-all]
        [-g|--strip-debug]
        [--strip-unneeded]
        [-K symbolname|--keep-symbol=symbolname]
        [--keep-file-symbols]
        [--keep-section-symbols]
        [-N symbolname|--strip-symbol=symbolname]
        [--strip-unneeded-symbol=symbolname]
        [-G symbolname|--keep-global-symbol=symbolname]
        [--localize-hidden]
        [-L symbolname|--localize-symbol=symbolname]
        [--globalize-symbol=symbolname]
        [--globalize-symbols=filename]
        [-W symbolname|--weaken-symbol=symbolname]
        [-w|--wildcard]
        [-x|--discard-all]
        [-X|--discard-locals]
        [-b byte|--byte=byte]
        [-i [breadth]|--interleave[=breadth]]
        [--interleave-width=width]
        [-j sectionpattern|--only-section=sectionpattern]
        [-R sectionpattern|--remove-section=sectionpattern]
        [--keep-section=sectionpattern]
        [--remove-relocations=sectionpattern]
        [-p|--preserve-dates]
        [-D|--enable-deterministic-archives]
        [-U|--disable-deterministic-archives]
        [--debugging]
        [--gap-fill=val]
        [--pad-to=address]
        [--set-start=val]
        [--adjust-start=incr]
        [--change-addresses=incr]
        [--change-section-address sectionpattern{=,+,-}val]
        [--change-section-lma sectionpattern{=,+,-}val]
        [--change-section-vma sectionpattern{=,+,-}val]
        [--change-warnings] [--no-change-warnings]
        [--set-section-flags sectionpattern=flags]
        [--set-section-alignment sectionpattern=align]
        [--add-section sectionname=filename]
        [--dump-section sectionname=filename]
        [--update-section sectionname=filename]
        [--rename-section oldname=newname[,flags]]
        [--long-section-names {enable,disable,keep}]
        [--change-leading-char] [--remove-leading-char]
        [--reverse-bytes=num]
        [--srec-len=ival] [--srec-forceS3]
        [--redefine-sym old=new]
        [--redefine-syms=filename]
        [--weaken]
        [--keep-symbols=filename]
        [--strip-symbols=filename]
        [--strip-unneeded-symbols=filename]
        [--keep-global-symbols=filename]
        [--localize-symbols=filename]
        [--weaken-symbols=filename]
        [--add-symbol name=[section:]value[,flags]]
        [--alt-machine-code=index]
        [--prefix-symbols=string]
        [--prefix-sections=string]
        [--prefix-alloc-sections=string]
        [--add-gnu-debuglink=path-to-file]
        [--only-keep-debug]
        [--strip-dwo]
        [--extract-dwo]
        [--extract-symbol]
        [--writable-text]
        [--readonly-text]
        [--pure]
        [--impure]
        [--file-alignment=num]
        [--heap=size]
        [--image-base=address]
        [--section-alignment=num]
        [--stack=size]
        [--subsystem=which:major.minor]
        [--compress-debug-sections]
        [--decompress-debug-sections]
        [--elf-stt-common=val]
        [--merge-notes]
        [--no-merge-notes]
        [--verilog-data-width=val]
        [-v|--verbose]
        [-V|--version]
        [--help] [--info]
        infile [outfile]

GNU 实用程序将objcopy一个目标文件的内容复制到另一个。 objcopy使用GNU BFD库来读取和写入目标文件。它可以以不同于源目标文件的格式写入目标目标文件。的确切行为objcopy由命令行选项控制。请注意，objcopy应该能够在任何两种格式之间复制完全链接的文件。但是，在任何两种格式之间复制可重定位目标文件可能无法按预期工作。

objcopy创建临时文件进行翻译，然后将其删除。 objcopy使用BFD完成所有翻译工作；它可以访问BFD中描述的所有格式 ，因此能够识别大多数格式而无需明确告知。请参阅使用 LD中的BFD。

objcopy可用于通过使用输出目标 ' 来生成 S 记录srec'（例如，使用 '-O srec'）。

objcopy可用于通过使用 ' 的输出目标生成原始二进制文件二进制'（例如，使用-O 二进制）。当 objcopy生成原始二进制文件时，它本质上会生成输入目标文件内容的内存转储。所有符号和重定位信息都将被丢弃。内存转储将从复制到输出文件的最低部分的加载地址开始。

在生成 S 记录或原始二进制文件时，使用它可能会有所帮助-S删除包含调试信息的部分。在某些情况下-R将有助于删除包含二进制文件不需要的信息的部分。

注意——objcopy不能改变其输入文件的字节顺序。如果输入格式具有字节序（某些格式没有）， objcopy则只能将输入复制到具有相同字节序或没有字节序的文件格式（例如，'srec'）。（但是，请参阅--反向字节选项。）

infile
outfile
分别是输入和输出文件。如果您不指定outfile，则objcopy创建一个临时文件并将结果破坏性地重命名为infile。

-I bfdname
--input-target=bfdname
考虑源文件的对象格式为bfdname，而不是试图推断它。有关详细信息，请参阅目标选择。

-O bfdname
--output-target=bfdname
使用对象格式bfdname写入输出文件。有关详细信息，请参阅目标选择。

-F bfdname
--target=bfdname
使用bfdname作为输入和输出文件的对象格式；即，只需将数据从源传输到目标，无需转换。有关详细信息，请参阅目标选择。

-B bfdarch
--binary-architecture=bfdarch
在将无体系结构的输入文件转换为目标文件时很有用。在这种情况下，输出架构可以设置为bfdarch。如果输入文件具有已知的bfdarch ，则此选项将被忽略。您可以通过引用转换过程创建的特殊符号来访问程序内部的此二进制数据。这些符号称为 _binary_ objfile _start、_binary_ objfile _end 和 _binary_ objfile _size。例如，您可以将图片文件转换为目标文件，然后使用这些符号在您的代码中访问它。

-j sectionpattern
--only-section=sectionpattern
仅将指定部分从输入文件复制到输出文件。这个选项可能不止一次给出。请注意，不恰当地使用此选项可能会使输出文件无法使用。sectionpattern中接受通配符。

如果sectionpattern的第一个字符是感叹号 (!)，则不会复制匹配的部分，即使之前使用--仅部分在同一命令行上否则会复制它。例如：

  --only-section=.text.* --only-section=!.text.foo
将复制所有匹配 '.text.*' 但不匹配 '.text.foo' 部分的 sectinos。

-R sectionpattern
--remove-section=sectionpattern
从输出文件中删除任何与sectionpattern匹配的部分。这个选项可能不止一次给出。请注意，不恰当地使用此选项可能会使输出文件无法使用。sectionpattern中接受通配符。同时使用 -j和-R选项一起导致未定义的行为。

如果sectionpattern的第一个字符是感叹号 (!)，那么即使之前使用--删除部分在同一命令行上否则会删除它。例如：

  --remove-section=.text.* --remove-section=!.text.foo
将删除与模式“.text.*”匹配的所有部分，但不会删除“.text.foo”部分。

--keep-section=sectionpattern
从输出文件中删除部分时，保留与 sectionpattern匹配的部分。

--remove-relocations=sectionpattern
从输出文件中删除与sectionpattern匹配的任何部分的非动态重定位。这个选项可能不止一次给出。请注意，不恰当地使用此选项可能会使输出文件无法使用，并尝试删除动态重定位部分，例如 '.rela.plt' 来自可执行文件或共享库 --remove-relocations=.plt不管用。sectionpattern中接受通配符。例如：

  --remove-relocations=.text.*
将删除匹配模式 '.text.*' 的所有部分的重定位。

如果sectionpattern的第一个字符是感叹号 (!)，那么即使之前使用--remove-relocations在同一命令行上，否则会导致重定位被删除。例如：

  --remove-relocations=.text.* --remove-relocations=!.text.foo
将删除与模式“.text.*”匹配的部分的所有重定位，但不会删除“.text.foo”部分的重定位。

-S
--strip-all
不要从源文件中复制重定位和符号信息。还会删除调试部分。

-g
--strip-debug
不要从源文件中复制调试符号或部分。

--strip-unneeded
除了调试符号和由 --strip-debug.

-K symbolname
--keep-symbol=symbolname
剥离符号时，请保留符号symbolname ，即使它通常会被剥离。这个选项可能不止一次给出。

-N symbolname
--strip-symbol=symbolname
不要从源文件中复制符号symbolname 。这个选项可能不止一次给出。

--strip-unneeded-symbol=symbolname
除非重定位需要，否则不要从源文件中复制符号symbolname 。这个选项可能不止一次给出。

-G symbolname
--keep-global-symbol=symbolname
只保留符号symbolname全局。将所有其他符号设置为文件本地，以便它们在外部不可见。这个选项可能不止一次给出。注意：此选项不能与--globalize-符号要么 --globalize-symbols选项。

--localize-hidden
在 ELF 对象中，将所有具有隐藏或内部可见性的符号标记为本地。此选项适用于特定于符号的本地化选项，例如-L.

-L symbolname
--localize-symbol=symbolname
将名为symbolname的全局或弱符号转换为局部符号，使其在外部不可见。这个选项可能不止一次给出。注意 - 唯一符号不会被转换。

-W symbolname
--weaken-symbol=symbolname
使符号symbolname弱。这个选项可能不止一次给出。

--globalize-symbol=symbolname
为符号symbolname 提供全局范围，使其在定义它的文件之外可见。这个选项可能不止一次给出。注意：此选项不能与-G要么--keep-global-symbol选项。

-w
--wildcard
允许在其他命令行选项中使用的symbolname中的正则表达式。问号 (?)、星号 (*)、反斜杠 (\) 和方括号 ([]) 运算符可用于符号名称中的任何位置。如果符号名称的第一个字符是感叹号 (!)，则该符号的开关意义相反。例如：

  -w -W !foo -W fo*
将导致 objcopy 削弱除符号“foo”之外的所有以“fo”开头的符号。

-x
--discard-all
不要从源文件中复制非全局符号。

-X
--discard-locals
不要复制编译器生成的本地符号。（这些通常以'开头大号' 要么 '.'。）

-b byte
--byte=byte
如果已通过--交错选项然后开始字节范围以保持在第字节字节。 byte可以在 0 到width -1 的范围内，其中 width是由--交错选项。

-i [breadth]
--interleave[=breadth]
仅从每个宽度字节中复制一个范围。（标题数据不受影响）。选择范围中的哪个字节开始复制- 字节选项。选择范围的宽度 --交错宽度选项。

此选项对于创建文件以对ROM进行编程很有用。它通常与srec输出目标一起使用。请注意， objcopy如果您不指定 - 字节选项也是如此。

默认交错宽度为 4，因此使用- 字节设置为 0， objcopy将每四个字节中的第一个字节从输入复制到输出。

--interleave-width=width
当与--交错选项，一次复制宽度 字节。要复制的字节范围的开始由- 字节选项，范围的范围设置与--交错选项。

此选项的默认值为 1。width 的值加上由- 字节选项不得超过由设置的交错宽度--交错选项。

此选项可用于为在 32 位总线中交错的两个 16 位闪存创建映像，方法是通过-b 0 -i 4 --interleave-width=2 和-b 2 -i 4 --interleave-width=2到两个objcopy 命令。如果输入是“12345678”，那么输出将分别是“1256”和“3478”。

-p
--preserve-dates
将输出文件的访问日期和修改日期设置为与输入文件的访问日期和修改日期相同。

-D
--enable-deterministic-archives
以确定性模式运行。复制存档成员和写入存档索引时，对 UID、GID、时间戳使用零，并对所有文件使用一致的文件模式。

如果二进制工具配置了 --enable-deterministic-archives, 则默认开启此模式。它可以用'禁用-U' 选项，如下。

-U
--disable-deterministic-archives
不要在确定性模式下运行。这是相反的-D上面的选项：复制存档成员和写入存档索引时，使用它们的实际 UID、GID、时间戳和文件模式值。

这是默认设置，除非二进制工具配置了 --enable-deterministic-archives.

--debugging
如果可能，转换调试信息。这不是默认设置，因为仅支持某些调试格式，并且转换过程可能很耗时。

--gap-fill val
用val填充部分之间的间隙。此操作适用于段的加载地址(LMA)。这是通过增加具有较低地址的部分的大小并填充使用val创建的额外空间来完成的。

--pad-to address
将输出文件填充到加载地址address。这是通过增加最后一节的大小来完成的。额外的空间用指定的值填充- 填补空白（默认为零）。

--set-start val
将新文件的起始地址（也称为入口地址）设置为val。并非所有目标文件格式都支持设置起始地址。

--change-start incr
--adjust-start incr
通过添加incr更改起始地址（也称为入口地址） 。并非所有目标文件格式都支持设置起始地址。

--change-addresses incr
--adjust-vma incr
通过添加incr更改所有部分的 VMA 和 LMA 地址以及起始地址。某些目标文件格式不允许任意更改节地址。请注意，这不会重新定位这些部分；如果程序希望将节加载到某个地址，并且此选项用于更改节以使它们加载到不同的地址，则程序可能会失败。

--change-section-address sectionpattern{=,+,-}val
--adjust-section-vma sectionpattern{=,+,-}val
设置或更改任何与sectionpattern匹配的部分的 VMA 地址和 LMA 地址。如果 '=' 时，节地址设置为val。否则，将val添加到段地址或从段地址中减去。请参阅下面的评论 --更改地址， 更多。如果sectionpattern与输入文件中的任何部分都不匹配，则会发出警告，除非 --no-change-warnings用来。

--change-section-lma sectionpattern{=,+,-}val
设置或更改与sectionpattern匹配的任何部分的 LMA 地址 。LMA 地址是在程序加载时将段加载到内存中的地址。通常这与 VMA 地址相同，即程序运行时段的地址，但在某些系统上，尤其是那些程序保存在 ROM 中的系统上，两者可能不同。如果 '=' 时，节地址设置为val。否则， 将val添加到段地址或从段地址中减去。请参阅下面的评论--更改地址， 更多。如果 sectionpattern与输入文件中的任何部分都不匹配，则会发出警告，除非--no-change-warnings用来。

--change-section-vma sectionpattern{=,+,-}val
设置或更改任何与 sectionpattern匹配的部分的 VMA 地址。VMA 地址是程序开始执行后该段所在的地址。通常这与 LMA 地址相同，后者是将节加载到内存中的地址，但在某些系统上，尤其是那些程序保存在 ROM 中的系统上，两者可能不同。如果 '=' 时，节地址设置为 val。否则，将val添加到段地址或从段地址中减去。请参阅下面的评论--更改地址， 更多。如果sectionpattern与输入文件中的任何部分都不匹配，则会发出警告，除非 --no-change-warnings用来。

--change-warnings
--adjust-warnings
如果--change-section-address要么--change-section-lma要么 --change-section-vma已使用，并且节模式不匹配任何节，发出警告。这是默认设置。

--no-change-warnings
--no-adjust-warnings
不发出警告，如果--change-section-address要么 --adjust-section-lma要么--调整部分-vma使用，即使节模式不匹配任何节。

--set-section-flags sectionpattern=flags
为与sectionpattern匹配的任何部分设置标志。flags参数是一个逗号分隔的 标志名称字符串。公认的名字是'分配', '内容', '加载', '空载', '只读', '代码', '数据', '只读存储器', '排除', '分享'， 和 '调试'。你可以设置'内容' 没有内容的部分的标志，但清除它没有意义 '内容' 确实有内容的部分的标志 - 只需删除该部分即可。并非所有标志对所有目标文件格式都有意义。特别是'分享' 标志只对 COFF 格式文件有意义，对 ELF 格式文件没有意义。

--set-section-alignment sectionpattern=align
为匹配sectionpattern的任何部分设置对齐方式。 align以字节为单位指定对齐方式，并且必须是 2 的幂，即 1、2、4、8……。

--add-section sectionname=filename
在复制文件时添加一个名为sectionname的新部分。新部分的内容取自文件filename。该部分的大小将是文件的大小。此选项仅适用于可以支持具有任意名称的部分的文件格式。注意 - 可能需要使用--set-section-flags 选项来设置新创建的部分的属性。

--dump-section sectionname=filename
将名为sectionname的部分的内容放入文件 filename中，覆盖之前可能存在的任何内容。此选项与--添加部分. 此选项类似于--仅部分选项除了它不创建格式化文件外，它只是将内容转储为原始二进制数据，而不应用任何重定位。可以多次指定该选项。

--update-section sectionname=filename
将名为sectionname的部分的现有内容替换 为文件filename的内容。该部分的大小将调整为文件的大小。sectionname的部分标志 将保持不变。对于 ELF 格式文件，段到段的映射也将保持不变，这是无法使用的--删除部分其次是 --添加部分. 可以多次指定该选项。

注意 - 可以使用--重命名部分和 --更新部分从一个命令行更新和重命名一个部分。在这种情况下，将原始部分名称传递给 --更新部分，以及原始和新的部分名称 --重命名部分.

--add-symbol name=[section:]value[,flags]
在复制文件时添加一个名为name的新符号。可以多次指定此选项。如果给出了部分，则符号将与该部分相关联并相对于该部分，否则它将是一个 ABS 符号。指定未定义的部分将导致致命错误。没有检查该值，它将按规定进行。可以指定符号标志，但并非所有标志对所有目标文件格式都有意义。默认情况下，符号将是全局的。特殊标志 'before= othersym ' 将在指定的 othersym前面插入新符号​​，否则符号将按照它们出现的顺序添加到符号表的末尾。

--rename-section oldname=newname[,flags]
将部分从oldname重命名为newname，可选择将部分的标志更改为进程中的标志。与使用链接器脚本执行重命名相比，这具有优势，因为输出保留为目标文件并且不会成为链接的可执行文件。此选项接受与 --sect-section-flags选项。

当输入格式为二进制时，此选项特别有用，因为这将始终创建一个名为 .data 的部分。例如，如果您想创建一个名为 .rodata 的包含二进制数据的部分，您可以使用以下命令行来实现它：

  objcopy -I 二进制 -O <输出格式> -B <架构> \
   --rename-section .data=.rodata,alloc,load,readonly,data,contents \
   <输入二进制文件> <输出对象文件>
--long-section-names {enable,disable,keep}
在处理COFF 和PE-COFF对象格式时控制长节名称的处理。默认行为，'保持'，如果输入文件中存在长段名称，则保留长段名称。这 '使能够' 和 '禁用' 选项强制启用或禁用输出对象中长节名称的使用；什么时候 '禁用' 生效时，输入对象中的任何长节名称都将被截断。这 '使能够' 选项只会在输入中存在长节名时发出长节名；这与 '保持'，但未定义 '使能够' 选项可能会强制在输出文件中创建一个空字符串表。

--change-leading-char
一些目标文件格式在符号的开头使用特殊字符。最常见的此类字符是下划线，编译器通常会在每个符号之前添加下划线。此选项告诉objcopy在目标文件格式之间转换时更改每个符号的前导字符。如果目标文件格式使用相同的前导字符，则此选项无效。否则，它将根据需要添加一个字符、删除一个字符或更改一个字符。

--remove-leading-char
如果全局符号的第一个字符是目标文件格式使用的特殊符号前导字符，则删除该字符。最常见的符号前导字符是下划线。此选项将从所有全局符号中删除前导下划线。如果您想将具有不同符号名称约定的不同文件格式的对象链接在一起，这将很有用。这不同于 --change-leading-char因为无论输出文件的目标文件格式如何，它总是在适当的时候更改符号名称。

--reverse-bytes=num
使用输出内容反转节中的字节。部分长度必须能被给定的值整除，才能进行交换。反转发生在执行交织之前。

此选项通常用于为有问题的目标系统生成 ROM 映像。例如，在某些目标板上，从 8 位 ROM 中提取的 32 位字以 little-endian 字节顺序重新组合，而不管 CPU 字节顺序如何。根据编程模型，可能需要修改 ROM 的字节序。

考虑一个包含以下八个字节的部分的简单文件： 12345678.

使用 '--reverse-bytes=2' 对于上面的例子，输出文件中的字节是有序的21436587。

使用 '--reverse-bytes=4' 对于上面的例子，输出文件中的字节是有序的43218765。

通过使用 '--reverse-bytes=2' 对于上面的例子，后跟 '--reverse-bytes=4' 在输出文件上，第二个输出文件中的字节将被排序34127856。

--srec-len=ival
仅对 srec 输出有意义。将生成的 Srecord 的最大长度设置为ival。此长度涵盖地址、数据和 crc 字段。

--srec-forceS3
仅对 srec 输出有意义。避免生成 S1/S2 记录，创建仅限 S3 的记录格式。

--redefine-sym old=new
将符号名old更改为new。当您尝试将两个您没有来源的事物链接在一起并且存在名称冲突时，这可能很有用。

--redefine-syms=filename
申请--重新定义符号到文件文件名中列出的每个符号对“旧 新” 。 filename只是一个平面文件，每行有一个符号对。行注释可以由散列字符引入。这个选项可能不止一次给出。

--weaken
将文件中的所有全局符号更改为弱。这在构建一个对象时很有用，该对象将使用-R链接器的选项。此选项仅在使用支持弱符号的目标文件格式时有效。

--keep-symbols=filename
申请--保持符号文件filename中列出的每个符号的选项 。 filename只是一个平面文件，每行有一个符号名称。行注释可以由散列字符引入。这个选项可能不止一次给出。

--strip-symbols=filename
申请--strip-符号文件filename中列出的每个符号的选项 。 filename只是一个平面文件，每行有一个符号名称。行注释可以由散列字符引入。这个选项可能不止一次给出。

--strip-unneeded-symbols=filename
申请--strip-不需要的符号文件filename中列出的每个符号的选项。 filename只是一个平面文件，每行有一个符号名称。行注释可以由散列字符引入。这个选项可能不止一次给出。

--keep-global-symbols=filename
申请--keep-global-symbol文件filename中列出的每个符号的选项。 filename只是一个平面文件，每行有一个符号名称。行注释可以由散列字符引入。这个选项可能不止一次给出。

--localize-symbols=filename
申请--localize-符号文件filename中列出的每个符号的选项 。 filename只是一个平面文件，每行有一个符号名称。行注释可以由散列字符引入。这个选项可能不止一次给出。

--globalize-symbols=filename
申请--globalize-符号文件filename中列出的每个符号的选项 。 filename只是一个平面文件，每行有一个符号名称。行注释可以由散列字符引入。这个选项可能不止一次给出。注意：此选项不能与-G要么--keep-global-symbol 选项。

--weaken-symbols=filename
申请--弱化符号文件filename中列出的每个符号的选项 。 filename只是一个平面文件，每行有一个符号名称。行注释可以由散列字符引入。这个选项可能不止一次给出。

--alt-machine-code=index
如果输出架构具有备用机器代码，请使用第 index个代码而不是默认代码。这在机器被分配了官方代码并且工具链采用新代码但其他应用程序仍然依赖于正在使用的原始代码的情况下很有用。对于基于 ELF 的体系结构，如果索引 替代项不存在，则该值被视为要存储在 ELF 标头的 e_machine 字段中的绝对数。

--writable-text
将输出文本标记为可写。此选项对所有目标文件格式都没有意义。

--readonly-text
使输出文本写保护。此选项对所有目标文件格式都没有意义。

--pure
将输出文件标记为按需分页。此选项对所有目标文件格式都没有意义。

--impure
将输出文件标记为不纯。此选项对所有目标文件格式都没有意义。

--prefix-symbols=string
使用string为输出文件中的所有符号添加前缀。

--prefix-sections=string
在输出文件中的所有部分名称前加上string前缀。

--prefix-alloc-sections=string
使用string为输出文件中所有已分配部分的所有名称添加前缀 。

--add-gnu-debuglink=path-to-file
创建一个 .gnu_debuglink 部分，其中包含对 文件路径的引用并将其添加到输出文件中。注意： path-to-file处的文件必须存在。添加 .gnu_debuglink 部分的部分过程涉及将调试信息文件内容的校验和嵌入到该部分中。

如果调试信息文件构建在一个位置，但稍后将安装到另一个位置，则不要使用安装位置的路径。这--add-gnu-debuglink 选项将失败，因为安装的文件尚不存在。而是将调试信息文件放在当前目录中并使用 --add-gnu-debuglink没有任何目录组件的选项，如下所示：

objcopy --add-gnu-debuglink=foo.debug
在调试时，调试器将尝试在一组已知位置中查找单独的调试信息文件。这些位置的确切集合取决于所使用的分布，但通常包括：

* The same directory as the executable.
* A sub-directory of the directory containing the executable
称为 .debug

* A global debug directory such as /usr/lib/debug.
只要在调试器运行之前将调试信息文件安装到这些位置之一，一切都应该正常工作。

--keep-section-symbils
剥离文件时，也许与--strip-debug要么 --strip-不需要, 保留任何指定节名称的符号，否则会被剥离。

--keep-file-symbols
剥离文件时，也许与--strip-debug要么 --strip-不需要, 保留任何指定源文件名的符号，否则会被剥离。

--only-keep-debug
剥离文件，删除不会被剥离的任何部分的内容--strip-debug并保持调试部分完好无损。在 ELF 文件中，这会保留输出中的所有音符部分。

注意 - 被剥离部分的部分标题被保留，包括它们的大小，但部分的内容被丢弃。保留部分标题，以便其他工具可以将 debuginfo 文件与实际可执行文件匹配，即使该可执行文件已被重定位到不同的地址空间。

目的是将此选项与 --add-gnu-debuglink创建一个两部分的可执行文件。一个是剥离的二进制文件，它将在 RAM 和分发中占用更少的空间，第二个是调试信息文件，仅在需要调试能力时才需要。创建这些文件的建议过程如下：

正常链接可执行文件。假设它被调用 foo然后......
运行objcopy --only-keep-debug foo foo.dbg以创建包含调试信息的文件。
运行objcopy --strip-debug foo以创建剥离的可执行文件。
运行objcopy --add-gnu-debuglink=foo.dbg foo 以将调试信息的链接添加到剥离的可执行文件中。
注意——选择.dbg作为调试信息文件的扩展名是任意的。该--only-keep-debug步骤也是可选的。你可以这样做：

正常链接可执行文件。
复制foo到 foo.full
跑步objcopy --strip-debug foo
跑步objcopy --add-gnu-debuglink=foo.full foo
即，指向的文件--add-gnu-debuglink可以是完整的可执行文件。它不必是由 --only-keep-debug转变。

注意—此开关仅适用于完全链接的文件。在调试信息可能不完整的目标文件上使用它是没有意义的。除了 gnu_debuglink 功能目前只支持一个包含调试信息的文件名，而不是基于一个对象文件的多个文件名。

--strip-dwo
删除所有 DWARF .dwo 部分的内容，保留剩余的调试部分和所有符号不变。此选项旨在供编译器用作-gsplit-dwarf选项，它在 .o 文件和单独的 .dwo 文件之间拆分调试信息。编译器在同一个文件中生成所有调试信息，然后使用--extract-dwo将 .dwo 部分复制到 .dwo 文件的选项，然后--strip-dwo从原始 .o 文件中删除这些部分的选项。

--extract-dwo
提取所有 DWARF .dwo 部分的内容。见 --strip-dwo选项了解更多信息。

--file-alignment num
指定文件对齐方式。文件中的部分将始终以该数字的倍数的文件偏移量开始。默认为 512。[此选项特定于 PE 目标。]

--heap reserve
--heap reserve,commit
指定要保留（并可选择提交）用作此程序的堆的内存字节数。[此选项特定于 PE 目标。]

--image-base value
使用value作为程序或 dll 的基地址。这是加载程序或 dll 时将使用的最低内存位置。为了减少重新定位和提高 dll 性能的需要，每个 dll 都应该有一个唯一的基地址，并且不与任何其他 dll 重叠。可执行文件的默认值为 0x400000，dll 的默认值为 0x10000000。[此选项特定于 PE 目标。]

--section-alignment num
设置 PE 标头中的节对齐字段。内存中的部分总是从这个数字的倍数的地址开始。默认为 0x1000。[此选项特定于 PE 目标。]

--stack reserve
--stack reserve,commit
指定要保留（并可选择提交）用作此程序堆栈的内存字节数。[此选项特定于 PE 目标。]

--subsystem which
--subsystem which:major
--subsystem which:major.minor
指定程序将在其下执行的子系统。的合法值为, , native, windows, console, posix, efi-app, efi-bsd, efi-rtd和sal-rtd. xbox您也可以选择设置子系统版本。数字值也被 接受。[此选项特定于 PE 目标。]

--extract-symbol
保留文件的部分标志和符号，但删除所有部分数据。具体来说，选项：

删除所有部分的内容；
将每个部分的大小设置为零；和
将文件的起始地址设置为零。
此选项用于构建.symVxWorks 内核的文件。它也可以是减小文件大小的有用方法--just-symbols 链接器输入文件。

--compress-debug-sections
使用来自 ELF ABI 的带有 SHF_COMPRESSED 的 zlib 压缩 DWARF 调试部分。注意 - 如果压缩实际上会使一个部分 变大，那么它不会被压缩。

--compress-debug-sections=none
--compress-debug-sections=zlib
--compress-debug-sections=zlib-gnu
--compress-debug-sections=zlib-gabi
对于 ELF 文件，这些选项控制 DWARF 调试部分的压缩方式。 --compress-debug-sections=none相当于--decompress-debug-sections. --compress-debug-sections=zlib和 --compress-debug-sections=zlib-gabi相当于 --compress-debug-sections. --compress-debug-sections=zlib-gnu使用 zlib 压缩 DWARF 调试部分。调试部分被重命名为以'开头.zdebug' 代替 '.调试'。注意 - 如果压缩实际上会使一个部分变大，那么它不会被压缩也不会被重命名。

--decompress-debug-sections
使用 zlib 解压缩 DWARF 调试部分。恢复压缩节的原始节名。

--elf-stt-common=yes
--elf-stt-common=no
对于 ELF 文件，这些选项控制是否应将常用符号转换为STT_COMMONorSTT_OBJECT类型。 --elf-stt-common=是将通用符号类型转换为 STT_COMMON.--elf-stt-common=否将通用符号类型转换为STT_OBJECT.

--merge-notes
--no-merge-notes
对于 ELF 文件，尝试（或不尝试）通过删除重复的注释来减小任何 SHT_NOTE 类型部分的大小。

-V
--version
显示版本号objcopy。

--verilog-data-width=bytes
对于 Verilog 输出，此选项控制为每个输出数据元素转换的字节数。输入目标控制转换的字节顺序。

-v
--verbose
详细输出：列出所有修改的目标文件。在档案的情况下，'对象复制 -V' 列出档案的所有成员。

--help
显示选项的摘要objcopy。

--info
显示一个列表，显示所有可用的体系结构和对象格式。

objdump
^^^^^^^^^^^
objdump [-a|--archive-headers]
        [-b bfdname|--target=bfdname]
        [-C|--demangle[=style] ]
        [-d|--disassemble[=symbol]]
        [-D|--disassemble-all]
        [-z|--disassemble-zeroes]
        [-EB|-EL|--endian={big | little }]
        [-f|--file-headers]
        [-F|--file-offsets]
        [--file-start-context]
        [-g|--debugging]
        [-e|--debugging-tags]
        [-h|--section-headers|--headers]
        [-i|--info]
        [-j section|--section=section]
        [-l|--line-numbers]
        [-S|--source]
        [--source-comment[=text]]
        [-m machine|--architecture=machine]
        [-M options|--disassembler-options=options]
        [-p|--private-headers]
        [-P options|--private=options]
        [-r|--reloc]
        [-R|--dynamic-reloc]
        [-s|--full-contents]
        [-W[lLiaprmfFsoORtUuTgAck]|
         --dwarf[=rawline,=decodedline,=info,=abbrev,=pubnames,=aranges,=macro,=frames,=frames-interp,=str,=str-offsets,=loc,=Ranges,=pubtypes,=trace_info,=trace_abbrev,=trace_aranges,=gdb_index,=addr,=cu_index,=links]]
        [-WK|--dwarf=follow-links]
        [-WN|--dwarf=no-follow-links]
        [-L|--process-links]
        [--ctf=section]
        [-G|--stabs]
        [-t|--syms]
        [-T|--dynamic-syms]
        [-x|--all-headers]
        [-w|--wide]
        [--start-address=address]
        [--stop-address=address]
        [--no-addresses]
        [--prefix-addresses]
        [--[no-]show-raw-insn]
        [--adjust-vma=offset]
        [--dwarf-depth=n]
        [--dwarf-start=n]
        [--ctf-parent=section]
        [--no-recurse-limit|--recurse-limit]
        [--special-syms]
        [--prefix=prefix]
        [--prefix-strip=level]
        [--insn-width=width]
        [--visualize-jumps[=color|=extended-color|=off]
        [-U method] [--unicode=method]
        [-V|--version]
        [-H|--help]
        objfile…
        
objdump显示有关一个或多个目标文件的信息。这些选项控制要显示的特定信息。这些信息对使用编译工具的程序员最有用，而不是只希望他们的程序编译和工作的程序员。

objfile ... 是要检查的目标文件。当您指定档案时，objdump显示每个成员对象文件的信息。

在这里作为备选方案显示的多头和空头形式的期权是等价的。列表中的至少一个选项 -a,-d,-D,-e,-f,-g,-G,-h,-H,-p,-P,-r,-R,-s,-S,-t,-T ,-V,-x必须给出。

-a
--archive-header
如果任何objfile文件是档案，则显示档案头信息（格式类似于 'ls -l'）。除了您可以列出的信息 '艺术电视', 'objdump -a' 显示每个归档成员的目标文件格式。

--adjust-vma=offset
转储信息时，首先将偏移量添加到所有段地址。如果节地址与符号表不对应，这很有用，当使用不能表示节地址的格式（例如 a.out）将节放在特定地址时，可能会发生这种情况。

-b bfdname
--target=bfdname
指定目标文件的目标代码格式为 bfdname。此选项可能不是必需的；objdump可以自动识别多种格式。

例如，

objdump -b oasys -m vax -h fu.o
显示节标题中的摘要信息（-H） 的 fu.o，这是明确标识的（-m) 作为由 Oasys 编译器生成的格式的 VAX 对象文件。您可以列出可用的格式-一世选项。有关详细信息，请参阅目标选择。

-C
--demangle[=style]
将 ( demangle ) 低级符号名称解码为用户级名称。除了删除系统前面的任何初始下划线外，这使得 C++ 函数名称可读。不同的编译器有不同的修饰风格。可选的 demangling 样式参数可用于为您的编译器选择合适的 demangling 样式。参见c++filt，了解更多关于去重的信息。

--recurse-limit
--no-recurse-limit
--recursion-limit
--no-recursion-limit
启用或禁用在对字符串进行分解时执行的递归量的限制。由于名称修饰格式允许无限级别的递归，因此可以创建字符串，其解码将耗尽主机上可用的堆栈空间量，从而触发内存故障。该限制试图通过将递归限制为 2048 级嵌套来防止这种情况发生。

默认情况下启用此限制，但可能需要禁用它才能破解真正复杂的名称。但是请注意，如果禁用递归限制，则可能会耗尽堆栈，并且任何有关此类事件的错误报告都将被拒绝。

-g
--debugging
显示调试信息。这试图解析存储在文件中的 STABS 调试格式信息，并使用类似 C 的语法将其打印出来。如果未找到 STABS 调试，则此选项将退回到-W打印文件中任何 DWARF 信息的选项。

-e
--debugging-tags
喜欢-G, 但信息是以与 ctags 工具兼容的格式生成的。

-d
--disassemble
--disassemble=symbol
显示输入文件中机器指令的汇编程序助记符。此选项仅反汇编那些预期包含指令的部分。如果给出了可选的symbol 参数，则显示从 symbol开始的汇编助记符。如果symbol是一个函数名，那么反汇编将在函数的末尾停止，否则它将在遇到下一个符号时停止。如果符号不匹配， 则不会显示任何内容。

请注意，如果--dwarf=关注链接选项被启用，那么链接调试信息文件中的任何符号表都将被读取并在反汇编时使用。

-D
--disassemble-all
喜欢-d，但反汇编所有部分的内容，而不仅仅是那些预期包含指令的部分。

此选项对代码段中指令的反汇编也有微妙的影响。选项时-d实际上，objdump 将假定代码部分中存在的任何符号都出现在指令之间的边界上，并且它将拒绝跨过这样的边界进行反汇编。选项时-D是有效的，但是这个假设被压制了。这意味着可以输出-d和-D例如，如果数据存储在代码段中，则不同。

如果目标是 ARM 体系结构，则此开关还具有强制反汇编器对代码段中的数据片段进行解码的效果，就好像它们是指令一样。

请注意，如果--dwarf=关注链接选项被启用，那么链接调试信息文件中的任何符号表都将被读取并在反汇编时使用。

--no-addresses
反汇编时，不要在每一行或符号和重定位偏移上打印地址。结合--no-show-raw-insn 这对于比较编译器输出可能很有用。

--prefix-addresses
反汇编时，在每一行打印完整的地址。这是较旧的反汇编格式。

-EB
-EL
--endian={big|little}
指定目标文件的字节顺序。这只影响拆卸。这在反汇编不描述字节顺序信息的文件格式时很有用，例如 S 记录。

-f
--file-headers
显示来自每个objfile文件 的整体标题的摘要信息。

-F
--file-offsets
反汇编节时，每当显示符号时，也会显示即将转储的数据区域的文件偏移量。如果零被跳过，那么当反汇编恢复时，告诉用户有多少个零被跳过以及反汇编恢复位置的文件偏移量。转储部分时，显示转储开始位置的文件偏移量。

--file-start-context
指定在显示 interlisted 源代码/反汇编时（假设-S) 从尚未显示的文件中，将上下文扩展到文件的开头。

-h
--section-headers
--headers
显示来自目标文件的节标题的摘要信息。

文件段可能被重定位到非标准地址，例如通过使用-文本,-Tdata， 要么-Tbss选项 ld。但是，某些目标文件格式，例如 a.out，不存储文件段的起始地址。在这些情况下，虽然ld正确地重新定位了这些部分，但使用 '对象转储 -h' 列出文件节标题无法显示正确的地址。相反，它显示了对目标隐含的常用地址。

请注意，在某些情况下，一个部分可以同时设置 READONLY 和 NOREAD 属性。在这种情况下，NOREAD 属性优先，但objdump会报告两者，因为标志位的确切设置可能很重要。

-H
--help
打印选项摘要objdump并退出。

-i
--info
显示一个列表，显示所有可用于规范的架构和对象格式-b要么-m.

-j name
--section=name
仅显示部分名称的信息。

-L
--process-links
显示在链接到主文件的单独 debuginfo 文件中找到的非调试部分的内容。此选项自动暗示-WK选项，并且仅显示其他命令行选项请求的部分。

-l
--line-numbers
使用与显示的目标代码或重定位相对应的文件名和源代码行号标记显示（使用调试信息）。仅适用于-d,-D， 要么-r.

-m machine
--architecture=machine
指定反汇编目标文件时要使用的体系结构。这在反汇编不描述架构信息的目标文件时很有用，例如 S 记录。您可以列出可用的架构-一世选项。

如果目标是 ARM 架构，则此开关具有附加效果。它将反汇编限制为仅由machine指定的体系结构支持的那些指令。如果由于输入文件不包含任何架构信息而需要使用此开关，但还希望反汇编所有使用的指令-马尔姆.

-M options
--disassembler-options=options
将目标特定信息传递给反汇编程序。仅在某些目标上支持。如果需要指定多个反汇编程序选项，则需要指定多个-M选项可以使用，也可以一起放入逗号分隔的列表中。

对于弧，数字信号处理器控制DSP指令的打印， spfp选择打印FPX单精度FP指令，dpfp选择打印FPX双精度FP指令，quarkse_em选择特殊 QuarkSE-EM 指令的打印，福普达选择打印双精度辅助指令，fpus选择打印 FPU 单精度 FP 指令，而fpud 选择打印 FPU 双精度 FP 指令。此外，可以选择使用十六进制打印所有立即数十六进制. 默认情况下，短立即数使用十进制表示形式打印，而长立即数值则以十六进制形式打印。

中央处理器=...允许在反汇编指令时强制执行特定的 ISA，覆盖-m值或 ELF 文件中的任何内容。这对于选择 ARC EM 或 HS ISA 可能很有用，因为它们的架构是相同的，并且反汇编程序依赖于私有 ELF 标头数据来决定代码是用于 EM 还是 HS。此选项可能会被指定多次 - 只会使用最新的值。有效值与汇编程序相同 -mcpu=...选项。

如果目标是 ARM 体系结构，则此开关可用于选择在反汇编程序期间使用哪个寄存器名称集。指定 -M reg-names-std（默认）将选择 ARM 指令集文档中使用的寄存器名称，但寄存器 13 称为“sp”，寄存器 14 称为“lr”，寄存器 15 称为“pc”。指定 -M 注册名称-apcs将选择 ARM 过程调用标准使用的名称集，同时指定-M reg-names-raw只会使用'r' 后跟注册号。

APCS 寄存器命名方案也有两种变体，由-M reg-names-atpcs和-M reg-names-special-atpcs它使用 ARM/Thumb 过程调用标准命名约定。（使用普通寄存器名称或特殊寄存器名称）。

此选项还可用于 ARM 架构，通过使用开关强制反汇编器将所有指令解释为 Thumb 指令--disassembler-options=force-thumb. 这在尝试反汇编其他编译器生成的拇指代码时很有用。

对于 AArch64 目标，此开关可用于设置是否将指令反汇编为最通用的指令，使用-M 无别名 选项或是否应生成说明注释作为反汇编中的注释使用-M 笔记.

对于 x86，一些选项与-m 开关，但允许更细粒度的控制。

x86-64
i386
i8086
为给定的架构选择反汇编。

intel
att
在 intel 语法模式和 AT&T 语法模式之间进行选择。

amd64
intel64
在 AMD64 ISA 和 Intel64 ISA 之间进行选择。

intel-mnemonic
att-mnemonic
在 intel 助记符模式和 AT&T 助记符模式之间进行选择。注：intel-mnemonic暗示intel和 att-mnemonic暗示att。

addr64
addr32
addr16
data32
data16
指定默认地址大小和操作数大小。如果或 稍后出现在选项字符串中 x86-64，这五个选项将被覆盖。i386i8086

suffix
在 AT&T 模式下以及在 Intel 模式下对于一组有限的指令，指示反汇编程序打印助记符后缀，即使后缀可以由操作数推断，或者对于某些指令，执行模式的默认值也是如此。

对于 PowerPC，-M争论生的选择硬件insns而不是别名的拆卸。例如，您将看到rlwinm而不是clrlwi，addi 而不是li。全部-mgas支持选择 CPU 的参数 。这些是： 403,405,440,464,476, 601,603,604,620,7400, 7410,7450,7455,750cl, 821,850,860,a2,书本, booke32,细胞,com,e200z4, e300,e500,e500mc,e500mc64, e500x2,e5500,e6500,efs, 电源4,电源5,电源6,电源7, 电源8,电源9,电源10,ppc, ppc32,ppc64,ppc64bridge,ppcps, 压水机,pwr2,pwr4,pwr5,pwr5x, pwr6,pwr7,pwr8,pwr9,pwr10, pwrx,泰坦， 和虚拟机. 32和64修改默认或先前的 CPU 选择，分别禁用和启用 64 位 insns。此外，阿尔替韦,任何,htm,vsx， 和专业为之前或之后的CPU 选择添加功能。 任何将反汇编 binutils 已知的任何操作码，但在​​操作码具有两种不同含义或不同参数的情况下，您可能看不到您期望的反汇编。如果您在没有选择 CPU 的情况下进行反汇编，则会从 BFD 从目标文件头中收集的信息中选择默认值，但结果可能与您预期的不同。

对于 MIPS，此选项控制反汇编指令中指令助记符名称和寄存器名称的打印。可以将以下多项选择指定为逗号分隔的字符串，并忽略无效选项：

no-aliases
打印“原始”指令助记符而不是一些伪指令助记符。即，打印'daddu'或'or'而不是'move'，'sll'而不是'nop'等。

msa
反汇编 MSA 指令。

virt
反汇编虚拟化 ASE 指令。

xpa
反汇编扩展物理地址 (XPA) ASE 指令。

gpr-names=ABI
打印适用于指定 ABI 的 GPR（通用寄存器）名称。默认情况下，根据被反汇编的二进制文件的 ABI 选择 GPR 名称。

fpr-names=ABI
根据指定的 ABI 打印 FPR（浮点寄存器）名称。默认情况下，打印 FPR 编号而不是名称。

cp0-names=ARCH
根据ARCH指定的 CPU 或体系结构打印 CP0（系统控制协处理器；协处理器 0）寄存器名称 。默认情况下，CP0 寄存器名称是根据被反汇编的二进制文件的体系结构和 CPU 选择的。

hwr-names=ARCH
根据ARCHrdhwr指定的 CPU 或体系结构 打印 HWR（指令使用的硬件寄存器）名称。默认情况下，根据要反汇编的二进制文件的体系结构和 CPU 选择 HWR 名称。

reg-names=ABI
打印适合所选 ABI 的 GPR 和 FPR 名称。

reg-names=ARCH
根据所选 CPU 或体系结构打印特定于 CPU 的寄存器名称（CP0 寄存器和 HWR 名称）。

对于上面列出的任何选项，ABI或 ARCH可以指定为 '数字' 为所选类型的寄存器打印数字而不是名称。您可以使用列出ABI和ARCH的可用值- 帮助选项。

对于 VAX，您可以指定函数入口地址-M 条目：0xf00ba. 您可以多次使用它来正确反汇编不包含符号表（如 ROM 转储）的 VAX 二进制文件。在这些情况下，函数入口掩码将被解码为 VAX 指令，这可能会导致函数的其余部分被错误地反汇编。

-p
--private-headers
打印特定于目标文件格式的信息。打印的确切信息取决于目标文件格式。对于某些目标文件格式，不会打印任何附加信息。

-P options
--private=options
打印特定于目标文件格式的信息。参数选项是一个逗号分隔的列表，取决于格式（选项列表与帮助一起显示）。

对于 XCOFF，可用的选项有：

header
aout
sections
syms
relocs
lineno,
loader
except
typchk
traceback
toc
ldinfo
并非所有对象格式都支持此选项。特别是 ELF 格式不使用它。

-r
--reloc
打印文件的重定位条目。如果与-d要么 -D，重定位打印穿插拆卸。

-R
--dynamic-reloc
打印文件的动态重定位条目。这仅对动态对象有意义，例如某些类型的共享库。至于-r, 如果与-d要么 -D，重定位打印穿插拆卸。

-s
--full-contents
显示请求的任何部分的全部内容。默认情况下会显示所有非空部分。

-S
--source
如果可能，显示与反汇编混合的源代码。暗示 -d.

--source-comment[=txt]
如-S选项，但所有源代码行都以txt前缀显示。通常txt将是一个注释字符串，可用于区分汇编代码和源代码。如果未提供 txt ，则将使用默认字符串“#”（哈希后跟一个空格）。

--prefix=prefix
指定前缀以在使用时添加到绝对路径 -S.

--prefix-strip=level
指示要从硬连线的绝对路径中剥离多少个初始目录名称。没有就没有效果--前缀=前缀。

--show-raw-insn
反汇编指令时，以十六进制和符号形式打印指令。这是默认设置，除非 --前缀地址用来。

--no-show-raw-insn
反汇编指令时，不要打印指令字节。这是默认情况下--前缀地址用来。

--insn-width=width
反汇编指令时在单行 显示宽度字节。

--visualize-jumps[=color|=extended-color|=off]
通过在起始地址和目标地址之间绘制 ASCII 艺术来可视化保留在函数内部的跳转。可选的=颜色参数使用简单的终端颜色为输出添加颜色。或者=扩展颜色参数将使用 8 位颜色添加颜色，但这些可能不适用于所有终端。

如果需要禁用可视化跳跃之前启用后的选项然后使用 可视化跳跃=关闭.

-W[lLiaprmfFsoORtUuTgAckK]
--dwarf[=rawline,=decodedline,=info,=abbrev,=pubnames,=aranges,=macro,=frames,=frames-interp,=str,=str-offsets,=loc,=Ranges,=pubtypes,=trace_info,=trace_abbrev,=trace_aranges,=gdb_index,=addr,=cu_index,=links,=follow-links]
显示文件中 DWARF 调试部分的内容（如果存在）。压缩的调试部分在显示之前会自动（临时）解压缩。如果开关后面有一个或多个可选字母或单词，则只会转储那些类型的数据。字母和单词指的是以下信息：

a
=abbrev
显示'的内容.debug_abbrev' 部分。

A
=addr
显示'的内容.debug_addr' 部分。

c
=cu_index
显示'的内容.debug_cu_index'和/或'.debug_tu_index' 部分。

f
=frames
显示 ' 的原始内容.debug_frame' 部分。

F
=frames-interp
显示 ' 的解释内容.debug_frame' 部分。

g
=gdb_index
显示'的内容.gdb_index'和/或'.debug_names' 部分。

i
=info
显示'的内容。调试信息' 部分。注意：此选项的输出也可以通过使用 --矮人深度和--矮人开始选项。

k
=links
显示'的内容.gnu_debuglink', '.gnu_debugaltlink' 和 '.debug_sup' 部分，如果有的话。还显示指向单独的 dwarf 对象文件 (dwo) 的任何链接，如果它们由 '。调试信息' 部分。

K
=follow-links
显示在链接的单独调试信息文件中找到的任何选定调试部分的内容。如果同一调试部分存在于多个文件中，这可能会导致显示多个版本。

另外，在显示 DWARF 属性时，如果发现表单引用了单独的调试信息文件，那么也会显示引用的内容。

注意 - 在某些发行版中，此选项默认启用。它可以通过禁用ñ调试选项。通过配置 binutils 时可以选择默认值 --enable-follow-debug-links=yes要么 --enable-follow-debug-links=no选项。如果不使用这些，则默认启用以下调试链接。

N
=no-follow-links
禁用以下链接到单独的调试信息文件。

l
=rawline
显示'的内容.debug_line' 原始格式的部分。

L
=decodedline
显示 ' 的解释内容.debug_line' 部分。

m
=macro
显示'的内容.debug_macro'和/或'.debug_macinfo' 部分。

o
=loc
显示'的内容.debug_loc'和/或'.debug_loclists' 部分。

O
=str-offsets
显示'的内容.debug_str_offsets' 部分。

p
=pubnames
显示'的内容.debug_pubnames'和/或'.debug_gnu_pubnames' 部分。

r
=aranges
显示'的内容.debug_aranges' 部分。

R
=Ranges
显示'的内容.debug_ranges'和/或'.debug_rnglists' 部分。

s
=str
显示'的内容.debug_str', '.debug_line_str'和/或'.debug_str_offsets' 部分。

t
=pubtype
显示'的内容.debug_pubtypes'和/或'.debug_gnu_pubtypes' 部分。

T
=trace_aranges
显示'的内容.trace_aranges' 部分。

u
=trace_abbrev
显示'的内容.trace_abbrev' 部分。

U
=trace_info
显示'的内容.trace_info' 部分。

注意：显示 ' 的内容.debug_static_funcs', '.debug_static_vars' 和 'debug_weaknames' 部分当前不受支持。

--dwarf-depth=n
.debug_info将部分的转储限制为n个子项。这仅适用于--调试转储=信息. 默认打印所有DIE；n的特殊值 0也会产生这种效果。

如果n的值不为零， 则不会打印n级或更深的 DIE 。n的范围是从零开始的。

--dwarf-start=n
仅打印以编号为n的 DIE 开头的 DIE 。这仅适用于--调试转储=信息.

如果指定，此选项将禁止打印任何标头信息和编号为n的 DIE 之前的所有 DIE 。只会打印指定 DIE 的兄弟姐妹和孩子。

这可以与--矮人深度.

--dwarf-check
启用额外检查以确保 Dwarf 信息的一致性。

--ctf[=section]
显示指定 CTF 节的内容。CTF 部分本身包含许多子部分，所有子部分都按顺序显示。

默认情况下，显示名为.ctf的部分的名称，即 .ctf 发出的名称ld。

--ctf-parent=member
如果 CTF 部分包含模糊定义的类型，它将包含许多 CTF 字典的存档，所有字典都继承自一个包含明确类型的字典。此成员默认命名为.ctf，就像包含它的部分一样，但可以ctf_link_set_memb_name_changer 在链接时使用该函数更改此名称。查看由使用名称更改器重命名父存档成员的链接器创建的 CTF 存档时，--ctf-父级可用于指定用于父级的名称。

-G
--stabs
显示请求的任何部分的全部内容。显示 ELF 文件中 .stab 和 .stab.index 和 .stab.excl 部分的内容。.stab这仅在调试符号表条目在 ELF 部分中携带的系统（例如 Solaris 2.0）上有用 。在大多数其他文件格式中，调试符号表条目与链接符号交错，并且在--syms 输出。

--start-address=address
开始显示指定地址的数据。这会影响输出-d,-r和-s选项。

--stop-address=address
停止在指定地址显示数据。这会影响输出-d,-r和-s选项。

-t
--syms
打印文件的符号表条目。这类似于'纳米' 程序，虽然显示格式不同。输出的格式取决于转储文件的格式，但主要有两种类型。一个看起来像这样：

[ 4](sec 3)(fl 0x00)(ty 0)(scl 3) (nx 1) 0x00000000 .bss
[ 6](秒 1)(fl 0x00)(ty 0)(scl 2) (nx 0) 0x00000000 弗雷德
其中方括号内的数字是符号表中条目的编号，sec编号是节编号， fl值是符号的标志位，ty编号是符号的类型，scl编号是符号的存储类和nx值是与符号关联的辅助条目的数量。最后两个字段是符号的值及其名称。

另一种常见的输出格式，通常用于基于 ELF 的文件，如下所示：

00000000 ld .bss 00000000 .bss
00000000 g .text 00000000 弗雷德
这里第一个数字是符号的值（有时称为它的地址）。下一个字段实际上是一组字符和空格，表示在符号上设置的标志位。这些字符如下所述。接下来是与符号关联的部分，如果该部分是绝对的（即不与任何部分连接），则为*ABS* ，如果在转储文件中引用了该部分，但未在此处定义，则为 *UND* 。

在节名之后是另一个字段，一个数字，对于常用符号来说是对齐方式，对于其他符号来说是大小。最后显示符号的名称。

标志字符分为 7 组，如下所示：

l
g
u
!
符号是局部 (l)、全局 (g)、唯一全局 (u)、既不是全局也不是局部（空格）或既全局又局部 (!)。由于各种原因，一个符号既不能是本地的也不能是全局的，例如，因为它用于调试，但如果它既是本地的又是全局的，它可能是一个错误的指示。唯一的全局符号是对标准 ELF 符号绑定集的 GNU 扩展。对于这样的符号，动态链接器将确保在整个过程中只有一个具有此名称和类型的符号在使用中。

w
符号是弱（w）或强（空格）。

C
符号表示构造函数（C）或普通符号（空格）。

W
该符号是警告 (W) 或正常符号（空格）。警告符号的名称是在警告符号后面的符号被引用时显示的消息。

I
i
该符号是对另一个符号 (I) 的间接引用，是在 reloc 处理期间要评估的函数 (i) 或普通符号（空格）。

d
D
该符号是调试符号 (d) 或动态符号 (D) 或普通符号（空格）。

F
f
O
符号是函数 (F) 或文件 (f) 或对象 (O) 的名称，或者只是普通符号（空格）。

-T
--dynamic-syms
打印文件的动态符号表条目。这仅对动态对象有意义，例如某些类型的共享库。这类似于'纳米' 给定程序时-D(- 动态的） 选项。

输出格式类似于--syms 选项，除了在符号名称之前插入一个额外字段，提供与符号关联的版本信息。如果版本是解析对符号的无版本引用时使用的默认版本，则按原样显示，否则将其放入括号中。

--special-syms
当显示符号时，包括目标认为在某些方面是特殊的并且用户通常不会感兴趣的符号。

-U [d|i|l|e|x|h]
--unicode=[default|invalid|locale|escape|hex|highlight]
控制字符串中 UTF-8 编码的多字节字符的显示。默认（--unicode=默认) 就是不给他们特殊待遇。这--unicode=语言环境选项显示当前语言环境中的序列，可能支持也可能不支持。选项 --unicode=十六进制和--unicode=无效将它们显示为用尖括号或花括号括起来的十六进制字节序列。

这--unicode=转义选项将它们显示为转义序列（\uxxxx）和--unicode=高亮选项将它们显示为以红色突出显示的转义序列（如果输出设备支持）。着色旨在引起人们对可能不期望出现的 unicode 序列的关注。

-V
--version
打印版本号objdump并退出。

-x
--all-headers
显示所有可用的头信息，包括符号表和重定位条目。使用-X相当于指定所有 -a -f -h -p -r -t.

-w
--wide
为超过 80 列的输出设备格式化一些行。也不要在显示符号名称时截断它们。

-z
--disassemble-zeroes
通常，反汇编输出将跳过零块。该选项指示反汇编器反汇编这些块，就像任何其他数据一样。

ranlib
^^^^^^^^
ranlib [--plugin name] [-DhHvVt] archive

ranlib生成档案内容的索引并将其存储在档案中。该索引列出了由作为可重定位目标文件的存档成员定义的每个符号。

你可以使用'nm -s' 要么 'nm --print-armap' 列出这个索引。

具有这种索引的存档加速了与库的链接，并允许库中的例程相互调用，而无需考虑它们在存档中的位置。

GNU ranlib程序是 GNU 的另一种 形式ar；running ranlib完全等同于执行 'ar -s'。见ar。

-h
-H
--help
显示 的使用信息ranlib。

-v
-V
--version
显示版本号ranlib。

-D
以确定性模式运行。符号映射存档成员的标题将显示 UID、GID 和时间戳为零。使用此选项时，多次运行将产生相同的输出文件。

如果二进制工具配置了 --enable-deterministic-archives, 则默认开启此模式。它可以用'禁用-U' 选项，如下所述。

-t
更新档案符号图的时间戳。

-U
不要在确定性模式下运行。与上面的-D选项意义相反：存档索引将获取实际的 UID、GID、时间戳和文件模式值。

如果二进制工具没有配置 --enable-deterministic-archives, 则默认开启此模式。
 
size
^^^^^^^
size [-A|-B|-G|--format=compatibility]
     [--help]
     [-d|-o|-x|--radix=number]
     [--common]
     [-t|--totals]
     [--target=bfdname] [-V|--version]
     [objfile…] 
 
 GNU 实用程序在其参数列表中列出每个二进制文件objfile的size部分大小和总大小。默认情况下，如果文件是存档，则为每个文件或每个模块生成一行输出。

objfile … 是要检查的文件。如果未指定，a.out则将使用该文件。

命令行选项具有以下含义：

-A
-B
-G
--format=compatibility
使用这些选项之一，您可以选择GNU size的输出是否类似于 System V的输出size（使用-一种， 要么--format=sysv），或伯克利size（使用-B， 要么 --格式=伯克利）。默认是类似于 Berkeley 的单行格式。或者，您可以选择 GNU 格式输出（使用-G， 要么--格式=gnu)，这类似于 Berkeley 的输出格式，但大小的计算方式不同。

以下是来自的输出的伯克利（默认）格式示例 size：

$ size --format=伯克利ranlib大小
   文本数据 bss dec 十六进制文件名
 294880 81920 11592 388392 5ed28
 294880 81920 11888 388688 5ee50尺寸
Berkeley 样式输出只计算列中的数据text ，而不是data列中的数据，dec和列都分别以十进制和十六进制 显示、和 hex 列的总和。textdatabss

GNU 格式只计算列中的只读数据data，而不是列中的数据，并且只在列中显示、 和列text的总和一次。这textdatabsstotal--基数选项可用于更改所有列的数字基数。以下是使用 GNU 约定显示的相同数据：

$ 大小 --format=GNU ranlib 大小
      文本数据 bss 总文件名
    279880 96920 11592 388392
    279880 96920 11888 388688 尺寸
这是相同的数据，但显示更接近 System V 约定：

$ size --format=SysV ranlib 大小
运行库：
截面尺寸地址
.text 294880 8192
.data 81920 303104
.bss 11592 385024
总计 388392


尺寸 ：
截面尺寸地址
.text 294880 8192
.data 81920 303104
.bss 11888 385024
总计 388688
--help
显示可接受的论点和选项的摘要。

-d
-o
-x
--radix=number
使用这些选项之一，您可以控制每个部分的大小是否以十进制 (-d， 要么--radix=10); 八进制 (-o， 要么--radix=8); 或十六进制 (-X， 要么 --radix=16）。在--radix=数字，仅支持三个值（8、10、16）。总大小总是以两个基数给出；十进制和十六进制-d要么-X输出，或者如果您使用的是八进制和十六进制-o.

--common
打印每个文件中常见符号的总大小。当使用 Berkeley 或 GNU 格式时，这些都包含在 bss 大小中。

-t
--totals
显示列出的所有对象的总数（仅限伯克利或 GNU 格式模式）。

--target=bfdname
指定objfile的目标代码格式为 bfdname。此选项可能不是必需的；size可以自动识别多种格式。有关详细信息，请参阅目标选择。

-V
--version
显示版本号size。


strings
^^^^^^^^^
 
 strings [-afovV] [-min-len]
        [-n min-len] [--bytes=min-len]
        [-t radix] [--radix=radix]
        [-e encoding] [--encoding=encoding]
        [-U method] [--unicode=method]
        [-] [--all] [--print-file-name]
        [-T bfdname] [--target=bfdname]
        [-w] [--include-all-whitespace]
        [-s] [--output-separator sep_string]
        [--help] [--version] file…
 
 对于给定的每个文件，GNU strings打印至少 4 个字符长的可打印字符序列（或下面的选项给出的数字），然后是不可打印的字符。

根据字符串程序的配置方式，它将默认显示它可以在每个文件中找到的所有可打印序列，或者仅显示可加载的初始化数据部分中的那些序列。如果文件类型无法识别，或者如果字符串正在从标准输入读取，那么它将始终显示它可以找到的所有可打印序列。

为了向后兼容，任何出现在命令行选项之后的文件-也将被完整扫描，无论是否存在任何-d选项。

strings主要用于确定非文本文件的内容。

-a
--all
-
扫描整个文件，无论它包含哪些部分或这些部分是否已加载或初始化。通常这是默认行为，但可以配置字符串，以便 -d是默认值。

这-选项取决于位置，并强制字符串对后面提到的任何文件执行完整扫描- 在命令行上，即使-d已指定选项。

-d
--data
仅打印文件中已初始化、加载的数据部分的字符串。这可能会减少输出中的垃圾量，但也会使字符串程序暴露于用于扫描和加载节的 BFD 库中可能存在的任何安全漏洞。可以配置字符串，以便此选项是默认行为。在这种情况下-一种选项可用于避免使用 BFD 库，而只打印文件中找到的所有字符串。

-f
--print-file-name
在每个字符串之前打印文件名。

--help
在标准输出上打印程序使用摘要并退出。

-min-len
-n min-len
--bytes=min-len
打印长度至少为 min-len 个字符的可显示字符序列。如果未指定，则使用默认的最小长度 4。可显示字符和不可显示字符之间的区别取决于 -e和-U选项。序列总是以控制字符终止，例如换行符和回车符，而不是制表符。

-o
喜欢 '-到'。其他一些版本的strings有-o 表现得像'-td' 反而。由于我们无法同时兼容这两种方式，因此我们只是选择了一种。

-t radix
--radix=radix
在每个字符串之前打印文件中的偏移量。单个字符参数指定偏移的基数——'○'对于八进制，'X' 十六进制，或 'd' 代表十进制。

-e encoding
--encoding=encoding
选择要查找的字符串的字符编码。可能的编码值为：'s' = 单 7 位字节字符（ASCII、ISO 8859 等，默认），'小号' = 单 8 位字节字符，'b' = 16 位大端序，'l' = 16 位小端序，'乙' = 32 位双端序，'大号' = 32 位小端。用于查找宽字符串。('l' 和 'b' 适用于例如 Unicode UTF-16/UCS-2 编码）。

-U [d|i|l|e|x|h]
--unicode=[default|invalid|locale|escape|hex|highlight]
控制字符串中 UTF-8 编码的多字节字符的显示。默认（--unicode=默认) 是不给他们特殊待遇，而是依赖于 --编码选项。此选项的其他值自动启用--编码=S.

这--unicode=无效选项将它们视为非图形字符，因此不是有效字符串的一部分。所有剩余的选项都将它们视为有效的字符串字符。

这--unicode=语言环境选项在当前语言环境中显示它们，可能支持也可能不支持 UTF-8 编码。这 --unicode=十六进制选项将它们显示为包含在<>字符之间的十六进制字节序列。这--unicode=转义 选项将它们显示为转义序列（\uxxxx）和 --unicode=高亮选项将它们显示为以红色突出显示的转义序列（如果输出设备支持）。着色旨在引起人们对可能不期望出现的 unicode 序列的关注。

-T bfdname
--target=bfdname
指定系统默认格式以外的目标代码格式。有关详细信息，请参阅目标选择。

-v
-V
--version
在标准输出上打印程序版本号并退出。

-w
--include-all-whitespace
默认情况下，显示的字符串中包含制表符和空格字符，但不包含其他空白字符，例如换行符和回车符。这-w选项对此进行了更改，以便所有空白字符都被视为字符串的一部分。

-s
--output-separator
默认情况下，输出字符串由换行符分隔。此选项允许您提供要用作输出记录分隔符的任何字符串。与 –include-all-whitespace 一起使用，其中字符串可能在内部包含换行符。
 
 
 strip
 ^^^^^^^
 strip [-F bfdname |--target=bfdname]
      [-I bfdname |--input-target=bfdname]
      [-O bfdname |--output-target=bfdname]
      [-s|--strip-all]
      [-S|-g|-d|--strip-debug]
      [--strip-dwo]
      [-K symbolname|--keep-symbol=symbolname]
      [-M|--merge-notes][--no-merge-notes]
      [-N symbolname |--strip-symbol=symbolname]
      [-w|--wildcard]
      [-x|--discard-all] [-X |--discard-locals]
      [-R sectionname |--remove-section=sectionname]
      [--keep-section=sectionpattern]
      [--remove-relocations=sectionpattern]
      [-o file] [-p|--preserve-dates]
      [-D|--enable-deterministic-archives]
      [-U|--disable-deterministic-archives]
      [--keep-section-symbols]
      [--keep-file-symbols]
      [--only-keep-debug]
      [-v |--verbose] [-V|--version]
      [--help] [--info]
      objfile…


GNU strip丢弃目标文件 objfile中的所有符号。目标文件列表可能包括档案。必须至少给出一个目标文件。

strip修改在其参数中命名的文件，而不是以不同的名称写入修改后的副本。

-F bfdname
--target=bfdname
将原始objfile视为具有目标代码格式bfdname的文件，并以相同格式重写它。有关详细信息，请参阅目标选择。

--help
显示选项摘要strip并退出。

--info
显示一个列表，显示所有可用的体系结构和对象格式。

-I bfdname
--input-target=bfdname
将原始objfile视为具有目标代码格式bfdname的文件。有关详细信息，请参阅目标选择。

-O bfdname
--output-target=bfdname
将objfile替换为输出格式为bfdname的文件。有关详细信息，请参阅目标选择。

-R sectionname
--remove-section=sectionname
从输出文件中删除任何名为sectionname的部分，除了将被删除的任何部分。这个选项可能不止一次给出。请注意，不恰当地使用此选项可能会使输出文件无法使用。通配符 '*' 可以在sectionname的末尾给出。如果是这样，那么任何以sectionname开头的部分都将被删除。

如果sectionpattern的第一个字符是感叹号 (!)，那么即使之前使用--删除部分在同一命令行上否则会删除它。例如：

  --remove-section=.text.* --remove-section=!.text.foo
将删除与模式“.text.*”匹配的所有部分，但不会删除“.text.foo”部分。

--keep-section=sectionpattern
从输出文件中删除部分时，保留与 sectionpattern匹配的部分。

--remove-relocations=sectionpattern
从输出文件中删除与 sectionpattern匹配的任何部分的重定位。这个选项可能不止一次给出。请注意，不恰当地使用此选项可能会使输出文件无法使用。sectionpattern中接受通配符。例如：

  --remove-relocations=.text.*
将删除与模式“.text.*”匹配的所有部分的重定位。

如果sectionpattern的第一个字符是感叹号 (!)，那么即使之前使用--remove-relocations在同一命令行上，否则会导致重定位被删除。例如：

  --remove-relocations=.text.* --remove-relocations=!.text.foo
将删除与模式“.text.*”匹配的部分的所有重定位，但不会删除“.text.foo”部分的重定位。

-s
--strip-all
删除所有符号。

-g
-S
-d
--strip-debug
仅删除调试符号。

--strip-dwo
删除所有 DWARF .dwo 部分的内容，保留剩余的调试部分和所有符号不变。有关详细信息，请参阅本节中对该选项的说明objcopy。

--strip-unneeded
除了调试符号和由 --strip-debug.

-K symbolname
--keep-symbol=symbolname
剥离符号时，请保留符号symbolname ，即使它通常会被剥离。这个选项可能不止一次给出。

-M
--merge-notes
--no-merge-notes
对于 ELF 文件，尝试（或不尝试）通过删除重复的注释来减小任何 SHT_NOTE 类型部分的大小。默认是尝试这种减少，除非剥离调试或 DWO 信息。

-N symbolname
--strip-symbol=symbolname
从源文件中删除符号symbolname 。该选项可以多次给出，并且可以与除 -K.

-o file
将剥离的输出放在file中，而不是替换现有文件。使用此参数时，只能指定一个objfile 参数。

-p
--preserve-dates
保留文件的访问和修改日期。

-D
--enable-deterministic-archives
以确定性模式运行。复制存档成员和写入存档索引时，对 UID、GID、时间戳使用零，并对所有文件使用一致的文件模式。

如果二进制工具配置了 --enable-deterministic-archives, 则默认开启此模式。它可以用'禁用-U' 选项，如下。

-U
--disable-deterministic-archives
不要在确定性模式下运行。这是相反的-D上面的选项：复制存档成员和写入存档索引时，使用它们的实际 UID、GID、时间戳和文件模式值。

这是默认设置，除非二进制工具配置了 --enable-deterministic-archives.

-w
--wildcard
允许在其他命令行选项中使用的symbolname中的正则表达式。问号 (?)、星号 (*)、反斜杠 (\) 和方括号 ([]) 运算符可用于符号名称中的任何位置。如果符号名称的第一个字符是感叹号 (!)，则该符号的开关意义相反。例如：

  -w -K !foo -K fo*
将导致 strip 仅保留以字母“fo”开头的符号，但丢弃符号“foo”。

-x
--discard-all
删除非全局符号。

-X
--discard-locals
删除编译器生成的本地符号。（这些通常以'开头大号' 要么 '.'。）

--keep-section-symbols
剥离文件时，也许与--strip-debug要么 --strip-不需要, 保留任何指定节名称的符号，否则会被剥离。

--keep-file-symbols
剥离文件时，也许与--strip-debug要么 --strip-不需要, 保留任何指定源文件名的符号，否则会被剥离。

--only-keep-debug
剥离文件，清空任何不会被剥离的部分的内容--strip-debug并保持调试部分完好无损。在 ELF 文件中，这也会保留输出中的所有注释部分。

注意 - 被剥离部分的部分标题被保留，包括它们的大小，但部分的内容被丢弃。保留部分标题，以便其他工具可以将 debuginfo 文件与实际可执行文件匹配，即使该可执行文件已被重定位到不同的地址空间。

目的是将此选项与 --add-gnu-debuglink创建一个两部分的可执行文件。一个是剥离的二进制文件，它将在 RAM 和分发中占用更少的空间，第二个是调试信息文件，仅在需要调试能力时才需要。创建这些文件的建议过程如下：

正常链接可执行文件。假设它被调用 foo然后......
运行objcopy --only-keep-debug foo foo.dbg以创建包含调试信息的文件。
运行objcopy --strip-debug foo以创建剥离的可执行文件。
运行objcopy --add-gnu-debuglink=foo.dbg foo 以将调试信息的链接添加到剥离的可执行文件中。
注意——选择.dbg作为调试信息文件的扩展名是任意的。该--only-keep-debug步骤也是可选的。你可以这样做：

正常链接可执行文件。
复制foo到foo.full
跑步strip --strip-debug foo
跑步objcopy --add-gnu-debuglink=foo.full foo
即，指向的文件--add-gnu-debuglink可以是完整的可执行文件。它不必是由 --only-keep-debug转变。

注意—此开关仅适用于完全链接的文件。在调试信息可能不完整的目标文件上使用它是没有意义的。除了 gnu_debuglink 功能目前只支持一个包含调试信息的文件名，而不是基于一个对象文件的多个文件名。

-V
--version
显示 的版本号strip。

-v
--verbose
详细输出：列出所有修改的目标文件。在档案的情况下，'剥离 -v' 列出档案的所有成员。
 
addr2line
^^^^^^^^^^
addr2line [-a|--addresses]
          [-b bfdname|--target=bfdname]
          [-C|--demangle[=style]]
          [-r|--no-recurse-limit]
          [-R|--recurse-limit]
          [-e filename|--exe=filename]
          [-f|--functions] [-s|--basename]
          [-i|--inlines]
          [-p|--pretty-print]
          [-j|--section=name]
          [-H|--help] [-V|--version]
          [addr addr …]
 
 addr2line将地址转换为文件名和行号。给定可执行文件中的地址或可重定位对象部分中的偏移量，它使用调试信息来确定与其关联的文件名和行号。

要使用的可执行或可重定位对象由-e 选项。默认是文件a.out. 可重定位对象中要使用的部分由-j选项。

addr2line有两种操作模式。

首先，在命令行中指定十六进制地址，并addr2line显示每个地址的文件名和行号。

第二步，addr2line从标准输入读取十六进制地址，并在标准输出上打印每个地址的文件名和行号。在这种模式下，addr2line可以在管道中使用来转换动态选择的地址。

输出的格式是'文件名：LINENO'。默认情况下，每个输入地址都会生成一行输出。

两个选项可以在每个 ' 之前生成额外的行文件名：LINENO' 行（按此顺序）。

如果-一种使用选项然后显示一行输入地址。

如果-F选项被使用，然后一行带有 '功能名称' 被展示。这是包含地址的函数的名称。

一个选项可以在 ' 之后生成额外的行文件名：LINENO' 线。

如果-一世使用选项并且由于编译器内联而存在给定地址的代码，然后显示其他行。一到两行额外的行（如果 -F使用选项）为每个内联函数显示。

或者，如果-p使用选项然后每个输入地址生成一个长的输出行，其中包含地址、函数名、文件名和行号。如果 -一世选项也已被使用，那么任何内联函数都将以相同的方式显示，但在单独的行上，并以文本 ' 为前缀（内联）'。

如果无法确定文件名或函数名， addr2line将在其位置打印两个问号。如果无法确定行号，addr2line将打印 0。

在这里作为备选方案显示的多头和空头形式的期权是等价的。

-a
--addresses
在函数名、文件和行号信息之前显示地址。地址印有'0x' 前缀以便于识别它。

-b bfdname
--target=bfdname
指定目标文件的目标代码格式为 bfdname。

-C
--demangle[=style]
将 ( demangle ) 低级符号名称解码为用户级名称。除了删除系统前面的任何初始下划线外，这使得 C++ 函数名称可读。不同的编译器有不同的修饰风格。可选的 demangling 样式参数可用于为您的编译器选择合适的 demangling 样式。参见c++filt，了解更多关于去重的信息。

-e filename
--exe=filename
指定应为其转换地址的可执行文件的名称。默认文件是a.out.

-f
--functions
显示函数名称以及文件和行号信息。

-s
--basenames
仅显示每个文件名的基础。

-i
--inlines
如果地址属于被内联的函数，则返回到第一个非内联函数的所有封闭范围的源信息也将被打印。例如，如果maininlines callee1which inlinescallee2和 address 是 from callee2，则 和 的源信息callee1也main 将被打印。

-j
--section
读取相对于指定部分而不是绝对地址的偏移量。

-p
--pretty-print
使输出更人性化：每个位置都打印在一行上。如果选项-一世已指定，所有封闭范围的行都以 ' 为前缀（内联）'。

-r
-R
--recurse-limit
--no-recurse-limit
--recursion-limit
--no-recursion-limit
启用或禁用在对字符串进行分解时执行的递归量的限制。由于名称修饰格式允许无限级别的递归，因此可以创建字符串，其解码将耗尽主机上可用的堆栈空间量，从而触发内存故障。该限制试图通过将递归限制为 2048 级嵌套来防止这种情况发生。

默认情况下启用此限制，但可能需要禁用它才能破解真正复杂的名称。但是请注意，如果禁用递归限制，则可能会耗尽堆栈，并且任何有关此类事件的错误报告都将被拒绝。

这-r选项是的同义词 --无递归限制选项。这-R选项是的同义词--递归限制选项。

请注意，此选项仅在以下情况下有效-C要么 --demangle选项已启用。

readelf
^^^^^^^^^^^^

readelf [-a|--all]
        [-h|--file-header]
        [-l|--program-headers|--segments]
        [-S|--section-headers|--sections]
        [-g|--section-groups]
        [-t|--section-details]
        [-e|--headers]
        [-s|--syms|--symbols]
        [--dyn-syms|--lto-syms]
        [--sym-base=[0|8|10|16]]
        [--demangle=style|--no-demangle]
        [--quiet]
        [--recurse-limit|--no-recurse-limit]
        [-U method|--unicode=method]
        [-n|--notes]
        [-r|--relocs]
        [-u|--unwind]
        [-d|--dynamic]
        [-V|--version-info]
        [-A|--arch-specific]
        [-D|--use-dynamic]
        [-L|--lint|--enable-checks]
        [-x <number or name>|--hex-dump=<number or name>]
        [-p <number or name>|--string-dump=<number or name>]
        [-R <number or name>|--relocated-dump=<number or name>]
        [-z|--decompress]
        [-c|--archive-index]
        [-w[lLiaprmfFsoORtUuTgAck]|
         --debug-dump[=rawline,=decodedline,=info,=abbrev,=pubnames,=aranges,=macro,=frames,=frames-interp,=str,=str-offsets,=loc,=Ranges,=pubtypes,=trace_info,=trace_abbrev,=trace_aranges,=gdb_index,=addr,=cu_index,=links]]
        [-wK|--debug-dump=follow-links]
        [-wN|--debug-dump=no-follow-links]
        [-P|--process-links]
        [--dwarf-depth=n]
        [--dwarf-start=n]
        [--ctf=section]
        [--ctf-parent=section]
        [--ctf-symbols=section]
        [--ctf-strings=section]
        [-I|--histogram]
        [-v|--version]
        [-W|--wide]
        [-T|--silent-truncation]
        [-H|--help]
        elffile…
readelf显示有关一个或多个 ELF 格式目标文件的信息。这些选项控制要显示的特定信息。

elffile ... 是要检查的目标文件。支持 32 位和 64 位 ELF 文件，以及包含 ELF 文件的存档。

该程序执行与 objdump_ _

在这里作为备选方案显示的多头和空头形式的期权是等价的。除 ' 外至少有一个选项-v' 要么 '-H' 必须给出。

-a
--all
相当于指定--文件头, --程序头文件,--sections,--符号, --relocs,- 动态的,- 笔记, - 版本信息,--arch-specific,- 放松, --section-groups和--直方图.

注意 - 此选项不启用--使用动态本身，因此如果命令行上不存在该选项，则不会显示动态符号和动态重定位。

-h
--file-header
显示文件开头的 ELF 标头中包含的信息。

-l
--program-headers
--segments
显示文件段标头中包含的信息（如果有）。

--quiet
禁止“无符号”诊断。

-S
--sections
--section-headers
显示文件的节标题中包含的信息（如果有）。

-g
--section-groups
显示文件的节组中包含的信息（如果有）。

-t
--section-details
显示详细的部分信息。暗示-S.

-s
--symbols
--syms
显示文件符号表部分中的条目（如果有）。如果符号有与之关联的版本信息，那么也会显示。版本字符串显示为符号名称的后缀，前面有一个 @ 字符。例如 'foo@VER_1'。如果版本是解析对符号的无版本引用时使用的默认版本，则它显示为一个后缀，前面有两个 @ 字符。例如 'foo@@VER_2'。

--dyn-syms
显示文件的动态符号表部分中的条目（如果有）。输出格式与使用的格式相同 --syms选项。

--lto-syms
显示文件中任何 LTO 符号表的内容。

--sym-base=[0|8|10|16]
强制符号表的大小字段使用给定的基数。任何无法识别的选项都将被视为'0'。 --sym-base=0 表示默认行为和遗留行为。对于小于 100000 的数字，这会将大小输出为十进制。对于大小 100000 和更大的十六进制表示法，将使用 0x 前缀。 --sym-base=8将以八进制给出符号大小。 --sym-base=10将始终以十进制给出符号大小。 --sym-base=16将始终以带有 0x 前缀的十六进制给出符号大小。

-C
--demangle[=style]
将 ( demangle ) 低级符号名称解码为用户级名称。这使得 C++ 函数名称可读。不同的编译器有不同的修饰风格。可选的 demangling 样式参数可用于为您的编译器选择合适的 demangling 样式。参见c++filt，了解更多关于去重的信息。

--no-demangle
不要对低级符号名称进行拆解。这是默认设置。

--recurse-limit
--no-recurse-limit
--recursion-limit
--no-recursion-limit
启用或禁用在对字符串进行分解时执行的递归量的限制。由于名称修饰格式允许无限级别的递归，因此可以创建字符串，其解码将耗尽主机上可用的堆栈空间量，从而触发内存故障。该限制试图通过将递归限制为 2048 级嵌套来防止这种情况发生。

默认情况下启用此限制，但可能需要禁用它才能破解真正复杂的名称。但是请注意，如果禁用递归限制，则可能会耗尽堆栈，并且任何有关此类事件的错误报告都将被拒绝。

-U [d|i|l|e|x|h]
--unicode=[default|invalid|locale|escape|hex|highlight]
控制标识符名称中非 ASCII 字符的显示。默认（--unicode=语言环境要么--unicode=默认) 是将它们视为多字节字符并在当前语言环境中显示它们。此选项的所有其他版本将字节视为 UTF-8 编码值并尝试解释它们。如果它们不能被解释或者如果--unicode=无效使用选项然后它们显示为十六进制字节序列，括在花括号字符中。

使用--unicode=转义选项将字符显示为 unicode 转义序列（\uxxxx）。使用 --unicode=十六进制将字符显示为用尖括号括起来的十六进制字节序列。

使用--unicode=高亮会将字符显示为 unicode 转义序列，但假设输出设备支持着色，它也会以红色突出显示它们。着色旨在提醒人们注意可能不存在的 unicode 序列。

-e
--headers
显示文件中的所有标题。相当于-h -l -S.

-n
--notes
显示 NOTE 段和/或部分的内容（如果有）。

-r
--relocs
显示文件的重定位部分的内容（如果有的话）。

-u
--unwind
显示文件展开部分的内容（如果有）。当前仅支持 IA64 ELF 文件的展开部分以及 ARM 展开表 ( .ARM.exidx/ .ARM.extab)。如果您的架构尚未实现支持，您可以尝试使用转储.eh_frames部分 的内容--debug-dump=帧要么--debug-dump=frames-interp 选项。

-d
--dynamic
显示文件的动态部分的内容（如果有）。

-V
--version-info
显示文件中版本部分的内容，如果它们存在的话。

-A
--arch-specific
显示文件中特定于体系结构的信息（如果有）。

-D
--use-dynamic
显示符号时，此选项readelf使用文件动态部分中的符号哈希表，而不是符号表部分。

显示重定位时，此选项使readelf 显示动态重定位而不是静态重定位。

-L
--lint
--enable-checks
显示有关正在检查的文件可能存在问题的警告消息。如果单独使用，则将检查文件的所有内容。如果与转储选项之一一起使用，则只会为正在显示的内容生成警告消息。

-x <number or name>
--hex-dump=<number or name>
将指定部分的内容显示为十六进制字节。编号通过节表中的索引标识特定节；任何其他字符串标识目标文件中具有该名称的所有部分。

-R <number or name>
--relocated-dump=<number or name>
将指定部分的内容显示为十六进制字节。编号通过节表中的索引标识特定节；任何其他字符串标识目标文件中具有该名称的所有部分。该部分的内容将在显示之前重新定位。

-p <number or name>
--string-dump=<number or name>
将指定部分的内容显示为可打印字符串。编号通过节表中的索引标识特定节；任何其他字符串标识目标文件中具有该名称的所有部分。

-z
--decompress
请求被转储的部分X,R要么 p选项在显示之前被解压缩。如果部分未压缩，则它们按原样显示。

-c
--archive-index
显示包含在二进制归档文件头部分的文件符号索引信息。执行与吨 命令ar，但不使用 BFD 库。见ar。

-w[lLiaprmfFsOoRtUuTgAckK]
--debug-dump[=rawline,=decodedline,=info,=abbrev,=pubnames,=aranges,=macro,=frames,=frames-interp,=str,=str-offsets,=loc,=Ranges,=pubtypes,=trace_info,=trace_abbrev,=trace_aranges,=gdb_index,=addr,=cu_index,=links,=follow-links]
显示文件中 DWARF 调试部分的内容（如果存在）。压缩的调试部分在显示之前会自动（临时）解压缩。如果开关后面有一个或多个可选字母或单词，则只会转储那些类型的数据。字母和单词指的是以下信息：

a
=abbrev
显示'的内容.debug_abbrev' 部分。

A
=addr
显示'的内容.debug_addr' 部分。

c
=cu_index
显示'的内容.debug_cu_index'和/或'.debug_tu_index' 部分。

f
=frames
显示 ' 的原始内容.debug_frame' 部分。

F
=frames-interp
显示 ' 的解释内容.debug_frame' 部分。

g
=gdb_index
显示'的内容.gdb_index'和/或'.debug_names' 部分。

i
=info
显示'的内容。调试信息' 部分。注意：此选项的输出也可以通过使用 --矮人深度和--矮人开始选项。

k
=links
显示'的内容.gnu_debuglink', '.gnu_debugaltlink' 和 '.debug_sup' 部分，如果有的话。还显示指向单独的 dwarf 对象文件 (dwo) 的任何链接，如果它们由 '。调试信息' 部分。

K
=follow-links
显示在链接的单独调试信息文件中找到的任何选定调试部分的内容。如果同一调试部分存在于多个文件中，这可能会导致显示多个版本。

另外，在显示 DWARF 属性时，如果发现表单引用了单独的调试信息文件，那么也会显示引用的内容。

注意 - 在某些发行版中，此选项默认启用。它可以通过禁用ñ调试选项。通过配置 binutils 时可以选择默认值 --enable-follow-debug-links=yes要么 --enable-follow-debug-links=no选项。如果不使用这些，则默认启用以下调试链接。

N
=no-follow-links
禁用以下链接到单独的调试信息文件。

l
=rawline
显示'的内容.debug_line' 原始格式的部分。

L
=decodedline
显示 ' 的解释内容.debug_line' 部分。

m
=macro
显示'的内容.debug_macro'和/或'.debug_macinfo' 部分。

o
=loc
显示'的内容.debug_loc'和/或'.debug_loclists' 部分。

O
=str-offsets
显示'的内容.debug_str_offsets' 部分。

p
=pubnames
显示'的内容.debug_pubnames'和/或'.debug_gnu_pubnames' 部分。

r
=aranges
显示'的内容.debug_aranges' 部分。

R
=Ranges
显示'的内容.debug_ranges'和/或'.debug_rnglists' 部分。

s
=str
显示'的内容.debug_str', '.debug_line_str'和/或'.debug_str_offsets' 部分。

t
=pubtype
显示'的内容.debug_pubtypes'和/或'.debug_gnu_pubtypes' 部分。

T
=trace_aranges
显示'的内容.trace_aranges' 部分。

u
=trace_abbrev
显示'的内容.trace_abbrev' 部分。

U
=trace_info
显示'的内容.trace_info' 部分。

注意：显示 ' 的内容.debug_static_funcs', '.debug_static_vars' 和 'debug_weaknames' 部分当前不受支持。

--dwarf-depth=n
.debug_info将部分的转储限制为n个子项。这仅适用于--调试转储=信息. 默认打印所有DIE；n的特殊值 0也会产生这种效果。

如果n的值不为零， 则不会打印n级或更深的 DIE 。n的范围是从零开始的。

--dwarf-start=n
仅打印以编号为n的 DIE 开头的 DIE 。这仅适用于--调试转储=信息.

如果指定，此选项将禁止打印任何标头信息和编号为n的 DIE 之前的所有 DIE 。只会打印指定 DIE 的兄弟姐妹和孩子。

这可以与--矮人深度.

-P
--process-links
显示在链接到主文件的单独 debuginfo 文件中找到的非调试部分的内容。此选项自动暗示-wK选项，并且仅显示其他命令行选项请求的部分。

--ctf[=section]
显示指定 CTF 节的内容。CTF 部分本身包含许多子部分，所有子部分都按顺序显示。

默认情况下，显示名为.ctf的部分的名称，即 .ctf 发出的名称ld。

--ctf-parent=member
如果 CTF 部分包含模糊定义的类型，它将包含许多 CTF 字典的存档，所有字典都继承自一个包含明确类型的字典。此成员默认命名为.ctf，就像包含它的部分一样，但可以ctf_link_set_memb_name_changer 在链接时使用该函数更改此名称。查看由使用名称更改器重命名父存档成员的链接器创建的 CTF 存档时，--ctf-父级可用于指定用于父级的名称。

--ctf-symbols=section
--ctf-strings=section
指定 CTF 文件可以从中继承字符串和符号的另一个节的名称。默认情况下，使用.symtab及其链接的字符串表。

如果其中任何一个--ctf-符号要么--ctf-字符串已指定，其他也必须指定。

-I
--histogram
在显示符号表的内容时显示桶列表长度的直方图。

-v
--version
显示 readelf 的版本号。

-W
--wide
不要破坏输出行以适应 80 列。默认情况下 readelf，为 64 位 ELF 文件断开节标题和段列表行，以便它们适合 80 列。此选项会导致 readelf打印每个节标题。每个段都是单行，在超过 80 列的终端上可读性更高。

-T
--silent-truncation
通常当 readelf 显示符号名称时，它必须截断名称以适应 80 列显示，它会[...]在名称中添加后缀 of。此命令行选项禁用此行为，允许再显示 5 个名称的字符并恢复 readelf 的旧行为（发布 2.35 之前）。

-H
--help
显示 . 理解的命令行选项readelf。
 
 
elfedit
^^^^^^^^^
elfedit [--input-mach=machine]
        [--input-type=type]
        [--input-osabi=osabi]
        [--input-abiversion=version]
        --output-mach=machine
        --output-type=type
        --output-osabi=osabi
        --output-abiversion=version
        --enable-x86-feature=feature
        --disable-x86-feature=feature
        [-v|--version]
        [-h|--help]
        elffile… 
 
 elfedit更新具有匹配 ELF 机器和文件类型的 ELF 文件的 ELF 头和程序属性。这些选项控制 ELF 标头和程序属性中的更新方式和字段。

elffile … 是要更新的 ELF 文件。支持 32 位和 64 位 ELF 文件，以及包含 ELF 文件的存档。

在这里作为备选方案显示的多头和空头形式的期权是等价的。至少其中一项--输出-马赫, --输出类型,--输出-osabi, --output-abiversion, --enable-x86-feature和--disable-x86-feature 必须给出选项。

--input-mach=machine
将匹配的输入 ELF 机器类型设置为machine。如果 --输入-马赫未指定，它将匹配任何 ELF 机器类型。

支持的 ELF 机器类型是i386、IAMCU、L1OM、 K1OM和x86-64。

--output-mach=machine
将 ELF 标头中的 ELF 机器类型更改为machine。支持的 ELF 机器类型与--输入-马赫.

--input-type=type
将匹配的输入 ELF 文件类型设置为type。如果 - 输入类型未指定，它将匹配任何 ELF 文件类型。

支持的 ELF 文件类型是rel、exec和dyn。

--output-type=type
将 ELF 标头中的 ELF 文件类型更改为type。支持的 ELF 类型与- 输入类型.

--input-osabi=osabi
将匹配的输入 ELF 文件 OSABI 设置为osabi。如果 --input-osabi未指定，它将匹配任何 ELF OSABI。

支持的 ELF OSABI 包括：无、HPUX、NetBSD、 GNU、Linux（GNU的别名）、 Solaris、AIX、Irix、 FreeBSD、TRU64、Modesto、OpenBSD、OpenVMS、 NSK、AROS和FenixOS。

--output-osabi=osabi
将 ELF 标头中的 ELF OSABI 更改为osabi。支持的 ELF OSABI 与--input-osabi.

--input-abiversion=version
将匹配的输入 ELF 文件 ABIVERSION 设置为version。 版本必须介于 0 和 255 之间。如果--input-abiversion 未指定，它将匹配任何 ELF ABIVERSIONs。

--output-abiversion=version
将 ELF 标头中的 ELF ABIVERSION 更改为version。 版本必须在 0 到 255 之间。

--enable-x86-feature=feature
在机器类型为i386或x86-64的exec或dyn ELF 文件中设置程序属性中的功能位。支持的功能是ibt、shstk、lam_u48和 lam_u57。

--disable-x86-feature=feature
清除机器类型为i386或x86-64的exec或 dyn ELF 文件中程序属性中的功能位。支持的功能与--enable-x86-feature.

笔记：--enable-x86-feature和--disable-x86-feature 仅在带有 ' 的主机上可用地图' 支持。

-v
--version
显示版本号elfedit。

-h
--help
显示 . 理解的命令行选项elfedit。


常用选项
^^^^^^^^^^^
本手册中描述的所有程序都支持以下命令行选项。

@file
从文件中读取命令行选项。读取的选项被插入以代替原始的@file选项。如果文件 不存在或无法读取，则该选项将按字面意思处理，而不是删除。

文件中的选项由空格分隔。通过将整个选项括在单引号或双引号中，可以在选项中包含空格字符。通过在要包含的字符前加上反斜杠，可以包含任何字符（包括反斜杠）。该文件本身可能包含额外的@file选项；任何此类选项都将被递归处理。

--help
显示程序支持的命令行选项。

--version
显示程序的版本号。
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 


