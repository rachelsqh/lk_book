ARM架构
^^^^^^^^
linux 内核ARM架构部分
""""""""""""""""""""
- arm内核参考：https://www.kernel.org/doc/html/latest/arm/index.html；
- arm架构内核文档：https://www.kernel.org/doc/html/latest/arm/index.html
  - arm 中断架构：https://www.kernel.org/doc/html/latest/arm/interrupts.html
    总结：
  - UEFI：https://www.kernel.org/doc/html/latest/arm/uefi.html


- arm64架构内核参考：https://www.kernel.org/doc/html/latest/arm64/index.html
  - ACPI表：这个遵循ACPI规范：
    - 对于 arm64 上的 ACPI注意事项参考文档（架构差异）：https://www.kernel.org/doc/html/latest/arm64/acpi_object_usage.html
  - AArch64中的AMU，参考手册：https://www.kernel.org/doc/html/latest/arm64/amu.html
  - ARMv8 server的 ACPI,参考手册：https://www.kernel.org/doc/html/latest/arm64/arm-acpi.html
  - 非对称 32 位 SoC（big.LITTLE）参考文档：https://www.kernel.org/doc/html/latest/arm64/asymmetric-32bit.html	
  - AArch64 Linux引导，参考文档：https://www.kernel.org/doc/html/latest/arm64/booting.html
  - ARM64 CPU 功能寄存器，参考文档：https://www.kernel.org/doc/html/latest/arm64/cpu-feature-registers.html
  - ARM64 ELF hwcaps，参考文档：https://www.kernel.org/doc/html/latest/arm64/elf_hwcaps.html
  - ARM64 上的巨大 TLB 页面：https://www.kernel.org/doc/html/latest/arm64/hugetlbpage.html
  - AArh64内存布局：https://www.kernel.org/doc/html/latest/arm64/memory.html
  - AArch64 Linux 中的内存标记扩展 (MTE)，参考文档：https://www.kernel.org/doc/html/latest/arm64/memory-tagging-extension.html
  - perf:https://www.kernel.org/doc/html/latest/arm64/perf.html
  - AArch64 Linux 中的指针认证:https://www.kernel.org/doc/html/latest/arm64/pointer-authentication.html
  - 勘误：https://www.kernel.org/doc/html/latest/arm64/silicon-errata.html
  - 对 AArch64 Linux 的可扩展矢量扩展支持：https://www.kernel.org/doc/html/latest/arm64/sve.html
  - AArch64 标记地址 ABI：https://www.kernel.org/doc/html/latest/arm64/tagged-address-abi.html
  - AArch64 Linux 中的标记虚拟地址：https://www.kernel.org/doc/html/latest/arm64/tagged-pointers.html
  - linux 各功能在AArch64中的支持情况：https://www.kernel.org/doc/html/latest/arm64/features.html
  
arm内存管理：
^^^^^^^^^^^
linux kernel集中了世界顶尖程序员们的编程智慧，犹记操作系统课上老师讲操作系统的四大功能：进程调度 内存管理 设备驱动 网络。从事嵌入式软件开发工作，对设备驱动和网络接触的比较多。而进程调度和内存管理接触少之有少，更多的是敬而远之。
我的理解，想在内核开发上有更深层次的技术进步，应该对内核的内存管理进程调度等深层技术有一定的理解。不过这2块内容是内核最核心的部分，实际内核开发工作中涉及较少，很少有问题点来切入进去进行研究，网上也没有系统的资料进行讲解，学习起来谈何容易。
本着我不入地狱，谁入地狱的原则，这段时间利用工作之余对内存管理进行了一些研究，结合之前工作中遇到的一些内存管理的问题，对内存管理的框架有了一点理解。只能说，这一点点了解让我更加敬畏内核，不要说进程调度了，单单内存管理就够写一本500页的书了！
我的理解内存管理可以分为页表机制和内存分配机制两大块，这段时间的研究也仅仅让我对页表机制有些理解。先写几篇文章把页表机制写出来吧，把页表机制细节理通，再去学习内存分配机制，如bootmem和slab伙伴系统。
进程调度是比内存管理更大的坑，如果有幸从内存管理的大坑中活着爬出来，再去跳进程调度的坑吧。
忽然感觉自己肩上的担子很重，这也许就是博客的作用吧，分享自己的知识给大家，也让大家督促自己去进一步学习。
但是自己还只是参加工作上才几年的小学生，尝试着去分析这些高深的知识，肯定会有纰漏和错误的地方，不过既然分享出来，就是让大家和我一起去伪存真，进行完善。所以希望大家多提见解，一起努力！

