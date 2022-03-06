x86架构级内存管理总结
--------------------------
.. code-block:: c
   :caption: struct_task --> mm
   :emphasize-lines: 4,5
   :linenos:
   
   
缺页中断：看硬件映射
--------------------------
.. code-block:: c
   :caption: struct_task --> mm
   :emphasize-lines: 4,5
   :linenos:
   
   # 硬件（X86架构）

异常类型：Fault

####描述：

CR0寄存器设置PG标志，处理器通过页转化机制将一个线性地址转换为一个物理地址时探测到以下条件之一：

1. 地址转换需要的页目录或页表入口项的P(present)标志为空，表示包含内容的页表或页不在物理内存中。
2. 程序没有足够的特权访问指定的页面（也就是说，运行在应用模式的程序试图访问超级模式的页）。如果设置了CR4的SMAP标志，运行在超级模式的程序试图访问用户模式地址的数据也可以触发缺页中断。如果设置了CR4中的PKE或PKS标志，访问包含保护密钥的线性地址时，保护密钥权限寄存器可能产生缺页中断。（理解上需要进一步确认）
3. 运行在用户模式的代码试图向一个只读页面里写数据。如果使能了CR0的WP标志，运行在超级用户模式的程序试图向一个只读页面写数据时也会触发缺页中断。
4. 从禁止了可执行位的页面中取指令。如果设置了CR4的SMEP位，运行在超级模式下的代码试图从用户地址空家获取指令时也会触发页错误。
5. 页结构入口的保留位设置为1时。（就是写了保留位）
6. 对不是shadow-stack页的页进行shadow-stack访问。
7. enclave访问，违反访问控制要求时，此时异常称为SGX-induced page fault.处理器使用后面描述的错误码来从传统的页错误中区分SGX产生的页错误。

异常处理程序可以通过重启程序或任务从页不在内存中恢复回来，保持程序的连续性。它还可以在权限冲突后重新启动程序或任务，但是造成特权侵犯的问题可能是无法纠正的。

#### 异常错误码

处理器向page-fault处理程序提供两项信息来诊断异常并从中恢复：

1. ##### 堆栈上的错误代码

   page-fault的错误码的格式与其他异常的错误码格式不同。处理器按照如下规则设置错误码：

   1. bit 0: P flag:如果用于地址转换的页结构项的P标志为0则线性地址不进行转换，
   2. bit 1: W/R:如果导致页面错误异常的访问是写入，则此标志为1；否则为0。此标志描述访问导致page-fault 异常的访问不是分页指定的访问权限。
   3. bit 2:U/S:  如果用户模式访问导致了page-fault异常，则此标志为1；如果是超级模式访问导致，则此标志为0。此标志描述导致page-fault异常的访问不是分页指定的访问权限。 
   4. bit 3:RSVD:  如果因为因为在用于转换该地址的分页结构项中设置了保留位导致线性地址没有转换，则此标志为1。
   5. bit 4:I/D:如果导致页面错误异常的访问是指令获取，则此标志为1。此标志描述导致页面错误异常的访问不是分页指定的访问权限。
   6. bit 5:PK:对具有保护密钥数据的线性地址的数据访问导致的错误，保护密钥权限寄存器不允许访问该地址，则此标志为1。
   7. bit 1:SS:因为对shadow-stck访问（包括enclave模式中的shadow-stack访问）导致的page-fault异常设置次标志为1。此标志描述由于访问权限冲突导致page-fault异常。
   8. bit 15:SGX:如果异常与分页无关，而是由违反SGX特定的访问控制要求而导致的，则此标志为1。因为只有在没有普通page-fault的情况下才可能发生这种冲突，所以仅当P标志（位0）为1并且RSVD标志（位3）和PK标志（位5）都为0时才设置该标志。

2. ##### CR2寄存器的内容

   处理器将生成异常的32位线性地址加载到CR2寄存器。page-fault处理程序可以使用此地址定位相应的页目录和页表条目。在执行页page-fault处理程序期间，可能会发生另一个page-fault错误；处理程序应在发生第二个page-fault之前保存CR2寄存器的内容。如果page-fault是由页面级别保护冲突引起的，则在发生错误时设置页面目录项中的访问标志。IA-32处理器关于相应页表条目中的访问标志的行为是特定于model的，而不是体系结构定义的。 

#### 保存的指令指针

CS和EIP寄存器保存的内容通常指向生成异常的指令。如果在任务切换期间发生page-fault异常，CS和EIP寄存器可能会指向新任务的第一条指令（如“程序状态更改”部分所述）。

