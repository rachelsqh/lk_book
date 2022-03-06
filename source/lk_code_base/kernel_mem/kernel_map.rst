内核内存布局
-----------
systemap:这个里面是符号与地址对应关系。

vmlinux内核内存布局
------------------
img

其中的赋值并不占用空间，这些参数在其他地方存储，此处只是对其中的值进行赋值。
我们再重新看链接脚本的作用：


内核的加载
----------
内核加载完成后参考如上，也就是说跳转到startup64时就已经完成内核的整个定位了。我们看说明：
















--------------------------------------------------------------------------
startup_64:
	/*
	 * At this point the CPU runs in 64bit mode CS.L = 1 CS.D = 0,
	 * and someone has loaded an identity mapped page table
	 * for us.  These identity mapped page tables map all of the
	 * kernel pages and possibly all of memory.
	 *
	 * %rsi holds a physical pointer to real_mode_data.
	 *
	 * We come here either directly from a 64bit bootloader, or from
	 * arch/x86/boot/compressed/head_64.S.
	 *
	 * We only come here initially at boot nothing else comes here.
	 *
	 * Since we may be loaded at an address different from what we were
	 * compiled to run at we first fixup the physical addresses in our page
	 * tables and then reload them.
	 */

	/* Set up the stack for verify_cpu(), similar to initial_stack below */

此时 CPU 运行在 64 位模式 CS.L = 1 CS.D = 0，并且有人为我们加载了身份映射页表。 这些身份映射页表映射所有内核页面，可能还映射所有内存。 %rsi 拥有一个指向 real_mode_data 的物理指针。我们可以直接从 64 位引导加载程序或从 arch/x86/boot/compressed/head_64.S 来。 我们最初只是在启动时来到这里，没有其他东西来到这里。 由于我们可能被加载到与我们编译运行的地址不同的地址，我们首先修复页表中的物理地址，然后重新加载它们。

从这个位置往下看的时候记得所有的映射已经做好了

	leaq	(__end_init_task - FRAME_SIZE)(%rip), %rsp //栈：用的init_task的__end_init_task,空出了FRAME_SIZE的空间，这一块是在数据段中已经存在

	leaq	_text(%rip), %rdi   //相对寻址，rdi 指向_text地址
	pushq	%rsi
	call	startup_64_setup_env //加载GDT和IDT，加载了数据段：CS，SS。。。。。


	popq	%rsi

	/* Now switch to __KERNEL_CS so IRET works reliably */
	pushq	$__KERNEL_CS
	leaq	.Lon_kernel_cs(%rip), %rax
	pushq	%rax
	lretq	// 跳转：rsp - sizeof(rax) : rsp,即__KERNEL_CS --> CS,rax --> IP，此时GDT，IDT的加载才起作用，加载的32位寄存器
//从此以后寻址就进入偏移了。

.Lon_kernel_cs:
	UNWIND_HINT_EMPTY //调试用，编译内核时由objtool使用


	/* Sanitize CPU configuration
	 *	This is a common code for verification whether CPU supports
 * 	long mode and SSE or not. It is not called directly instead this
 *	file is included at various places and compiled in that context.
 *	This file is expected to run in 32bit code.  Currently:
 *
 *	arch/x86/boot/compressed/head_64.S: Boot cpu verification
 *	arch/x86/kernel/trampoline_64.S: secondary processor verification
 *	arch/x86/kernel/head_32.S: processor startup
 *
 *	verify_cpu, returns the status of longmode and SSE in register %eax.
 *		0: Success    1: Failure
 *
 *	On Intel, the XD_DISABLE flag will be cleared as a side-effect.
 *
 * 	The caller needs to check for the error code and take the action
 * 	appropriately. Either display a message or halt.
 */
	*/
	call verify_cpu//校验

	/*
	 * Perform pagetable fixups. Additionally, if SME is active, encrypt
	 * the kernel and retrieve the modifier (SME encryption mask if SME
	 * is active) to be added to the initial pgdir entry that will be
	 * programmed into CR3.
	 */
	 
	leaq	_text(%rip), %rdi
	pushq	%rsi
	call	__startup_64//这个处理一些冲定位的问题，非常难理解，用设问题，页表建立？

	popq	%rsi

	/* Form the CR3 value being sure to include the CR3 modifier */
	addq	$(early_top_pgt - __START_KERNEL_map), %rax //rax = sme_get_me_mask(); rax += $(early_top_pgt - __START_KERNEL_map);我们假设没有开启复杂功能，则sme_get_me_mask() = 0; 此处rax = $(early_top_pgt - __START_KERNEL_map)
	jmp 1f