由于内存管理是比较抽象的知识，因此本着三个原则去研究：
（1）带着一些问题去研究，提出一些疑问，由这些疑问点去切入
（2）将抽象的知识画出逻辑图，使框架更加清晰
（3）实例化，尽量由实际的设备来对内存管理进行研究

本系列文章是基于ARM架构，linux内核版本号3.4.55

linux内核的页表机制简单说来就是管理着设备真实物理地址与虚拟地址的一个动态或静态的映射，是基于硬件的MMU进行的，处理器必须提供MMU内存管理单元，linux的页表机制才能正常工作，两者相辅相成。
学习内核的内存管理如果脱离了MMU的硬件原理，只去学习其软件逻辑，真的很难懂。说到底，软件代码的逻辑是为硬件服务，只是为了充分发挥硬件的各项功能，因此学习linux的内存管理机制，首先要学习下该处理器架构下MMU的工作原理，这样对我们理解页表机制的逻辑很有帮助。（作为底层软件工程师，没事翻翻datasheet很有用啊，多从硬件思维去考虑问题）
MMU是处理器核内部的硬件逻辑，因此只有在处理器核的datasheet中才会有详细的说明，ARM的MMU逻辑对于不同版本处理器大同小异，我手头有一个ARM920T的手册，详细阅读了MMU一章，我有以下几个疑问需要解决：

一 MMU利用TLB进行PA（物理地址）和VA（虚拟地址）之间的转换，处理器寻址是直接在TLB中进行地址匹配。但内核初始化时会在内存中建立页表，页表与TLB什么关系？
ARM的MMU中分别有64个指令TLB和数据TLB，处理器寻址时的虚实地址转换是MMU在TLB之间进行匹配完成映射的，但是在内核初始化中会在内存中建立页表swapper_pg_dir（该过程可看我的另一篇博文：http://blog.csdn.net/skyflying2012/article/details/41447843），并将该地址配置到CP15寄存器中。这个页表跟ARM的TLB什么关系？
920T的MMU一章我找到了答案，如下：


CPU在访问VA（虚拟地址）时，TLB硬件完成VA到PA（物理地址）的转换，但是如果没有该VA的TLB entry，MMU的硬件单元translation table walk hardware（页表索引单元）会索引CP15寄存器c0提供的内存页表，进行地址转换，获取PA进行访问。并且会将该页表信息更新到TLB中，页表跟TLB可不是一个概念，TLB是针对于内存页表的一个缓存硬件！
也就是说ARM的MMU不仅使用TLB进行地址转换，还能够对内存中提供的页表进行解析并地址转换，而TLB中存储的是CPU最常用的一些地址。TLB速度快，这样可以加快地址转换效率。
如果都找不到该VA的页表信息，MMU会向CPU发出异常（根据data还是instruct不同，发出data abort或者instruct abort），异常处理函数中进行页表填充。
这也就让我明白了为什么在内核初始化的create_mapping函数（内存映射的关键函数，以后会讲到）以及缺页异常处理函数do_page_fault中看到的都是对内存页表的更新，而没有操作TLB。因为ARM的MMU本身就会使用内存页表！
并且ARM直接操作TLB比较复杂，不如操作内存页表，让MMU根据内存页表自己去更新TLB。
当然了，ARM的MMU所能操作的内存页表也是有固定格式的，这就是我们的下一个问题了。