####  程序状态更改

page-fault异常通常不会导致程序状态更改，因为导致生成异常的指令不会被执行。在page-fault异常处理程序纠正了冲突（例如，将页面加载到内存中）之后，程序或任务能恢复执行。 

当在任务切换期间产生page-fault异常时，程序状态可能会更改，如下所示。在任务切换期间，在以下任何操作期间都可能发生page-fault异常：

1. 当将任务的状态写入该任务的TSS时。
2. 在读取GDT以定位新任务的TSS描述符时。
3. 读取新任务的TSS。
4. 从新任务中读取与段选择器关联的段描述符时。
5. 在读取新任务的LDT时，验证存储在新TSS中的段寄存器。

在最后两种情况下，异常发生在新任务的上下文中。指令指针指向新任务的第一条指令，而不是导致任务切换的指令（或在中断的情况下执行的最后一条指令）。如果操作系统的设计允许在任务切换期间发生page-fault异常，则应通过任务门调用page-fault处理程序。(注意任务门间是否互斥？很重要的问题)

如果在任务切换期间发生page-fautl，则处理器将在生成异常之前从新的TSS（不执行任何其他限制，是否存在或类型检查）中加载所有状态信息。因此，page-fault处理程序不应假定使用CS，SS，DS，ES，FS和GS寄存器中的段选择子时不会引起其他异常。（可结合Interrupt 10进行理解）

其他异常处理信息

应格外小心，以确保在显式堆栈切换期间发生的异常不会导致处理器使用无效的堆栈指针（SS：ESP）。 为16位IA-32处理器编写的软件通常使用一对指令更改新堆栈，例如：

```ruby
MOV SS, AX
MOV SP, StackTop
```

在32位IA-32处理器上执行此代码时，在SS寄存器加载了段选择子之后，ESP寄存器加载段选择子之前可能会出现page-fault，通用保护错误（#GP）或对齐检查错误（#AC）。  此时，堆栈指针的两个部分（SS和ESP）是不一致。 新的堆栈段正在与旧的堆栈指针一起使用。

如果异常处理程序切换到定义良好的堆栈（即该处理程序是任务或特权更高的处理过程），则处理器不会使用不一致的堆栈指针。 但是，如果在相同的特权级别并从同一任务调用异常处理程序，则处理器将尝试使用不一致的堆栈指针。

在faulting任务中处理页面错误，常规保护或对齐检查异常的系统（带有陷阱门或中断门）中，以与异常处理程序相同的特权级别执行的软件应使用LSS指令而不是一对MOV指令初始化新堆栈。 当异常处理程序以特权级别0（正常情况）运行时，问题仅限于以特权级别0运行的过程或任务，通常是特权级别0的过程或任务。通常是操作系统内核。（待确认，这一段，脑袋不太清楚）

# linux 内核代码分析

kernel:v5.10.13

#### 异常程序基本代码架构：

```ruby
arch/x86/include/asm/trapnr.h:
#define X86_TRAP_PF	14  /*  Page Fault */

arch/x86/kernel/idt.c

/* Interrupt gate ：中断门*/
struct idt_bits {
    u16	ist	: 3, 
    	zero:5,
    	type:5,
    	dpl:2,
    	p:1;
    } __attribute__((packed));

struct idt_data { /* 中断门格式 */
    unsigned int vector; /* 向量号：#PF --> 14*/
    unsigned int segment; /* 代码段 */
    struct idt_bits bits; /* 权限相关设置 */
    const void *addr; /* 处理程序地址 */
};

#define  G(\_vector,\_addr,\_ist,\_type,\_dpl,\_segment) \ /* 初始化struct idt_data结构 */

	{	\
		.vector = \_vector, \
		.bits.ist = \_ist, \
		.bits.type = \_type, \
		.bits.dpl = \_dpl, \
		.bits.p = 1,	\
		.addr = \_addr,	\
		.segment = \_segment,	\
}
#define DEFAULT_STACK 0
enum {
    	GATE_INTERRUPT =0xE,
    	GATE_TRAP = 0xF,
    	GATE_CALL = 0xC,
    	GATE_TASK = 0x5,
};
#define DPL0	0x0  /* 注意: linux 只使用了这两种优先级 */
#define DPL3 	0x3

#define GDT_ENTRY_KERNEL_CS	12
#define GDT_ENTRY_KERNEL_DS	13

#define __KERNEL_CS	(GDT_ENTRY_KERNEL_CS * 8)
#define __KERNEL_DS	(GDT_ENTRY_KERNEL_DS * 8)
#define __USER_DS	(GDT_ENTRY_DEFAULT_USER_DS * 8 + 3)
#define __USER_CS	(GDT_ENTRY_DEFAULT_USER_CS * 8 + 3)
#define __ESPFIX_SS	(GDT_ENTRY_ESPFIX_SS * 8)

#define INTG(_vector,_addr)
	G(_vector,_addr,DEFAULT_STACK,GATE_INTERRUPT,DPL0,__KERNEL_CS)
INTG(X86_TRAP_PF,asm_exc_page_fault);/* 中断异常表入口 */

```
#### 定义 asm_exc_page_fault

