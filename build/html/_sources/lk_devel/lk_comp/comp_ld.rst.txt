
ld 在linux中的应用
-----------------

1. ld 将多个对象文件和归档文件进行组合，重定位他们的数据并绑定引用的符号。通常情况下编译一个程序的最后一步就是运行ld。
2. ld 接收利用AT&T 链接编辑指令语言符号编写的链接指令语言文件，来对链接过程提供显式并全面的控制。
3. 最近版本的利用通用BPF库来对目标文件进行操作。ld 文件可以通过多种不同的格式来读，合并，写目标文件，例如COFF 或 a.out。不同的格式可以被链接产生任何一种可用的目标文件（难道ELF和COFF可以一块链接？应该是不是这个意思吧？
4. 除了灵活性，gnu 链接器在提供诊断信息方面比其他连接器更有优势。相较其他链接器遇到错误就放弃执行，ld可以继续执行，允许识别其他的错误（或者，某些情况下，即使出现错误也要获取输出文件）。


gnu连接器ld 涵盖广泛，并最大程度上与其他连接器兼容。这种情况下，就有很多选项来控制连接器的行为。

命令行选项
^^^^^^^^^
 - 虽然ld提供了大量命令行选项，针对特定的应用场景并不会涉及很多指令。例如unix目标文件链接文件：
   -  ld -o output /lib/crt0.o hello.o -lc  (-lc = -llibc.so)
 - ld 的一些命令行选项可以在命令行的任何位置指定,但是想'-l'或'-T'这样与文件相关的选型,会让文件在命令行指定的与目标文件和其他文件选项相对位置插入文件内容,
 - 命令行中 目标文件不能夹在选项和其变量中间。
 - 用不同的参数重复使用非文件选项不会有影响,或者覆盖掉选项指定的参数(右边覆盖左边的).对于多次指定会有实际意义的特殊选项,在后面说明.
 - 通常情况下,连接器只针对至少一个目标文件进行操作,但是也可以通过'-l'来指定其他格式的二进制输入文件,'-R'指定脚本另名语言.如果没有指定二进制输入文件,连接器不产生任何输出,并打印消息:"No input files".
 - 如果连接器不能识别目标文件的格式,会假定这个文件为连接器脚本.以这种方式指定的脚本会扩充用于链接的主链接器脚本（默认链接器脚本或使用“-T”指定的链接器脚本） (注意是扩充,不是代替).这个特性允许链接器链接一个看起来是一个对象或档案的文件，但实际上只是定义了一些符号值，或者使用 INPUT 或 GROUP 加载其他对象。 以这种方式指定脚本只会增加主链接脚本，额外的命令放在主脚本之后； 使用“-T”选项完全替换默认链接描述文件，但请注意 INSERT 命令的效果。 
 - 对于名称为单个字母的选项，选项参数必须紧跟在选项字母之后且中间没有空格，或者作为单独的参数紧跟在需要它们的选项后面。 
 - 针对由多个字符组成的命令选项，前面加一个或两个破折号，如"-trace-symbol"和"--trace-symbol"(两种方式是等效的）。一种情况例外：以消协"o"开头的选项只能前加两个破折号，这样是为了降低与“-o"冲突的几率。如”-omagic"设置输出文件为“majic”，而“--omagic"为输出设置NMAGIC标识。多字符选项与其参数间需要通过"="链接 或紧跟在选项后面，如 “--trace-symbol foo”与“--trace-symbol=foo”是等价的。
 - 如果通过编译器（如gcc）间接调用连接器（ld),则所有的连接器命令行选项都要加上‘-W1，’前缀，如：
   - gcc -W1,--start-group foo.o bar.o -W1,--end-group ,如果不加前缀，编译器会默认丢弃连接器选项，从而导致链接错误。
 - 当采用编译器调用连接器时，如果链接参数和其值之间用空格做为间隔符，会增加冲突的可能性。编译器容易把选项传递给连接器，却把值传给编译器。最简单的解决方法就是用组合方式(单字符选项和多字符选项采用不同的方式），如：
   - gcc foo.o bar.o -W1,-eENTRY -W1,-Map=a.map
 - 通用命令行表:
   - @file:从文件中读命令行参数。选项会插入到@file的位置。如果文件不存在，或不能读，就将文件名字当选项，并不能删除。文件中的选项由空格分割。通过将整个选项括在单引号或双引号中，可以在选项中包括空格字符。每个字符（包括反斜杠）可能需要包含以反斜杠开头的字符。文件中可以包含另问的@file选项;这样的选项采用递归方式进行处理。
   - -a keyword: 与 HP/UX兼容。
   - --audit AUDITLIB:将AUDITLIB 添加到dynamic section 的DT_AUDIT入口，将AUDITLIB添加到动态部分的DT_AUDIT条目中。 不会检查AUDITLIB的存在，也不会使用指定的DT SONAME不会检查AUDITLIB的存在，也不会使用在库中指定的DT SONAME。 如果多次指定，则DT_AUDIT将包含用冒号分隔的审核接口列表。 如果链接器在搜索共享库时找到带有审核条目的对象，则它将在输出文件中添加相应的DT_DEPAUDIT条目。 此选项仅在支持rtld-audit接口的ELF平台上有意义。 
   - -b input-format
     --format=input-format
   - -c MRI-commandfile
     --mri-script=MRI-commandfile
   - -d
     -dc
     -dp
   - --depaudit AUDITLIB
     -P AUDITLIB
   - --enable-non-contiguous-regions
   - --enable-non-contiguous-regions-warnings
   - -e entry
     --entry=entry
   - --exclude-libs lib ,lib ,...
   - --exclude-modules-for-implib module ,module ,...
   - -E
     --export-dynamic
     --no-export-dynamic
   - --export-dynamic-symbol=glob
   - --export-dynamic-symbol-list=file
   - -EB
   - -EL
   - -f name
     --auxiliary=name
   - -F name
     --filter=name
   - -fini=name
   - -g
   - -G value
     --gpsize=value
   - -h name
     -soname=name
   - -i
   - -init=name
   - -l namespec
     --library=namespec
   - -L searchdir
     --library-path=searchdir
   - -m emulation
   - -M
     --print-map
   - --print-map-discarded
     --no-print-map-discarded
   - -n
     --nmagic
   - -N
     --omagic
   - --no-omagic
   - -o output
     --output=output
   - --dependency-file=depfile
   - -O level
   - -plugin name
   - --push-state
   - --pop-state
   - -q
     --emit-relocs
   - --force-dynamic
   - -r
     --relocatable
   - -R filename
     --just-symbols=filename
   - -s
     --strip-all
   - -S
     --strip-debug
   - --strip-discarded
     --no-strip-discarded
   - -t
     --trace
   - -T scriptfile
     --script=scriptfile
   - -dT scriptfile
     --default-script=scriptfile
   - -u symbol
     --undefined=symbol
   - --require-defined=symbol
   - -Ur
   - --orphan-handling=MODE
   - --unique[=SECTION ]
   - -v
     --version
     -V
   - -x
     --discard-all
   - -X
     --discard-locals
   - -y symbol
     --trace-symbol=symbol
   - -Y path
   - -z keyword
   - -( archives -)
     --start-group archives --end-group
   - --accept-unknown-input-arch
     --no-accept-unknown-input-arch
   - --as-needed
     --no-as-needed
   - --add-needed
     --no-add-needed
   - -assert keyword
   - -Bdynamic
     -dy
     -call_shared
   - -Bgroup
   - -Bstatic
     -dn
     -non_shared
     -static
   - -Bsymbolic
   - -Bsymbolic-functions
   - --dynamic-list=dynamic-list-file
   - --dynamic-list-data
   - --check-sections
     --no-check-sections
   - --copy-dt-needed-entries
     --no-copy-dt-needed-entries
   - --cref
   - --no-define-common
   - --force-group-allocation
   - --defsym=symbol =expression
   - --demangle[=style ]
     --no-demangle
   - -Ifile
     --dynamic-linker=file
   - --no-dynamic-linker
   - --embedded-relocs
   - --disable-multiple-abs-defs
   - --fatal-warnings
     --no-fatal-warnings
   - --force-exe-suffix
   - --gc-sections
     --no-gc-sections
   - --print-gc-sections
     --no-print-gc-sections
   - --gc-keep-exported
   - --print-output-format
   - --print-memory-usage
   - --help
   - --target-help
   - -Map=mapfile
   - --no-keep-memory
   - --no-undefined
     -z defs
   - --allow-multiple-definition
     -z muldefs
   - --allow-shlib-undefined
     --no-allow-shlib-undefined
   - --no-undefined-version
   - --default-symver
   - --default-imported-symver
   - --no-warn-mismatch
   - --no-warn-search-mismatch
   - --no-whole-archive
   - --noinhibit-exec
   - -nostdlib
   - --oformat=output-format
   - --out-implib file
   - -pie
     --pic-executable
   - -qmagic
   - -Qy
   - --relax
     --no-relax
   - --retain-symbols-file=filename
   - -rpath=dir
   - -rpath-link=dir
   - -shared
     -Bshareable
   - --sort-common
     --sort-common=ascending
     --sort-common=descending
   - --sort-section=name
   - --sort-section=alignment
   - --spare-dynamic-tags=count
   - --split-by-file[=size ]
   - --split-by-reloc[=count ]
   - --stats
   - --sysroot=directory
   - --task-link
   - --traditional-format
   - --section-start=sectionname =org
   - -Tbss=org
     -Tdata=org
     -Ttext=org
   - -Ttext-segment=org
   - -Trodata-segment=org
   - -Tldata-segment=org
   - --unresolved-symbols=method
   - --dll-verbose
     --verbose[=NUMBER ]
   - --version-script=version-scriptfile
   - --warn-common
   - --warn-constructors
   - --warn-multiple-gp
   - --warn-once
   - --warn-section-align
   - --warn-textrel
   - --warn-alternate-em
   - --warn-unresolved-symbols
   - --error-unresolved-symbols
   - --whole-archive
   - --wrap=symbol
   - --eh-frame-hdr
     --no-eh-frame-hdr
   - --no-ld-generated-unwind-info
   - --enable-new-dtags
     --disable-new-dtags
   - --hash-size=number
   - --hash-style=style
   - --compress-debug-sections=none
     --compress-debug-sections=zlib
     --compress-debug-sections=zlib-gnu
     --compress-debug-sections=zlib-gabi
   - --reduce-memory-overheads
   - --build-id
     --build-id=style


环境变量
^^^^^^^^^
可以使用环境变量GNUTARGET，LDEMULATION和COLLECT_NO_DEMANGLE更改ld的行为。
- GNUTARGET
- LDEMULATION
- COLLECT_NO_DEMANGLE

链接脚本
^^^^^^^^

- 每一次链接都由一个链接脚本来控制。链接脚本利用链接命令语言写成。
- 链接脚本的主要目的就是描述输入文件中的节如何映射到输出文件中，控制输出文件的内存布局。大多数链接脚本只做这些工作。如果需要，链接脚本可以借助后面描述指令直接做其他操作。
- 链接过程一定要使用一个链接脚本，如果没有显式提供，连接器将使用缺省链接脚本来进行链接操作，可通过“--verbose”命令行选项来进行查看。命令行参数‘-r'或’-N‘ 会对缺省脚本有影响。
- 可通过“-T”参数来指定自己的链接脚本。
- 也可以将链接脚本作为连接器的输入文件来隐式指定链接脚本。

链接脚本的基本概念
"""""""""""""""
用于描述链接脚本语言的关键词

- 链接脚跟将所有输入文件组合为一个输出文件.输出文件和每个输入文件以对象文件格式的数据格式存在.每个文件成为对象文件.输出文件经常被成为可执行文件,我们也可以称其为对象文件.
- 每个对象文件包含一些列的节.
- 我们通常称输入文件中的一个节为输入节,输出文件中的节为输出节
- 对象文件中的每个节有节名和大小.多数节有对应的数据块作为其节内容.
- 一个节可以被标注为可加载的(loadable),也就是说当输出文件运行时需要将这个节加载到内存中
- 一个没有对应节内容的节可能是可分配的(allocatable),也就是说需要在内存中为其预留,但不初始化内容(一些情况下需要将其内容设置为全零)
- 一个节如果既不时可加载的也不是可分配的通常情况下包含的是调试信息.
- 每个可加载或可分配的输出节有两个地址:地一个是VMA(虚拟内存地址)是输出文件运行时节的地址；第二个是LMA(加载内存地址),这是节加载时的地址(是这样吗?).多数情况下这两个值是一样的.一个不一样的例子是当数据节加载进ROM,然后在RAM运行时(程序启动,通常用于初始化ROM基础系统中的全局变量时).此时ROM地址可能是LMA,RAM地址是VMA.
- objdump -h 选项来查看对象文件中的节:通过objdump -h vmlinux  查看各个节信息，会发现这两个地址是不同的(虚拟地址/物理地址:可见被加载时与运行时寻址方式会发生变化,其实真是地址是没有移动的)，但可以与vmlinux.ld对应起来。（理解加载过程，其实只要理解了ELF就够了）
- 每个目标文件有一个符号列表，称为符号表。一个符号可以是定义的或未定义的。每个符号有一个名字，每个定义的符号有一个地址，如果将一个C或C++程序编译成一个目标文件，需要为每个定义的符号和全局或静态变量获取一个定义的符号。输出文件中每个未定义的函数或全局变量变为未定义符号。
- nm程序用于查看目标文件中的符号，或通过objdump -t来查看，（注意程序里如何调用符号表？）


链接脚本格式：
""""""""""""
- 链接脚本是一系列指令组成的文本文件。
- 每个命令可以是关键字，通常后面跟有参数，或指定为一个符号。
- 通常用“;”做为命令分割符。
- 空格在通常情况下是被忽略的。
- 通常可以直接输入诸如文件名或格式名之类的字符串。
- 如果文件名字中出现“，”被解析为分割文件名字的符号时，需要用双引号将文件名引起来。
- 在文件名中不能用双引号。
- 可用/* */做注释。用法与C中相似。注释与空格等价

简单的链接脚本
""""""""""""

很多脚本是非常简单的。最简单的脚本只有一个命令：“SECTIONS”。使用SECTIONS指令来描述输出文件的内存布局。

SECTIONS是一个很强大的命令。假设程序只包含代码，初始化数据和未初始化数据，对应'.text','.data'和'.bss'节。假设输入文件也只包含这些节。

数据加载到0x10000，数据从0x8000000开始，如下

.. code-block:: c
   :caption: c test
   :linenos:
   SECTIONS
   {
    . = 0x1000;
    .text : { *(.text)}
    . = 0x08000000;
    .data : {*(.data)}
    .bss : { *(.bss)}
    }


- '.'：位置计数器,如果不重新设定，则顺序递增。SECTIONS命令的开始，定位计数器的值为“0”。这是一个简单而完整的链接描述文件.
- 第二行定义一个输出节'.text',冒号是必需的语法，可以暂时忽略.在输出节名称后的花括号内，列出 应放入此输出部分的输入部分的名称,'*'通配符匹配所有文件,表达式'*(.text)'意思是所有输入文件中的'.text'输入节.
- 定义'.text节时设定定位计数器为'0x10000',连接器将设置输出文件中'.text'节加载地址为'0x10000'.
- 设置'. data'节地址为'0x8000000',将'.bss'防止在其后.连接器会处理节按需要的格式对其,如果需要需要增加计数器的值.在'.data'和'.bss'节之间连接器可能会增加一个空白区域.


简单的链接脚本指令
""""""""""""""""

设置入口点
*********
设置入口指令:ENTRY（symbol),参数是一个符号名,可以有多种方式来设置入口点.连接器会按照以下顺序尝试设置入口点,当尝试成功时就停止:
1. 命令行参数’-e'
2. 链接接本中ENTRY(symbol)命令
3. 目标特定符号的值（如果已定义）； 对于许多目标，这是开始，但例如基于 PE 和 BeOS 的系统检查可能的入口符号列表，匹配找到的第一个符号。 
4. '.text'节的第一个字节
5. 地址0