二 看内核代码，ARM LINUX使用的二级页表映射，那么ARM的MMU硬件如何完成VA到PA的转换？
ARM的MMU使用内存页表如何完成地址转换，手册一张图将MMU操作页表的几种方式列了出来，如下：

如果这张图能够完全看懂，ARM的MMU硬件的地址转换就算完全明白了。可以看出ARM的MMU完成地址转换的方式有好多种，总体分为2种，section-mapping和page-mapping。linux的二级页表方式属于page-mapping，不过2种方式的映射linux内核都用到了，这个以后再说。
我们交给CPU的内存页表（写入CP15的C0）是一级页表（也可以称为页目录）地址，该页表总共有4096个索引，每个索引占4 bytes，单个表项可以映射1MB的地址空间，这样16KB大小的页表就可以囊括32位CPU可以寻址到的最大4GB空间。
可以想象，查询这4096个索引，仅需32位虚拟地址的高12位，CPU首先获取页目录基地址（TTB），加上待转换虚拟地址的高12位，即获取了该虚拟地址的页目录项。这个过程对于section-mapping和page-mapping都是一样的，那如何区分映射方式呢，关键在与页目录项的最低2bit，如下：

MMU根据页目录项最低2bit来判断接下来该如何操作，全0，无效页目录，MMU会向CPU发出缺页异常。page-mapping又会细分为coarse page table（粗页表）和fine page table（细页表），区别在于二级页表映射的是64K/4K页还是1K页，linux内核采用的是4K页，因此本文章着重说明粗页表中的4K页。
接下来来看section-mapping和page-mapping的具体虚实地址转换原理。

这张图说明白了page-mapping方式中4K页的工作原理，是一个二级页表方式，可以细分为5步：
（1）MMU由CP15的C0取出TTB（页目录）基址，与VA（虚拟地址）高12位相加，获取该VA在页目录中的对应页目录项值。
（2）MMU获取页目录项最低2bit，是01，说明本次映射的1MB数据为4k小页的page-mapping。
（3）MMU获取页目录项的高22位（页表是256X4=1K，所以页表基址是1K对齐的）是页表基地址，与VA的中间8位相加，即该VA的对应页表项地址，从而获取VA对应的页表项值（page table entry）
（4）MMU获取页表项值的高20位，这就是该4K页对应的物理地址了，与VA低12位相加（也就是4K页内的偏移），这就是VA对应的物理地址了
（5）MMU访问该物理地址，进行CPU给出的读写操作

上面说明了2种映射方式的虚实地址转换逻辑，可以看出，不管section-mapping还是page-mapping，在一级页表中都是完成1MB地址的映射，而page-mapping的第二级页表项中完成4K页的映射。
因此不管第一级页表项还是第二级页表项中除了存储物理地址，还会有很多bit是空余的，这些空余的bit完成了对所映射地址的访问权限以及操作属性的控制，主要包括AP位（access permission）和cache属性位，对于section-mapping，其控制位在第一级页表中（因为它只有一级啊），section-mapping的第一级页表项位定义如下：

--------------------------
如果采用单层的段映射，内存中有个段映射表，表中有4096个表项，每个表项的大小是4Byte，所以这个段映射表的大小是16KB，而且其位置必须与16KB边界对齐。每个段表项可以寻址1MB大小的地址空间。当CPU访问内存时，32位虚拟地址的高12位（bit[31:20]）用作访问段映射表的索引，从表中找到相应的表项。每个表项提供了一个12位的物理段地址，以及相应的标志位，如可读、可写等标志位。将这个12位物理地址和虚拟地址的低20位拼凑在一起，就得到32位的物理地址。