SYM_CODE_END(startup_64)

1:

	/* Enable PAE mode, PGE and LA57 */
	movl	$(X86_CR4_PAE | X86_CR4_PGE), %ecx
#ifdef CONFIG_X86_5LEVEL
	testl	$1, __pgtable_l5_enabled(%rip)
	jz	1f
	orl	$X86_CR4_LA57, %ecx
1:
#endif
	movq	%rcx, %cr4  //设置页管理

	/* Setup early boot stage 4-/5-level pagetables. */
	addq	phys_base(%rip), %rax  //phys_base:__startup_64中进行值设置

	/*
	 * For SEV guests: Verify that the C-bit is correct. A malicious
	 * hypervisor could lie about the C-bit position to perform a ROP
	 * attack on the guest by writing to the unencrypted stack and wait for
	 * the next RET instruction.
	 * %rsi carries pointer to realmode data and is callee-clobbered. Save
	 * and restore it.
	 */
	pushq	%rsi
	movq	%rax, %rdi
	call	sev_verify_cbit
	popq	%rsi

	/* Switch to new page-table */
	movq	%rax, %cr3 //设置CR3

	/* Ensure I am executing from virtual addresses */
	movq	$1f, %rax //设置地址
	ANNOTATE_RETPOLINE_SAFE
	jmp	*%rax  //指针形式访问。对指针还是模糊
1:
	UNWIND_HINT_EMPTY

	/*
	 * We must switch to a new descriptor in kernel space for the GDT
	 * because soon the kernel won't have access anymore to the userspace
	 * addresses where we're currently running on. We have to do that here
	 * because in 32bit we couldn't load a 64bit linear address.
	 */
	lgdt	early_gdt_descr(%rip)  //加载64位GDT？

	/* set up data segments */
	xorl %eax,%eax
	movl %eax,%ds
	movl %eax,%ss
	movl %eax,%es

	/*
	 * We don't really need to load %fs or %gs, but load them anyway
	 * to kill any stale realmode selectors.  This allows execution
	 * under VT hardware.
	 */
	movl %eax,%fs
	movl %eax,%gs

	/* Set up %gs.
	 *
	 * The base of %gs always points to fixed_percpu_data. If the
	 * stack protector canary is enabled, it is located at %gs:40.
	 * Note that, on SMP, the boot cpu uses init data section until
	 * the per cpu areas are set up.
	 */
	movl	$MSR_GS_BASE,%ecx
	movl	initial_gs(%rip),%eax  //所以gs主要作用：percpu?
	movl	initial_gs+4(%rip),%edx
	wrmsr

	/*
	 * Setup a boot time stack - Any secondary CPU will have lost its stack
	 * by now because the cr3-switch above unmaps the real-mode stack
	 */
	movq initial_stack(%rip), %rsp//这个有些难理解

	/* Setup and Load IDT */
	pushq	%rsi
	call	early_setup_idt //重新加载idt
	popq	%rsi

	/* Check if nx is implemented */
	movl	$0x80000001, %eax
	cpuid
	movl	%edx,%edi

	/* Setup EFER (Extended Feature Enable Register) */
	movl	$MSR_EFER, %ecx
	rdmsr
	btsl	$_EFER_SCE, %eax	/* Enable System Call */
	btl	$20,%edi		/* No Execute supported? */
	jnc     1f
	btsl	$_EFER_NX, %eax
	btsq	$_PAGE_BIT_NX,early_pmd_flags(%rip)