```ruby
arch/x86/include/asm/idtentry.h /* 定义如下 */

#define DECLARE_IDTENTRY_ERRORCODE(vector,func) \
	idtentry vector asm_##func func has_error_code=1

#define DECLARE_IDTENTRY_RAW_ERRORCODE(vector,func)	\
		DECLARE_IDTENTRY_ERRORCODE(vector,func)

DECLARE_IDTENTRY_RAW_ERRORCODE(X86_TRAP_PF,exc_page_fault)
展开后定义为:
idtentry X86_TRAP_PF asm_exc_page_fault  exc_page_fault has_error_code=1
```

```ruby
arch/x86/entry/entry_64.S
/* 函数定义
 * idtentry - Macro to generate entry stubs for simple IDT entries
     @vector: Vector number
     @asmsym: ASM symbol for the entry point
     @cfunc:  C function to be called
     @has_error_code: Hardware pushed error code on stack
     The macro emits code to set up the kernel context for straight forward and simple IDT entries.No IST stack,no paranoid entry checks. 
*/

.macro idtentry vector asmsym cfunc has_error_code:req
SYM_CODE_START(\asmsym)
	UNWIND_HINT_IRET_REGS offset=\has_error_code * 8
	ASM_CLAC
	.if \has_error_code == 0 /* 针对没有错误码的中断\/异常 */
		pushq $-1
	.endif
	.if \vector == X86_TRAP_BP /* 断点:用于调试 */
		testb $3,CS-ORIG_RAX(%rsp)
		jnz .Lfrom_usermode_no_gap_\@ /* 用户空间 */
		.rept 6
		pushq 5 * 8(%rsp)
		.endr
		UNWIND_HINT_IRET_REGS offset=8
.Lfrom_usermode_no_gap_\@:
	.endif
	idtentry_body \cfunc \has_error_code /* 主题 */
_ASM_NOKPROBE(\asmsym)
SYM_CODE_END(\asmsym)
.endm

/* 函数主题定义
idtentry_body: Macro to emit code calling the C function
         @cfunc:C function to the called
         @has_error_code:	Hardware pushed error code on stack
*/
.macro idtentry_body cfunc has_error_code:req
	call error_entry /* Save all registers in pt_regs,and switch GS if needed */
	UNWIND_HINT_REGS
	movq %rsp,%rdi
	.if \has_error_code == 1
		movq ORIG_RAX(%rsp),%rsi
		movq $-1,ORIG_RAX(%rsp)
      .endif
	call \cfunc /* exc_page_fault */
	jmp error_return
.endm		
```

#### 调用exc_page_fault

```ruby
arch/x86/inclue/asm/idtentry.h

/*
    DEFINE_IDTENTRY_RAW_ERRORCODE - Emit code for raw IDT entry points
        @func:	Function name of the entry point
        @func is called from ASM entry code with interrupts disabled
        
        The macro is written so it acts as function definition,Append the body with a pair of curly brackets.
           
        Contrary to DEFINE_IDTENTRY_ERRORCODE() this does not invoke the irqentry_enter/exit() helpers before and after the body invocation.This needs to be done in the body itself if applicable.Use if extra work is required before the enter\/exit() helpers are invoked.
*/

#define DEFINE_IDTENTRY_RAW_ERRORCODE(func)
	__visible noinstr void func(struct pt_regs *regs,unsigned long error_code)
```