如果采用页表映射的方式，段映射表就变成一级映射表（First Level table，在Linux内核中称为PGD），其表项提供的不再是物理段地址，而是二级页表的基地址。32位虚拟地址的高12位（bit[31:20]）作为访问一级页表的索引值，找到相应的表项，每个表项指向一个二级页表。以虚拟地址的次8位（bit[19:12]）作为访问二级页表的索引值，得到相应的页表项，从这个页表项中找到20位的物理页面地址。最后将这20位物理页面地址和虚拟地址的低12位拼凑在一起，得到最终的32位物理地址。这个过程在ARM32架构中由MMU硬件完成，软件不需要接入。
https://www.coolcou.com/linux-kernel/linux-kernel-memory-management/the-linux-kernel-arm32-page-table-mapping.html



arm系统调用
^^^^^^^^^^^

当用户空间的程序调用swi指令发起内核服务请求的时候，实际上程序其实是完成了一次“穿越”，该进程从用户态穿越到了内核态。这个过程有点象周末你在家里看片，突然有些内急，随手按下了pause按键，电影里面的世界嘎然而止了。程序世界亦然，一个swi后，用户空间的代码执行暂停了、stack（用户栈）上的数据，正文段、静态数据区、heap去的数据……一切都停下来了，程序的执行突然就转入另外一个世界，使用的栈变成了内核栈、正在执行的正文段程序变成vector_swi开始的binary code、与之匹配数据区也变化了……

一切是怎么发生的呢？CPU只有一套而已，这里硬件做了哪些动作？软件又搞了什么鬼？穿越到另外的世界当然有趣，但是如何找到回来的路？这一切疑问希望能在这样的一篇文档中讲述清楚。

本文的代码来自4.4.6内核，用ARM处理器为例子描述。

 

二、构建内核栈上的用户现场

代码如下（忽略Cortex-M处理器的代码，忽略THUMB指令集的代码）：


当执行vector_swi的时候，硬件已经做了不少的事情，包括：

（1）将CPSR寄存器保存到SPSR_svc寄存器中，将返回地址（用户空间执行swi指令的下一条指令）保存在lr_svc中

（2）设定CPSR寄存器的值。具体包括：CPSR.M = '10011'（svc mode），CPSR.I = '1'（disable IRQ），CPSR.IT = '00000000'（TODO），CPSR.J = '0'（），CPSR.T = SCTLR.TE（J和Tbit和Instruction set state有关，和本文无关），CPSR.E = SCTLR.EE（字节序定义，和本文无关）。

（3）PC设定为swi异常向量的地址

随后的行为都是软件行为了，因为代码中涉及压栈动作，所以首先要确定的就是当前在哪里这个问题。sp_svc早在进程切换的时候就已经设定好了，就是该进程的内核栈。


代码我们就不走读了，很简单，大家可自行阅读即可。顺便一提的是：你看到这个保存的现场是不是觉得很熟悉？可以看看ARM中断处理这篇文档，中断保存的现场和系统调用是一样的。另外，保存在内核栈上的用户空间现场并不是全部的HW Context，HW context是一段内存中的数据，保存了某个时刻CPU的全部状态，不仅仅是core register，还有很多CPU内部其他的HW block状态，例如FPU的寄存器和状态。这时候，问题来了，在通过系统调用进入内核态的时候，仅仅保存core register够不够？够不够是和系统调用接口的约定相关，实际上，对于linux，我们约定如下：内核态的代码是不允许执行浮点操作指令的（这里用FPU例子，其他类似），如果一定要这样的话，那么需要在内核使用FPU的代码前后增加FPU上下文的保存和恢复的代码，确保在返回用户空间的时候，FPU的上下文是保持不变的。

