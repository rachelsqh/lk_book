linux 异常表:extable
--------------------------
.. code-block:: c
   :caption: struct_task --> mm
   :emphasize-lines: 4,5
   :linenos:
   
   
异常表：用于存储发生pf的地址的处理方式。
   
   
   
   ```
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
```

我们观察struct exception_table_entry结构：

```
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
	int insn, fixup, handler;
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

```

kernel/vmlinux.ld

```
. = ALIGN(16); __ex_table : AT(ADDR(__ex_table) - 0xffffffff80000000) { __start___ex_table = .; KEEP(*(__ex_table)) __stop___ex_table = .; }
```

操作函数：

```
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

	e = search_kernel_exception_table(addr);
	if (!e)
		e = search_module_extables(addr);
	if (!e)
		e = search_bpf_extables(addr);
	return e;
}

```

添加项的时机：模块或系统调用莫个地址处：这个主要用在pf处使用，就是当某个地址访问有问题时本身带的处理方法。

