内核级异常处理
---------------

这部分的理解是非常清晰的。

进程运行在内核模式时，通常需要访问由不被信任的程序传递过来的处于用户模式的内存地址。为了自我保护，内核需要对这个地址进行校验。

历史：

旧版本linux通过verify_area(int type,const void *addr,unsigned longsize)函数来实现（后来被access_ok()函数代替）。这个函数判断这个地址范围是否能进行特定类型（读/写）的访问。verify_read需要查询包含地址的vma。正常情况下（正确运行的程序），这个测试是成功的。只在少数有bug的程序中会出现错误的情况。在一些内核分析测试中，这种验证会占用大量时间。

为了解决这个问题，Linux决定由虚拟内存硬件来处理这个测试。

实现原理为：

当内存试图访问一个当前不能访问的地址时，CPU产生一个页错误一场，并调用页错误处理程序：

.. code-block:: sh
   :caption: 缺页异常处理程序
   :emphasize-lines: 2
   :linenos:
   
   void do_page_fault(struct pt_regs *regs, unsigned long error_code)
   

在arch/x86/mm/fault.c。堆栈上的参数由 arch/x86/entry/entry_32.S 中汇编程序设置。参数 regs 是指向堆栈上保存的寄存器的指针，error_code 包含异常的原因代码。

do_page_fault 首先从控制寄存器CR2读取不可访问地址。如果地址在进程的虚拟地址空间中，可能会以为页面没有换进来，写保护或类似的问题产生错误。我们更关心另外一种情况：地址无效，没有包含地址的vma。这种情况下，内核跳转到bad_area 标签。

此时通过产生一场的指令的地址（如regs->eip)来查询一个继续运行的地址（fixup)。如果查询成功，错误处理程序修改返回地址，并返回。一场处理在这个地址上继续运行。

fixup指向的位置
由于我们跳转到 fixup 的内容，fixup指向的是可执行代码。此代码隐藏在用户访问宏中。以 arch/x86/include/asm/uaccess.h 中定义的 get_user 宏作为示例。这个定义有点难以理解，所以让我们看看预处理器和编译器生成的代码。选择在 drivers/tty/sysrq.c 中 get_user 调用进行详细检查。

sysrq.c,1155行：

.. code-block:: sh
   :caption: get_user
   :emphasize-lines: 2
   :linenos:
   
   get_user(c, buf);
   
预处理输出：

插入一下：如何与编译某些内核文件。

.. code-block:: sh
   :caption: 与编译sysrq.c
   :emphasize-lines: 2
   :linenos:
   
   make driver/tty/sysrq.i  //预编译

相关代码为：

.. code-block:: sh
   :caption: get_user
   :emphasize-lines: 2
   :linenos:
   (
     {
   	might_fault(); 
   	(
   	  { 
   	    int __ret_gu;
   	    register __typeof__( __builtin_choose_expr(sizeof(*(buf))<=sizeof(char),(unsigned char)0,__builtin_choose_expr(sizeof(*(buf))<=sizeof(short),(unsigned short)0,__builtin_choose_expr(sizeof(*(buf))<=sizeof(int),(unsigned int)0,__builtin_choose_expr(sizeof(*(buf))<=sizeof(long),(unsigned long)0,0ULL))))) __val_gu asm("%""rdx"); (void)0; 
   	    asm volatile("call __" "get_user" "_%P4" : "=a" (__ret_gu), "=r" (__val_gu), "+r" (current_stack_pointer) : "0" (buf), "i" (sizeof(*(buf)))); (c) = ( __typeof__(*(buf))) __val_gu; __builtin_expect(__ret_gu, 0); }); 
   	    
   	    }
   	)
   

结构定义：


汇编代码：




section __ex_table
.pushsection __ex_table


所有__ex_table成员放入__ex_table节中

arch/x86/kernel/vmlinux.lds

. = ALIGN(16); __ex_table : AT(ADDR(__ex_table) - 0xffffffff80000000) { __start___ex_table = .; KEEP(*(__ex_table)) __stop___ex_table = .; }


我们再来看定义：

extern struct exception_table_entry __start___ex_table[];
extern struct exception_table_entry __stop___ex_table[];


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


也就是说 __ex_table节是struct exception_table_entry结构的列表。


流程为：

那么，如果内核模式的故障没有合适的 vma 发生，实际会发生什么呢？

1. 访问无效地址：

.. code-block:: sh
   :caption: 访问无效地址
   :emphasize-lines: 2
   :linenos:

   > c017e7a5 <do_con_write+e1> movb   (%ebx),%dl
   
2. MMU 产生异常

3. CPU 调用 do_page_fault

4. 执行页面错误调用 search_exception_table (regs->eip == c017e7a5);

5. search_exception_table 在异常表中查找地址c017e7a5（即ELF 部分__ex_table 的内容）并返回相关故障句柄代码c0199ff5 的地址。

6. do_page_fault 修改自己的返回地址指向故障句柄代码并返回。

7. 在故障处理代码中继续执行。
8. 如下注意：
   a. EAX 变为 -EFAULT (== -14)
   b. DL 变为零（我们从用户空间“读取”的值）
   c. 在本地标签 2 处继续执行（错误用户访问后立即执行的指令地址）。

步骤8a到8c以某种方式模拟了故障指令。

就是这样，主要是。如果您查看我们的示例，您可能会问为什么我们在异常处理程序代码中将 EAX 设置为 -EFAULT。嗯，get_user 宏实际上返回一个值：0，如果用户访问成功，-EFAULT 表示失败。我们的原始代码没有测试这个返回值，但是 get_user 中的内联汇编代码尝试返回 -EFAULT。GCC 选择 EAX 来返回这个值。

注意：由于异常表的构建方式需要排序，因此仅对 .text 部分中的代码使用异常。任何其他部分都将导致异常表无法正确排序，并且异常将失败。

当 64 位支持添加到 x86 Linux 时，情况发生了变化。不是通过将两个条目从 32 位扩展到 64 位来将异常表的大小加倍，而是使用了一个聪明的技巧来将地址存储为与表本身的相对偏移量。汇编代码从：





在 v4.6 中，异常表条目扩展了一个新字段“处理程序”。这也是 32 位宽并包含第三个相对函数指针，它指向以下之一：

1. int ex_handler_default(const struct exception_table_entry *fixup)
这是仅跳转到修复代码的遗留案例

2. int ex_handler_fault(const struct exception_table_entry *fixup)
本案例提供在 entry->insn 发生的陷阱的故障编号。它用于区分页面错误和机器检查。

可以轻松添加更多功能。

CONFIG_BUILDTIME_TABLE_SORT 允许通过主机实用程序脚本/排序表对内核映像的链接后的 __ex_table 部分进行排序。它将符号 main_extable_sort_needed 设置为 0，避免在引导时对 __ex_table 部分进行排序。通过对异常表进行排序，在运行时发生异常时，我们可以通过二进制搜索快速查找 __ex_table 条目。

这不仅仅是启动时间优化，一些体系结构要求对这个表进行排序，以便在启动过程中相对较早地处理异常。例如，i386 在启用分页支持之前就使用了这种形式的异常处理！


具体代码分析：

do_page_fault:


























