GCC 在linux 内核中的应用
----------------------




重要选项
---------------

控制输出种类的选项
^^^^^^^^^^^^^^^
编译可能涉及四个阶段 ：预处理，编译，汇编和链接，始终按该顺序进行。 GCC能够将多个文件预处理并编译为多个汇编程序输入文件或一个汇编程序输入文件； 然后，每个汇编程序输入文件都会生成一个目标文件，并将所有目标文件（新编译的和指定为输入的那些目标文件）的链接合并到一个可执行文件中。 

对于任何给定的输入文件，文件名后缀确定完成哪种编译：

file.c ：需要预处理的C源代码。
file.i ：不需要预处理的C源代码。
file.h ：C头文件（or C, C++ header file to be turned into an Ada spec (via the ‘-fdump-ada-spec’ switch）

而已利用’-x'选项来准确制定输入语言：

-x language：为后面的输入文件指定使用的语言（代替处理器利用文件名后缀来确定输入语言）。这个选项接受所有下一个'-x'选项出来前的输入文件采用指定的语言。语言的名字可以指定为：

.. code-block:: c
   :caption: 可以指定的语言
   :emphasize-lines: 2
   :linenos:
   
   c c-header cpp-output
   c++ c++-header c++-cpp-output
   objective-c objective-c-header objective-c-cpp-output
   objective-c++ objective-c++-header objective-c++-cpp-output
   assembler assembler-with-cpp
   ada
   d
   f77 f77-cpp-input f95 f95-cpp-input
   go
   brig
   
   
-x none: 关闭指定语言，接下来的文件通过文件后缀名来确定其语言类型。（回到没有使用'-x'的时候）。

如果只想要某些编译阶段，您可以使用“ -x”（或使用特定文件名后缀）告诉gcc从哪里开始，以及选项“ -c”，“-S”或“ -E”之一来说明gcc的停止位置。 请注意，某些组合（例如，“-x cpp-output -E”）指示gcc完全不执行任何操作。 

- -c ：编译或汇编源文件，但不链接。最终的输出是每个源文件对应的目标文件的形式。默认目标文件名字用'.o'代替源文件的后缀，如'.c','.i','.s'等。忽略掉不能识别的输入文件。
- -S ：编译阶段后停止;不进行汇编。输出非汇编输出文件对应的汇编代码文件。默认，输出的汇编文件用'.s'代替输入文件的后缀'.c','.i'等。忽略掉不需要编译的文件。
- -E ：预处理阶段后停止;不运行编译过程。以预处理源代码形式做为输出，忽略掉不需要预处理的文件。
- -o file:
- -dumpbase dumpbase
- -dumpbase-ext auxdropsuf
- -dumpdir dumppfx
- -v 
- -### Like ‘-v’ except the commands are not executed and arguments are quoted
  unless they contain only alphanumeric characters or ./-_. This is useful for
  shell scripts to capture the driver-generated command lines.
- --help 
- --target-help
- --help={class|[^]qualifier}[,...]
- --version
- -pass-exit-codes
- -pipe
- -specs=file
- -wrapper
- -ffile-prefix-map=old=new
- -fplugin=name.so
- -fplugin-arg-name-key=value
- -fdump-ada-spec[-slim]
- -fada-spec-parent=unit
- @file


 控制C语言的选项
 ^^^^^^^^^^^^^^^
- ansi：C模式中，与 ‘-std=c90'等价。
- -std=：
- -fgnu89-inline：
- -fpermitted-flt-eval-methods=style
- -aux-info filename
- -fallow-parameterless-veriadic-functions
- -fno-asm
- -fno-builtin
- -fno-builtin-function
- -fgimple
- -fhosted
- -ffreestanding
- -fopenacc
- -fopenacc-dim=geom
- -fopenacc-kernels=mode
- -fopenmp
- -fopenmp-simd
- -fgnu-tm
- -fms-extensions
- -fplan9-extensions
- -fcond-mismatch
- -flax-vector-conversions
- -funsigned-char
- -fsigned-char
- -fsigned-bitfields/-funsigned-bitfields/fno-signed-bitfields/-fno-unsigned-bitfields
- -fsso-struct=endianness


控制诊断消息格式的选项
^^^^^^^^^^^^^^^^^^^^^^^
- -fmessage-length=n
- -fdiagnostics-plain-output
- -fdiagnostics-show-location=once
- -fdiagnostics-show-location=every-line
- -fdiagnostics-color[=WHEN]/-fno-diagnostics-color
- -fdiagnostics-urls[=WHEN]
- -fno-diagnostics-show-option
- -fno-diagnostics-show-caret
- -fno-diagnostics-show-labels
- -fno-diagnostics-show-cwe
- -fno-diagnostics-show-line-numbers
- -fdiagnostics-minimum-margin-width=width
- -fdiagnostics-parseable-fixits
- -fdiagnostics-generate-patch
- -fdiagnostics-show-template-tree
- -fno-elide-type
- -fdiagnostics-path-format=KIND
- -fdiagnostics-show-path-depths
- -fno-show-column
- -fdiagnostics-column-unit=UNIT
- -fdiagnostics-column-origin=ORIGIN
- -fdiagnostics-format=FORMAT


 请求或禁止警告的选项
 ^^^^^^^^^^^^^^^^^^
 
- -fsyntax-only
- -fmax-errors=n
- -w
- -Werror
- -Werror=
- -Wfatal-errors
- -Wpedantic/-pedantic
- -pedantic-errors
- -Wall
- -Wextra
- -Wabi
- -Wchar-subscripts
- -Wno-coverage-mismatch
- -Wno-cpp
- -Wdouble-promotion
- -Wduplicate-decl-specifier
- -Wformat/-Wformat=n
- -Wno-format-contains-nul
- -Wno-format-extra-args
- -Wformat-overflow/-Wformat-overflow=level
- -Wno-format-zero-length
- -Wformat-nonliteral
- -Wformat-security
- -Wformat-signedness
- -Wformat-truncation/-Wformat-truncation=level
- -Wformat-y2k
- -Wnonnull
- -Wnonnull-compare
- -Wnull-dereference
- -Winit-self (C, C++, Objective-C and Objective-C++ only)
- -Wno-implicit-int (C and Objective-C only)


控制静态分析的选项
^^^^^^^^^^^^^^^
- -fanalyzer
- -Wanalyzer-too-complex
- -Wno-analyzer-double-fclose
- -Wno-analyzer-double-free
- -Wno-analyzer-exposure-through-output-file
- -Wno-analyzer-file-leak
- -Wno-analyzer-free-of-non-heap
- -Wno-analyzer-malloc-leak
- -Wno-analyzer-mismatching-deallocation
- -Wno-analyzer-possible-null-argument
- -Wno-analyzer-possible-null-dereference
- -Wno-analyzer-null-argument
- -Wno-analyzer-null-dereference
- -Wno-analyzer-shift-count-negative
- -Wno-analyzer-shift-count-overflow
- -Wno-analyzer-stale-setjmp-buffer
- -Wno-analyzer-tainted-array-index
- -Wno-analyzer-unsafe-call-within-signal-handler
- -Wno-analyzer-use-after-free
- -Wno-analyzer-use-of-pointer-in-stale-stack-frame
- -Wno-analyzer-write-to-const
- -Wno-analyzer-write-to-string-literal
- -fanalyzer-call-summaries
- -fanalyzer-checker=name
- -fno-analyzer-feasibility
- -fanalyzer-fine-grained
- -fanalyzer-show-duplicate-count
- -fno-analyzer-state-merge
- -fno-analyzer-state-purge
- -fanalyzer-transitivity
- -fanalyzer-verbose-edges
- -fanalyzer-verbose-state-changes
- -fanalyzer-verbosity=level
- -fdump-analyzer
- -fdump-analyzer-stderr
- -fdump-analyzer-callgraph
- -fdump-analyzer-exploded-graph
- -fdump-analyzer-exploded-nodes
- -fdump-analyzer-exploded-nodes-2
- -fdump-analyzer-exploded-nodes-3
- -fdump-analyzer-json
- -fdump-analyzer-state-purge
- -fdump-analyzer-supergraph


调试程序选项
^^^^^^^^^^^^^^

- -g:

- -ggdb

- -gdwarf

  -gdwarf-version

- -gstabs

- -gstabs+

- -gxcoff

- -gxcoff+

- -gvms

- -glevel
  -ggdblevel
  -gstabslevel
  -gxcofflevel
  -gvmslevel

- -fno-eliminate-unused-debug-symbols

- -femit-class-debug-always

- -fno-merge-debug-strings

- -fdebug-prefix-map=old=new

- -fvar-tracking

- -fvar-tracking-assignments

- -gsplit-dwarf

- -gdwarf32
  -gdwarf64

- -gdescribe-dies

- -gpubnames

- -ggnu-pubnames

- -fdebug-types-section

- -grecord-gcc-switches
  -gno-record-gcc-switches

- -gstrict-dwarf

- -gno-strict-dwarf

- -gas-loc-support

- -gno-as-loc-support

- -gas-locview-support

- -gno-as-locview-support

- -gcolumn-info
  -gno-column-info

- -gstatement-frontiers
  -gno-statement-frontiers

- -gvariable-location-views
  -gvariable-location-views=incompat5
  -gno-variable-location-views

- -ginternal-reset-location-views
  -gno-internal-reset-location-views

- -ginline-points
  -gno-inline-points

- -gz[=type]

- -femit-struct-debug-baseonly

- -femit-struct-debug-reduced

- -femit-struct-debug-detailed[=spec-list]

- -fno-dwarf2-cfi-asm

- -fno-eliminate-unused-debug-types

  
控制优化的选项
^^^^^^^^^^^^^

这些选项控制各种优化。

如果没有任何优化选项，编译器的目标是降低编译开销，并使调试产生预期的结果。语句是不独立的，。。。。。。

开打开优化标志会使编译器尝试以牺牲编译时间和调试能力为代价来提高性能和/或代码大小

不是所有的优化都是通过标识来控制的。只有有标识对应的才能直接控制。

大多数优化可以通过'-O0'或者'-O'没有设置数字时完全禁止，这种情况下即使单独指定了优化选项也是无效的。同样的，'-Og'抑制许多优化。

依赖于目标平台和GCC的配置，在每个' -O '级别上可能会启用一组略微不同的优化，而不是这里列出的那些。可以通过 GCC '-Q --help=optimizers'查看每个级别的优化集。（已经验证）


.. code-block:: c
   :caption: 可以指定的语言
   :emphasize-lines: 2
   :linenos:
   rachel@rachel:~$ gcc -Q --help=optimizers
   The following options control optimizations:
  -O<number>                  
  -Ofast                      
  -Og                         
  -Os                         
  -faggressive-loop-optimizations       [enabled]
  -falign-functions                     [disabled]
  -falign-functions=          
  -falign-jumps                         [disabled]
  -falign-jumps=              
  -falign-labels                        [disabled]
  -falign-labels=             
  -falign-loops                         [disabled]
  -falign-loops=              
  -fallocation-dce                      [enabled]
  -fallow-store-data-races              [disabled]
  -fassociative-math                    [disabled]
  -fassume-phsa                         [available in BRIG]
  -fasynchronous-unwind-tables          [enabled]
  -fauto-inc-dec                        [enabled]
  -fbranch-count-reg                    [disabled]
  -fbranch-probabilities                [disabled]
  -fcaller-saves                        [disabled]
  -fcode-hoisting                       [disabled]
  -fcombine-stack-adjustments           [disabled]
  -fcompare-elim                        [disabled]
  -fconserve-stack                      [disabled]
  -fcprop-registers                     [disabled]
  -fcrossjumping                        [disabled]
  -fcse-follow-jumps                    [disabled]
  -fcx-fortran-rules                    [disabled]
  -fcx-limited-range                    [disabled]
  -fdce                                 [enabled]
  -fdefer-pop                           [disabled]
  -fdelayed-branch                      [disabled]
  -fdelete-dead-exceptions              [disabled]
  -fdelete-null-pointer-checks          [enabled]
  -fdevirtualize                        [disabled]
  -fdevirtualize-speculatively          [disabled]
  -fdse                                 [disabled]
  -fearly-inlining                      [enabled]
  -fexceptions                          [available in Modula-2]
  -fexcess-precision=[fast|standard]    [default]
  -fexpensive-optimizations             [disabled]
  -ffast-math                 
  -ffinite-loops                        [disabled]
  -ffinite-math-only                    [disabled]
  -ffloat-store                         [disabled]
  -fforward-propagate                   [disabled]
  -ffp-contract=[off|on|fast]           fast
  -ffp-int-builtin-inexact              [enabled]
  -ffunction-cse                        [enabled]
  -fgcse                                [disabled]
  -fgcse-after-reload                   [disabled]
  -fgcse-las                            [disabled]
  -fgcse-lm                             [enabled]
  -fgcse-sm                             [disabled]
  -fgraphite                            [disabled]
  -fgraphite-identity                   [disabled]
  -fguess-branch-probability            [disabled]
  -fhandle-exceptions                   -fexceptions
  -fhoist-adjacent-loads                [disabled]
  -fif-conversion                       [disabled]
  -fif-conversion2                      [disabled]
  -findirect-inlining                   [disabled]
  -finline                              [disabled]
  -finline-atomics                      [enabled]
  -finline-functions                    [disabled]
  -finline-functions-called-once        [disabled]
  -finline-small-functions              [disabled]
  -fipa-bit-cp                          [disabled]
  -fipa-cp                              [disabled]
  -fipa-cp-clone                        [disabled]
  -fipa-icf                             [disabled]
  -fipa-icf-functions                   [disabled]
  -fipa-icf-variables                   [disabled]
  -fipa-profile                         [disabled]
  -fipa-pta                             [disabled]
  -fipa-pure-const                      [disabled]
  -fipa-ra                              [disabled]
  -fipa-reference                       [disabled]
  -fipa-reference-addressable           [disabled]
  -fipa-sra                             [disabled]
  -fipa-stack-alignment                 [enabled]
  -fipa-vrp                             [disabled]
  -fira-algorithm=[CB|priority]         CB
  -fira-hoist-pressure                  [enabled]
  -fira-loop-pressure                   [disabled]
  -fira-region=[one|all|mixed]          [default]
  -fira-share-save-slots                [enabled]
  -fira-share-spill-slots               [enabled]
  -fisolate-erroneous-paths-attribute   [disabled]
  -fisolate-erroneous-paths-dereference         [disabled]
  -fivopts                              [enabled]
  -fjump-tables                         [enabled]
  -fkeep-gc-roots-live                  [disabled]
  -flifetime-dse                        [enabled]
  -flifetime-dse=<0,2>                  2
  -flimit-function-alignment            [disabled]
  -flive-patching                       -flive-patching=inline-clone
  -flive-patching=[inline-only-static|inline-clone]     [default]
  -flive-range-shrinkage                [disabled]
  -floop-interchange                    [disabled]
  -floop-nest-optimize                  [disabled]
  -floop-parallelize-all                [disabled]
  -floop-unroll-and-jam                 [disabled]
  -flra-remat                           [disabled]
  -fmath-errno                          [enabled]
  -fmodulo-sched                        [disabled]
  -fmodulo-sched-allow-regmoves         [disabled]
  -fmove-loop-invariants                [disabled]
  -fnon-call-exceptions                 [disabled]
  -fnothrow-opt                         [available in C++, ObjC++]
  -fomit-frame-pointer                  [disabled]
  -fopt-info                            [disabled]
  -foptimize-sibling-calls              [disabled]
  -foptimize-strlen                     [disabled]
  -fpack-struct                         [disabled]
  -fpack-struct=<number>      
  -fpartial-inlining                    [disabled]
  -fpatchable-function-entry= 
  -fpeel-loops                          [disabled]
  -fpeephole                            [enabled]
  -fpeephole2                           [disabled]
  -fplt                                 [enabled]
  -fpredictive-commoning                [disabled]
  -fprefetch-loop-arrays                [enabled]
  -fprintf-return-value                 [enabled]
  -fprofile-partial-training            [disabled]
  -fprofile-reorder-functions           [disabled]
  -freciprocal-math                     [disabled]
  -free                                 [disabled]
  -freg-struct-return                   [enabled]
  -frename-registers                    [enabled]
  -freorder-blocks                      [disabled]
  -freorder-blocks-algorithm=[simple|stc]       simple
  -freorder-blocks-and-partition        [disabled]
  -freorder-functions                   [disabled]
  -frerun-cse-after-loop                [disabled]
  -freschedule-modulo-scheduled-loops   [disabled]
  -frounding-math                       [disabled]
  -frtti                                [available in C++, D, ObjC++]
  -fsave-optimization-record            [disabled]
  -fsched-critical-path-heuristic       [enabled]
  -fsched-dep-count-heuristic           [enabled]
  -fsched-group-heuristic               [enabled]
  -fsched-interblock                    [enabled]
  -fsched-last-insn-heuristic           [enabled]
  -fsched-pressure                      [disabled]
  -fsched-rank-heuristic                [enabled]
  -fsched-spec                          [enabled]
  -fsched-spec-insn-heuristic           [enabled]
  -fsched-spec-load                     [disabled]
  -fsched-spec-load-dangerous           [disabled]
  -fsched-stalled-insns                 [disabled]
  -fsched-stalled-insns-dep             [enabled]
  -fsched-stalled-insns-dep=<number> 
  -fsched-stalled-insns=<number> 
  -fsched2-use-superblocks              [disabled]
  -fschedule-fusion                     [enabled]
  -fschedule-insns                      [disabled]
  -fschedule-insns2                     [disabled]
  -fsection-anchors                     [disabled]
  -fsel-sched-pipelining                [disabled]
  -fsel-sched-pipelining-outer-loops    [disabled]
  -fsel-sched-reschedule-pipelined      [disabled]
  -fselective-scheduling                [disabled]
  -fselective-scheduling2               [disabled]
  -fshort-enums                         [enabled]
  -fshort-wchar                         [disabled]
  -fshrink-wrap                         [disabled]
  -fshrink-wrap-separate                [enabled]
  -fsignaling-nans                      [disabled]
  -fsigned-zeros                        [enabled]
  -fsimd-cost-model=[unlimited|dynamic|cheap]   unlimited
  -fsingle-precision-constant           [disabled]
  -fsplit-ivs-in-unroller               [enabled]
  -fsplit-loops                         [disabled]
  -fsplit-paths                         [disabled]
  -fsplit-wide-types                    [disabled]
  -fsplit-wide-types-early              [disabled]
  -fssa-backprop                        [enabled]
  -fssa-phiopt                          [disabled]
  -fstack-check=[no|generic|specific] 
  -fstack-clash-protection              [disabled]
  -fstack-protector                     [disabled]
  -fstack-protector-all                 [disabled]
  -fstack-protector-explicit            [disabled]
  -fstack-protector-strong              [disabled]
  -fstack-reuse=[all|named_vars|none]   all
  -fstdarg-opt                          [enabled]
  -fstore-merging                       [disabled]
  -fstrict-aliasing                     [disabled]
  -fstrict-enums                        [available in C++, ObjC++]
  -fstrict-volatile-bitfields           [enabled]
  -fthread-jumps                        [disabled]
  -fno-threadsafe-statics               [available in C++, ObjC++]
  -ftoplevel-reorder                    [disabled]
  -ftracer                              [disabled]
  -ftrapping-math                       [enabled]
  -ftrapv                               [disabled]
  -ftree-bit-ccp                        [disabled]
  -ftree-builtin-call-dce               [disabled]
  -ftree-ccp                            [disabled]
  -ftree-ch                             [disabled]
  -ftree-coalesce-vars                  [disabled]
  -ftree-copy-prop                      [disabled]
  -ftree-cselim                         [enabled]
  -ftree-dce                            [disabled]
  -ftree-dominator-opts                 [disabled]
  -ftree-dse                            [disabled]
  -ftree-forwprop                       [enabled]
  -ftree-fre                            [disabled]
  -ftree-loop-distribute-patterns       [disabled]
  -ftree-loop-distribution              [disabled]
  -ftree-loop-if-convert                [enabled]
  -ftree-loop-im                        [enabled]
  -ftree-loop-ivcanon                   [enabled]
  -ftree-loop-optimize                  [enabled]
  -ftree-loop-vectorize                 [disabled]
  -ftree-lrs                            [disabled]
  -ftree-parallelize-loops=<number>     1
  -ftree-partial-pre                    [disabled]
  -ftree-phiprop                        [enabled]
  -ftree-pre                            [disabled]
  -ftree-pta                            [disabled]
  -ftree-reassoc                        [enabled]
  -ftree-scev-cprop                     [enabled]
  -ftree-sink                           [disabled]
  -ftree-slp-vectorize                  [disabled]
  -ftree-slsr                           [disabled]
  -ftree-sra                            [disabled]
  -ftree-switch-conversion              [disabled]
  -ftree-tail-merge                     [disabled]
  -ftree-ter                            [disabled]
  -ftree-vectorize            
  -ftree-vrp                            [disabled]
  -funconstrained-commons               [disabled]
  -funroll-all-loops                    [disabled]
  -funroll-completely-grow-size         [disabled]
  -funroll-loops                        [disabled]
  -funsafe-math-optimizations           [disabled]
  -funswitch-loops                      [disabled]
  -funwind-tables                       [disabled]
  -fvar-tracking                        [enabled]
  -fvar-tracking-assignments            [enabled]
  -fvar-tracking-assignments-toggle     [disabled]
  -fvar-tracking-uninit                 [disabled]
  -fvariable-expansion-in-unroller      [disabled]
  -fvect-cost-model=[unlimited|dynamic|cheap]   [default]
  -fversion-loops-for-strides           [disabled]
  -fvpt                                 [disabled]
  -fweb                                 [enabled]
  -fwrapv                               [disabled]
  -fwrapv-pointer                       [disabled]


程序工具选项
^^^^^^^^^^
GCC支持一系列命令行选项来控制向代码中增加运行时工具。例如，一个旨在收集程序分析统计信息，以找到它的热点，代码覆盖性分析或者由于知道程序优化。另一类程序插装是添加运行时检查，以检测编程错误，如无效指针解引用或数组越界访问，以及蓄意的恶意攻击，如栈破坏或c++虚函数表劫持。还有一个通用钩子，可以用来实现其他形式的跟踪或函数级检测，用于调试或程序分析目的。

- -p

  -pg

- -fprofile-arcs

- --coverage

- -ftest-coverage

- -fprofile-abs-path

- -fprofile-dir=path

- -fprofile-generate

- -fprofile-generate=path

- -fprofile-info-section

  -fprofile-info-section=name

- -fprofile-note=path

- -fprofile-prefix-path=path

- -fprofile-update=method

- -fprofile-filter-files=regex

- -fprofile-exclude-files=regex

- -fprofile-reproducible=[multithreaded|parallel-runs|serial]

- -fsanitize=address

- -fsanitize=kernel-address

- -fsanitize=hwaddress

- -fsanitize=kernel-hwaddress

- -fsanitize=pointer-compare

- -fsanitize=pointer-subtract

- -fsanitize=thread

- -fsanitize=leak

- -fsanitize=undefined

- -fno-sanitize=all

- -fasan_shadow-offset=number

- -fsanitize-sections=s1,s2,...

- -fsanitize-recover[=opts]

- -fsanitize-address-use-after-scope

- -fsanitize-undefined-trap-on-error

- -fsanitize-coverage=trace-pc

- -fsanitize-coverage=trace-cmp

- -fcf-protection=[full|branch|return|none|check]

- -fstack-protector

- -fstack-protector-all

- -fstack-protector-strong

- -fstack-protector-explicit

- -fstack-check

- -fstack-clash-protection

- -fstack-limit-register=reg

  -fstack-limit-symbol=sym

  -fno-stack-limit

- -fsplit-stack

- -fvtable-verify=[std|preinit|none]

- -fvtv-debug

- -fvtv-counts

- -finstrument-functions

- -finstrument-functions-exclude-file-list=file,file,...

- -finstrument-functions-exclude-function-list=sym,sym,...

- -fpatchable-function-entry=N[,M]


控制预处理器的选项
^^^^^^^^^^^^^^^^^

这些指令控制C预处理，它在实际编译之前在每个C源文件上运行。

如果使用了'-E'选项，则只进行预处理。其中一些选项只有与' -E '一起使用才有意义，因为它们会导致预处理器输出不适合实际编译。

除了列出的选项，控制include文件路径搜索选项在s3.16中描述，控制预处理诊断的选项在s3.8中列出。

- -D name: 与定义name为一个宏，定义为'1'
- -D name=definition：
- -U name:
- -include file:
- -imacros file:
- -undef
- -pthread
- -M
- -MM
- -MF file
- -MG
- -MP
- -MT target
- -MQ target
- -MD
- -MMD
- -fpreprocessed
- -fdirectives-only
- -fdollars-in-identifiers
- -fextended-identifiers
- -fno-canonical-system-headers
- -fmax-include-depth=depth
- -ftabstop=width
- -ftrack-macro-expansion[=level]
- -fmacro-prefix-map=old-new
- -fexec-charset-charset
- -fwide-exec-charset=charset
- -finput-charset-charset
- -fpch-deps
- -fpch-preprocess
- -fworking-directory
- -A predicate=answer
- -A -predicate=answer
- -C
- -CC
- -P
- -traditional
- -traditional-cpp
- -trigraphs
- -remap
- -H
- -dletters
- -fdebug-cpp
- -Wp,option
- -Xpreprocessor option
- -no-integrated-cpp
- -flarge-source-files


将选项传递给汇编器
^^^^^^^^^^^^^^^^

向汇编器传递参数。

- -Wa,option
- -Xassembler option


链接选项
^^^^^^^^^

当编译器将目标文件链接到可执行输出文件时，这些选项就会发挥作用。如果编译器不执行链接步骤，它们就没有意义。

- object-file-name

- -c

  -S

  -E

- -flinker-output=type

- -fuse-ld=bfd

- -fuse-ld=gold

- -fuse-ld=lld

- -llibrary

  -l library

- -lobjc

- -nostartfiles

- -nodefaultlibs

- -nolibc

- -nostdlib

- -e entry

  --entry=entry

- -pie

- -no-pie

- -static-pie

- -pthread

- -r

- -rdynamic

- -s

- -static

- -shared

- -shared-libgcc

  -static-libgcc

- -static-libasan

- -static-libtsan

- -static-liblsan

- -static-libubsan

- -static-libstdc++

- -symbolic

- -T script

- -Xlinker option

- -WI,option

- -u symbol

- -z keyword

目录搜索选项
^^^^^^^^^^^^^

这些选项指定头文件，库和编译器的搜索目录。

- -I dir

  -iquote dir

  -isystem dir

  -idirafter dir

- -I-

- -iprefix prefix

- -iwithprefix dir

  -iwithprefixbefore dir

- -isysroot dir

- -imultilib dir

- -nodtdinc

- -nostdinc++

- -iplugindir=dir

- -Ldir

- -Bprefix

- -no-canonical-prefixes

- --sysroot=dir

- --no-sysroot-suffix


代码生成约定的选项:
^^^^^^^^^^^^^^^^^^
不依赖机器的选项控制代码生成的接口约定。

机器相关的选项
^^^^^^^^^^^^
eBPF选项
"""""""""
- -mframe-limit=bytes
- -mkernel=version
- -mbig-endian
- -mlittle-endian
- -mxbpf


x86 Options
""""""""""""
'-m'选项指定为x86系列计算机

- -march=cpu-type

- -mtune=cpu-type

- -mcpu=cpu-type

- -mfpmath=unit

- -masm=dialect

- -mieee-fp
  -mno-ieee-fp

- -m80387
  -mhard-float

- -mno-80387
  -msoft-float

- -mno-fp-ret-in-387

- -mno-fancy-math-387

- -malign-double
  -mno-align-double

- -m96bit-long-double
  -m128bit-long-double

- -mlong-double-64
  -mlong-double-80
  -mlong-double-128

- -malign-data=type

- -mlarge-data-threshold=threshold

- -mrtd

- -mregparm=num

- -msseregparm

- -mvect8-ret-in-mem

- -mpc32
  -mpc64
  -mpc80

- -mstackrealign

- -mpreferred-stack-boundary=num

- -mincoming-stack-boundary=num

- -mmmx
  -msse
  -msse2
  -msse3
  -mssse3
  -msse4
  -msse4a
  -msse4.1
  -msse4.2
  -mavx
  -mavx2
  -mavx512f
  -mavx512pf
  -mavx512er
  -mavx512cd
  -mavx512vl
  -mavx512bw
  -mavx512dq
  -mavx512ifma
  -mavx512vbmi
  -msha
  -maes
  -mpclmul
  -mclflushopt
  -mclwb
  -mfsgsbase
  -mptwrite
  -mrdrnd
  -mf16c
  -mfma

  -mpconfig
  -mwbnoinvd
  -mfma4
  -mprfchw
  -mrdpid
  -mprefetchwt1
  -mrdseed
  -msgx
  -mxop
  -mlwp
  -m3dnow
  -m3dnowa
  -mpopcnt
  -mabm
  -madx
  -mbmi
  -mbmi2
  -mlzcnt
  -mfxsr
  -mxsave
  -mxsaveopt
  -mxsavec
  -mxsaves
  -mrtm
  -mhle
  -mtbm
  -mmwaitx
  -mclzero
  -mpku
  -mavx512vbmi2
  -mavx512bf16
  -mgfni
  -mvaes
  -mwaitpkg
  -mvpclmulqdq
  -mavx512bitalg
  -mmovdiri
  -mmovdir64b
  -menqcmd
  -muintr
  -mtsxldtrk
  -mavx512vpopcntdq
  -mavx512vp2intersect
  -mavx5124fmaps
  -mavx512vnni
  -mavxvnni
  -mavx5124vnniw
  -mcldemote
  -mserialize
  -mamx-tile
  -mamx-int8

  -mamx-bf16
  -mhreset
  -mkl
  -mwidekl

- -mdump-tune-features

- -mtune-ctrl=feature-list

- -mno-default

- -mcld

- -mvzeroupper

- -mprefer-avx128

- -mprefer-vector-width=opt

- -mcx16

- -msahf

- -mmovbe

- -mshstk

- -mcrc32

- -mrecip

- -mrecip=opt

- -mveclibabi=type

- -mabi=name

- -mforce-indirect-call

- -mmanual-endbr

- -mcall-ms2sysv-xlogues

- -mtls-dialect=type

- -mpush-args
  -mno-push-args

- -maccumulate-outgoing-args

- -mthreads

- -mms-bitfields
  -mno-ms-bitfields

- -mno-align-stringops

- -minline-all-stringops

- -minline-stringops-dynamically

- -mstringop-strategy=alg

- -mmemcpy-strategy=strategy

- -mmemset-strategy=strategy

- -momit-leaf-frame-pointer

- -mtls-direct-seg-refs
  -mno-tls-direct-seg-refs

- -msse2avx
  -mno-sse2avx

- -mfentry
  -mno-fentry

- -mrecord-mcount
  -mno-record-mcount

- -mnop-mcount
  -mno-nop-mcount

- -minstrument-return=type

- -mrecord-return
  -mno-record-return

- -mfentry-name=name

- -mfentry-section=name

- -mskip-rax-setup
  -mno-skip-rax-setup

- -m8bit-idiv
  -mno-8bit-idiv

- -mavx256-split-unaligned-load
  -mavx256-split-unaligned-store

- -mstack-protector-guard=guard
  -mstack-protector-guard-reg=reg
  -mstack-protector-guard-offset=offset

- -mgeneral-regs-only

- -mindirect-branch=choice

- -mfunction-return=choice

- -mindirect-branch-register

- -m32
  -m64
  -mx32
  -m16
  -miamcu

- -mno-red-zone

- -mcmodel=small

- -mcmodel=kernel

- -mcmodel=medium

- -mcmodel=large

- -maddress-mode=long

- -maddress-mode=short

- -mneeded
  -mno-needed
  
  
指定子流程和传递给它们的开关
^^^^^^^^^^^^^^^^^^^^^^^^


影响GCC的环境变量
^^^^^^^^^^^^^^^
这节描述影响GCC操作的很多环境变量。一些变量用于指定搜索目录或不同类型文件前缀。有些用于指定编译环境的其他方面。

注意，你也可以使用' -B '， ' -I '和' -L '等选项指定搜索位置。它们优先于使用环境变量指定的位置，而环境变量又优先于GCC配置指定的位置。

- LANG
  LC_CTYPE
  LC_MESSAGES
  LC_ALL
- TMPDIR
- GCC_COMPARE_DEBUG
- GCC_EXEC_PREFIX
- COMPILER_PATH
- LIBRARY_PATH
- LANG

一些环境变量影响到预处理的行为。

- CPATH
  C_INCLUDE_PATH
  CPLUS_INCLUDE_PATH
  OBJC_INCLUDE_PATH
- DEPENDENCIES_OUTPUT
- SUNPRO_DEPENDENCIES
- SOURCE_DATE_EPOCH

使用预编译头
^^^^^^^^^^^

打的项目每个源文件通常有很多头文件。编译器一遍又一遍地处理这些头文件所花费的时间几乎占了构建项目所需的全部时间。为了让构建速度更快，GCC允许预编译一个头文件。

如果需要，使用'-x'选项将头文件看作简单的C头文件，像编译其他文件一样简单编译一个预编译的头文件。我们需要一个像make的工具来保持预编译头文件修改后处于up-to-date状态。

当编译中出现#include时，将搜索预编译头文件。

。。。。。。

当满足一天条件时才能使用预编译头文件：

- 当预编译的头文件能在编译中应用时;
- A
- The
- Any
- if
- the
- each
- some
- address



C实现定义的行为
-------------

 C语言扩展
 ----------
 
 GNU C提供了ISO 标准C中找不到的很多语言特性（"-pedantic"会对这些扩展特性的应用打印警告信息).要在条件编译中测试这些功能的可用性，可以通过测试是否存在宏定义\_\_GNUC\_\_来实现。GCC下会定义这个宏。

我们只关注针对C语言部分。

ISO C99中的某些功能不在C90中的一些功能，GCC会把些特性当作C90模式的扩展进行包含。

表达式中的陈述和声明
^^^^^^^^^^^^^^^^^^

括号中的复合语句可能会在GNU C中作为表达式出现。这使您可以在表达式中使用循环，switch和局部变量。

 回想一下，复合语句是用大括号括起来的一系列语句； 在此构造中，大括号被括号括起来。 例如： 
 
.. code-block:: c
   :caption: test
   :emphasize-lines: 2
   :linenos:
   
   ({ int y = foo (); int z;	if (y > 0) z = y;	else z = - y;	z; })

对于foo（）的绝对值是一个有效的表达式（尽管比必需的复杂一些）。 

复合语句中的最后一件事应该是一个表达式，后跟一个分号； 该子表达式的值用作整个构造的值。 （如果最后在花括号内使用其他类型的语句，则该构造的类型为void，因此实际上没有任何值。） 

这种应用方式在保证宏定义的安全性时尤其有用。如以下定义方式：


.. code-block:: c
   :caption: test
   :emphasize-lines: 2
   :linenos:
   
   #define max(a,b) ((a) > (b) ? (a) : (b))
   
但是此定义计算a或b两次，如果操作数有副作用，则结果会很差。 在GNU C中，如果您知道操作数的类型（此处为int），则可以通过如下定义宏来避免此问题： 

.. code-block:: c
   :caption: test
   :emphasize-lines: 2
   :linenos:
   
   #define maxint(a,b) \	({int _a = (a), _b = (b); _a > _b ? _a : _b; })

请注意，引入变量声明（如我们在maxint中所做的那样）可能会导致变量阴影，因此，在使用max宏的此示例中，可以产生正确的结果： 

变量阴影？：对变量有影响？
......


本地声明的标签
^^^^^^^^^^^^^^^

标签作为值
^^^^^^^^^^^^^

嵌套函数
^^^^^^^^^^^^


非本地Gotos
^^^^^^^^^^^^^


构造函数调用
^^^^^^^^^^^^


用typeof引用Type
^^^^^^^^^^^^^^^


省略操作数的条件
^^^^^^^^^^^^^^^

命名地址空间
^^^^^^^^^^^
作为扩展，GNU C支持ISO / IEC DTR 18037的N1275草案中定义的命名地址空间。随着技术报告草案的更改，对GCC中命名地址空间的支持也将不断发展。 任何目标的调用约定也可能会更改。 目前，只有AVR，M32C，RL78和x86目标设备支持通用地址空间以外的地址空间。 地址空间标识符可以与其他任何C类型限定符（例如const或volatile）完全一样地使用。 有关更多详细信息，请参见N1275文档。 

总结：

x86命名地址空间
"""""""""""""
在x86目标上，可以将变量声明为相对于％fs或％gs段。 

- \_\_seg\_fs\\\_\_seg\_gs:

  使用相应的段替代前缀访问该对象。 必须通过特定于操作系统的某种方法来设置相应的段基础。 这些地址空间不被认为是通用（平面）地址空间的子空间，而不是需要昂贵的系统调用来检索段基础。 这意味着需要显式强制转换才能在这些地址空间和通用地址空间之间转换指针。 在实践中，应用程序应转换为uintptr_t并应用其先前安装的段基础偏移量。 

  当支持这些地址空间时，将定义预处理器符号\_\_SEG\_FS和\_\_SEG\_GS。 

总结：


长度为零的数组
^^^^^^^^^^^^^

没有成员的结构
^^^^^^^^^^^^^


可变长度的数组
^^^^^^^^^^^^^


参数可变的宏
^^^^^^^^^^^^

转义换行符的较宽松规则
^^^^^^^^^^^^^^^^^^^



Non-Lvalue 数组可能带有下标
^^^^^^^^^^^^^^^^^^^^^^^^^

无效指针和函数指针的算法
^^^^^^^^^^^^^^^^^^^^^^^

可变参数函数中的指针参数
^^^^^^^^^^^^^^^^^^^^^^

带有限定符的数组的指针按预期工作
^^^^^^^^^^^^^^^^^^^^^^^^^^^^




非常数初始化器
^^^^^^^^^^^^^



复合字面量
^^^^^^^^^^^^^^


指定的初始化程序
^^^^^^^^^^^^^^^^


case范围
^^^^^^^^^^^


转换为联合类型
^^^^^^^^^^^



混合声明，标签和代码
^^^^^^^^^^^^^^^^^

声明函数的属性
^^^^^^^^^^^^^

常用函数属性 
""""""""""""


BPF功能属性 
"""""""""""""

x86函数属性
""""""""""""

指定变量的属性
^^^^^^^^^^^^^

通用变量属性
""""""""""""

x86 变量属性
""""""""""""

指定类型的属性
^^^^^^^^^^^^^


通用类型属性
""""""""""
x86类型属性
""""""""""

标签属性
^^^^^^^^^^

枚举器属性
^^^^^^^^^^


语句属性
^^^^^^^

属性语法
^^^^^^^

原型和旧式函数定义
^^^^^^^^^^^^^^^


标识符名称中的美元符号
^^^^^^^^^^^^^^^^^^^^

常量中的字符ESC
^^^^^^^^^^^^^^


确定函数，类型或变量的对齐方式
^^^^^^^^^^^^^^^^^^^^^^^^^



内联函数的速度与宏一样快
^^^^^^^^^^^^^^^^^^^^^^


什么时候访问易失对象？
^^^^^^^^^^^^^^^^^^



如何在C代码中使用内联汇编语言
^^^^^^^^^^^^^^^^^^^^^^^^^




备用关键字
^^^^^^^^^^


不完整的枚举类型
^^^^^^^^^^^^^^^



函数名称为字符串
^^^^^^^^^^^^^^^


获取函数的返回地址或帧地址
^^^^^^^^^^^^^^^^^^^^^^


通过内置函数使用矢量指令
^^^^^^^^^^^^^^^^^^^^^


支持offsetof
^^^^^^^^^^^^


用于原子内存访问的旧版__sync内置函数
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^



内存模型感知原子操作的内置函数
^^^^^^^^^^^^^^^^^^^^^^^^^



内置函数，可通过溢出检查执行算术运算
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^


x86特定于事务性内存的内存模型扩展
^^^^^^^^^^^^^^^^^^^^^^^^^^^^


对象大小检查内置函数
^^^^^^^^^^^^^^^^^^^^

GCC提供的其他内置函数
^^^^^^^^^^^^^^^^^^^^^


特定于目标机器的内置函数
^^^^^^^^^^^^^^^^^^^^^^^^
在某些目标机器上，GCC支持许多特定于这些机器的内置函数，通常这些函数会生成对特定机器指令的调用，但允许编译器调度这些调用。


BPF内置功能
"""""""""""

x86内置函数
""""""""""""""

x86事务性内存固有
"""""""""""""""

x86控制流保护本质
""""""""""""""

特定于特定目标计算机的格式检查
^^^^^^^^^^^^^^^^^^^^^^^^^^
接受的用语说明
^^^^^^^^^^^^^

未命名的结构和联合字段
^^^^^^^^^^^^^^^^^^


线程本地存储
^^^^^^^^^^^^^

使用“ 0b”前缀的二进制常量 
^^^^^^^^^^^^^^^^^^^^^^

C 扩展部分在linux 内核中的应用
^^^^^^^^^^^^^^^^^^^^^^^^^^^


二进制兼容性
____________



gcov
-----------



linux 内核编程中GCC的高级应用
---------------------------