1:	wrmsr				/* Make changes effective */

	/* Setup cr0 */
	movl	$CR0_STATE, %eax
	/* Make changes effective */
	movq	%rax, %cr0

	/* zero EFLAGS after setting rsp */
	pushq $0
	popfq

	/* rsi is pointer to real mode structure with interesting info.
	   pass it to C */
	movq	%rsi, %rdi

.Ljump_to_C_code:
	/*
	 * Jump to run C code and to be on a real kernel address.
	 * Since we are running on identity-mapped space we have to jump
	 * to the full 64bit address, this is only possible as indirect
	 * jump.  In addition we need to ensure %cs is set so we make this
	 * a far return.
	 *
	 * Note: do not change to far jump indirect with 64bit offset.
	 *
	 * AMD does not support far jump indirect with 64bit offset.
	 * AMD64 Architecture Programmer's Manual, Volume 3: states only
	 *	JMP FAR mem16:16 FF /5 Far jump indirect,
	 *		with the target specified by a far pointer in memory.
	 *	JMP FAR mem16:32 FF /5 Far jump indirect,
	 *		with the target specified by a far pointer in memory.
	 *
	 * Intel64 does support 64bit offset.
	 * Software Developer Manual Vol 2: states:
	 *	FF /5 JMP m16:16 Jump far, absolute indirect,
	 *		address given in m16:16
	 *	FF /5 JMP m16:32 Jump far, absolute indirect,
	 *		address given in m16:32.
	 *	REX.W + FF /5 JMP m16:64 Jump far, absolute indirect,
	 *		address given in m16:64.
	 */
	pushq	$.Lafter_lret	# put return address on stack for unwinder
	xorl	%ebp, %ebp	# clear frame pointer
	movq	initial_code(%rip), %rax
	pushq	$__KERNEL_CS	# set correct cs
	pushq	%rax		# target address in negative space
	lretq	
.Lafter_lret:

之里面最困难理解的就是 __startup_64()：




















----------------------------------------------------------------------------------------------------





----------------------------------------------------------------------------------------------------

unsigned long __attribute__((__section__(".head.text"))) __startup_64(unsigned long physaddr, //这个函数真的看得迷迷糊糊,这是一个黑洞，非常重要，内核内存映像最后一个黑洞。
      struct boot_params *bp)
{
 unsigned long vaddr, vaddr_end;
 unsigned long load_delta, *p;
 unsigned long pgtable_flags;
 pgdval_t *pgd;
 p4dval_t *p4d;
 pudval_t *pud;
 pmdval_t *pmd, pmd_entry;
 pteval_t *mask_ptr;
 bool la57;
 int i;
 unsigned int *next_pgt_ptr;

 la57 = check_la57_support(physaddr);

 if (physaddr >> (0 ? 52 : 46))
  for (;;);

 load_delta = physaddr - (unsigned long)(_text - (0xffffffff80000000UL));
 if (load_delta & ~(~(((1UL) << 21)-1)))
  for (;;);
 sme_enable(bp);
 load_delta += sme_get_me_mask();

 pgd = fixup_pointer(&early_top_pgt, physaddr);
 p = pgd + ((((0xffffffff80000000UL)) >> 39) & (512 - 1));

 if (la57)
  	*p = (unsigned long)level4_kernel_pgt;
 else
  	*p = (unsigned long)level3_kernel_pgt;
 
 *p += ((((pteval_t)(1)) << 0)|(((pteval_t)(1)) << 1)|(((pteval_t)(1)) << 2)|(((pteval_t)(1)) << 5)| 0|(((pteval_t)(1)) << 6)| 0| 0) - (0xffffffff80000000UL) + load_delta;

 if (la57) {
  	p4d = fixup_pointer(&level4_kernel_pgt, physaddr);
 	p4d[511] += load_delta;
 }