```ruby
arch/x86/mm/fault.c

DEFINE_IDTENTRY_RAW_ERRORCODE(exc_page_fault) /* 函数展开如下：即最终执行函数*/
/* __visible noinstr void exc_page_fault(struct pt_regs *regs,unsigned long error_code) */
{
	unsigned long address = read_cr2();/* 产生page fault 的虚拟地址(线性地址?) */
	irqentry_state_t state;
	prefetchw(&current->mm->mmap_lock);
/*
	For KVM
*/
	if(kvm_handle_async_pf(regs,(u32)address))/* 这部分不做说明 */
		return;
/*
        Entry handling for valid #PF from kernel mode is slightly different:RCU is already watching and
            rcu_irq_enter() must not be invoked because a kernel fault on a user space address might sleep.
        In case the fault hit a RCU idle region the conditional entry code reenabled RCU to avoid subsequent wreckage which helps debugability.
*/
	state = irqentry_enter(regs);
    instrumentation_begin();
	handle_page_fault(regs,error_code,address);/* 函数主题 */
	instrumentation_end();
	irqentry_exit(regs,state);
}
```

#### 调用handle_page_fault：

```ruby
static __always_inline void handle_page_fault(struct pt_regs *regs,unsigned long error_code unsigned long address)
{
	trace_page_fault_entries(regs,error_code,address);
	if(unlikely(kmmio_fault(regs,address)))
		return;
	if(unlikely(fault_in_kernel_space(address))) {
		do_kern_addr_fault(regs,error_code,address);/* 主体:内核空间 */
	} else {
		do_user_addr_fault(regs,error_code,address); /* 主体:应用空间 */
		local_irq_disable();
	}
}
```
##### do_kern_addr_fault

```ruby
/*
    注意，内核处理部分，因为内核在满足某些要求时不存在换出的问题，具体分析如下。
*/
static void do_kern_addr_fault(struct pt_regs *regs,unsigned long hw_error_code,unsigned long address)
{
/*
        Protection keys exceptions only happen on user pages.We have no user pages in the kernel portion of the address spae,so do not expect them here.
*/
WARN_ON_ONCE(hw_error_code & X86_PF_PK);
/*
       Was the fault spurious,caused by lazy TLB invalidation?
*/
if(spurious_kernel_fault(hw_error_code,address))/* 虚拟#PF，这个要深入分析。 */
	return;
/*
        kprobes do not want to hook the spurious faults:
*/
if(kprobe_page_fault(regs,X86_TRAP_PF)) /* kprobe来处理page fault:这个要做实验，很重要 */
	return;
/*
        Note,despite being a "bad area",there are quite a few acceptable reasons to get here,such a erratum fixups and handling kernel code that can fault,like get_user().
        Do not take the mm semaphore here.If we fixup a prefetch fault we could otherwise deadlock;
*/
bad_area_nosemaphore(regs,hw_error_code,address);/* 进行详细处理 */
}
```
##### do_user_addr_fault