处理文件的命令
*************

- INCLUDE filename:包含链接文件。从当前目录，或从-L指定的目录查找。最多可进行10级嵌套。INCLUDE可放置在顶层，MEMORY或SECTIONS命令中，或输出节描述中
- INPUT(file , file , ...)
  INPUT(file file ...):链接一个文件。如果INPUT（-lfile), 将链接libfile.a,与命令行参数’-l'等价.如果隐式链接脚本中加入的INPUT指令，可能会影响文件搜索 。
- GROUP(file , file , ...)
  GROUP(file file ...):与INPUT相似，除了已命名的文件应全部为归档文件，然后重复搜索它们，直到未创建新的未定义引用为止。（不是很理解）
- AS_NEEDED(file , file , ...)
  AS_NEEDED(file file ...):
- OUTPUT(filename):命名输出文件，就像在命令行使用‘-o filename'指定输出文件是一样的，如果同时使用，则命令行参数优先。 如果都不指定，默认输出文件为a.out
- SEARCH_DIR(path):与’-L path'类似
- STARTUP（filename):与INPUT命令类似，但此中的文件做为第一个链接的文件。当使用入口点始终是第一个文件的开始的系统时，这可能很有用。 

处理目标文件格式的命令：两个
************************
- OUTPUT_FORMAT(bfdname )
  OUTPUT_FORMAT(default , big , little ):与‘--oformat bfdname'命令行参数类似。如果同时使用，命令行参数优先。  三个参数时，与'-EB' 或'-EL'功能类似，如：OUTPUT_FORMAT(elf32-bigmips,elf32-bigmips,elf32-littlemips):默认'elf32-bigmips',但命令行如果用了‘-EL’参数，则输出文件则为‘elf32-littlemips'格式。
- TARGET（bfdname):读输入文件时使用，与命令行' -b bfdname'相似。


为内存区域指定别名
****************
REGION_ALIAS(alias,region)

- .text 代码
- .rodata:只读数据
- .data 可读写
- .bss 读写 数据初始化为0

目的是提供一个链接器命令文件，该文件包含一个系统独立部分，该部分定义输出部分，以及一个系统独立部分，将输出部分映射到系统上可用的内存区域。我们的嵌入式系统带有三种不同的内存设置A，B和C： 

所以 ROM加载在内核层面与内核只需要设置，而如何解析加载是引导的问题。

(......)


其他链接脚本命令
***************

- ASSERT（exp,message):exp如果为0，则连接器推出并打印错误码和信息（message,注意PROVIDEd,关于初始化值部分，有些注意的地方
- EXTERN（symbol symbol ...):强制将符号作为未定义符号输入到输出文件中,这样做可能会触发例如标准库中其他模块的链接,可一次设置多个符号，多次使用。其作用与’-u‘命令行参数一致。
- FORCE_COMMON_ALLOCATION:功能与’-d‘命令行参数一致：即使指定了可重定位的输出文件，也可以使ld为公共符号分配空间（’-r')
- INHIBIT_COMMON_ALLOCATION:与‘--no-define-common'命令行参数功能相同：即使对于不可重定位的输出文件，也可以将地址分配给通用符号。 
- FORCE_GROUP_ALLOCATION:--force-group-allocation
- INSERT [AFTER | BEFORE] output_section:"-T"
- NOCROSSREFS(section section ...):注意其使用输出文件的节名字。
- NOCROSSREFS_TO(tosection fromsection ...)
- OUTPUT_ARCH(bfdarch);ld '-f' 参数
- LD_FEATURE(string):用于改变ld 行为，如“SANE_EXPR"然后将脚本中的绝对符号和数字简单地视作数字 .


