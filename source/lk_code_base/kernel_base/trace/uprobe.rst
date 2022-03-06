.. Rachel's E-book documentation master file, created by
   sphinx-quickstart on Sun Jan  9 16:38:00 2022.
   You can adapt this file completely to your liking, but it should at least
   contain the root `toctree` directive.

用户空间探针（uprobes)
--------------------------

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
^^^^^^^^^^^^^^^^^^^^
kernel/trace/trace_uprobe.c

debugfs:

- /sys/kernel/debug/tracing/uprobe_events 
- /sys/kernel/debug/tracing/uprobe_profile：配置文件：
  

要启用此功能，请使用 CONFIG_UPROBE_EVENTS=y 构建内核。通过 /sys/kernel/debug/tracing/uprobe_events 添加探测点，uprobe 事件接口希望用户计算对象中探测点的偏移量。可以使用 /sys/kernel/debug/tracing/dynamic_events 代替 uprobe_events。该接口也将提供对其他动态事件的统一访问。

uprobe_tracer
""""""""""""""
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
"""""""""
您可以通过 /sys/kernel/debug/tracing/uprobe_profile 检查每个事件的探测命中总数。第一列是文件名，第二列是事件名称，第三列是探测命中数。

使用示例
""""""""
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







-------------


初步总结，这些总归还是在于在某个对应的进程应用地址空间加入跳转指令，hook代码则在内核代码中实现：应该是int3处理程序中完成。（待确定）。