```ruby
/*
    用户空间,这部分是重点
*/
static inline void do_user_addr_fault(struct pt_regs *regs,unsigned long hw_error_code,unsigned long address)
{
	struct vm_area_struct *vma;
	struct task_struct *tsk;
	struct mm_struct *mm;/* 针对task_struct 的内存空间 */
	vm_fault_t fault;
	unsigned int flags = FAULT_FALG_DEFAULT;
	
	tsk = current;
	mm = tsk->mm;
	/*
        Kprobes do not want to hook the spurious faults: 
	*/
	if(unlikely(kprobe_page_fault(regs,X86_TRAP_PF)))
		return;
     /* 
        Reserved bits are never expected to be set on entries in the user portion of the page tables.
	*/
     if(unlikely(hw_error_code & X86_PF_RSVD))
         pgtable_bad(regs,hw_error_code,address)
	/* 
         If SMAP is on,check for invalid kernel(supervisor) access to user
             page in the user address space.The odd case here is WRUSS which,according to the preliminary documentation,does not respect
                 SMAP and will have the USER bit set so,in all cases,SMAP enforcement appears to be consistent whith the USER bit */
                     
	if(unlikely(cpu_feature_enabled(X86_FEATURE_SMAP) &&
		!(hw_error_code & X86_PF_USER) &&
		!(regs->flags & X86_EFLAGS_AC)))
	{
			bad_area_nosemaphore(regs,hw_error_code,address);
			return;
	}
	/*
            If we are in an interrupt,have no user context or are running 
        	in a region with pagefaults disabled then we must not take the fault
	*/      
	if(unlikely(faulthandler_disabled() || !mm)){
		bad_area_nosemaphore(regs,hw_error_code,address);
		return;
	}
	/*
            It is safe to allow irqs after cr2 has been saved and the vmalloc fault has been handled.
            User-mod eregisters count as a user access even for any potential system fault or CPU buglet
	*/
	if(user_mode(regs)) {
		local_irq_enable();
		flags |= FAULT_FLAG_USER;
	} else {
		if(regs->flags & X86_EFLAGS_IF)
			local_irq_enable();
	}
	perf_sw_event(PERF_COUNT_SW_PAGE_PAULTS,1,regs,address);
	if(hw_error_code & X86_PF_WRITE)
		flags |= FAULT_FLAG_WRITE;
	if(hw_error_code & X86_PF_INSTR)
		flags |= FAULT_FLAG_INSTRUCTION;
	/*
	 只描述X86_64:
        Faults in the vsyscall page might need emulation:...
        此处不作为重点进行分析
	*/
	if(is_vsyscall_vaddr(address)) {
		if(emulate_vsyscall(hw_error_code),regs,address))
			return;
	}
	/*
		保证正确处理顺序的措施
	*/
	
	if(unlikely(!mmap_read_trylock(mm))) {
		if(!user_mode(regs) && !search_exception_tables(regs->ip)) {
			/*
                Fault from code in kernel from which we do not expect faults.
			*/
			bad_area_nosemaphore(regs,hw_error_code,address);
			return;
		}
retry:
		mmap_read_lock(mm);
	} else {
	/*
            The above down_read_trylock() might have succeeded in which case we will have missed the might_sleep() from down_read():
	*/
		might_sleep();
	}
	vma = find_vma(mm,address); /* 处理主体:找到包含虚拟地址的内存空间 */
	if(unlikely(!vma)) {/* 不在进程地址空间中 */
		bad_area(regs,hw_error_code,address);
		return;
	}
	
	if(likely(vma->vm_start <= address)) /* 包含在空间中 */
		goto good_area;
	
	if(unlikely(!(vma->vm_flags & VM_GROWSDOWN))) {/* 此时 address < vm_start,若非向小增加,则出错. */
		bad_area(regs,hw_error_code,address);
		return;
	}
	
	if(unlikely(expand_stack(vma,address))) { /* 若非栈,则... */
		bad_area(regs,hw_error_code,address);
		return;
	}
	/*
            处理主题:Ok,we have a good vm_area for this memory access,so we can handle it ...
	*/
	
good_area:
	if(unlikely(access_error(hw_error_code,vma))) {/* 因为权限等错误引起的#PG */
		bad_area_access_error(regs,hw_error_code,address,vma);
		return;
	}
	/*
	*/
	fault = handle_mm_fault(vma,address,flags,regs);/* 处理主体 */
	/*
            Quick path to respond to signals
	*/
	if(fault_signal_pending(fault,regs)) {
		if(!user_mode(regs))
			no_context(regs,hw_error_code,address,SIGBUS,BUS_ADRERR);
		return;
	}
	/*
            If we need to retry the mmap_lock has already been released,and if there is fatal signal pending there 
        is no guarantee that we made any progress.Handle this case first.
	*/
	if(unlikely((fault & VM_FAULT_RETRY) && (flags & FAULT_FLAG_ALLOW_RETRY))) {
		flags |= FAULT_FLAG_TRIED;
		goto retry;
	}
	
	mmap_read_unlock(mm);
	if(unlikely(fault & VM_FAULT_ERROR)) {/* 错误处理 */
		mm_fault_error(regs,hw_error_code,address,fault);
		return;
	}
	check_v8086_mode(regs,address,tsk);
}
	
```
###### handle_mm_fault

```ruby
/*
    By the time we get here,we already hold the mm semaphore
	The mmap_lock may have been released depending on flags and our return value.See filemap_fault() and __lock_page_or_retry().
*/
vm_fault_t handle_mm_fault(struct vm_area_struct *vma,unsigned long address,unsigned int flags,struct pt_regs *regs)
{
	vm_fault_t ret;
	__set_current_state(TASK_RUNNING);
	count_vm_event(PGFAULT); /* 计数 */
	count_memcg_event_mm(vma->vm_mm,PGFAULT);
	/*
        do counter updates before entering really critical section
	*/
	check_sync_rss_stat(current); /* 统计数据更新 */
	if(!arch_vma_access_permitted(vma,flags & FAULT_FLAG_WRITE,flags & FAULT_FLAG_INSTRUCTION,flags & FAULT_FLAG_REMOTE))
		return VM_FAULT_SIGSEGV;
	/*
            Enable the memcg OOM handling for faults triggered in user space.Kernel faults are handled more gracefully.
	*/
	if(flags & FAULT_FLAG_USER)
		mem_cgroup_enter_user_fault();
        
	if(unlikely(is_vm_hugetlb_page(vma)))
		ret = hugetlb_fault(vma->vm_mm,vma,address,flags);/* 处理主体 */
	else
		ret = __handle_mm_fault(vma,address,falgs); /* 处理主体 */
        
	if(flags & FAULT_FLAG_USER) {
		mem_cgroup_exit_user_fault();
		/*
            The task may have entered a memcg OOM situation but if the allocation error was handled gracefully(no VM_FAULT_OOM),there is no need to kill anything.Just clean up the OOM state peacefully.
		*/
		if(task_in_memcg_oom(current) && !(ret & VM_FAULT_OOM))
			mem_cgroup_oom_synchronize(false);
	}
	
	mm_account_fault(regs,address,flags,ret);/* 统计信息更新 */
	
	return ret;
}
```
###### 调用__handle_mm_fault