最后一个有趣的问题是：为何r0被两次压栈？一个是r0，另外一个是old r0。其实在系统调用过程中，r0有两个角色，一个是传递参数，另外一个是返回值。刚进入系统调用现场的时候，old r0和r0其实都是保存了本次系统调用的参数，而在完成系统调用之后，r0保存了返回用户空间的return value。不过你可能觉得用一个r0就OK了，具体为何如此我们后面还会进行描述。

 系统调用有两种规范，一种是老的OABI（系统调用号来自swi指令中），另外一种是ARM ABI，也就是EABI（系统调用号来自r7）。如果想要兼容旧的OABI，那么我们需要定义OABI_COMPAT，这会带来一点系统调用的开销，同时让内核变大一点，对应的好处是使用旧的OABI规格的用户程序也可以运行在内核之上。当然，如果我们确定用户空间只是服从EABI规范，那么可以考虑不定义CONFIG_OABI_COMPAT。

相关的代码如下：

整个代码比较简单，就是用进入系统调用时候压入内核栈的值来进行用户现场的恢复，其中一个细节是内核栈的操作，在调用movs    pc, lr 返回用户空间现场之前，add    sp, sp, #\offset + S_FRAME_SIZE指令确保用户栈上是空的。此外，我们需要考虑返回用户空间时候的r0设置问题，毕竟它承载了本次系统调用的返回值，这时候的r0有两种情况：

（1）在没有pending work的情况下（fast等于1），r0保存了sys_xxx函数的返回值

（2）在有pending work的情况下（fast等于0），struct pt_regs（返回用户空间的现场）中的r0保存了sys_xxx函数的返回值

restore_user_regs还有一个参数叫做offset，我们知道，在进入系统调用的时候，我们把参数5和参数6压入栈上，因此产生了到pt_regs 8个字节的偏移，这里需要补偿回来。







arm硬件架构
""""""""""""
- arm系列contex的a，r，m的区别：从cortex开始，分为三个系列，a系列，r系列，m系列。
  - m系列与arm7相似，不能跑操作系统（只能跑ucos2），偏向于控制方面，说白了就是一个高级的单片机。
  - a系列主要应用在人机互动要求较高的场合，比如pda，手机，平板电脑等。a系列类似于cpu，与arm9和arm11相对应，都是可以跑草错系统的。linux等。
  - r系列，是实时控制。主要应用在对实时性要求高的场合。
  
  arm7和m3，m4是同一类型。这三个里面，arm7是最早的arm产品。m3是cortex m系列的过渡品，其低端市场被cortex m0的高端替代， 其高端市场又被cortex m4的低端取代。现在m系列，是m4内核的。典型的芯片是st公司和飞思卡尔公司的。
  arm9 和cortex a8 是一个类型的，都是跑操作系统的，现在的高端手机，三星，htc等智能手机，就是用的cortex a8，cortex a9 内核的芯片作为cpu。
  - ARM7,ARM9属于v4T或v5E架构
  - ARM11属于v6架构
  - Contex属于v7架构
  ARM7,ARM9的区别在于是否有MMU(存储器管理单元)或MPU(存储器保护单元)
  架构上v5E相比v4T则是在于v5E新加入的增强型DSP(数字信号处理)指令,v4T则是Thumb指令集的加入,v6架构则是开始支持SIMD以及Thumb2的问世.


RISC:精简指令集合：
- 指令数目 < 100;
- 指令格式 < 4;
- 寻址方式 < 4;
- 指令字长:等长；
- 可访存指令：只有取数/存数指令；
- 各种指令使用频率：相差很大；
- 各种指令执行时间：绝大多数在一个周期内完成；
- 优化编译实现：容易；
- 程序源代码长度：较长；
- 控制器实现方式：绝大多数为硬布线控制；
- 软件系统开发时间：较长；
ARM是RISC指令集合。而ARM在看到移动设备对64位计算的需求后，于2011年发布了ARMv8 64位架构。ARMv8使用了两种执行模式，AArch32和AArch64。顾名思义，一个运行32位代码，一个运行64位代码。ARM设计的巧妙之处，是处理器在运行中可以无缝地在两种模式间切换

