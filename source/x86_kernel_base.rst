linux X86内核基础
-----------------
x86通用基础架构(硬件）
^^^^^^^^^^^^^^^^^^^^


linux/x86启动协议总结
^^^^^^^^^^^^^^^^^^^^
设备树
^^^^^^^^^
X86功能标志
^^^^^^^^^^^
x86拓扑
^^^^^^^^^
内核级异常处理
^^^^^^^^^^^^
硬件原理
"""""""
软件处理
"""""""

内核栈
^^^^^^^^^^^

内核entry
^^^^^^^^^^^

ORC unwinder
^^^^^^^^^^^^^^
与objtool关系


零页面
^^^^^^^^^^^^

TLB
^^^^^^^^^^^^^

mtrr
^^^^^^^^^^^

PAT
^^^^^^^^^^^^
linux IOMMU
^^^^^^^^^^^^^^

Intel(R) TXT
^^^^^^^^^^^^^

AMD内存加密
^^^^^^^^^^^^

页表隔离（PTI）
^^^^^^^^^^^^^^

mds
^^^^^^^^

intel microcode 加载
^^^^^^^^^^^^^^^^^^^^^

Intel(R) RDT
^^^^^^^^^^^^^^^^^^

TSX Async Abort (TAA) mitigation
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

总线锁检测与处理
^^^^^^^^^^^^^^^^

IO-APIC
^^^^^^^^^^^

x86_64支持
^^^^^^^^^^^
内存管理
""""""""

- 注意事项：
	- 诸如“-23 TB”之类的负地址是以字节为单位的绝对地址，从 64 位地址空间的顶部向下计数。以绝对地址和与顶部的距离表示法都更容易理解布局。例如 0xffffe90000000000 == -23 TB，它比 64 位地址空间的顶部 (ffffffffffffffff) 低 23 TB。请注意，随着我们越来越接近地址空间的顶部，符号会从 TB 变为 GB，然后是 MB/KB。
	- “16M TB”乍一看可能很奇怪，但它比“16 EB”更容易可视化大小表示法，很少有人会一眼认出是 16 EB。它还很好地展示了 64 位地址空间有多大。

- 具有 4 级页表的完整虚拟内存映射
========================================================================================================================
    Start addr    |   Offset   |     End addr     |  Size   | VM area description
========================================================================================================================
                  |            |                  |         |
 0000000000000000 |    0       | 00007fffffffffff |  128 TB | user-space virtual memory, different per mm
__________________|____________|__________________|_________|___________________________________________________________
                  |            |                  |         |
 0000800000000000 | +128    TB | ffff7fffffffffff | ~16M TB | ... huge, almost 64 bits wide hole of non-canonical
                  |            |                  |         |     virtual memory addresses up to the -128 TB
                  |            |                  |         |     starting offset of kernel mappings.
__________________|____________|__________________|_________|___________________________________________________________
                                                            |
                                                            | Kernel-space virtual memory, shared between all processes:
____________________________________________________________|___________________________________________________________
                  |            |                  |         |
 ffff800000000000 | -128    TB | ffff87ffffffffff |    8 TB | ... guard hole, also reserved for hypervisor
 ffff880000000000 | -120    TB | ffff887fffffffff |  0.5 TB | LDT remap for PTI
 ffff888000000000 | -119.5  TB | ffffc87fffffffff |   64 TB | direct mapping of all physical memory (page_offset_base)
 ffffc88000000000 |  -55.5  TB | ffffc8ffffffffff |  0.5 TB | ... unused hole
 ffffc90000000000 |  -55    TB | ffffe8ffffffffff |   32 TB | vmalloc/ioremap space (vmalloc_base)
 ffffe90000000000 |  -23    TB | ffffe9ffffffffff |    1 TB | ... unused hole
 ffffea0000000000 |  -22    TB | ffffeaffffffffff |    1 TB | virtual memory map (vmemmap_base)
 ffffeb0000000000 |  -21    TB | ffffebffffffffff |    1 TB | ... unused hole
 ffffec0000000000 |  -20    TB | fffffbffffffffff |   16 TB | KASAN shadow memory
__________________|____________|__________________|_________|____________________________________________________________
                                                            |
                                                            | Identical layout to the 56-bit one from here on:
____________________________________________________________|____________________________________________________________
                  |            |                  |         |
 fffffc0000000000 |   -4    TB | fffffdffffffffff |    2 TB | ... unused hole
                  |            |                  |         | vaddr_end for KASLR
 fffffe0000000000 |   -2    TB | fffffe7fffffffff |  0.5 TB | cpu_entry_area mapping
 fffffe8000000000 |   -1.5  TB | fffffeffffffffff |  0.5 TB | ... unused hole
 ffffff0000000000 |   -1    TB | ffffff7fffffffff |  0.5 TB | %esp fixup stacks
 ffffff8000000000 | -512    GB | ffffffeeffffffff |  444 GB | ... unused hole
 ffffffef00000000 |  -68    GB | fffffffeffffffff |   64 GB | EFI region mapping space
 ffffffff00000000 |   -4    GB | ffffffff7fffffff |    2 GB | ... unused hole
 ffffffff80000000 |   -2    GB | ffffffff9fffffff |  512 MB | kernel text mapping, mapped to physical address 0
 ffffffff80000000 |-2048    MB |                  |         |
 ffffffffa0000000 |-1536    MB | fffffffffeffffff | 1520 MB | module mapping space
 ffffffffff000000 |  -16    MB |                  |         |
    FIXADDR_START | ~-11    MB | ffffffffff5fffff | ~0.5 MB | kernel-internal fixmap range, variable size and offset
 ffffffffff600000 |  -10    MB | ffffffffff600fff |    4 kB | legacy vsyscall ABI
 ffffffffffe00000 |   -2    MB | ffffffffffffffff |    2 MB | ... unused hole
__________________|____________|__________________|_________|___________________________________________________________



- 具有 5 级页表的完整虚拟内存映射

使用 56 位地址，用户空间内存扩展了 512 倍，从 0.125 PB 到 64 PB。所有内核映射都向下移动到 -64 PB 起始偏移量，并且许多区域扩展以支持支持的更大物理内存。

========================================================================================================================
    Start addr    |   Offset   |     End addr     |  Size   | VM area description
========================================================================================================================
                  |            |                  |         |
 0000000000000000 |    0       | 00ffffffffffffff |   64 PB | user-space virtual memory, different per mm
__________________|____________|__________________|_________|___________________________________________________________
                  |            |                  |         |
 0100000000000000 |  +64    PB | feffffffffffffff | ~16K PB | ... huge, still almost 64 bits wide hole of non-canonical
                  |            |                  |         |     virtual memory addresses up to the -64 PB
                  |            |                  |         |     starting offset of kernel mappings.
__________________|____________|__________________|_________|___________________________________________________________
                                                            |
                                                            | Kernel-space virtual memory, shared between all processes:
____________________________________________________________|___________________________________________________________
                  |            |                  |         |
 ff00000000000000 |  -64    PB | ff0fffffffffffff |    4 PB | ... guard hole, also reserved for hypervisor
 ff10000000000000 |  -60    PB | ff10ffffffffffff | 0.25 PB | LDT remap for PTI
 ff11000000000000 |  -59.75 PB | ff90ffffffffffff |   32 PB | direct mapping of all physical memory (page_offset_base)
 ff91000000000000 |  -27.75 PB | ff9fffffffffffff | 3.75 PB | ... unused hole
 ffa0000000000000 |  -24    PB | ffd1ffffffffffff | 12.5 PB | vmalloc/ioremap space (vmalloc_base)
 ffd2000000000000 |  -11.5  PB | ffd3ffffffffffff |  0.5 PB | ... unused hole
 ffd4000000000000 |  -11    PB | ffd5ffffffffffff |  0.5 PB | virtual memory map (vmemmap_base)
 ffd6000000000000 |  -10.5  PB | ffdeffffffffffff | 2.25 PB | ... unused hole
 ffdf000000000000 |   -8.25 PB | fffffbffffffffff |   ~8 PB | KASAN shadow memory
__________________|____________|__________________|_________|____________________________________________________________
                                                            |
                                                            | Identical layout to the 47-bit one from here on:
____________________________________________________________|____________________________________________________________
                  |            |                  |         |
 fffffc0000000000 |   -4    TB | fffffdffffffffff |    2 TB | ... unused hole
                  |            |                  |         | vaddr_end for KASLR
 fffffe0000000000 |   -2    TB | fffffe7fffffffff |  0.5 TB | cpu_entry_area mapping
 fffffe8000000000 |   -1.5  TB | fffffeffffffffff |  0.5 TB | ... unused hole
 ffffff0000000000 |   -1    TB | ffffff7fffffffff |  0.5 TB | %esp fixup stacks
 ffffff8000000000 | -512    GB | ffffffeeffffffff |  444 GB | ... unused hole
 ffffffef00000000 |  -68    GB | fffffffeffffffff |   64 GB | EFI region mapping space
 ffffffff00000000 |   -4    GB | ffffffff7fffffff |    2 GB | ... unused hole
 ffffffff80000000 |   -2    GB | ffffffff9fffffff |  512 MB | kernel text mapping, mapped to physical address 0
 ffffffff80000000 |-2048    MB |                  |         |
 ffffffffa0000000 |-1536    MB | fffffffffeffffff | 1520 MB | module mapping space
 ffffffffff000000 |  -16    MB |                  |         |
    FIXADDR_START | ~-11    MB | ffffffffff5fffff | ~0.5 MB | kernel-internal fixmap range, variable size and offset
 ffffffffff600000 |  -10    MB | ffffffffff600fff |    4 kB | legacy vsyscall ABI
 ffffffffffe00000 |   -2    MB | ffffffffffffffff |    2 MB | ... unused hole
__________________|____________|__________________|_________|___________________________________________________________

体系结构定义了一个 64 位的虚拟地址。实现可以支持更少。当前支持的是 48 位和 57 位虚拟地址。位 63 到最高有效位是符号扩展的。如果您将它们解释为无符号，这会导致用户空间和内核地址之间出现漏洞。

直接映射覆盖系统中的所有内存，直到最高内存地址（这意味着在某些情况下它还可以包括 PCI 内存孔）。

我们在 64Gb 大虚拟内存窗口中的“efi_pgd”PGD 中映射 EFI 运行时服务（这个大小是任意的，如果需要，可以稍后提高）。这些映射不是任何其他内核 PGD 的一部分，并且仅在 EFI 运行时调用期间可用。

请注意，如果启用了 CONFIG_RANDOMIZE_MEMORY，则所有物理内存、vmalloc/ioremap 空间和虚拟内存映射的直接映射都是随机的。它们的顺序被保留，但它们的基数将在启动时提前偏移。

在这里更改任何内容时要非常小心与 KASLR。KASLR 地址范围不得与除 KASAN 影子区域外的任何内容重叠，这是正确的，因为 KASAN 禁用了 KASLR。

对于 4 级和 5 级布局，最后 2MB 孔中的 STACKLEAK_POISON 值：ffffffffffff4111




SVA
^^^^^^^^^^

SGX
^^^^^^^^^^^^
x86架构的特性状态
^^^^^^^^^^^^^^^

x86 特定的 ELF 辅助向量
^^^^^^^^^^^^^^^^^^^^^^


在用户空间应用程序中使用 XSTATE 特性
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Xtensa架构总结
^^^^^^^^^^^^^^