```ruby
/*
    By the time we get here,we already hold the mm semaphore
	The mmap_lock may have been released depending on flags and our return value.See filemap_fault() and __lock_page_or_retry().
*/
static vm_fault_t __handle_mm_fault(struct vm_area_struct *vma,unsigned long address,unsigned int flags)
{
	struct vm_fault vmf = { /* 这个结构我们需要注意 */
		.vma = vma,
		.address = address & PAGE_MASK,
		.flags = flags,
		.pgoff = linear_page_index(vma,address);
		.gfp_mask = __get_fault_gfp_mask(vma),
	};
	unsigned int dirty = flags & FAULT_FLAG_WRITE;
	struct mm_struct *mm = vma->vm_mm;
	pgd_t *pgd;
	p4d_t *p4d;
	vm_fault_t ret;
	pgd = pgd_offset(mm,address);
	p4d = p4d_alloc(mm,pgd,address);
	if(!p4d)
		return VM_FAULT_OOM;
	vmf.pud = pud_alloc(mm,p4d,address);
	if(!vmf.pud)
		return VM_FAULT_OOM;
retry_pud:
	if(pud_none(*vmf.pud) && __transparent_hugepage_enabled(vma)) {
		ret = ceate_huge_pud(&vmf);
		if(!(ret & VM_FAULT_FALLBACK))
			return ret;
	} else {
		pud_t orig_pud = *vmf.pud;
		
		barrier();
		if(pud_trans_huge(orig_pud) || pud_devmap(orig_pud)) {
			/*
				*/
			if(dirty && !pud_write(orig_pud)) {
				ret = wp_huge_pud(&vmf,orig_pud);
				if(!(ret & VM_FAULT_FALLBACK))
					return ret;
			} else {
				huge_pud_set_accessed(&vmf,orig_pud);
				return 0;
			}
		}
	}
	vmf.pmd = pmd_alloc(mm,vmf.pud,address);
	if(!vmf.pmd)
		return VM_FAULT_OOM;
	/*
	*/
	if(pud_trans_unstable(vmf.pud))
		goto retry_pud;
	if(pmd_none(*vmf.pmd) && __transparent_hugepage_enabled(vma)) {
		ret = create_huge_pmd(&vmf);
		if(!(ret & VM_FAULT_FALLBACK))
			return ret;
	} else {
		pmd_t orig_pmd = *vmf.pmd;
		barrier();
		if(unlikely(is_swap_pmd(orig_pmd))) {
			VM_BUG_ON(thp_migration_supported() &&
				!is_pmd_migration_entry(orig_pmd));
			if(is_pmd_migration_entry(orig_pmd))
				pmd_migration_entry_wait(mm,vmf.pmd);
			return 0;
		}
		if(pmd_trans_huge(orig_pmd) || pmd_devmap(orig_pmd)) {
			if(pmd_protnone(orig_pmd) && vma_is_acessible(vma))
				return do_huge_pmd_numa_page(&vmf,orig_pmd);
			if(dirty && !pmd_write(orig_pmd)) {
				ret = wp_huge_pmd(&vmf,orig_pmd);
				if(!(ret & VM_FAULT_FALLBACK))
					return ret;
			} else {
				huge_pmd_set_accessed(&vmf,orig_pmd);
				return 0;
			}	
		}
	}
	return handle_pte_fault(&vmf);/* 处理主体 */
}
```
###### 调用handle_pte_fault