给符号赋值
"""""""""""

可以在链接脚本中给一个符号赋值。这将定义符号并将其放入具有全局范围的符号表中。

- 可以采用C风格的赋值操作。

.. code-block:: c
   :caption: c test
   :linenos:
   symbol = expression ;
   symbol += expression ;
   symbol -= expression ;
   symbol *= expression ;
   symbol /= expression ;
   symbol <<= expression ;
   symbol >>= expression ;
   symbol &= expression ;
   symbol |= expression ;


第一种情况将符号定义为表达式的值。 在其他情况下，必须已定义符号，并将相应地调整值。
- ’.'表示定位计数器。通常仅在SECTIONS命令中使用。表达式后的分号是必需的,
- 您可以自行将符号分配编写为命令， 也可以将其编写为SECTIONS命令中的语句，也可以将其作为SECTIONS命令中输出节描述的一部分。

HIDDEN：不会导出的符号：HIDDEN(symbol = expression )
****************************************************

.. code-block:: c
   :caption: c test
   :linenos:
   
   HIDDEN(floating_point = 0);
   SECTIONS
   {
   .text :
   {
   *(.text)
   HIDDEN(_etext = .);
   }
   HIDDEN(_bdata = (. + 3) & ~ 3);
   .data : { *(.data) }
   }

在这种情况下，这三个符号在该模块外部均不可见。

PROVIDE
************
PROVIDE：在某些情况下，希望链接描述文件仅在符号被引用且未由链接中包含的任何对象定义的情况下定义符号。例如，传统的链接器定义了符号“ etext”。 但是，ANSI C要求用户能够使用“ etext”作为函数名称，而不会遇到错误。 PROVIDE关键字可用于 仅在引用但未定义的情况下定义符号，例如“ etext”。 语法为PROVIDE（symbol = expression）。 

.. code-block:: c
   :caption: c test
   :linenos:
   
   SECTIONS
  {
  .text :
  {
  *(.text)
  _etext = .;
  PROVIDE(etext = .);
  }
  }

在这个例子中，如果程序定义了‘_etext",链接器将给出多个定义错误。否则，就会使用定义。如果程序引用了“ etext”但未定义它，则链接器将使用链接器脚本中的定义。 （如果程序中用了没有定义的符号，则会检查链接脚本中的定义，如果有，就有链接脚本中的符号定义。）注意：PROVIDE指令认为要定义一个通用符号 ，即使此类符号可以与PROVIDE将创建的符号结合使用 。在考虑构造函数和析构函数列表符号（例如“ __CTOR_LIST__”）时，这一点尤其重要，因为它们通常被定义为通用符号

PROVIDE_HIDDEN
***************

PROVIDE_HIDDEN:与provide类似，EFL格式文件，符号将会隐藏不导出。

源码引用
*******

源代码中访问链接脚本中定义的变量是不直观的。特别是链接脚本符号与高级语言中的变量定义不同，是没有值的符号。进一步分析前，认识到编译器经常将源代码中的名字会转换为不同的名字存入符号表中时。例如，Fortran 编译器通常会在符号名前后加入下划线，C++则会更广发改名。因此，在源代码中使用的变量名称与在链接描述文件中定义的相同变量的名称之间可能会有差异。也就是说在源码中使用的变量的名字和链接脚本中定义的变量点可能会有差异。例如在C语言中引用链接脚本变量的方式如下：extern int fool;但在链接脚本中可能定义如下： _foo = 1000;( 也就是说，链接脚本中变量应该与编译器修改后的符号名字一致）其他例子假定名称不需要改变。当一个符号在如C一样的高级语言中声明，会发生两件事情。第一编译器会在程序内存中保留足够的空间来保存符号的值。第二编译器在符号表中传见一个保存符号地址的入口（entry)。如符号表包含保存符号值的内存块的地址。所以，文件范围内，下面的C声明中：

.. code-block:: c
   :caption: c test
   :linenos:
   
   int foo = 1000;
   
   
在符号表中创建一个'foo'的入口。这个入口包含一个'int'长度的内存块的地址，内存块初始化为1000。
当程序引用编译器产生代码中的符号时，首先访问符号表查找到符号内存块的地址，然后代码从内存块中读出数值。所以：

.. code-block:: c
   :caption: c test
   :linenos:
   
   int *a = &foo;
   
产生foo在符号表中的符号，获取其地址，然后将地址复制到变量'a'对应的内存块中。
链接脚本符号声明，相比之下，在符号表中创建一个符号入口，但没为其指定内存。也就是说是一个没有值的地址。例如，链接脚本定义如下：

.. code-block:: c
   :caption: c test
   :linenos:
   
   foo = 1000;
   
在符号表中创建一个'foo'的入口，包含地址定位为1000的地址（所以这是个地址？不是值？）。但在地址1000中没有指定值。也就意味着不能访问链接脚本中定义的符号，因为没有值。所能做的只能是访问链接脚本定义的符号的地址。所以当源码使用链接脚本中定义的符号时只能使用符号的地址，不能试图使用其中的值。例如假设你想将一个称为'.ROM'的节的内容复制到一个称为'.FLASH'的节中，链接脚本中的定义如下：


.. code-block:: c
   :caption: c test
   :linenos:
   
   start_of_ROM = .ROM;
   end_of_ROM = .ROM + sizeof (.ROM);
   start_of_FLASH = .FLASH;


复制方式如下：

.. code-block:: c
   :caption: c test
   :linenos:
   
   extern char start_of_ROM,end_of_ROM,start_of_FLASH; (为啥我感觉这样是不对的呢？）
   memcpy (& start_of_FLASH, & start_of_ROM, & end_of_ROM - & start_of_ROM);

注意 '&'操作符的使用。这样使用是正确的，也可以将变量看作一个向量或数组，使用方式可以如下：

.. code-block:: c
   :caption: c test
   :linenos:
   
   extern char start_of_ROM[], end_of_ROM[], start_of_FLASH[];（我就采用这个方式了）
   memcpy (start_of_FLASH, start_of_ROM, end_of_ROM - start_of_ROM);

SECTIONS 命令
"""""""""""""

告诉连接器如何将输入节映射到输出节，输出节如何在内存中存放。格式如下：

.. code-block:: c
   :caption: c test
   :linenos:
   
   SECTIONS
   {
    sections-command
   sections-command
   ...
   }

 每个sections-command可能是以下任意一个：

   1. ENTRY命令
   2. 符号赋值
   3. 输出节描述
   4. 覆盖性描述
   
为了方便在这些命令中使用位置计数器，SECTIONS命令中允许使用ENTRY命令和符号赋值。 这样可以方便链接脚本更容易理解，以在输出文件的布局中的有意义的位置使用这些命令。 

输出节描述和叠加描述描述如下:如果链接脚本中没有使用SECTIONS命令，链接器会将每个输入节放置在一个名称相同的输出节中，以在输入文件出现的顺序存放。第一个节会被存放在地址0

输出节的完整描述为
***************

.. code-block:: c
   :caption: c test
   :linenos:
   
   section [address ] [(type )] :
  	[AT(lma )]
  	[ALIGN(section_align ) | ALIGN_WITH_INPUT]
  	[SUBALIGN(subsection_align )]
  	[constraint ]
  	{
  		output-section-command
  		output-section-command
  	...   
  	} [>region ] [AT>lma_region ] [:phdr :phdr ...] [=fillexp ] [,]


大多数输出节不会使用大多数的节属性。节附近的空格是必须的。明确的节名。冒号和花括号也是必须的。如果使用fillexp，并且下一部分命令看起来像是表达式的延续，则可能需要在末尾加逗号。 换行符和其他空格是可选的。
 每个output-section-command都是下面的一种：
 
 - 符号赋值
 - 输入节描述
 - 直接包含数据的值
 - 特定的输出节关键字

输出节名
*******

输出节名：节名必须与格式相符，例如a.out节名字只能是'.text','.data','.bss'。如果输出格式支持任意数量的节，但是带有数字而不是名称（Oasys就是这种情况），则应使用带引号的数字字符串来提供名称。 节名可以由任何字符序列组成，但是包含任何不寻常字符（例如逗号）的名称必须加引号。 

输出节名称“ / DISCARD /”是特殊的 ，后面描述。

输出节地址
********

输出节地址：这个地址是输出节的VMA类型地址（虚拟地址）。这个地址是可选的，但是，如果提供，则输出地址将完全按照指定的方式设置。 如果没指定，则按照以下原则指定。地址将根据对齐要求进行调整。输出节地址按如下规则产生：
- 如果为该节设置了输出存储区域，则将其添加到该区域，并且其地址将是该区域中的下一个空闲地址。 
- 如果已使用MEMORY命令创建了一个存储区域列表，则选择具有与该节兼容的属性的第一个区域来包含它。 该部分的输出地址将是该区域中的下一个空闲地址； 
- 如果未指定任何存储区域，或者与该部分不匹配，则输出地址将基于位置计数器的当前值进行设置。

例如：

.. code-block:: c
   :caption: c test
   :linenos:
   
   .text . : { *(.text) }和
   .text : { *(.text) }

这两个有细微不同。第一个会将.text输出节的地址设置为定位寄存器的当前值。第二个会设置为当前值并符合严格对齐要求,地址可以是任何表达式;例如，如果要求节 0x10字节对其，节地址的低四位为0，可以这样设置

.. code-block:: c
   :caption: c test
   :linenos:
   
   .text ALIGN(0x10) : { *(.text) }
   

ALIGN 返回基于当前地址计数器 与要求对其数后产生的地址。为一个节制定地址会修改定位计数器的值，前提是节不能为空。（空节是要忽略的）.


输入节描述
*********

输入节描述：最常见的输出节命令是输入节描述。输入节描述是最基本的链接脚本操作。输出节告诉连接器程序的内存布局。可以使用输入节描述来告诉链接器如何将输入文件映射到您的内存布局中。

输入节基本信息

输入节描述由文件名组成，可以选择在文件名后加上括号中的节名列表。 文件名和节名可以用通配符方式，后面会进一步描述。最通用的输入节描述是在输出节中包含所有具有特定名称的输入节，例如，要包括所有输入的“ .text”部分：

.. code-block:: c
   :caption: c test
   :linenos:
   
    *(.text)
    
'*'是一个通配符，适配所有文件名字。用EXCLUDE_FILE列表用于排除掉适配的文件，如：


.. code-block:: c
   :caption: c test
   :linenos:
   
    EXCLUDE_FILE (*crtend.o *otherfile.o) *(.ctors)
    
 EXCLUDE_FILE 也可以放置在节列表中，如：
 
.. code-block:: c
   :caption: c test
   :linenos:
   
   *(EXCLUDE_FILE (*crtend.o *otherfile.o) .ctors)
   
注意：最前面那个'*'通配任意文件。以上两种方式功能等效。如果节列表包含多个节，则支持EXCLUDE FILE的两种语法非常有用，如下所述。 

包含多个节（两种方式）


.. code-block:: c
   :caption: c test
   :linenos:
   
    *(.text .rdata)
    *(.text) *(.rdata)


 第一种会出现.text 和 .rdata交替放置的情况。第二种情况则会聚集存储。
 
 用EXCLUDE_FILE作用与多于一个节时，如果排除项在部分列表中，则排除项仅适用于紧随其后的部分：

.. code-block:: c
   :caption: c test
   :linenos:
   *(EXCLUDE_FILE (*somefile.o) .text .rdata)

则只针对文件列表的.text节，如果想排除掉其中的.rdata节，则需要这样写：

.. code-block:: c
   :caption: c test
   :linenos:
   (EXCLUDE_FILE (*somefile.o) .text EXCLUDE_FILE (*somefile.o) .rdata)


 *或者，将EXCLUDE_FILE 放置在节列表外面，在选择的输入文件前面，这将适用于所有节。前面例子可以这样重写：

.. code-block:: c
   :caption: c test
   :linenos:
   
   EXCLUDE_FILE (*somefile.o) *(.text .rdata)


 您可以指定文件名以包括特定文件中的节。 如果一个或多个文件包含需要位于内存中特定位置的特殊数据，则可以执行此操作。 例如：

.. code-block:: c
   :caption: c test
   :linenos:
   data.o(.data)


要基于输入节的节标记来精炼包含的节,可以使用INPUT_SECTION_FLAGS。这是对ELF部分使用Section标头标志的简单示例 

.. code-block:: c
   :caption: c test
   :linenos:
   SECTIONS {
       .text : { INPUT_SECTION_FLAGS (SHF_MERGE & SHF_STRINGS) *(.text) }
       .text2 : { INPUT_SECTION_FLAGS (!SHF_WRITE) *(.text) }
    }

输出节.text包含输入节中节头标志设置了SHF_MERGE和SHF_STRINGS的*(.text)。输出节.text2包含输入节中节头标志清楚了SHF_WRITE的*(.text)节。

个人理解：INPUT_SECTION_FLAGS针对节头的flags项。

也可以通过写一个模式来适配归档文件中的文件，':'是必须的，':'附近不能有空格。(个人问题：归档文件原理，不理解的)

- 'archive:file':匹配归档中的文件
- 'archive:':匹配整个归档文件
- ':file': 不匹配归档中的文件

“归档”和“文件”之一或两者都可以包含shell通配符。

在基于DOS文件系统上，链接器将假定单个字母后跟一个冒号是驱动器说明符，因此“ c：myfile.o”是一个简单的文件说明 ，而非归档文件匹配。

例如，您不能通过在INPUT命令中使用“ archive：file”从存档中提取文件。

- archive:file'也可以用在EXCLUDE_FILE列表中，但可能不会出现在其他链接描述文件上下文中。如不能通过在INPUT命令中使用“ archive：file”从存档中提取文件。
- 如果使用了没有节列表的文件名字，则输入文件中的所有节会包含在输出节中。这是不常用的，但有时可能会有用，例如：data.o


当您使用的文件名不是“ archive：file”说明符且不包含任何通配符时，链接器将首先查看您是否还在链接器命令行或INPUT命令中指定了文件名。如果指定了，链接器将尝试将其作为输入文件打开，就像它出现在命令行中一样。 如果没有指定，连接器将尝试将其作为输入文件（注意，这是哪个文件？那个没有匹配到内容的文件名）打开，就像它出现在命令行中一样。请注意，这与INPUT命令不同，因为链接器不会在归档搜索路径中搜索文件。


输入节通配符模式

在输入节描述中，文件名字或节名或两者同时是通配符模式。通配符描述如下：

- '*'：匹配任意个数的字符
- '?'：匹配任意一个字符
- '[chars]':匹配任何字符的单个实例，通配符将不匹配“ /”字符 （用于在Unix上分隔目录名称 )。由单个“ *”字符组成的模式是一个例外 ;将始终匹配任何文件名，无论它是否包含“ /” .在一个节名中，匹配符号也可匹配'/'字符。
- '\ ':引用以下字符

文件名通配符模式仅与在命令行或INPUT命令中明确指定的文件匹配。链接器不会搜索扩展通配符匹配的目录。
如果文件名与多个通配符模式匹配，或者文件名显式出现并且也由通配符模式匹配，则链接器将使用链接器脚本中的第一个匹配项。例如，此输入节描述的顺序可能有误，因为将不使用“ data.o”规则（如何理解？）：

.. code-block:: c
   :caption: c test
   :linenos:
   
   .data : { *(.data) }
   .data1 : { data.o(.data) }

通常，链接器将按通配符匹配的顺序放置文件和节。 您可以使用SORT_BY_NAME关键字更改此关键字，该关键字会出现在括号中的通配符模式之前（例如SORT_BY_NAME（.text *））。 当使用SORT_BY_NAME关键字时，链接器将文件或节按名称升序排序，然后再将它们放置在输出文件中。
SORT_BY_ALIGNMENT与SORT_BY_NAME类似。 SORT_BY_ALIGNMENT将对部分进行排序 在将它们放入输出文件之前，先按对齐的降序排列。 在较小的对齐方式之前放置较大的对齐方式可以减少所需的填充量。 SORT_BY_INIT_PRIORITY也类似于SORT_BY_NAME。 SORT_BY_INIT_PRIORITY将把节按在节名称中编码的GCC初始化优先级属性的升序排列，然后再将它们放入输出文件中。 在.init_array.NNNNN和.fini_array.NNNNN中，NNNNN是初始化优先级。 在.ctors.NNNNN和.dtors.NNNNN中，NNNNN为65535减去初始优先级。 
SORT是SORT_BY_NAME的别名。 当链接描述文件中有嵌套的节排序命令时，节排序命令最多可以有1个嵌套级别。 

1. SORT_BY_NAME（SORT_BY_ALIGNMENT（通配符部分模式））。 如果两个节的名称相同，它将首先按名称对输入节进行排序，然后按对齐方式对输入节进行排序。
2.  SORT_BY_ALIGNMENT（SORT_BY_NAME（通配符部分模式））。 它将首先按对齐方式对输入节进行排序，如果两个节具有相同的对齐方式，则将按名称进行排序。
3. SORT_BY_NAME（SORT_BY_NAME（通配符部分模式））与SORT_BY_NAME（通配符部分模式）相同。
4. SORT_BY_ALIGNMENT（通配符部分模式）（SORT_BY_ALIGNMENT）与SORT_BY_ALIGNMENT（通配符部分模式）相同。
5. 所有其他嵌套节排序命令均无效。 

当同时使用命令行节排序选项和链接程序脚本节排序命令时，节排序命令始终优先于命令行选项。如果未嵌套链接程序脚本中的节排序命令，则命令行选项将使 该部分排序命令将被视为嵌套排序命令。 

1. 具有“ --sort-sections对齐方式”的SORT_BY_NAME（通配符部分模式）等效于SORT_BY_NAME（SORT_BY_ALIGNMENT（通配符部分模式））。
2. 具有“ --sort section name”的SORT_BY_ALIGNMENT（通配符部分模式）等效于SORT_BY_ALIGNMENT（SORT_BY_NAME（通配符部分模式））。

如果链接脚本中的节排序命令是嵌套的，则命令行选项将被忽略。

SORT_NONE通过忽略命令行节排序选项来禁用节排序。如果您对输入节的去向感到困惑，请使用“ -M”链接器选项来生成地图文件。 映射文件精确显示了输入节如何映射到输出节。 

本示例说明了如何使用通配符模式对文件进行分区。 此链接描述文件指示链接器将所有“ .text”部分放置在“ .text”中，并将所有“ .bss”部分放置在“ .bss”中 

链接器会将所有文件中的“ .data”部分放在“ .DATA”中以大写字母开头的位置； 对于所有其他文件，链接器会将“ .data”部分放在“ .data”中。

.. code-block:: c
   :caption: c test
   :linenos:``
   SECTIONS {
    .text : { *(.text) }
    .DATA : { [A-Z]*(.data) }
    .data : { *(.data) }
    .bss : { *(.bss) }
 }
 

输入部分的通用符号。

通用符号需要一些注意事项，在很多目标文件格式的通用符号中不包含在特定的输入节中。链接器将常见符号放置在名字为“ COMMON”的输入节中。

可以在“ COMMON”节中使用文件名，就像在其他任何输入节中一样。



可以使用此命令将特定输入文件中的公共符号放置在一个节中，而将其他输入文件中的公共符号放置在另一个节中。



在多数情况下，输入文件中的通用符号将会放置的输出文件中的’.bss'节中，例如：

.. code-block:: c
   :caption: c test
   :linenos:``
   .bss { *(.bss) *(COMMON)}

可能有多于一个COMMON节，如MIPS中的“COMMON”和'.scommon'。方便各自映射。

在旧版本的连接器脚本可以看到'[COMMON]'。认为已经过时了。与'*(COMMON)'等效。

输入节和垃圾收集（Input Section and Garbage Collection）

使用链接时垃圾收集时（"--gc-sections"),如果将节标注为不能修改，很多时候说明节是有用的。通过KEEP（）关键字来实现，如：

KEEP(*(.init)) or KEEP(SORT_BY_NAME(*)(.ctors))。

输入节例子：



它们分别指示输出部分的开始地址和结束地址。 注意：大多数节名称不能用C标识符表示，因为它们包含一个'.'字符。



注意：如果有多个匹配条件，应该输入只匹配一次输出吧。（要确认）

注意：大多数节名称不能用C标识符表示，因为它们包含一个“.”字符。 

输出节数据

通过将BYTE，SHORT，LONG，QUAD或SQUAD用作输出节命令，可以在输出节中包含数据的显式字节。 每个关键字后跟一个括号中的表达式，该表达式提供要存储的值 。表达式的值存储在位置计数器的当前值。 BYTE，SHORT，LONG和QUAD命令存储一个，两个，四个和八个字节 。存储字节后，位置计数器将增加所存储的字节数。 

例如，这将存储字节1，后跟符号“ addr”的四个字节值 

BYTE(1)
LONG(addr)

关于字节序，如果没有显式指定，则用第一个输入目标文件的字节序。

注意：这些指令只在节描述符内部使用，不能在节描述符之间使用，所以以下连接器会产生错误：

SECTIONS { .text : { *(.text) } LONG(1) .data : { *(.data) } }

应该这样写：

SECTIONS { .text : { *(.text) ; LONG(1) } .data : { *(.data) } }

可以使用FILL命令设置当前节的填充样式。其后是括号中的表达式。该节中任何其他未指定的内存区域（例如，由于输入节的必需对齐而留下的间隙）都填充有表达式的值，并根据需要重复进行。FILL语句覆盖节定义中发生该点之后的内存位置。通过包含多个FILL语句，可以在一个输出节中的不同部分用不同的填充模式。

如采用’0x90'填充未指定的内存区域：

FILL(0x90909090) 

FILL指令与‘=fillexp’输出节属性效果一致，但只影响节中FILL指令后的部分。如果同时使用了这两种方式，则FILL指令优先。


输出节关键字
**********

以下两个关键字可以作为输出节命令。

- CREATE_OBJECT_SYMBOLS:命令告诉里那节气为每个输入文件创建一个符号。符号的名字与对应输入文件的名字相同。符号对应的节将出现在出现此命令的输出节中。通常用于a.out目标文件格式。一般不用于其他目标文件格式。
- CONSTRUCTIORS：当使用a.out对象文件格式进行链接时，链接器使用不寻常的set构造来支持C ++全局构造函数和析构函数。 链接不支持任意节的目标文件格式（例如ECOFF和XCOFF）时，链接器将按名称自动构造结构。 对于这些目标文件格式，CONSTRUCTORS命令告诉链接器将构造函数信息放置在CONSTRUCTORS命令出现的输出部分。 对于其他目标文件格式，将忽略CONSTRUCTORS命令。
 符号__CTOR_LIST__标识全局构建开始，__CTOR_END__标识结束。同样的 __DTOR_LIST__ 和 __DTOR_END__标识析构器。
 列表中的第一个字代表入口数，以一个为0的字结束。编译器必须安排实际运行的代码。针对这些目标文件格式 GNU C++ 通常从子进程__main中调用构造函数。对__main的调用会自动插入到main的启动代码中。gnu C ++通常通过使用atexit或直接从函数出口运行析构函数。
 像COFF，ELF支持节属性的目标文件格式，GNU C++通常情况下将全局构建和析构函数分别放置在.ctors 和 .dtors节中。将以下序列放入您的链接器脚本中，将构建gnu C ++运行时代码希望看到的表的种类。 
.. code-block:: c
   :caption: c test
   :linenos:``
   __CTOR_LIST__ = .;
     LONG((__CTOR_END__ - __CTOR_LIST__) / 4 - 2)
     *(.ctors)
     LONG(0)
     __CTOR_END__ = .;
     __DTOR_LIST__ = .;
     LONG((__DTOR_END__ - __DTOR_LIST__) / 4 - 2)
      *(.dtors)
      LONG(0)
     __DTOR_END__ = .;

   如果您使用的是gnu C ++支持的初始化优先级，可以控制全局构造函数的运行顺序。
   
.. code-block:: c
   :caption: c test
   :linenos:``
   ld --verbose可进行查看
   
   必须对构造函数进行排序来保证运行时构造函数的执行顺序的正确性。当使用CONSTRUCTORS命令时，使用‘SORT_BY_NAME(CONSTRUCTORS)'代替。当使用 .ctors 和 .dtors'*节时，使用'*(SORT_BY_NAME(.ctors))'和'*(SORT_BY_NAME(.dtors))'来代替'*(.ctors)'和'*(.dtors)'来实现排序。通常情况下编译器和连接器会自动处理这些操作，不需要手动处理。如果使用的是C ++并编写自己的链接描述文件，则可能需要考虑这一点。

.. code-block:: c
   :caption: c test
   :linenos:``
   .foo : { *(.foo) }


输出节中丢弃部分
*************

连接器通常不创建不包含任何内容的节。当引用任何输入文件中可能存在或可能不存在的输入节时，这是为了方便起见。


输出节属性
*********

输出节的完整描述如下：

.. code-block:: c
   :caption: c test
   :linenos:``
   .foo : { *(.foo) }
   
   section [address ] [(type )] :
   [AT(lma )]
   [ALIGN(section_align ) | ALIGN_WITH_INPUT]
   [SUBALIGN(subsection_align )]
   [constraint ]
   {
   output-section-command
   output-section-command
   ...
   } [>region ] [AT>lma_region ] [:phdr :phdr ...] [=fillexp ]


前面我们已经描述了节,地址和输出节命令.接下来对剩下的节属性进行描述.

输出节类型（type)
****************

每个输出节可以有一个类型（type)。类型是一个存在括号中的关键字。类型定义如下：

- NOLOAD：节标记为不能加载的节，所以当成寻运行是不加载进内存。
- DSECT
  COPY
  INFO
  OVERLAY:为了向后兼容。很少用到。有相同的作用：节应该标记为不可非陪的，所以当程序运行时并不会为节分配内存

连接器通常基于映射到输出节的输入节的属性来设置输出节的属性。可以使用节类型来进行覆盖。如以下的脚本例子，‘ROM’节在内存地址‘0’处，不需要在程序运行时进行加载：

.. code-block:: c
   :caption: c test
   :linenos:``
   .foo : { *(.foo) }
   
   SECTIONS {
   ROM 0 (NOLOAD) : { ... }
   ...
   }


输出节 LMA
**********

每个节有一个虚拟地址（VMA）和一个加载地址（LMA：Load address),VMA已经描述过。加载地址通过AT或AT > keywords来指定。指定LMA是可选的。

AT关键字后边跟一个表达式作为参数。指定节加载的确切地址。AT > 关键字 以一个内存区域的名字作为参数。LMA设置为区域中下一个空闲地址，并满足节的对齐要求。

如果没有为一个可分配的节指定AT和AT >制定，连接器会使用启发式方式来确定加载地址：

- 如果节指定了VMA地址，则将LMA设置为VMA的值。
- 如果节是不能分配的，则LMA设置为VMA的值。
- 否则，如果找到与当前节兼容的内存区域，这个区域包含至少一个节，然后设置LMA，因此VMA和LMA之间的差异与所定位区域中最后一部分的VMA和LMA之间的差值相同。 
- 如果未声明内存区域，则在上一步中使用覆盖整个地址空间的默认区域。 
- 如果没有找到合适的去怒，或者前面没有节，则将LMA设置为VMA相同的值。

此功能旨在使其易于构建ROM影响。例如，以下脚本创建了三个输出节：‘.text’，从0x1000开始，‘.mdata'即使其VMA为0x2000,也是加载到'.text'节的后面。‘.bss’含有未初始化的数据，在地址0x3000。符号 _data定义的值为0x2000，表示定位计数器包含VMA的值，而不是LMA的值。（个人理解：所以没有显示定义LMA时，默认LMA = LMA = .)




.. code-block:: c
   :caption: c test
   :linenos:``
   .foo : { *(.foo) }
   ECTIONS

  {

   .text 0x1000: { *(.text) _etext = .; }

   .mdata 0x2000 :
    AT ( ADDR (.text) + SIZEOF (.text) )
     { _data = . ; *(.data); _edata = . ; }
    .bss 0x3000 :
    { _bstart = .; *(.bss) *(COMMON); _BEND = .;}
   }
   
使用以上脚本的程序生成的运行时初始化代码会包含像下面一样的代码，从ROM 映像中复制初始化数据到运行时地址。请注意，此代码如何利用链接器脚本定义的符号。

.. code-block:: c
   :caption: c test
   :linenos:
   
   extern char _etext,_data,_edata,_bstart,_bend;
   char *src = &_etext;
   char *dst = &_data;
   /* ROM has data at end of text;copy it;*/
   while(dst < &_edata)

  *dst++ = *src++;
  /* Zero bss. */
  for (dst = &_bstart;dst < &_bend; dst++)

  *dst = 0;


强制输出对齐

可以利用ALIGN来增加输出节的对齐。作为替代方案，您可以使用ALIGN_WITH_INPUT属性强制执行VMA和LMA之间的差异在整个输出部分中保持完整。


强制输入对齐

通过使用SUBALIGN来强制输入节在输出节中对齐。指定的值覆盖掉输入节中给出的对齐值。不过更大或更小。

输出节约束

可以通过ONLY_IF_RO和ONLY_IF_RW来指定仅当所有输入节同时为只读或读写时才创建对应的输出节。

输出节区域

可以使用'>region'来将一个已经定义的内存区域指定给一个节。如

.. code-block:: c
   :caption: c test
   :linenos:
   
   MEMORY { rom : ORIGIN = 0x1000, LENGTH = 0x1000 }
   SECTIONS { ROM : { *(.text) } >rom }
   
   
输出节 Phdr设置

可以使用‘：phdr'将节指定到一定定义的程序段中。如果一个节同时指定到一个或多个段，然后，所有后续分配的部分也将被分配给这些段，除非显式使用':phdr'进行修改。可以使用:NONE来告诉连接器不要将节放入任何一个段中（这是不用加载吗？）。

例如：   
   
.. code-block:: c
   :caption: c test
   :linenos:
   PHDRS { text PT_LOAD ; }
   SECTIONS { .text : { *(.text) } : text }
   
输出节填充

可以使用'= fillexp'设置整个部分的填充模式。其中fillexp是一个表达式。输出节中任何其他未使用的内存区域（如，输入节中因为对齐要求产生的空洞）将会填充为指定的值，根据需要进行重复。如果填充表达式是一个简单的十六制数字，如以'0x'开头的十六制数字的字符串没有以'k'或'M'结尾，然后可以使用任意长的十六进制数字序列来指定填充模式 ;打头的零也成为模式的一部分。其他情况下，包含额外的括号或'+',填充模式是表达式值的四个最低有效字节.所有情况下，数字采用大端模式。

可以使用输出节中的FILL命令来更改填充值。

.. code-block:: c
   :caption: c test
   :linenos:
   
   SECTIONS { .text : { *(.text) } =0x90909090 }



Overlay 描述
************

提供了一个容易的方法来描述作为单个内存映像的一部分加载并在相同的地址运行的节。运行时，

某种覆盖管理器将根据需要在运行时内存地址中复制覆盖部分，也许是简单地操纵寻址位。

这种方法是有效的，例如，当某个内存区域比另一个区域更快。用OVERLAY命令来描述Overlays。OVERLAY命令与输出节描述符一样，在SECTIONS命令中使用。完整语法如下：


.. code-block:: c
   :caption: c test
   :linenos:
   
   OVERLAY [start ] : [NOCROSSREFS] [AT ( ldaddr )]
   {
   secname1
   {
   output-section-command
   output-section-command
   ...
   } [:phdr ...] [=fill ]
   secname2
   {
   output-section-command
   output-section-command
   ...
   } [:phdr ...] [=fill ]
   ...
   } [>region ] [:phdr ...] [=fill ] [,]


除了OVERLAY（关键字）外其他都是可选的，每个节必须有一个名字（secname1 和 secname2)。OVERLAY构建中节的定义与SECTIONS构建中的节定义是相同的，只是OVERLAY中节的定义不指定地址和内存区域。

如果使用的fill指令，并且后面的节指令与前面的表达式看起来一样时，需要在fill指令结束后加一个’，'(逗号）
......


内存指令
"""""""

链接器的默认配置允许分配所有可用内存。 您可以使用MEMORY命令覆盖此内容。 



上面格式没有弄好，下面继续：


PHDRS命令
""""""""

ELF目标文件格式使用程序标头，也称为段。 程序标头描述了如何将程序加载到内存中。 您可以使用带有-p选项的objdump程序将其打印出来。 

在本机ELF系统上运行ELF程序时，系统加载程序会读取程序标头，以弄清楚如何加载程序。 仅在正确设置程序头的情况下，这才起作用。 本手册没有详细介绍系统加载程序如何解释程序头。 有关更多信息，请参见ELF ABI。 

默认情况下，链接器将创建合理的程序头。 但是，在某些情况下，您可能需要更精确地指定程序头。 为此，您可以使用PHDRS命令。 当链接器在链接器脚本中看到PHDRS命令时，它将不创建指定程序头以外的任何程序头。 

链接器在生成ELF输出文件时仅注意PHDRS命令，在其他情况下，链接器将仅忽略PHDRS。

这是PHDRS命令的语法。 单词PHDRS，FILEHDR，AT和FLAGS是关键字。 

.. code-block:: c
   :caption: c test
   :linenos:
   PHDRS
  {
	name type [ FILEHDR ] [ PHDRS ] [ AT ( address ) ]
		[ FLAGS ( flags ) ] ;
  }


该名称仅用于链接描述文件的SECTIONS命令中的参考。 它不会放入输出文件中。 程序标头名称存储在单独的名称空间中，不会与符号名称，文件名或节名称冲突。 每个程序头必须有一个不同的名称。 标头是按顺序处理的，通常它们以升序加载地址的顺序映射到段。 

某些程序头类型描述了系统加载程序将从文件中加载的内存段。 在链接器脚本中，可以通过将可分配的输出节放置在段中来指定这些段的内容。 您可以使用'：phdr'输出节属性来将节放置在特定的句段中。 请参见第73.3.6节[输出部分Phdr]，第73页。通常将某些部分放在一个以上的段中。 这仅意味着一个内存段包含另一段。 您可以重复‘：phdr’，对应该包含该节的每个段使用一次。

如果使用'：phdr'将一个节放在一个或多个段中，则链接器会将所有未指定'：phdr'的后续可分配节放在同一段中。 这是为了方便起见，因为通常将一整套连续部分放在单个段中。 您可以使用：NONE覆盖默认段，并告诉链接器不要将该段放在任何段中。 

您可以在程序标头类型之后使用FILEHDR和PHDRS关键字来进一步描述段的内容。 FILEHDR关键字意味着该段应包含ELF文件头。 PHDRS关键字表示该细分受众群应包含 ELF程序标头本身。 如果应用于可装入段（PT_LOAD），则所有先前的可装入段都必须具有以下关键字之一。 

类型可以是以下之一。 数字表示关键字的值。 

- PT_NULL (0)：
- PT_LOAD (1)：
- PT_DYNAMIC (2)：
- PT_INTERP (3)：
- PT_NOTE (4)：
- PT_SHLIB (5)：
- PT_PHDR (6)：
- PT_TLS (7)：

expression：该表达式给出程序头的数字类型。 这可以用于上面未定义的类型。 

您可以使用AT表达式指定将段加载到内存中的特定地址。 这与用作输出节属性的AT命令相同（请参见第3.6.8.2节[输出节LMA]，第71页）。 程序头的AT命令将覆盖输出部分属性。 

链接器通常会根据组成段的段来设置段标志。 您可以使用FLAGS关键字来显式指定段标志。 标志的值必须是整数。 它用于设置程序头的p_flags字段。 这是PHDRS的示例。 这显示了在本机ELF系统上使用的一组典型的程序头。 

.. code-block:: c
   :caption: c test
   :linenos:
   PHDRS
   {
	headers PT_PHDR PHDRS ;
	interp PT_INTERP ;
	text PT_LOAD FILEHDR PHDRS ;
	data PT_LOAD ;
	dynamic PT_DYNAMIC ;
   }
   SECTIONS
   {
	. = SIZEOF_HEADERS;
	.interp : { *(.interp) } :text :interp
	.text : { *(.text) } :text
	.rodata : { *(.rodata) } /* defaults to :text */
	...
	. = . + 0x1000; /* move to a new page in memory */
	.data : { *(.data) } :data
	.dynamic : { *(.dynamic) } :data :dynamic
	...
   }



VERSION 指令
""""""""""""

使用ELF时，链接器支持符号版本。 符号版本仅在使用共享库时有用。 当动态链接程序运行可能已与共享库的早期版本链接的程序时，它可以使用符号版本来选择函数的特定版本。 

您可以直接在主链接程序脚本中包含版本脚本，也可以将版本脚本作为隐式链接程序脚本提供。 您也可以使用“ --version-script”链接器选项。 

VERSION命令的语法很简单 ：

.. code-block:: c
   :caption: c test
   :linenos:
   VERSION { version-script-commands }


版本脚本命令的格式与Sun的链接程序在Solaris 2.5中使用的格式相同。 版本脚本定义了版本节点树。 您可以在版本脚本中指定节点名称和相互依赖性。 您可以指定将哪些符号绑定到哪个版本节点，并且可以将一组指定的符号减少到本地范围，以使它们在共享库的外部不全局可见。 

演示版本脚本语言的最简单方法是使用一些示例。 



.. code-block:: c
   :caption: c test
   :linenos:
   VERS_1.1 {
	global:
	foo1;
	local:
	old*;
	original*;
	new*;
   };
   VERS_1.2 {
	foo2;
   } VERS_1.1;
   VERS_2.0 {
	bar1; bar2;
	extern "C++" {
		ns::*;
		"f(int, double)";
	};
   } VERS_1.2;



此示例版本脚本定义了三个版本节点。 定义的第一个版本节点为“ VERS_1.1”； 它没有其他依赖项。 该脚本将符号“ foo1”绑定到“ VERS_1.1”。它会将符号数量减少到本地范围，以使它们在 共享库 这是使用通配符模式完成的，因此任何名称以“ old”，“ original”或“ new”开头的符号都将匹配。 匹配文件名时，可用的通配符模式与Shell中使用的通配符模式相同（也称为“ globbing”）。 然而， 如果在双引号中指定符号名称，则该名称将被视为文字，而不是全局模式。 

接下来，版本脚本定义节点“ VERS_1.2”。 该节点取决于“ VERS_1.1”。 该脚本将符号“ foo2”绑定到版本节点“ VERS_1.2”。

最后，版本脚本定义了节点“ VERS_2.0”。 该节点取决于“ VERS_1.2”。

脚本将符号“ bar1”和“ bar2”绑定到版本节点“ VERS_2.0”。 

当链接器在库中定义的未特定绑定到版本节点的符号时，它将有效地将其绑定到库的未指定基本版本。 您可以通过在版本脚本中的某个位置使用“ global：*;”将所有其他未指定的符号绑定到给定的版本节点。 请注意，在全球规范中使用通配符有点疯狂，但最后一个版本节点除外。 其他地方的全局通配符冒着将符号意外添加到旧版本导出的集合中的风险。 这是错误的，因为较旧的版本应具有一组固定的符号。

版本节点的名称除了它们可能向阅读它们的人建议的含义外，没有其他特殊含义。 “ 2.0”版本也可能出现在“ 1.1”和“ 1.2”之间。 但是，这将是编写版本脚本的一种令人困惑的方法。 

如果节点名称是版本脚本中的唯一版本节点，则可以省略。 这样的版本脚本不会为符号分配任何版本，只会选择哪些符号在全局范围内可见，哪些不会。 


.. code-block:: c
   :caption: c test
   :linenos:
   { global: foo; bar; local: *; };


当您将应用程序链接到具有版本控制符号的共享库时，应用程序本身知道它需要的每个符号的版本，并且还知道要链接到的每个共享库中所需的版本节点。 因此，在运行时，动态加载程序可以进行快速检查，以确保所链接的库确实提供了应用程序解析所有动态符号所需的所有版本节点。 通过这种方式，动态链接器可以确定地知道它所需要的所有外部符号将是可解析的，而不必搜索每个符号引用。 

实际上，符号版本控制是进行SunOS进行次要版本检查的一种更为复杂的方法。 此处要解决的基本问题是，通常在需要时绑定对外部函数的引用，并且在应用程序启动时不会全部绑定。 如果共享库已过期，则可能缺少所需的接口； 当应用程序尝试使用该接口时，它可能会突然意外失败。 使用符号版本控制时，如果与该应用程序一起使用的库太旧，则用户在启动程序时将收到警告。

Sun的版本控制方法有多个GNU扩展。 其中的第一个功能是将符号绑定到源文件中的版本节点的能力，在该节点中已定义符号而不是在版本控制脚本中。 这样做主要是为了减轻图书馆维护人员的负担。 您可以通过输入以下内容来做到这一点： 


.. code-block:: c
   :caption: c test
   :linenos:
   __asm__(".symver original_foo,foo@VERS_1.1");


在C源文件中。 这会将功能“ original_foo”重命名为绑定到版本节点“ VERS_1.1”的“ foo”的别名。 “ local：”指令可用于阻止导出符号“ original_foo”。 “ .symver”指令优先于版本脚本。 

第二个GNU扩展是允许同一功能的多个版本出现在给定的共享库中。 这样，您可以对接口进行不兼容的更改，而无需增加共享库的主要版本号，同时仍然允许与旧接口链接的应用程序继续运行。 

为此，您必须在源文件中使用多个“ .symver”指令。 这是一个例子： 

.. code-block:: c
   :caption: c test
   :linenos:
   __asm__(".symver original_foo,foo@");__asm__(".symver old_foo,foo@VERS_1.1");__asm__(".symver old_foo1,foo@VERS_1.2");__asm__(".symver new_foo,foo@@VERS_2.0");


在此示例中，“ foo @”代表符号“ foo”绑定到未指定的基本版本。 包含此示例的源文件将定义4个C函数：“ original_foo”，“ old_foo”，“ old_foo1”和“ new_foo”。

当给定符号有多个定义时，需要某种方式指定默认版本，该符号的外部引用将绑定到该默认版本。 您可以使用“ .symver”指令的“ foo @@ VERS_2.0”类型执行此操作。 您只能以这种方式将符号的一个版本声明为默认版本。 否则，您将有效地对同一符号进行多个定义。 

如果希望将引用绑定到共享库中特定版本的符号，则可以使用方便的别名（即“ old_foo”），也可以使用“ .symver”指令专门绑定到外部 有关功能的版本。 您还可以在版本脚本中指定语言： 

.. code-block:: c
   :caption: c test
   :linenos:
   VERSION extern "lang" { version-script-commands }


支持的“ lang”是“ C”，“ C ++”和“ Java”。 链接器将在链接时遍历符号列表，并根据“ lang”对它们进行解映射，然后再将它们与“ version-script-commands”中指定的模式进行匹配。 默认的“ lang”是“ C”。 

杂乱的名称可能包含空格和其他特殊字符。 如上所述，您可以使用glob模式来匹配不匹配的名称，也可以使用双引号引起来的字符串来完全匹配该字符串。 在后一种情况下，请注意，版本脚本和demangler输出之间的细微差异（例如不同的空格）将导致不匹配。 由于分解器生成的确切字符串将来可能会更改，即使名称不正确，您也应该检查所有版本指令的行为是否符合升级时的预期。 


链接描述文件中的表达式
""""""""""""""""""

链接程序脚本语言中表达式的语法与C表达式相同。 所有表达式均以整数形式求值。 所有表达式都以相同的大小求值，如果主机和目标都为32位，则为32位，否则为64位。 

您可以在表达式中使用和设置符号值。

链接器定义了几个专用于表达式的内置函数。 

.. code-block:: c
   :caption: c test
   :linenos:
   _fourk_1 = 4K；
   _fourk_2 = 4096；
   _fourk_3 = 0x1000;
   _fourk_4 = 10000o


注意 - K 和 M 后缀不能与提到的基本后缀一起使用

常数
********
所有的常数都是整型变量。

与C中一样，链接器将以“ 0”开头的整数视为八进制，而将以“ 0x”或“ 0X”开头的整数视为十六进制。 或者，链接器接受后缀“ h”或“ H”代表十六进制，“ o”或“ O”代表八进制，“ b”或“ B”代表二进制，而“ d”或“ D”代表十进制。 

任何不带前缀或后缀的整数值均视为十进制。

 此外，可以使用后缀K和M分别将常数缩放1024或1024^2。 

例如，以下所有都是指相同的数量： 

.. code-block:: c
   :caption: c test
   :linenos:
   _fourk_1 = 4K;_fourk_2 = 4096;_fourk_3 = 0x1000;_fourk_4 = 10000o;


注意-K和M后缀不能与上述基本后缀一起使用。 


符号常量
*********

可以通过使用CONSTANT（name）运算符来引用特定于目标的常量，其中name是以下之一： 

- MAXPAGESIZE:目标的最大页面尺寸。 
- COMMONPAGESIZE:目标的默认页面大小。

因此，例如：

.. code-block:: c
   :caption: c test
   :linenos:
   .text ALIGN (CONSTANT (MAXPAGESIZE)) : { *(.text) }


将创建与目标所支持的最大页面边界对齐的文本部分。  

符号名称
*******
除非加引号，否则符号名称以字母，下划线或句点开头，并且可以包括字母，数字，下划线，句点和连字符。 未加引号的符号名称不得与任何关键字冲突。 您可以使用双引号将符号名称括起来，以指定包含奇数字符或与关键字名称相同的符号： 


.. code-block:: c
   :caption: c test
   :linenos:
   "SECTION" = 9;"with a space" = "also with a space" + 10;

由于符号可以包含许多非字母字符，因此用空格定界符号是最安全的。 例如，“ A-B”是一个符号，而“ A-B”是一个包含减法的表达式。 

孤儿节
******

孤立节是输入文件中存在的节，链接器脚本未将这些节显式放置到输出文件中。 链接器仍然可以通过查找或创建合适的输出节（将孤立的输入节放置在其中）将这些节复制到输出文件中。 

如果孤立输入节的名称与现有输出节的名称完全匹配，则孤立输入节将放置在该输出节的末尾。 

如果没有名称匹配的输出节，则将创建新的输出节。 

每个新的输出节将与放置在其中的孤立节具有相同的名称。 

如果有多个同名的孤立节，则将它们全部合并为一个新的输出节。 

如果创建了新的输出节以容纳孤立的输入节，则链接器必须决定相对于现有输出节将这些新的输出节放置在何处。 在大多数现代目标上，链接器会尝试将孤立的节放在同一属性的节之后，例如代码与数据，可加载与不可加载等。如果找不到具有匹配属性的节，或者您的目标缺少此支持，则链接器 孤立部分位于文件的末尾。 

命令行选项“ --orphan-handling”和“ --unique”（请参阅第2.1节[命令行选项]，第3页）可用于控制将孤儿放在哪个输出节中。 

位置计数器
**********

特殊链接器变量点“.”始终包含当前输出位置计数器。 自从。 总是引用输出节中的位置，它只能出现在SECTIONS命令中的表达式中。 这 . 符号可以出现在表达式中允许使用普通符号的任何位置。 

为分配一个值. 将导致位置计数器被移动。 这可用于在输出部分中创建孔. 位置计数器可能不会在输出部分内部向后移动，也可能不会在输出部分外部向后移动，如果这样做会创建具有重叠LMA的区域。 

.. code-block:: c
   :caption: c test
   :linenos:
   
   SECTIONS{	output :	{		file1(.text)		. = . + 1000;		file2(.text)		. += 1000;		file3(.text)	} = 0x12345678;}


在上一个示例中，“ file1”中的“ .text”部分位于输出部分“ output”的开头。 随后是一个1000字节的间隙。 然后出现“ file2”中的“ .text”部分，并且在“ file3”中“ .text”部分之前也有一个1000字节的间隙。 标记“ = 0x12345678”指定要在间隙中写入哪些数据（请参见第3.6.8.8节“输出节填充”，第73页）。

注意： . 实际上是指从当前包含对象的开头开始的字节偏移量. 通常，这是SECTIONS语句，其起始地址为0，因此为. 可以用作绝对地址。 如果 . 在节描述中使用“ 0”，但是它指的是距该节开始处的字节偏移量，而不是绝对地址。 因此，在这样的脚本中： 

.. code-block:: c
   :caption: c test
   :linenos:
   SECTIONS{	. = 0x100	.text: {		*(.text)		. = 0x200	}	. = 0x500	.data: {	*(.data)	. += 0x600	}}




即使“ .text”输入部分中的数据不足以填充该区域，也会为“ .text”部分分配0x100的起始地址，大小恰好为0x200字节。 （如果数据太多，将产生错误，因为这是向后移动的尝试）。 “ .data”部分将从0x500开始，并且在“ .data”输入部分的值结束之后和“ .data”输出部分本身结束之前还有0x600字节的额外空间。 

如果链接器需要放置孤立节，则在输出节语句之外将符号设置为位置计数器的值可能会导致意外的值。 例如，给出以下内容：


.. code-block:: c
   :caption: c test
   :linenos:
   
   SECTIONS{	start_of_text = . ;	.text: { *(.text) }	end_of_text = . ;	start_of_data = . ;	.data: { *(.data) }	end_of_data = . ;}


如果链接器需要放置一些输入部分，例如 .rodata（脚本中未提及），它可能会选择将该部分放在.text和.data之间。 您可能会认为链接器应该在上面的脚本的空白行上放置.rodata，但是空白行对链接器没有特别的意义。 同样，链接器也没有将上述符号名称与其部分相关联。 而是假定所有赋值或其他语句都属于上一个输出节，但对..的特殊情况除外，即，链接器将放置孤立的.rodata节，就像脚本编写如下


.. code-block:: c
   :caption: c test
   :linenos:
   SECTIONS{	start_of_text = . ;	.text: { *(.text) }	end_of_text = . ;	start_of_data = . ;	.rodata: { *(.rodata) }	.data: { *(.data) }	end_of_data = . ;}


这可能不是脚本作者对start_of_data的值的意图。 影响孤立节放置的一种方法是将位置计数器分配给它自己，因为链接器假定已将分配给。 正在设置下一个输出部分的起始地址，因此应与该部分分组。 所以你可以这样写： 

.. code-block:: c
   :caption: c test
   :linenos:
   SECTIONS{	start_of_text = . ;	.text: { *(.text) }	end_of_text = . ;	. = . ;	start_of_data = . ;	.data: { *(.data) }	end_of_data = . ;}


现在，孤立的.rodata节将放置在end_of_text和start_of_data之间。 

操作符
******

链接器识别具有标准绑定和优先级的标准C组算术运算符： 

![](/assets/images/linux_img/ld/ld_p89.png)


评估
****

链接器懒惰地计算表达式。 仅在绝对必要时才计算表达式的值。 

链接器需要一些信息，例如第一节的起始地址的值以及存储区域的起点和长度，以便进行任何链接。 当链接器读取链接器脚本时，将尽快计算这些值。 

但是，直到存储分配之后，其他值（例如符号值）才是未知的或不需要的。 当其他信息（例如输出节的大小）可用于符号分配表达式中时，稍后将评估此类值。 

直到分配后才能知道节的大小，因此直到分配后才执行取决于这些节的分配。 

某些表达式，例如取决于位置计数器“。”的表达式，必须在节分配期间求值。 

如果需要表达式的结果，但该值不可用，则将导致错误。 例如，以下脚本 


.. code-block:: c
   :caption: c test
   :linenos:
   SECTIONS{	.text 9+this_isnt_constant :	{ *(.text) }}


将导致错误消息“初始地址的非常量表达式”。 

表达式部分
********

地址和符号可以是相对的，也可以是绝对的。 节的相对符号是可重定位的。 如果您使用'-r'选项请求可重定位输出，则进一步的链接操作可能会更改部分相对符号的值。 另一方面，绝对符号将在所有进一步的链接操作中保留相同的值。 

链接器表达式中的某些术语是地址。 对于部分相对符号和返回地址的内置函数（例如ADDR，LOADADDR，ORIGIN和SEGMENT_START）而言，都是如此。 其他术语只是数字，或者是返回非地址值的内置函数，例如LENGTH。 一种复杂的情况是，除非您设置LD_FEATURE（“ SANE_EXPR”）（请参见第3.4.5节[其他命令]，第56页），否则数字和绝对符号将根据其位置而有所不同，以与ld的较早版本兼容。 

出现在输出节定义之外的表达式会将所有数字视为绝对地址。 出现在输出节定义内的表达式将绝对符号视为数字。 如果给出了LD_FEATURE（“ SANE_EXPR”），则绝对符号和数字在任何地方都将被简单地视为数字。 

在下面的简单示例中， 


.. code-block:: c
   :caption: c test
   :linenos:
   SECTIONS{	. = 0x100;	__executable_start = 0x100;	.data :	{		. = 0x10;		__data_start = 0x10;		*(.data)	}...}


两个都 . 和\_\_executable_start在前两个分配中均设置为绝对地址0x100，然后在两者中均设置为。 相对于后两个赋值中的.data节，\_\_data_start和\_\_data_start设置为0x10。 

对于涉及数字，相对地址和绝对地址的表达式，ld遵循以下规则来评估术语： 

- 对绝对地址或数字的一元运算，以及对两个绝对地址或两个数字或在一个绝对地址和一个数字之间的二进制运算，会将运算符应用于该值。 
- 对相对地址的一元运算，以及在同一节中或在一个相对地址和数字之间的两个相对地址的二进制运算，将运算符应用于地址的偏移量部分。
- 其他二进制运算，即，不在同一节中的两个相对地址之间，或在相对地址和绝对地址之间，首先将任何非绝对项转换为绝对地址，然后再应用运算符。 

每个子表达式的结果部分如下： 

- 仅涉及数字的运算将产生一个数字。 

- 比较结果“ &&”和“ ||”也是一个数字。

- 在LD_FEATURE（“ SANE_EXPR”）或在输出节定义内时，对同一节中两个相对地址或两个绝对地址（在上述转换之后）的其他二进制算术和逻辑运算的结果也是一个数字，否则为绝对地址。 

- 在相对地址或一个相对地址和一个数字上进行其他运算的结果是与相对操作数在同一部分中的相对地址。

- 对绝对地址的其他操作（经过上述转换）的结果是绝对地址。 

  

  如果表达式是相对的，则可以使用内置函数ABSOLUTE强制表达式为绝对。 例如，要创建一个绝对符号集，将其设置为输出节“ .data”末尾的地址： 


.. code-block:: c
   :caption: c test
   :linenos:
   SECTIONS{	.data : { *(.data) _edata = ABSOLUTE(.); }}


如果未使用“绝对”，则“ _edata”将相对于“ .data”部分。 使用LOADADDR也会强制表达式为绝对值，因为此特定的内置函数将返回绝对地址。 

内置函数
*******

链接描述文件语言包括许多内置函数，可用于链接描述文件表达式。 

- ABSOLUTE(exp ):返回表达式exp的绝对值（不可重定位，而不是非负值）。 主要用于将绝对值分配给节定义中的符号，其中符号值通常是节相对的。 请参阅第3.10.8节[表达部分]，第86页。

- ADDR(section ):返回命名部分的地址（VMA）。 您的脚本之前必须已经定义了该部分的位置。 在以下示例中，为start_of_output_1，symbol_1和symbol_2分配了等效值，不同的是symbol_1将相对于.output1部分，而另两个则是绝对值：

- 
	.. code-block:: c
   		:caption: c test
   		:linenos:
   		SECTIONS { ....output1 :{	start_of_output_1 = ABSOLUTE(.);	...}.output :{	symbol_1 = ADDR(.output1);	symbol_2 = start_of_output_1;}... }
  

  

- ALIGN(align )/ALIGN(exp ,align ):返回与下一个对齐边界对齐的位置计数器（.）或任意表达式。 单个操作数ALIGN不会更改位置计数器的值，它只是对其进行算术运算。 两个操作数ALIGN允许任意表达式向上对齐（ALIGN（align）等同于ALIGN（ABSOLUTE（.），align））。 

  这是一个示例，该示例将输出.data节与前一个节之后的下一个0x2000字节边界对齐，并将该节内的变量设置为与输入节后的下一个0x8000边界： 

  .. code-block:: c
   		:caption: c test
   		:linenos:
   		
   		SECTIONS { ...	.data ALIGN(0x2000): {	*(.data)	variable = ALIGN(0x8000);	}... }
  

  在此示例中，ALIGN的首次使用指定了节的位置，因为它被用作节定义的可选地址属性（请参见第3.6.3节[输出节地址]，第62页）。 ALIGN的第二种用法用于定义符号的值。 

  内建函数NEXT与ALIGN紧密相关。 

- ALIGNOF(section ):返回已命名节的对齐方式（以字节为单位）。 如果在评估时尚未分配该节，则链接器将报告错误。 在下面的示例中，.output节的对齐方式存储为该节中的第一个值。 

  
.. code-block:: c
   :caption: c test
   :linenos:
   SECTIONS{ ...	.output {		LONG (ALIGNOF (.output))		...	}... }
  

  

- BLOCK(exp ):这是ALIGN的同义词，以与较旧的链接程序脚本兼容。 设置输出部分的地址时最常见。 

- DATA_SEGMENT_ALIGN(maxpagesize , commonpagesize )

- DATA_SEGMENT_END(exp )

- DATA_SEGMENT_RELRO_END(offset , exp )

- DEFINED(symbol )

- LENGTH(memory )

- LOADADDR(section )

- LOG2CEIL(exp )

- MAX(exp1 , exp2 )

- MIN(exp1 , exp2 )

- NEXT(exp )

- ORIGIN(memory )

- SEGMENT_START(segment , default )

- SIZEOF(section )

- SIZEOF_HEADERS
  sizeof_headers

隐式链接脚本
"""""""""""

