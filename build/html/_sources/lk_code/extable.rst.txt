异常表:extable
^^^^^^^^^^^^^^^^^^^^^^

- 异常表：用于存储发生pf的地址的处理方式。
   
	.. code-block:: c
	   :caption: 异常表全局结构
	   :linenos:
		/*
		 * mutex protecting text section modification (dynamic code patching).
		 * some users need to sleep (allocating memory...) while they hold this lock.
		 *
		 * Note: Also protects SMP-alternatives modification on x86.
		 *
		 * NOT exported to modules - patching kernel text is a really delicate matter.
		 */
		DEFINE_MUTEX(text_mutex);

		extern struct exception_table_entry __start___ex_table[];
		extern struct exception_table_entry __stop___ex_table[];


- 我们观察struct exception_table_entry结构：

.. code-block:: c
   :caption: 异常表项结构
   :linenos:

	/*
	 * The exception table consists of triples of addresses relative to the
	 * exception table entry itself. The first address is of an instruction
	 * that is allowed to fault, the second is the target at which the program
	 * should continue. The third is a handler function to deal with the fault
	 * caused by the instruction in the first field.
	 *
	 * All the routines below use bits of fixup code that are out of line
	 * with the main instruction path.  This means when everything is well,
	 * we don't even have to jump over them.  Further, they do not intrude
	 * on our cache or tlb entries.
	 */

	struct exception_table_entry {
		int insn, /* 出错的指令 */
		    fixup, /* 继续的指令 */
		    handler; /* 错误处理句柄 */
	};
	struct pt_regs;

	#define ARCH_HAS_RELATIVE_EXTABLE

	#define swap_ex_entry_fixup(a, b, tmp, delta)			\
		do {							\
			(a)->fixup = (b)->fixup + (delta);		\
			(b)->fixup = (tmp).fixup - (delta);		\
			(a)->handler = (b)->handler + (delta);		\
			(b)->handler = (tmp).handler - (delta);		\
		} while (0)

	enum handler_type {
		EX_HANDLER_NONE,
		EX_HANDLER_FAULT,
		EX_HANDLER_UACCESS,
		EX_HANDLER_OTHER
	};

	extern int fixup_exception(struct pt_regs *regs, int trapnr,
				   unsigned long error_code, unsigned long fault_addr);
	extern int fixup_bug(struct pt_regs *regs, int trapnr);
	extern enum handler_type ex_get_fault_handler_type(unsigned long ip);
	extern void early_fixup_exception(struct pt_regs *regs, int trapnr);



- kernel/vmlinux.ld

.. code-block:: c
   :caption: 异常表在内核中的布局
   :linenos:

	. = ALIGN(16); __ex_table : AT(ADDR(__ex_table) - 0xffffffff80000000) { __start___ex_table = .; KEEP(*(__ex_table)) __stop___ex_table = .; }


- 操作函数：

.. code-block:: c
   :caption: 异常表项查询
   :linenos:

	/* Sort the kernel's built-in exception table */
	void __init sort_main_extable(void)
	{
		if (main_extable_sort_needed &&
		    &__stop___ex_table > &__start___ex_table) {
			pr_notice("Sorting __ex_table...\n");
			sort_extable(__start___ex_table, __stop___ex_table);
		}
	}
 
	/* Given an address, look for it in the kernel exception table */
	const
	struct exception_table_entry *search_kernel_exception_table(unsigned long addr)
	{
		return search_extable(__start___ex_table,
				      __stop___ex_table - __start___ex_table, addr);
	}

	/* Given an address, look for it in the exception tables. */
	const struct exception_table_entry *search_exception_tables(unsigned long addr)
	{
		const struct exception_table_entry *e;

		e = search_kernel_exception_table(addr);/* 内核代码中的异常表 */
		if (!e)
			e = search_module_extables(addr); /* 内核模块中的异常表 */
		if (!e)
			e = search_bpf_extables(addr); /* bpf中的异常表 */
		return e;
	}



- 添加项的时机：模块或系统调用莫个地址处：这个主要用在pf处使用，就是当某个地址访问有问题时本身自带处理方法。

编程示例
"""""""

struct module {
	......
	unsigned int num_exentires;
	struct exception_table_entry *extable;
	......
}

mod->extable = section_objs(info,"__ex_table",sizeof(*mod->extable),&mod->num_exentries);







struct bpf_prog_aux {
	......
	u32 num_exentries;
	struct exception_table_entry *extable;
	......
}

- 代码位置：/usr/src/k_mod_sample_extable
- 内核文档参考： https://kernel.org/doc/html/latest/x86/exception-tables.html