```ruby
/*
    By the time we get here,we already hold the mm semaphore
	The mmap_lock may have been released depending on flags and our return value.See filemap_fault() and __lock_page_or_retry().
*/
static vm_fault_t handle_pte_fault(struct vm_fault *vmf)
{
	pte_t entry;
	if(unlikely(pmd_none(*vmf->pmd))) {
	/*
	*/
		vmf->pte = NULL;
	} else {
	/* */
		if(pmd_devmap_trans_unstable(vmf->pmd))
			return 0;
		/*
		*/
		vmf->pte = pte_offset_map(vmf->pmd,vmf->address);
		vmf->orig_pte = *vmf->pte;
		/*
		*/
		barrier();
		if(pte_none(vmf->orig_pte)){
			pte_unmap(vmf->pte);
			vmf->pte = NULL;
		}
	}
	if(!vmf->pte){
		if(vma_is_anonymous(vmf->vma))
			return do_anonymous_page(vmf);/* 匿名映射:代码段,数据段,栈段: 第一次映射 */
		else
			return do_fault(vmf); /* 非匿名映射: 第一次映射*/
	}
	if(!pte_present(vmf->orig_pte))
		return do_swap_page(vmf); /* 交换分区部分 */
	if(pte_protnone(vmf->orig_pte) && vma_is_accessible(vmf->vma))
		return do_numa_page(vmf);/* */
	
	vmf->ptl = pte_lockptr(vmf->vma->vm_mm,vmf->pmd);
	spin_lock(vmf->ptl);
	entry = vmf->orig_pte;
	if(unlikely(!pte_same(*vmf->pte,entry))) {
		update_mmu_tlb(vmf->vma,vmf->address,vmf->pte);
		goto unlock;
	}
	
	if(vmf->flags & FAULT_FLAG_WRITE) {
		if(!pte_write(entry))
			return do_wp_page(vmf);
		entry = pte_mkdirty(entry);
	}
	entry = pte_mkyoung(entry);
	if(ptep_set_access_flags(vmf->vma,vmf->address,vmf->pte,entry,vmf->flags & FAULT_FLAG_WRITE)) {
		update_mmu_cache(vmf->vma,vmf->address,vmf->pte);
	} else {
	/* */
		if(vmf->flags & FAULT_FLAG_TRIED)
			goto unlock;
			/*
			*/
		if(vmf->flags & FAULT_FLAG_WRITE)
        	flush_tlb_fix_spurious_fault(vmf->vma,vmf->address);
	}
	
unlock:
	pte_unmap_unlock(vmf->pte,vmf->ptl);
	return 0;
}
```

```ruby
我们先看重要结构：
/*
	由pagefault处理程序填充，并传递给vma's的fault function句柄。vma's->fault负责返回一个以VM_FAULT_XX格式的位掩码标志，描述错误处理细节。
	MM层负责为页面分配填充gfp_mask，但是如果其实现需要不同的分配上下文，则故障处理程序可能会对其进行更改 。
	如果可能，应该使用pgoff来支持virtual_address。
*/
struct vm_fault {
	struct vm_area_struct *vma; /*目标VMA */
	unsigned ing flags; /* FAULT_FLAG_xxx 标志 */
	gfp_t gfp_mask; /*用于分配器的gfp掩码 */
	pgoff_t pgoff; /* 基于vma的逻辑页偏移 */
	unsigned long address; /* 产生错误的虚拟地址 */
	pmd_t *pmd; /* 与"address"对应的pmd项 */
	pud_t *pud; /* 与 "address"对应的pud项 */
	pte_t orig_pte; /* 发生错误时的PTE值 */
	struct page *cow_page; /* COW fault可能用的页处理程序 */
	struct page *page; /* 除非设置了VM_FAULT_NOPAGE（VM_FAULT_ERROR也暗示了这一点，如何理解？）,否则这个地方的->fault 处理程序应该返回一个页面*/
	/* 以下三个条目，仅在获取到ptl锁时才有效  */
	pte_t *pte; /*指向"address"对应的pte entry。如果页表没有分配，则为NULL。 */
	spinlock_t *ptl; /* 页表锁。
					 * 如果"pte"不为NULL，则保护页表，否则保护pmd。 */
	pgtable_t prealloc_pte; /* 预分配的PTE 页表。vm_ops->map_pages()从原子上下文调用alloc_set_pte()。

}
```

```ruby
下面看错误对应的处理方式：
!vmf->pte：第一次映射
		if(vma_is_anonymous(vmf->vma))
			return do_anonymous_page(vmf);/* 匿名映射 */
		else
			return do_fault(vmf); /* 文件映射 */
		分析：
!pte_present(vmf->orig_pte)：已经有PTE，如果只是不在内存，就在交换分区。
		do_swap_page(vmf);	
```