如果您指定了一个链接程序输入文件，但该链接程序无法将其识别为目标文件或存档文件，它将尝试将其作为链接程序脚本读取。 如果无法将文件解析为链接描述文件，则链接描述文件将报告错误。

隐式链接描述文件不会替换默认的链接描述文件。

通常，隐式链接描述文件仅包含符号分配或INPUT，GROUP或VERSION命令。 

由于隐式链接程序脚本而读取的所有输入文件都将在命令行中读取隐式链接程序脚本的位置处读取。 这可能会影响档案搜索。


机器相关的功能
^^^^^^^^^^^^

ld在某些平台上特有功能； 我们仅针对X86进行描述,此节没有相关内容需要总结.

BFD
^^^^^^

链接器使用BFD库访问对象文件和归档文件。 这些库允许链接器使用相同的例程对目标文件进行操作，而不管目标文件的格式如何。 只需创建一个新的BFD后端并将其添加到库中，就可以支持不同的目标文件格式。 但是，为了节省运行时内存，通常将链接器和关联的工具配置为仅支持可用的目标文件格式的一部分。可以使用objdump -i（请参阅GNU Binary Utilities中的“ objdump”一节）列出所有格式。 可用于您的配置。 

与大多数实现一样，BFD是多个冲突需求之间的折衷。 影响BFD设计的主要因素是效率：如果不涉及BFD，则在格式之间进行转换的任何时间都是不会花费的时间。 这部分被抽象的投资回报所抵消； 由于BFD简化了应用程序和后端，因此可能需要花费更多的时间和精力来优化算法以提高速度。 