 pud = fixup_pointer(&level3_kernel_pgt, physaddr);
 pud[510] += load_delta;
 pud[511] += load_delta;

 pmd = fixup_pointer(level2_fixmap_pgt, physaddr);
 for (i = 507; i > 507 - 2; i--)
  	pmd[i] += load_delta;
# 202 "arch/x86/kernel/head64.c"
 next_pgt_ptr = fixup_pointer(&next_early_pgt, physaddr);
 pud = fixup_pointer(early_dynamic_pgts[(*next_pgt_ptr)++], physaddr);
 pmd = fixup_pointer(early_dynamic_pgts[(*next_pgt_ptr)++], physaddr);

 pgtable_flags = ((((pteval_t)(1)) << 0)|(((pteval_t)(1)) << 1)| 0|(((pteval_t)(1)) << 5)| 0|(((pteval_t)(1)) << 6)| 0| 0) + sme_get_me_mask();

 if (la57) {
  	p4d = fixup_pointer(early_dynamic_pgts[(*next_pgt_ptr)++],
        physaddr);

  	i = (physaddr >> 39) % 512;
  	pgd[i + 0] = (pgdval_t)p4d + pgtable_flags;
  	pgd[i + 1] = (pgdval_t)p4d + pgtable_flags;

  	i = physaddr >> 39;
  	p4d[(i + 0) % 1] = (pgdval_t)pud + pgtable_flags;
  	p4d[(i + 1) % 1] = (pgdval_t)pud + pgtable_flags;
 } else {
  	i = (physaddr >> 39) % 512;
  	pgd[i + 0] = (pgdval_t)pud + pgtable_flags;
  	pgd[i + 1] = (pgdval_t)pud + pgtable_flags;
 }

 i = physaddr >> 30;
 
 pud[(i + 0) % 512] = (pudval_t)pmd + pgtable_flags;
 pud[(i + 1) % 512] = (pudval_t)pmd + pgtable_flags;

 pmd_entry = ((((pteval_t)(1)) << 0)|(((pteval_t)(1)) << 1)| 0|(((pteval_t)(1)) << 5)| 0|(((pteval_t)(1)) << 6)|(((pteval_t)(1)) << 7)|(((pteval_t)(1)) << 8)) & ~(((pteval_t)(1)) << 8);

 mask_ptr = fixup_pointer(&__supported_pte_mask, physaddr);
 pmd_entry &= *mask_ptr;
 pmd_entry += sme_get_me_mask();
 pmd_entry += physaddr;

 for (i = 0; i < (((_end - _text) + (((1UL) << 21)) - 1) / (((1UL) << 21))); i++) {
  	int idx = i + (physaddr >> 21);
	pmd[idx % 512] = pmd_entry + i * ((1UL) << 21);
 }
# 258 "arch/x86/kernel/head64.c"
 pmd = fixup_pointer(level2_kernel_pgt, physaddr);

 for (i = 0; i < pmd_index((unsigned long)_text); i++)
 	pmd[i] &= ~(((pteval_t)(1)) << 0);


 for (; i <= pmd_index((unsigned long)_end); i++)
 	if (pmd[i] & (((pteval_t)(1)) << 0))
 		pmd[i] += load_delta;


 for (; i < 512; i++)
 	pmd[i] &= ~(((pteval_t)(1)) << 0);

*fixup_long(&phys_base, physaddr) += load_delta - sme_get_me_mask();

sme_encrypt_kernel(bp);

if (mem_encrypt_active()) {
	vaddr = (unsigned long)__start_bss_decrypted;
	vaddr_end = (unsigned long)__end_bss_decrypted;
  	for (; vaddr < vaddr_end; vaddr += ((1UL) << 21)) {
   		i = pmd_index(vaddr);
   		pmd[i] -= sme_get_me_mask();
  	}
}

return sme_get_me_mask();
}



-----------------
ORC理解：