```ruby
do_anonymous_page(vmf): mm/memory.c
/*
    匿名映射
*/
static vm_fault_t do_anonymous_page(struct vm_fault *vmf)
{
	handle_userfault(vmf,VM_UFFD_MISSING);/* 分配页面 */
}
```

```ruby
do_fault(vmf): mm/memory.c
/*
    文件映射
*/
static vm_fault_t do_fault(struct vm_fault *vmf)
{
     !FAULT_FALG_WRITE:	do_read_fault(vmf);
     FAULT_FLAG_WRITE && (!VM_SHARED): do_cow_fault(vmf);
     FAULT_FALG_WRITE && VM_SHARED: do_shared_fault;
}
```

```ruby
do_swap_page(vmf)
/*
  顺便把交换分区给搞定了：
  交换分区应该做为文件系统处理吗？
  /proc/swaps
  mkswap /dev/...：将分区或文件创建成swap空间。
  
  kswapd 进程: mm/vmscan.c
  内存回收主要针对内存中的文件页面（file cache)和匿名页
  1：anon 匿名页内存主要回收手段是swap;
  2:file-backed的文件映射页，主要的释放手段是写回和清空。因为有硬盘文件对应，所以不走交换分区路径，直接写回，并清空内存（也就是说保存映射结构，但释放掉物理页）。 
*/
vm_fault_t do_swap_page(struct vm_fault *vmf)
{
	struct vm_area_struct *vma = vmf->vma;
	struct page *page = NULL,*swapcache;
	swp_entry_t entry;
	pte_t pte;
	int locked;
	int exclusive = 0;
	vm_fault_t ret = 0;
	void *shadow = NULL;

	entry = pte_to_swp_entry(vmf->orig_pte);
	delayacct_set_flag(DELAYACCT_PF_SWAPIN);
    page = lookup_swap_cache(entry,vma,vmf->address);
    swapcache = page;

}
```
#### 函数流程图

<pre><code class="language-mermaid">
graph TB
pgh0("asm_exc_page_fault") -->pgh1("exc_page_fault");
pgh1 --> pgh10("pre:上下文相关处理")
pgh10 --> pgh2("handle_page_fault");
pgh2 -."kmmio_fault:true".-> pgh20001("return")
pgh2-."kmmio_fault:false".-> pgh20002("fault_in_kernel_space")
pgh20002 -. "true:#PF发生在内核空间" .-> pgh20("do_kern_addr_fault:内核空间处理")
pgh20 -. "spurious fault" .-> pgh201("spurious_kernel_fault_check")
pgh20 -. "地址非法" .-> pgh202("bad_area_nosemaphore")
pgh202 -.-> pgh2020("is_f00f_bug")
pgh2020 -.-> pgh2021("no_context")
pgh2021 -.-> noc0("fixup_exception")
noc0 -."in_interrupt".-> noc1("return")
noc0 -."!in_interrupt".-> noc2("处理信号 SIGSEGV") -.-> noc3("return")
pgh2021 -.->noc4("!fixup_exception")
noc4 -."CONFIG_VMAP_STACK:stack overflow".->noc5("handle_stack_overflow")
noc5 -.->noc6("1:jmp 1b:kernel stack overflow:进入死循环:系统卡死")
noc4 -. "is_prefetch:" .-> noc8("return")
noc4 -. "is_errata93:" .-> noc9("return")
noc4 -. "CONFIG_EFI".-> noc10("efi_recover_from_page_fault")  -.-> noc11("oops")
noc4 -. "!CONFIG_EFI".->noc12("oops")
pgh2 -. "#PF发生在用户空间" .-> pgh21("do_user_addr_fault:用户空间处理")
pgh21 -."hw_error_code:合法缺页外的错误码".->pgh22("hw_error_code:处理")
pgh21 -."hw_error_code:缺页: 非进程空间：越界".-> pgh23("bad_area_acess_error:return")
pgh21 -."hw_error_code:缺页：vma进程空间内".->pgh24("handle_mm_fault")
pgh24 -.-> pgh25("__handle_mm_fault")
pgh25 -.-> pgh26("handle_pte_fault")
pgh26 -."匿名空间:无pte".->pgh27("do_anonymous_page")
pgh26 -."非匿名空间:无pte".->pgh28("do_fault")
pgh26 -."pte:!pte_present:交换空间".->pgh29("do_swap_page")
pgh2 --> pgh55("post:上下文处理:返回") --> pgh56("INTERRUPT_RETURN")
</code></pre>

#### 总结

以上是缺页中断的所有描述，针对linux 代码后期会完善所有注释，注意最终的页面分配，在内存部分进行整理。

