linux 栈跟踪管理：unwind_{frame,guess,orc},stacktrace
------------------------------------------------------
.. code-block:: c
   :caption: struct_task --> mm
   :emphasize-lines: 4,5
   :linenos:

对栈帧，栈跟踪基础及应用进行总结，分析

应用场景
^^^^^^^^
panic() --> dump_stack()


原理分析
^^^^^^^^

结构
"""""
struct unwind_state {


}


struct pt_regs {



}


struct stack_frame_user {


}

struct stack_trace_consume {

}

struct stack_trace {



}


函数说明
^^^^^^^^^
通用部分（导出）
"""""""""""""

- stack_trace_print:
  - 说明：
    /**
     * stack_trace_print - Print the entries in the stack trace
     * @entries:	Pointer to storage array
     * @nr_entries:	Number of entries in the storage array
     * @spaces:	Number of leading spaces to print
     */
     
- int stack_trace_snprint(char *buf, size_t size, const unsigned long *entries,
			unsigned int nr_entries, int spaces)

   /**
 * stack_trace_snprint - Print the entries in the stack trace into a buffer
 * @buf:	Pointer to the print buffer
 * @size:	Size of the print buffer
 * @entries:	Pointer to storage array
 * @nr_entries:	Number of entries in the storage array
 * @spaces:	Number of leading spaces to print
 *
 * Return: Number of bytes printed.
 */
 
 
- unsigned int stack_trace_save(unsigned long *store, unsigned int size,
			      unsigned int skip）
			  
   - /**
 * stack_trace_save - Save a stack trace into a storage array
 * @store:	Pointer to storage array
 * @size:	Size of the storage array
 * @skipnr:	Number of entries to skip at the start of the stack trace
 *
 * Return: Number of trace entries stored.
 */


- unsigned int stack_trace_save_tsk(struct task_struct *tsk, unsigned long *store,
				  unsigned int size, unsigned int skipnr)
				  
  - /**
 * stack_trace_save_tsk - Save a task stack trace into a storage array
 * @task:	The task to examine
 * @store:	Pointer to storage array
 * @size:	Size of the storage array
 * @skipnr:	Number of entries to skip at the start of the stack trace
 *
 * Return: Number of trace entries stored.
 */
 

- unsigned int stack_trace_save_regs(struct pt_regs *regs, unsigned long *store,
				   unsigned int size, unsigned int skipnr)
  - 说明
    /**
 * stack_trace_save_regs - Save a stack trace based on pt_regs into a storage array
 * @regs:	Pointer to pt_regs to examine
 * @store:	Pointer to storage array
 * @size:	Size of the storage array
 * @skipnr:	Number of entries to skip at the start of the stack trace
 *
 * Return: Number of trace entries stored.
 */
 
 
 
- int stack_trace_save_tsk_reliable(struct task_struct *tsk, unsigned long *store,
				  unsigned int size)
  - 说明
  /**
 * stack_trace_save_tsk_reliable - Save task stack with verification
 * @tsk:	Pointer to the task to examine
 * @store:	Pointer to storage array
 * @size:	Size of the storage array
 *
 * Return:	An error if it detects any unreliable features of the
 *		stack. Otherwise it guarantees that the stack trace is
 *		reliable and returns the number of entries stored.
 *
 * If the task is not 'current', the caller *must* ensure the task is inactive.
 */
 
 
 				  				   
- unsigned int stack_trace_save_user(unsigned long *store, unsigned int size)
  - 说明
    /**
 * stack_trace_save_user - Save a user space stack trace into a storage array
 * @store:	Pointer to storage array
 * @size:	Size of the storage array
 *
 * Return: Number of trace entries stored.
 */ 
 
-  unsigned int stack_trace_save(unsigned long *store, unsigned int size,
			      unsigned int skipnr)
   - 说明
   /**
 * stack_trace_save - Save a stack trace into a storage array
 * @store:	Pointer to storage array
 * @size:	Size of the storage array
 * @skipnr:	Number of entries to skip at the start of the stack trace
 *
 * Return: Number of trace entries stored
 */
 
 
- unsigned int stack_trace_save_tsk(struct task_struct *task,
				  unsigned long *store, unsigned int size,
				  unsigned int skipnr)
  - 说明
   /**
 * stack_trace_save_tsk - Save a task stack trace into a storage array
 * @task:	The task to examine
 * @store:	Pointer to storage array
 * @size:	Size of the storage array
 * @skipnr:	Number of entries to skip at the start of the stack trace
 *
 * Return: Number of trace entries stored
 */
 
 
 
- int stack_trace_save_tsk_reliable(struct task_struct *tsk, unsigned long *store,
				  unsigned int size)				  			      
  - 说明
  /**
 * stack_trace_save_tsk_reliable - Save task stack with verification
 * @tsk:	Pointer to the task to examine
 * @store:	Pointer to storage array
 * @size:	Size of the storage array
 *
 * Return:	An error if it detects any unreliable features of the
 *		stack. Otherwise it guarantees that the stack trace is
 *		reliable and returns the number of entries stored.
 *
 * If the task is not 'current', the caller *must* ensure the task is inactive.
 */
 
 
- unsigned int stack_trace_save_user(unsigned long *store, unsigned int size) 
  - 说明
  /**
 * stack_trace_save_user - Save a user space stack trace into a storage array
 * @store:	Pointer to storage array
 * @size:	Size of the storage array
 *
 * Return: Number of trace entries stored
 */
 
架构相关(底层实现）
"""""""""""""""

- void arch_stack_walk(stack_trace_consume_fn consume_entry, void *cookie,
		     struct task_struct *task, struct pt_regs *regs)
- int arch_stack_walk_reliable(stack_trace_consume_fn consume_entry,
			     void *cookie, struct task_struct *task)		     
			       
- static int copy_stack_frame(const struct stack_frame_user __user *fp,
		 struct stack_frame_user *frame) 
 
- void arch_stack_walk_user(stack_trace_consume_fn consume_entry, void *cookie,
			  const struct pt_regs *regs)
			  

底层uwind原理分析
^^^^^^^^^^^^^^^^^

内核文档：https://www.kernel.org/doc/html/latest/x86/orc-unwinder.html



内核配置：CONFIG_DWARF/CONFIG_FRAME
"""""""""""""""""""""""""""""""""

Choose kernel unwinder(ORC unwinder) -->
	- ORC unwinder -- CONFIG_UNWINDER_ORC -- unwind_orc.c (DWARF)
	- Frame pointer unwinder -- CONFIG_UNWINDER_FRAME_POINTER -- unwind_frame.c
	- Guess unwinder -- CONFIG_UNWINDER_GUESS  -- unwind_guess.c
			   				 
这个选项决定了panics,oopses,bugs,warnings,perf,/proc/<pid>/stack,livepatch,lockdep等使用的内核栈展开方法。


信息添加原理
""""""""""
引用文档：tools/objtool/Documentation/stack-validation.txt

- sorttable.h: 添加ORC unwind tables排序支持和其他更新
- ORC 数据由 objtool 生成
Objtool 通过与编译时堆栈元数据验证功能集成来生成 ORC 数据，在 tools/objtool/Documentation/stack-validation.txt 中有详细描述。在分析 .o 文件的所有代码路径后，它会创建一个 orc_entry 结构数组，以及与这些结构关联的指令地址的并行数组，并将它们分别写入 .orc_unwind 和 .orc_unwind_ip 部分。

ORC 代表糟糕的倒带能力。



