#if 0
/*
 * In asm, there are two kinds of code: normal C-type callable functions and
 * the rest.  The normal callable functions can be called by other code, and
 * don't do anything unusual with the stack.  Such normal callable functions
 * are annotated with the ENTRY/ENDPROC macros.  Most asm code falls in this
 * category.  In this case, no special debugging annotations are needed because
 * objtool can automatically generate the ORC data for the ORC unwinder to read
 * at runtime.
 *
 * Anything which doesn't fall into the above category, such as syscall and
 * interrupt handlers, tends to not be called directly by other functions, and
 * often does unusual non-C-function-type things with the stack pointer.  Such
 * code needs to be annotated such that objtool can understand it.  The
 * following CFI hint macros are for this type of code.
 *
 * These macros provide hints to objtool about the state of the stack at each
 * instruction.  Objtool starts from the hints and follows the code flow,
 * making automatic CFI adjustments when it sees pushes and pops, filling out
 * the debuginfo as necessary.  It will also warn if it sees any
 * inconsistencies.
 */
.macro UNWIND_HINT sp_reg:req sp_offset=0 type:req end=0
.Lunwind_hint_ip_\@:
	.pushsection .discard.unwind_hints  //就是在这个节中加入这么一段信息
		/* struct unwind_hint */
		.long .Lunwind_hint_ip_\@ - .
		.short \sp_offset
		.byte \sp_reg
		.byte \type
		.byte \end
		.balign 4
	.popsection
.endm

.macro UNWIND_HINT_EMPTY
	UNWIND_HINT sp_reg=ORC_REG_UNDEFINED type=UNWIND_HINT_TYPE_CALL end=1  //针对ORC专门来一段
.endm


#endif//end 



---------------------------------------------------------------------------------------------------
kasan


---------------------------------------------------------------------------------------------------


x86_64_start_kernel()

BUILD_BUG_ON():条件成立则编译错误：#define BUILD_BUG_ON(condition) ((void)sizeof(char[1 - 2*!!(condition)]))
-----------------
#define CFI_STARTPROC           .cfi_startproc
#define CFI_ENDPROC             .cfi_endproc

--------------------------------


请记住，我们删除early_top_pgt了函数中的所有条目，reset_early_page_table并且只保留了内核高映射。

该x86_64_start_kernel函数的最后一步是调用：

x86_64_start_reservations(real_mode_data);

带有real_mode_dataas 参数的函数。在x86_64_start_reservations与函数相同的源代码文件中定义的x86_64_start_kernel函数和外观：

------------------------------------------------
可以看到，它是我们进入内核入口点之前的最后一个函数——start_kernel函数。让我们看看它做了什么以及它是如何工作的。

start_kernel()
让我们看一下reserve_ebda_region功能。它从检查是否启用半虚拟化开始：if(paravirt_enabled) //这个理解很差

--> setup_arch -> 
early_reserve_memory()://内存管理框架第一步：