异构计算：ARM的big.LITTLE架构是一项Intel一时无法复制的创新。在big.LITTLE架构里，处理器可以是不同类型的。传统的双核或者四核处理器中包含同样的2个核或者4个核，每个核提供一样的性能，拥有相同的功耗。而ARM通过big.LITTLE向移动设备推出了异构计算。这意味着处理器中的核可以有不同的性能和功耗。当设备正常运行时，使用低功耗核，而当你需要高计算能力时，使用的是高性能的核。

ARM体系结构
************
- ARM架构版本：
  - v4/v4T:
  - v5:
  - v6:arm11
  - v7:cortex-axx ~ cortex-a15
- ARM指令集
  - thumb指令： 16位，相对ARM32更简化；
  - thumb2指令：16位与32位混合的指令集；
  - SIMD
  - XEON
  - TrustZone:安全架构
  - v7：LPAE（内存虚拟？）
  - v7：开始支持虚拟化。
- 术语
  - VFP：浮点向量；
  - SIMD：
  - LPAE：大物理内存
  - Virtualization:虚拟化；
  - big.LITILE:eg:A15 + A8:根据不同需求进行切换：这个跟多核切换是一样的，只是这是差、异构。

- A系列技术点
  - 哈佛结构（代码和数据是分开的）；
  - Thum2
  - VFP和 NEON；
  
  - 强大L2级别缓存；
  - DFT：接口
  - APB：接口
   
- 工具
  - qemu;
  - 交叉工具链；
  - uboot;
  - linux kernel;
  - busybox;
  
处理器模式
**********
- user:10000:大部分程序运行时的非特权模式；（唯一的非特权模式）
- FIQ:10001:进入FIQ中断异常；快速中断异常；高优先级
- IRQ：10010：进入IRQ中断异常；
- Supervisor(SVC):10011:管理调用指令被执行或reset;开机或rest。
- Monitor(MON):10110:安全扩展模式，只用于安全；检测信号？
- Abort(ABT):10111:存储访问异常；用户发生异常时，CPU退出执行，（与MCE有点像）
- Hyp(HYP):11010:虚拟化扩展；（超级监视者）
- Undef(UND):10011:未定义的指令执行的时候；（与X86里的异常对应）
- System(SYS):11111:特权模式，与用户模式共享寄存器；系统挂掉？这个对应MCE

寄存器
********
不同的模式对应不同的寄存器组：

- R0 ～ R12；通用寄存器；放通用数据，32bit;
	    各个模式的R0-R12与USR模式是共享的（除了FIQ，R8 - R12），PC，CPSR共享的；
  R13： SP；栈指针寄存器；
  R14：LR；链接寄存器；存储子程序返回地址；
  R15：PC；程序计数器；对应IP？
  （A/C）PSR:程序状态寄存器；应用程序状态寄存器/当前程序状态寄存器。
  	USR叫APSR，其他模式下叫CPSR，USR模式没有SPSR。存储的就是一个ARM指令。
  SRSR：已存储程序状态寄存器；
  
  CPSR与SRSR的切换。
  
程序返回，其实就是
  1.MOV PC，LR   （LR --> PC);
  2.跳转，BL
    eg: BL SP
  
  注意内存和寄存器的转换。
  
在thump:通用寄存器只能访问r0-r7;

CPSR指令格式：
M[4:0]:设置模式（9种模式的微编码：10000 ～ 11111）；
T：5：是否是thumb指令（1：是，0：不是）；
F：6：FIRQ：是否禁止FIQ（1：禁止，0：不禁止）；
A：8：是否disable异步abort;
E:9:操作存储的字节顺序；

	
  
注意哪些寄存器是共享的，哪些寄存器是独有的
  

linux arm分析
*************


  
解决方案
********
官网解决方案：支持--文档
ARM体系架构全在ARM官网：www.arm.com