您应该牢记的BFD解决方案的一个小缺陷是潜在的信息丢失。 使用BFD机制可能会在两个地方丢失有用的信息：转换期间和输出期间。 请参阅BFD内部文档中的BFD信息丢失。 

工作原理：BFD概述
"""""""""""""""

打开目标文件时，BFD子例程会自动确定输入目标文件的格式。 然后，他们在内存中建立一个带有例程的指针的描述符，该例程将用于访问目标文件数据结构的元素。 

由于需要来自目标文件的不同信息，因此BFD从文件的不同部分读取并处理它们。 例如，链接器的一个非常常见的操作是处理符号表。 每个BFD后端都提供了一个例程，用于在目标文件的符号表示和内部规范格式之间进行转换。 当链接器请求目标文件的符号表时，它通过内存指针从相关的BFD后端调用该例程，该BFD后端读取该表并将其转换为规范形式。 然后，链接器以规范形式运行。 链接结束并且链接器写入输出文件的符号表后，另一个BFD后端例程将被调用以获取新创建的符号表并将其转换为所选的输出格式。 

信息丢失
*******

在输出过程中信息可能会丢失。 BFD支持的输出格式不能提供相同的功能，可以以一种形式描述的信息无处可走。 一个示例是b.out中的对齐信息。 a.out格式文件中没有任何地方可以存储有关所包含数据的对齐信息，因此，当从b.out链接文件并生成a.out图像时，对齐信息将不会传播到输出文件。 （链接器仍将在内部使用对齐信息，因此可以正确执行链接）。 

另一个示例是COFF节名称。 COFF文件可能包含无限多个部分，每个部分都有一个文本部分名称。 如果链接的目标是没有很多节的格式（例如a.out）或没有名字的节（例如Oasys格式），则不能简单地完成链接。 您可以通过使用链接器命令语言描述所需的输入到输出部分映射来避免此问题。 

规范化过程中信息可能会丢失。 外部格式的BFD内部规范形式并不详尽； 输入格式的结构内部没有直接表示。 这意味着BFD后端无法维护 通过从外部格式到内部格式以及返回到外部格式之间的转换，实现所有可能的数据丰富性。

只有当应用程序读取一种格式并写入另一种格式时，此限制才是问题。每个BFD后端负责维护尽可能多的数据，并且内部BFD规范形式具有对BFD核心不透明的结构，并且仅导出到 后端。 以一种格式读取文件时，将为BFD和应用程序生成规范格式。 同时，后端可以保存任何可能会丢失的信息。 如果随后以相同格式写回数据，则后端例程将能够使用BFD内核提供的规范格式以及它先前准备的信息。 由于后端之间有很多共性，因此在将大字节序COFF链接或复制到小字节序COFF或将a.out链接到b.out时，不会丢失任何信息。 链接多种格式时，信息只会从格式与目的地不同的文件中丢失。 


BFD规范目标文件格式
*****************

当源格式提供的信息，规范格式存储的信息和目标格式需要的信息之间的重叠最少时，就会出现大概率的信息丢失。规范形式的简短说明可以帮助您了解在转换之间可以保留哪些类型的数据。 

- files:
- sections
- symbols
- relocation level
- line numbers