memblock.c:111:struct memblock memblock __initdata_memblock = {
-------------------------------------------------------------
__init属性
#define __init      __section(.init.text) __cold notrace


初始化过程完成后，内核将通过调用free_initmem函数释放这些部分。另请注意，它__init是用两个属性定义的：__cold和notrace。第一个cold属性的目的是标记该函数很少使用，编译器必须优化该函数的大小。第二个notrace定义为：
#define notrace __attribute__((no_instrument_function))

在start_kernel函数的定义中，您还可以看到__visible扩展为：
#define __visible __attribute__((externally_visible))

-----------------------------------------------------------
start_kernel:

char *command_line;
char *after_dashes;
第一个表示指向内核命令行的指针，第二个将包含函数的结果，该parse_args函数解析带有表单参数的输入字符串name=value，查找特定关键字并调用正确的处理程序。我们此时不会详细介绍这两个变量的相关细节，但会在下一部分中看到。在下一步中，我们可以看到对该set_task_stack_end_magic函数的调用。此函数获取地址init_task并将STACK_END_MAGIC( 0x57AC6E9D) 设置为它的金丝雀。init_task表示初始任务结构：

-----------------------------------------------------
INIT_TASK:
- 初始化进程状态为零或runnable。可运行进程是只等待 CPU 运行的进程。
- 初始化进程标志 -PF_KTHREAD这意味着 - 内核线程；
- 可运行任务列表；
- 进程地址空间；
- 初始化进程堆栈到&init_thread_infowhich is init_thread_union.thread_infoand initthread_unionhas type -thread_union其中包含thread_info和进程堆栈：

-------------------------------------

vmlinux.lds:init_task定义详细分析
-----------------------------------

在debug_object_early_init函数之后，我们可以看到函数的调用，boot_init_stack_canary其中填充了gcc 特性task_struct->canary的canary值。-fstack-protector此功能取决于CONFIG_CC_STACKPROTECTOR配置选项，如果禁用此选项，则不boot_init_stack_canary执行任何操作，否则它会根据随机池和TSC生成随机数：

--------------------------------------

CPU


------------------------------------------
下一步是特定于架构的初始化。Linux 内核通过调用setup_arch函数来完成它。这是一个非常大的函数start_kernel，我们没有时间在这部分考虑它的所有实现。在这里，我们只会开始做，并在下一部分继
该函数从内核的保留内存块_text开始_data，从_text符号开始（您可以从arch/x86/kernel/head_64.S中记住它）并在__bss_stop. 我们memblock用于保留内存块：

memblock_reserve(__pa_symbol(_text), (unsigned long)__bss_stop - (unsigned long)_text);
在我们得到_text符号的物理地址后，memblock_reserve就可以从_text到__bss_stop - _text.

所以这块是物理地址统计记录。

--------------------------------------------

initrd -->内存中：
early_reserve_initrd

u64 ramdisk_image = get_ramdisk_image();
u64 ramdisk_size  = get_ramdisk_size();
u64 ramdisk_end   = PAGE_ALIGN(ramdisk_image + ramdisk_size);

所有这些参数均取自boot_params


最后为初始 ramdisk 保留计算出的地址的内存块：

memblock_reserve(ramdisk_image, ramdisk_end - ramdisk_image);

------------------------------------------------
setup_arch:
early_trap_init函数。该函数初始化调试（#DB- 在TF设置 rflags 标志时引发）和int3( #BP) 中断门。

void __init early_trap_init(void)
{
        set_intr_gate_ist(X86_TRAP_DB, &debug, DEBUG_STACK);
        set_system_intr_gate_ist(X86_TRAP_BP, &int3, DEBUG_STACK);
        load_idt(&idt_descr);
}

我们已经set_intr_gate在前面关于中断的部分中看到了实现。这里有两个相似的函数set_intr_gate_ist和set_system_intr_gate_ist。这两个函数都接受三个参数：

- 中断号；
- 中断/异常处理程序的基地址；
- 第三个参数是 - Interrupt Stack Table。IST是 TSS 中的一个新机制，也是TSSx86_64的一部分。内核模式下的每个活动线程都有自己的内核堆栈，以千字节为单位。当一个线程在用户空间时，这个内核栈是空的。16

除了每个线程的堆栈之外，还有几个与每个 CPU 相关的专用堆栈。您可以在 linux 内核文档 - Kernel stacks中阅读有关这些堆栈的所有信息。x86_64提供允许special在任何事件期间切换到新堆栈的功能，例如不可屏蔽的中断等......并且此功能的名称是 - Interrupt Stack Table。每个 CPU最多可以有 7IST个条目，每个条目都指向专用堆栈。在我们的例子中，这是DEBUG_STACK.
？？？（所以这是GDT？）

set_intr_gate_ist并且set_system_intr_gate_ist按照相同的原理工作，set_intr_gate只有一个区别。这两个函数都检查中断号并_set_gate在内部调用：

BUG_ON((unsigned)n > 0xFF);
_set_gate(n, GATE_INTERRUPT, addr, 0, ist, __KERNEL_CS);


-------------------------------------------------------------
早期的 ioremap 初始化
下一步是 early 的初始化ioremap。通常有两种与设备通信的方式：

输入/输出端口；
设备内存。
我们已经outb/inb在关于 linux 内核启动过程的部分看到了第一种方法（说明） 。第二种方法是将 I/O 物理地址映射到虚拟地址。当 CPU 访问物理地址时，它可能指的是物理 RAM 的一部分，它可以映射到 I/O 设备的内存上。所以ioremap用来将设备内存映射到内核地址空间。

正如我上面写的，下一个函数是early_ioremap_init将 I/O 内存重新映射到内核地址空间，以便它可以访问它。我们需要为早期初始化代码初始化早期 ioremap，该代码需要在正常映射函数ioremap可用之前临时映射 I/O 或内存区域。该函数的实现在arch/x86/mm/ioremap.c中。在开始时，early_ioremap_init我们可以看到pmd指针pmd_t类型的定义（显示页面中间目录条目typedef struct { pmdval_t pmd; } pmd_t;where pmdval_tis unsigned long）并检查是否fixmap以正确的方式对齐：

early_ioremap_init分析

----------------------------------------------------------------
获取根设备：

ROOT_DEV = old_decode_dev(boot_params.hdr.root_dev);
initrd此代码获取稍后将在do_mount_root函数中安装的根设备的主要和次要编号。设备的主要编号标识与设备关联的驱动程序。驱动程序控制的设备上引用的次要编号。请注意，old_decode_dev它从boot_params_structure. 正如我们可以从 x86 linux 内核启动协议中看到的：

----------------------------------------------------------------------------------------------
我们在启动时获得的所有这些参数并存储在boot_params结构中。在此之后，我们需要设置 I/O 内存的末端。如您所知，内核的主要目的之一是资源管理。其中一种资源是内存。我们已经知道有两种与设备通信的方式是 I/O 端口和设备内存。有关注册资源的所有信息可通过以下方式获得：

- /proc/ioports - 提供当前注册的端口区域列表，用于与设备进行输入或输出通信；
- /proc/iomem - 为每个物理设备提供系统内存的当前映射。

如您所见，地址范围以其所有者以十六进制表示法显示。Linux 内核提供了用于以通用方式管理任何资源的 API。全局资源（例如 PIC 或 I/O 端口）可以分为子集 - 与任何硬件总线插槽相关。主要结构resource：

struct resource {
        resource_size_t start;
        resource_size_t end;
        const char *name;
        unsigned long flags;
        struct resource *parent, *sibling, *child;
};
呈现系统资源的树状子集的抽象。这个结构提供了资源覆盖的地址范围从start到end（resource_size_tisphys_addr_t或u64for ）、资源（您在输出中看到这些名称）和资源（在include/linux/ioport.h中定义的所有资源标志）的地址范围。最后一个是指向结构的三个指针。这些指针启用树状结构：x86_64name/proc/iomemflagsresource


每个资源子集都有根范围资源。因为iomem它被iomem_resource定义为：


struct resource iomem_resource = {
        .name   = "PCI mem",
        .start  = 0,
        .end    = -1,
        .flags  = IORESOURCE_MEM,
};
EXPORT_SYMBOL(iomem_resource);


void __init e820__memory_setup(void)
{
	char *who;

	/* This is a firmware interface ABI - make sure we don't break it: */
	BUILD_BUG_ON(sizeof(struct boot_e820_entry) != 20);

	who = x86_init.resources.memory_setup();

	memcpy(e820_table_kexec, e820_table, sizeof(*e820_table_kexec));
	memcpy(e820_table_firmware, e820_table, sizeof(*e820_table_firmware));

	pr_info("BIOS-provided physical RAM map:\n");
	e820__print_table(who);
}

--------------------------------------------------------------------------------
early_param("gbpages", parse_direct_gbpages_on);


early_param宏有两个参数：

- 命令行参数名称；
- 如果传递给定参数，将调用该函数。


#define early_param(str, fn) \
        __setup_param(str, fn, fn, 1)


#define __setup_param(str, unique_id, fn, early)                \
        static const char __setup_str_##unique_id[] __initconst \
                __aligned(1) = str; \
        static struct obs_kernel_param __setup_##unique_id      \
                __used __section(.init.setup)                   \
                __attribute__((aligned((sizeof(long)))))        \
                = { __setup_str_##unique_id, fn, early }

该宏定义__setup_str_*_id变量（其中*取决于给定的函数名称）并将其分配给给定的命令行参数名称。在下一行中，我们可以看到__setup_*变量类型的定义obs_kernel_param及其初始化。obs_kernel_param结构定义为：

struct obs_kernel_param {
        const char *str;
        int (*setup_func)(char *);
        int early;
};

并包含三个字段：

- 内核参数的名称；
- 设置某些东西的函数取决于参数；
- 字段确定参数是早期 (1) 还是非 (0)。
请注意，__set_param宏用__section(.init.setup)属性定义。这意味着所有内容都__setup_str_*将放置在该.init.setup部分中，此外，正如我们在include/asm-generic/vmlinux.lds.h中看到的那样，它们将放置在__setup_startand之间__setup_end：

#define INIT_SETUP(initsetup_align)                \
                . = ALIGN(initsetup_align);        \
                VMLINUX_SYMBOL(__setup_start) = .; \
                *(.init.setup)                     \
                VMLINUX_SYMBOL(__setup_end) = .;
                
                
                
                
                
现在我们知道了参数是如何定义的，让我们回到parse_early_param实现：                
 void __init parse_early_param(void)
{
        static int done __initdata;
        static char tmp_cmdline[COMMAND_LINE_SIZE] __initdata;

        if (done)
                return;

        /* All fall through to do_early_param. */
        strlcpy(tmp_cmdline, boot_command_line, COMMAND_LINE_SIZE);
        parse_early_options(tmp_cmdline);
        done = 1;
}               
                
--------------------------------------------------------------

pci

-------------------------------------------------------------

setup_log_buf

-------------------------------------------------------------
并重新定位initrd到具有该relocate_initrd功能的直接映射区域。在relocate_initrd函数的开头，我们尝试使用函数找到一个空闲区域memblock_find_in_range：


------------------------------------------------------------

dma_contiguous_reserve




-------------------------------------------------------------

kmemleak
cma
-------------------------------------------------------------
x86_init.paging.pagetable_init
#define native_pagetable_init        paging_init
--------------------------------------------------------------
vsyscall

--------------------------------------------------------------
smp
----------------------
其他

-------------------

setup_per_cpu_areas
------------------------

build_all_zonelists

cat /sys/devices/system/node/node0/numastat 

每一个node都是由struct pglist_datalinux 内核中的。每个节点都被分成许多特殊的块，称为 - zones。每个区域都由zone structlinux 内核中的 提供，并且具有以下类型之一：

------------------------------------------------------
正如我在上面所写的，所有节点都用内存中的pglist_dataor结构来描述。pg_data_t此结构在include/linux/mmzone.h中定义。build_all_zonelists来自mm / page_alloc.c构建一个有序zonelist（不同区域）DMA的函数DMA32，该命令指定选择或无法满足分配请求NORMAL时访问区域/节点。就这样。更多关于多处理器系统的内容将在特殊部分中。HIGH_MEMORYMOVABLEzonenodeNUMA

-----------------------------------------------------

在我们开始深入研究 linux 内核调度程序初始化过程之前，我们必须做几件事。首先是mm/page_alloc.cpage_alloc_init中的函数。这个函数看起来很简单：

-----------------------------------------------------------

jump_label_init

-----------------------------------------------------------
pidhash_init
----------------------------------------------
我们看一下idr_init_cache函数的实现：

---------------------------------------------
kmem_cache_create：告诉缓存问题，如何实现与slab 与page的关系
---------------------------------------
时钟源的作用：间隔，计数？


































