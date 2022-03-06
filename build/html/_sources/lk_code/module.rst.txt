linux 内核模块原理分析
^^^^^^^^^^^^^^^^^^^^
linux 内核模块编程基础
"""""""""""""""""""
内核模块数据结构
**************

- 每个模块由一个struct module表示

.. code-block:: c
	:caption: struct kset_uevent_ops
	:emphasize-lines: 4,5
	:linenos:
	struct module {
		enum module_state state;//记录模块状态，如正常，加载，卸载等：	MODULE_STATE_LIVE/MODULE_STATE_COMING/MODULE_STATE_GOING/MODULE_STATE_UNFORMED

		/* Member of list of modules */
		struct list_head list;

		/* Unique handle for this module */
		char name[MODULE_NAME_LEN];//模块名

	#ifdef CONFIG_STACKTRACE_BUILD_ID
		/* Module build ID */
		unsigned char build_id[BUILD_ID_SIZE_MAX];//build id 名
	#endif

		/* Sysfs stuff. */
		struct module_kobject mkobj;
		struct module_attribute *modinfo_attrs;
		const char *version;
		const char *srcversion;
		struct kobject *holders_dir;

		/* Exported symbols */
		const struct kernel_symbol *syms;
		const s32 *crcs;
		unsigned int num_syms;

	#ifdef CONFIG_CFI_CLANG
		cfi_check_fn cfi_check;
	#endif

		/* Kernel parameters. */
	#ifdef CONFIG_SYSFS
		struct mutex param_lock;
	#endif
		struct kernel_param *kp;
		unsigned int num_kp;

		/* GPL-only exported symbols. */
		unsigned int num_gpl_syms;
		const struct kernel_symbol *gpl_syms;
		const s32 *gpl_crcs;
		bool using_gplonly_symbols;

	#ifdef CONFIG_MODULE_SIG
		/* Signature was verified. */
		bool sig_ok;
	#endif

		bool async_probe_requested;

		/* Exception table */
		unsigned int num_exentries;
		struct exception_table_entry *extable;//异常表

		/* Startup function. */
		int (*init)(void);//初始化句柄

		/* Core layout: rbtree is accessed frequently, so keep together. */
		struct module_layout core_layout __module_layout_align;
		struct module_layout init_layout;

		/* Arch-specific module values */
		struct mod_arch_specific arch;

		unsigned long taints;	/* same bits as kernel:taint_flags */

	#ifdef CONFIG_GENERIC_BUG
		/* Support for BUG */
		unsigned num_bugs;
		struct list_head bug_list;
		struct bug_entry *bug_table;
	#endif

	#ifdef CONFIG_KALLSYMS
		/* Protected by RCU and/or module_mutex: use rcu_dereference() */
		struct mod_kallsyms __rcu *kallsyms;
		struct mod_kallsyms core_kallsyms;

		/* Section attributes */
		struct module_sect_attrs *sect_attrs;

		/* Notes attributes */
		struct module_notes_attrs *notes_attrs;
	#endif

		/* The command line arguments (may be mangled).  People like
		   keeping pointers to this stuff */
		char *args;

	#ifdef CONFIG_SMP
		/* Per-cpu data. */
		void __percpu *percpu;
		unsigned int percpu_size;
	#endif
		void *noinstr_text_start;
		unsigned int noinstr_text_size;

	#ifdef CONFIG_TRACEPOINTS
		unsigned int num_tracepoints;
		tracepoint_ptr_t *tracepoints_ptrs;
	#endif
	#ifdef CONFIG_TREE_SRCU
		unsigned int num_srcu_structs;
		struct srcu_struct **srcu_struct_ptrs;
	#endif
	#ifdef CONFIG_BPF_EVENTS
		unsigned int num_bpf_raw_events;
		struct bpf_raw_event_map *bpf_raw_events;
	#endif
	#ifdef CONFIG_DEBUG_INFO_BTF_MODULES
		unsigned int btf_data_size;
		void *btf_data;
	#endif
	#ifdef CONFIG_JUMP_LABEL
		struct jump_entry *jump_entries;
		unsigned int num_jump_entries;
	#endif
	#ifdef CONFIG_TRACING
		unsigned int num_trace_bprintk_fmt;
		const char **trace_bprintk_fmt_start;
	#endif
	#ifdef CONFIG_EVENT_TRACING
		struct trace_event_call **trace_events;
		unsigned int num_trace_events;
		struct trace_eval_map **trace_evals;
		unsigned int num_trace_evals;
	#endif
	#ifdef CONFIG_FTRACE_MCOUNT_RECORD
		unsigned int num_ftrace_callsites;
		unsigned long *ftrace_callsites;
	#endif
	#ifdef CONFIG_KPROBES
		void *kprobes_text_start;
		unsigned int kprobes_text_size;
		unsigned long *kprobe_blacklist;
		unsigned int num_kprobe_blacklist;
	#endif
	#ifdef CONFIG_HAVE_STATIC_CALL_INLINE
		int num_static_call_sites;
		struct static_call_site *static_call_sites;
	#endif

	#ifdef CONFIG_LIVEPATCH
		bool klp; /* Is this a livepatch module? */
		bool klp_alive;

		/* Elf information */
		struct klp_modinfo *klp_info;
	#endif

	#ifdef CONFIG_MODULE_UNLOAD
		/* What modules depend on me? */
		struct list_head source_list;
		/* What modules do I depend on? */
		struct list_head target_list;

		/* Destruction function. */
		void (*exit)(void);

		atomic_t refcnt;
	#endif

	#ifdef CONFIG_CONSTRUCTORS
		/* Constructor functions. */
		ctor_fn_t *ctors;
		unsigned int num_ctors;
	#endif

	#ifdef CONFIG_FUNCTION_ERROR_INJECTION
		struct error_injection_entry *ei_funcs;
		unsigned int num_ei_funcs;
	#endif
	} ____cacheline_aligned __randomize_layout;


- demo.c:

#include <linux/kernel.h>
#include <linux/module.h>
static int __init k_mod_sample_init(void)
{
	pr_info("%s:%d\n",__func__,__LINE__);
	return 0;
}

static void __exit k_mod_sample_exit(void)
{
	pr_info("%s:%d\n",__func__,__LINE__);
}

module_init(k_mod_sample_init)
module_exit(k_mod_sample_exit)
MODULE_LICENSE("GPL");


 demo.mod.c
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
	{ 0xc5850110, "printk" },
	{ 0xbdfb6dbb, "__fentry__" },
};

MODULE_INFO(depends, "");

- 模块组织：
/*
 * Mutex protects:
 * 1) List of modules (also safely readable with preempt_disable),
 * 2) module_use links,
 * 3) module_addr_min/module_addr_max.
 * (delete and add uses RCU list operations).
 */
static DEFINE_MUTEX(module_mutex);
static LIST_HEAD(modules);

- 加载过程分析图：（finit)
.. image:: ../../img/module_finit.svg
   :align: center


   
内核模块编译：kbuild
""""""""""""""""""
内核编译参考：

.. raw:: html
	
	<iframe src="../../lk_devel/lk_build/m_build.html" height="345px" width="100%"></iframe>
	
内核调试参考：

.. raw:: html
	
	<iframe src="../../lk_devel/lk_build/m_build.html" height="345px" width="100%"></iframe>
	
内核模块编译流程原理分析
""""""""""""""""""""
构建过程分析
***********
- make流程分析


A.make:$(MAKE) -C $(KDIR) M=$(PWD) modules
  
   1. cd $(KDIR)
   2. $M = $(PWD)
   3. make modules
B. kernel/Makefile
  
   1. KBUILD_EXTMOD := $(M) : make M=dir (外部模块目录)
   2. KBUILD_MODULES := 1
   3. build-dirs := $(KBUILD_EXTMOD)
   4. Makefile: modules:$(MODORDER) 等价于: modules: /root/for_work/rootkit/demo1/k_mod
      
      .. code-block:: c
	:caption: 代码片段
	:emphasize-lines: 4,5
	:linenos:
      
      		# External module support.
      		# When building external modules the kernel used as basis is considered
      		# read-only, and no consistency checks are made and the make
      		# system is not used on the basis kernel. If updates are required
      		# in the basis kernel ordinary make commands (without M=...) must
      		# be used.
      		#
      		# The following are the only valid targets when building external
      		# modules.
      		# make M=dir clean     Delete all automatically generated files
      		# make M=dir modules   Make all modules in specified dir
      		# make M=dir	       Same as 'make M=dir modules'
      		# make M=dir modules_install
      		#                      Install the modules built in the module directory
      		#                      Assumes install directory is already created
      
      		# We are always building only modules.
      		KBUILD_BUILTIN :=
      		KBUILD_MODULES := 1
      
      		build-dirs := $(KBUILD_EXTMOD)
      		PHONY += modules
      		modules: $(MODORDER)
      			$(Q)$(MAKE) -f $(srctree)/scripts/Makefile.modpost
      
      		$(MODORDER): descend
      			@:
      	
      		descend: $(build-dirs)
      		$(build-dirs): prepare      #此处prepare为空
      			$(Q)$(MAKE) $(build)=$@ \
      			single-build=$(if $(filter-out $@/, $(filter $@/%, $(KBUILD_SINGLE_TARGETS))),1) \
      			need-builtin=1 need-modorder=1
    
      Makefile中modules的前提条件:$(MODORDER)
      以上指令等价于 /root/for_work/rootkit/demo1/k_mod: make -f ./scripts/Makefile.build obj=/root/for_work/rootkit/demo1/k_mod  single-build=  need-builtin=1 need-modorder=1
   5. Makefile.build相关片段:
      
      .. code-block:: c
	:caption: 代码片段
	:emphasize-lines: 4,5
	:linenos:
	
      		__build: $(if $(KBUILD_BUILTIN), $(targets-for-builtin)) \
      			 $(if $(KBUILD_MODULES), $(targets-for-modules)) \
      			 $(subdir-ym) $(always-y)
      			@echo "target:" $(if $(KBUILD_BUILTIN), $(targets-for-builtin)) \  #(额外添加)
      	 		$(if $(KBUILD_MODULES), $(targets-for-modules)) \
      			 $(subdir-ym) $(always-y)
      			@:
   
      结果:#make 
      
      .. code-block:: c
	:caption: 代码片段
	:emphasize-lines: 4,5
	:linenos:
      
      		step "target:" /path/k_mod/k_m.mod /root/for_work/rootkit/demo1/k_mod/modules.order
  
  
       也就是说可以将目标理解为:
      
      .. code-block:: c
	:caption: 代码片段
	:emphasize-lines: 4,5
	:linenos:
		__build:/path/k_mod/k_m.mod /root/for_work/rootkit/demo1/k_mod/modules.order
               
               	k_m.mod:
               
                   	cmd_mod = { \
                    		echo $(if $($*-objs)$($*-y)$($*-m), $(addprefix $(obj)/, $($*-objs) $($*-y) $($*-m)), $(@:.mod=.o)); \
                    		$(undefined_syms) echo; \
                    		} > $@
                    
                   	 $(obj)/%.mod: $(obj)/%.o FORCE 
                    		$(call if_changed,mod)

      等价于:
      
      
      .. code-block:: c
	:caption: 代码片段
	:emphasize-lines: 4,5
	:linenos:
	
		/path/k_mod/k_m.mod: /path/k_mod/k_m.o FORCE


      通过/path/k_mod/Makefile我们知道k_m.o依赖:
      
      .. code-block:: c
	:caption: 代码片段
	:emphasize-lines: 4,5
	:linenos:
	
	obj-m := k_m.o
      	k_m-m := k_m1.o k_m2.o


      而Makefile.build中对*.o定义如下:编译目标文件:*.c --> *.o
      
      .. code-block:: c
	:caption: 代码片段
	:emphasize-lines: 4,5
	:linenos:
	
      	# Built-in and composite module parts
      	$(obj)/%.o: $(src)/%.c $(recordmcount_source) $(objtool_dep) FORCE
      		$(call if_changed_rule,cc_o_c)
      		@echo $(rule_cc_o_c)	#额外添加,具体信息在下一条说明
      		$(call cmd,force_checksrc)

      其规则展开为:
      
      .. code-block:: c
	:caption: 代码片段
	:emphasize-lines: 4,5
	:linenos:
	
		echo    @set -e;  echo '  CC [M]  /root/for_work/rootkit/demo1/k_mod/k_m1.o'; gcc -Wp,-MMD,/root/for_work/rootkit/demo1/k_mod/.k_m1.o.d -nostdinc -isystem /usr/lib/gcc/x86_64-linux-gnu/10/include -I./arch/x86/include -I./arch/x86/include/generated  -I./include -I./arch/x86/include/uapi -I./arch/x86/include/generated/uapi -I./include/uapi -I./include/generated/uapi -include ./include/linux/kconfig.h -include ./include/linux/compiler_types.h -D__KERNEL__ -fmacro-prefix-map=./= -Wall -Wundef -Werror=strict-prototypes -Wno-trigraphs -fno-strict-aliasing -fno-common -fshort-wchar -fno-PIE -Werror=implicit-function-declaration -Werror=implicit-int -Werror=return-type -Wno-format-security -std=gnu89 -mno-sse -mno-mmx -mno-sse2 -mno-3dnow -mno-avx -m64 -falign-jumps=1 -falign-loops=1 -mno-80387 -mno-fp-ret-in-387 -mpreferred-stack-boundary=3 -mskip-rax-setup -mtune=generic -mno-red-zone -mcmodel=kernel -DCONFIG_X86_X32_ABI -Wno-sign-compare -fno-asynchronous-unwind-tables -mindirect-branch=thunk-extern -mindirect-branch-register -fno-jump-tables -fno-delete-null-pointer-checks -Wno-frame-address -Wno-format-truncation -Wno-format-overflow -Wno-address-of-packed-member -O2 -fno-allow-store-data-races -Wframe-larger-than=2048 -fstack-protector-strong -Wno-unused-but-set-variable -Wimplicit-fallthrough -Wno-unused-const-variable -g -pg -mrecord-mcount -mfentry -DCC_USING_FENTRY -Wdeclaration-after-statement -Wvla -Wno-pointer-sign -Wno-stringop-truncation -Wno-zero-length-bounds -Wno-array-bounds -Wno-stringop-overflow -Wno-restrict -Wno-maybe-uninitialized -fno-strict-overflow -fno-stack-check -fconserve-stack -Werror=date-time -Werror=incompatible-pointer-types -Werror=designated-init -fcf-protection=none -Wno-packed-not-aligned  -DMODULE  -DKBUILD_BASENAME='"k_m1"' -DKBUILD_MODNAME='"k_m"' -c -o /root/for_work/rootkit/demo1/k_mod/k_m1.o /root/for_work/rootkit/demo1/k_mod/k_m1.c; scripts/basic/fixdep /root/for_work/rootkit/demo1/k_mod/.k_m1.o.d /root/for_work/rootkit/demo1/k_mod/k_m1.o 'gcc -Wp,-MMD,/root/for_work/rootkit/demo1/k_mod/.k_m1.o.d -nostdinc -isystem /usr/lib/gcc/x86_64-linux-gnu/10/include -I./arch/x86/include -I./arch/x86/include/generated  -I./include -I./arch/x86/include/uapi -I./arch/x86/include/generated/uapi -I./include/uapi -I./include/generated/uapi -include ./include/linux/kconfig.h -include ./include/linux/compiler_types.h -D__KERNEL__ -fmacro-prefix-map=./= -Wall -Wundef -Werror=strict-prototypes -Wno-trigraphs -fno-strict-aliasing -fno-common -fshort-wchar -fno-PIE -Werror=implicit-function-declaration -Werror=implicit-int -Werror=return-type -Wno-format-security -std=gnu89 -mno-sse -mno-mmx -mno-sse2 -mno-3dnow -mno-avx -m64 -falign-jumps=1 -falign-loops=1 -mno-80387 -mno-fp-ret-in-387 -mpreferred-stack-boundary=3 -mskip-rax-setup -mtune=generic -mno-red-zone -mcmodel=kernel -DCONFIG_X86_X32_ABI -Wno-sign-compare -fno-asynchronous-unwind-tables -mindirect-branch=thunk-extern -mindirect-branch-register -fno-jump-tables -fno-delete-null-pointer-checks -Wno-frame-address -Wno-format-truncation -Wno-format-overflow -Wno-address-of-packed-member -O2 -fno-allow-store-data-races -Wframe-larger-than=2048 -fstack-protector-strong -Wno-unused-but-set-variable -Wimplicit-fallthrough -Wno-unused-const-variable -g -pg -mrecord-mcount -mfentry -DCC_USING_FENTRY -Wdeclaration-after-statement -Wvla -Wno-pointer-sign -Wno-stringop-truncation -Wno-zero-length-bounds -Wno-array-bounds -Wno-stringop-overflow -Wno-restrict -Wno-maybe-uninitialized -fno-strict-overflow -fno-stack-check -fconserve-stack -Werror=date-time -Werror=incompatible-pointer-types -Werror=designated-init -fcf-protection=none -Wno-packed-not-aligned  -DMODULE  -DKBUILD_BASENAME='\''"k_m1"'\'' -DKBUILD_MODNAME='\''"k_m"'\'' -c -o /root/for_work/rootkit/demo1/k_mod/k_m1.o /root/for_work/rootkit/demo1/k_mod/k_m1.c' > /root/for_work/rootkit/demo1/k_mod/.k_m1.o.cmd; rm -f /root/for_work/rootkit/demo1/k_mod/.k_m1.o.d
                    @set -e
                      CC [M]  /root/for_work/rootkit/demo1/k_mod/k_m1.o


      注意其中的中间文件:.k_m1.o.d  .k_m1.o.cmd k_m1.o,其force_checksrc展开为(此处没有使能):
      
      .. code-block:: c
	:caption: 代码片段
	:emphasize-lines: 4,5
	:linenos:
	
		sparse -D__linux__ -Dlinux -D__STDC__ -Dunix -D__unix__ -Wbitwise -Wno-return-void -Wno-unknown-attribute -D__x86_64__ --arch=x86 -mlittle-endian -m64 -Wp,-MMD,/root/for_work/rootkit/demo1/k_mod/.k_m2.o.d -nostdinc -isystem /usr/lib/gcc/x86_64-linux-gnu/10/include -I./arch/x86/include -I./arch/x86/include/generated -I./include -I./arch/x86/include/uapi -I./arch/x86/include/generated/uapi -I./include/uapi -I./include/generated/uapi -include ./include/linux/kconfig.h -include ./include/linux/compiler_types.h -D__KERNEL__ -fmacro-prefix-map=./= -Wall -Wundef -Werror=strict-prototypes -Wno-trigraphs -fno-strict-aliasing -fno-common -fshort-wchar -fno-PIE -Werror=implicit-function-declaration -Werror=implicit-int -Werror=return-type -Wno-format-security -std=gnu89 -mno-sse -mno-mmx -mno-sse2 -mno-3dnow -mno-avx -m64 -falign-jumps=1 -falign-loops=1 -mno-80387 -mno-fp-ret-in-387 -mpreferred-stack-boundary=3 -mskip-rax-setup -mtune=generic -mno-red-zone -mcmodel=kernel -DCONFIG_X86_X32_ABI -Wno-sign-compare -fno-asynchronous-unwind-tables -mindirect-branch=thunk-extern -mindirect-branch-register -fno-jump-tables -fno-delete-null-pointer-checks -Wno-frame-address -Wno-format-truncation -Wno-format-overflow -Wno-address-of-packed-member -O2 -fno-allow-store-data-races -Wframe-larger-than=2048 -fstack-protector-strong -Wno-unused-but-set-variable -Wimplicit-fallthrough -Wno-unused-const-variable -g -pg -mrecord-mcount -mfentry -DCC_USING_FENTRY -Wdeclaration-after-statement -Wvla -Wno-pointer-sign -Wno-stringop-truncation -Wno-zero-length-bounds -Wno-array-bounds -Wno-stringop-overflow -Wno-restrict -Wno-maybe-uninitialized -fno-strict-overflow -fno-stack-check -fconserve-stack -Werror=date-time -Werror=incompatible-pointer-types -Werror=designated-init -fcf-protection=none -Wno-packed-not-aligned -DMODULE -DKBUILD_BASENAME="k_m2" -DKBUILD_MODNAME="k_m" /root/for_work/rootkit/demo1/k_mod/k_m2.c force



      k_m1.o k_m2.o链接为k_m.o,根据k_m.o 产生k_m.mod  modules.order:依赖 k_m.o
      
      .. code-block:: c
	:caption: 代码片段
	:emphasize-lines: 4,5
	:linenos:
	
      		# Rule to create modules.order file
      		#
      		# Create commands to either record .ko file or cat modules.order from
      		# a subdirectory
      		# Add $(obj-m) as the prerequisite to avoid updating the timestamp of
      		# modules.order unless contained modules are updated.
      		cmd_modules_order = { $(foreach m, $(real-prereqs), \
           		$(if $(filter %/modules.order, $m), cat $m, echo $(patsubst %.o,%.ko,$m));) :; } \
               	     	| $(AWK) '!x[$$0]++' - > $@
      		$(obj)/modules.order: $(obj-m) FORCE
            		$(call if_changed,modules_order)
 
 
      产生modules.order,至此完成Makefile中modules前提条件$(MODORDER) 的处理.
      Makefile modules 菜单指令:
      
      .. code-block:: c
	:caption: 代码片段
	:emphasize-lines: 4,5
	:linenos:
	
	modules: $(MODORDER)
      		$(Q)$(MAKE) -f $(srctree)/scripts/Makefile.modpost
      
   6. 即接下来看指令 make -f Makefile.modpost
      Makefile.modpost编译外部模块内容
      
      .. code-block:: c
	:caption: 代码片段
	:emphasize-lines: 4,5
	:linenos:
	
	__modpost:
	include include/config/auto.conf
      	include scripts/Kbuild.include
      
      	MODPOST = scripts/mod/modpost								\
      		$(if $(CONFIG_MODVERSIONS),-m)							\
      		$(if $(CONFIG_MODULE_SRCVERSION_ALL),-a)					\
      		$(if $(CONFIG_SECTION_MISMATCH_WARN_ONLY),,-E)					\
      		$(if $(KBUILD_MODPOST_WARN),-w) \
      		-o $@
      	# set src + obj - they may be used in the modules's Makefile
      	obj := $(KBUILD_EXTMOD)
      	src := $(obj)
      
      	# Include the module's Makefile to find KBUILD_EXTRA_SYMBOLS
      	include $(if $(wildcard $(KBUILD_EXTMOD)/Kbuild), \
      	             $(KBUILD_EXTMOD)/Kbuild, $(KBUILD_EXTMOD)/Makefile)
      
      	# modpost option for external modules
      	MODPOST += -e
      
      	input-symdump := Module.symvers $(KBUILD_EXTRA_SYMBOLS)
      	output-symdump := $(KBUILD_EXTMOD)/Module.symvers
      	# modpost options for modules (both in-kernel and external)
      	MODPOST += \
      		$(addprefix -i ,$(wildcard $(input-symdump))) \
      		$(if $(KBUILD_NSDEPS),-d $(MODULES_NSDEPS)) \
      		$(if $(CONFIG_MODULE_ALLOW_MISSING_NAMESPACE_IMPORTS)$(KBUILD_NSDEPS),-N)
      	# 'make -i -k' ignores compile errors, and builds as many modules as possible.
      	ifneq ($(findstring i,$(filter-out --%,$(MAKEFLAGS))),)
      	MODPOST += -n
      	endif
      
      	# Clear VPATH to not search for *.symvers in $(srctree). Check only $(objtree).
      	VPATH :=
      	$(input-symdump):
      		@echo >&2 'WARNING: Symbol version dump "$@" is missing.'
      		@echo >&2 '         Modules may not have dependencies or modversions.'
      
      	# Read out modules.order to pass in modpost.
      	# Otherwise, allmodconfig would fail with "Argument list too long".
      	quiet_cmd_modpost = MODPOST $@
           	 cmd_modpost = sed 's/ko$$/o/' $< | $(MODPOST) -T -
      
     	 $(output-symdump): $(MODORDER) $(input-symdump) FORCE
      		$(call if_changed,modpost)
      
      	targets += $(output-symdump)
      
      	__modpost: $(output-symdump)
      	ifneq ($(KBUILD_MODPOST_NOFINAL),1) 
      		$(Q)$(MAKE) -f $(srctree)/scripts/Makefile.modfinal :这一步通常情况下是要执行的
      	endif 	


      1. 第一步
         1. 创建单独的.o文件(如生成k_m1.o k_m2.o)
         2. 链接需要的.o 文件为\<module\>.o 文件(k_m1.o k_m2.o 链接为k_m.o)
         3. 生成\<module\>.mod文件,列出了初步的.o文件,(如k_m1.o k_m2.o)
         4. modules.order:列出所有的模块(k_m.ko)
      2. 第二步
         1. 查找所有在modules.order中列出的模块
      3. 第三步:修改模块ELF节中的信息,包含以下几个方面:与.mod.c相关
         1. Version magic(include/linux/vermagic.h来获取细节)
            1. Kernel release
            2. SMP is CONFIG_SMP
            3. PREEMPT is CONFIG_PREEMPT[_RT]
            4. GCC Version
         2. Module info
            1. Module version(MODULE_VERSION)
            2. Module alias'es(MODULE_ALIAS)
            3. Module license(MODULE_LICENSE)
            4. 参考include/linux/module.h来获取更多哦细节
      4. 第四步:仅用于允许外部模块中的模块版本控制，其中每个模块的 CRC 从 Module.symvers 文件中检索
      5.  Makefile.modpost参数设置:
         KBUILD_MODPOST_WARN:可以设置以避免在最终模块链接阶段出现未定义符号时出错.
         KBUILD_MODPOST_NOFINAL:可以设置用于忽略最后的模块链接.
   
      目前为止,我们完成了第一步,已经产生文件:k_m1.o k_m2.o k_m.o k_m.mod modules.order 现在进入第二步.
      Makefile.modpost关键信息:
   
      
      .. code-block:: c
	:caption: 代码片段
	:emphasize-lines: 4,5
	:linenos:
      
      	$(output-symdump): $(MODORDER) $(input-symdump) FORCE
      		$(call if_changed,modpost)
      
      	targets += $(output-symdump)
      
      	__modpost: $(output-symdump)
      	ifneq ($(KBUILD_MODPOST_NOFINAL),1) 
      		$(Q)$(MAKE) -f $(srctree)/scripts/Makefile.modfinal :这一步通常情况下是要执行的
      	endif
 
 
      $(output-symdump)定义:
      
      .. code-block:: c
	:caption: 代码片段
	:emphasize-lines: 4,5
	:linenos:
	
      	$(output-symdump): $(MODORDER) $(input-symdump) FORCE
      		$(call if_changed,modpost)


      等价于:
      
      .. code-block:: c
	:caption: 代码片段
	:emphasize-lines: 4,5
	:linenos:
	
      	/path/k_mod/Module.symvers: /path/kmod/modules.order Module.symvers
      		scripts/mod/modpost -m -o /path/k_mod/Module.symvers -e -i Module.symvers -T /path/k_mod/k_m.o
 
 
      1. Module.symvers:内核代码树文件,已经有了
      2.  /path/kmod/modules.order:第一步中已经产生
      3.  /path/k_mod/k_m.o: 第一步已经产生
      4. Module.symvers由指令 scripts/mod/modpost -m -o /path/k_mod/Module.symvers -e -i Module.symvers -T /path/k_mod/k_m.o产生.执行过程中产生文件 k_m.mod.c (scripts/mod/modpost.c代码段如下:),并产生 /path/k_mod/Module.symvers文件.
      
      .. code-block:: c
	:caption: 代码片段
	:emphasize-lines: 4,5
	:linenos:
	
         		err |= check_modname_len(mod);
         		err |= check_exports(mod);
         
         		add_header(&buf, mod);
         		add_intree_flag(&buf, !external_module);
         		add_retpoline(&buf);
         		add_staging_flag(&buf, mod->name);
         		err |= add_versions(&buf, mod);
         		add_depends(&buf, mod);
         		add_moddevtable(&buf, mod);
         		add_srcversion(&buf, mod);
         
         		sprintf(fname, "%s.mod.c", mod->name);
         		write_if_changed(&buf, fname);
     
   7.  最后一步产生最终的内核模块文件:make -f Makefile.modfinal
      $(MAKE) -f $(srctree)/scripts/Makefile.modfinal,目前已经产生的文件有:k_m1.o k_m2.o k_m.o k_m.mod modules.order   Module.symvers  k_m.mod.c
      
      .. code-block:: c
	:caption: 代码片段
	:emphasize-lines: 4,5
	:linenos:
	
	__modfinal: $(modules)
      		@:

      等价于
      
            
      .. code-block:: c
	:caption: 代码片段
	:emphasize-lines: 4,5
	:linenos:
	
	__modfinal:/path/k_mod/k_m.ko
      		@:
      
      
      进一步依赖
      
      .. code-block:: c
	:caption: 代码片段
	:emphasize-lines: 4,5
	:linenos:
	
	$(modules): %.ko: %.o %.mod.o $(ARCH_MODULE_LDS) FORCE
      		+$(call if_changed,ld_ko_o)
      
      等价于
      
      .. code-block:: c
	:caption: 代码片段
	:emphasize-lines: 4,5
	:linenos:
	
	k_m.ko: k_m.ko: k_m.o k_m.mod.o scripts/module.lds FORCE
      		ld -r -m elf_x86_64 --build-id=sha1 -T scripts/module.lds -o /root/for_work/rootkit/demo1/k_mod/k_m.ko /root/for_work/rootkit/demo1/k_mod/k_m.o /root/for_work/rootkit/demo1/k_mod/k_m.mod.o
      
      其中k_m.o scripts/module.lds都已经存在,只需要关注k_m.mo.o
      
      
      .. code-block:: c
	:caption: 代码片段
	:emphasize-lines: 4,5
	:linenos:
	
      	quiet_cmd_cc_o_c = CC [M]  $@
            cmd_cc_o_c = $(CC) $(c_flags) -c -o $@ $<
      
      %.mod.o: %.mod.c FORCE
      	$(call if_changed_dep,cc_o_c)

      等价于
      
      
      .. code-block:: c
	:caption: 代码片段
	:emphasize-lines: 4,5
	:linenos:
	
      	k_m.mod.o: k_m.mod.c FORCE
      		gcc -Wp,-MMD,/root/for_work/rootkit/demo1/k_mod/.k_m.mod.o.d -nostdinc -isystem /usr/lib/gcc/x86_64-linux-gnu/10/include -I./arch/x86/include -I./arch/x86/include/generated -I./include -I./arch/x86/include/uapi -I./arch/x86/include/generated/uapi -I./include/uapi -I./include/generated/uapi -include ./include/linux/kconfig.h -include ./include/linux/compiler_types.h -D__KERNEL__ -fmacro-prefix-map=./= -Wall -Wundef -Werror=strict-prototypes -Wno-trigraphs -fno-strict-aliasing -fno-common -fshort-wchar -fno-PIE -Werror=implicit-function-declaration -Werror=implicit-int -Werror=return-type -Wno-format-security -std=gnu89 -mno-sse -mno-mmx -mno-sse2 -mno-3dnow -mno-avx -m64 -falign-jumps=1 -falign-loops=1 -mno-80387 -mno-fp-ret-in-387 -mpreferred-stack-boundary=3 -mskip-rax-setup -mtune=generic -mno-red-zone -mcmodel=kernel -DCONFIG_X86_X32_ABI -Wno-sign-compare -fno-asynchronous-unwind-tables -mindirect-branch=thunk-extern -mindirect-branch-register -fno-jump-tables -fno-delete-null-pointer-checks -Wno-frame-address -Wno-format-truncation -Wno-format-overflow -Wno-address-of-packed-member -O2 -fno-allow-store-data-races -Wframe-larger-than=2048 -fstack-protector-strong -Wno-unused-but-set-variable -Wimplicit-fallthrough -Wno-unused-const-variable -g -pg -mrecord-mcount -mfentry -DCC_USING_FENTRY -Wdeclaration-after-statement -Wvla -Wno-pointer-sign -Wno-stringop-truncation -Wno-zero-length-bounds -Wno-array-bounds -Wno-stringop-overflow -Wno-restrict -Wno-maybe-uninitialized -fno-strict-overflow -fno-stack-check -fconserve-stack -Werror=date-time -Werror=incompatible-pointer-types -Werror=designated-init -fcf-protection=none -Wno-packed-not-aligned -DMODULE -DKBUILD_BASENAME="k_m.mod" -DKBUILD_MODNAME="k_m" -c -o /root/for_work/rootkit/demo1/k_mod/k_m.mod.o /root/for_work/rootkit/demo1/k_mod/k_m.mod.c


      至此,内核模块编译完成,有效文件包括:k_m1.o k_m2.o k_m.o k_m.mod modules.order   Module.symvers  k_m.mod.c k_m.mod.o k_m.ko
C. 编译流程图

.. image:: ../../img/module_make.svg
   :align: center

总结

  本文对内核模块的编译过程进行流程性说明，下一章针对符号、认证、调试等信息的处理细节进行描述，会对内核模块的二进制格式进行更深入分析。
  
  
构建模块过程中的的符号处理
"""""""""""""""""""""""
 demo 源文件

 k_m1.c

.. code-block:: c
	:caption: k_m1.c
	:emphasize-lines: 4,5
	:linenos:
	
	#include <linux/kernel.h>
	#include <linux/module.h>
	#include <linux/kprobes.h>
	#include <linux/sched.h>
	#include "./k_m.h"

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


 k_m2.c

.. code-block:: c
	:caption: k_m2.c
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

	int handler_fault(struct kprobe *p, struct pt_regs *regs, int trapnr)
	{
		pr_info("fault_handler: p->addr = 0x%p, trap #%dn", p->addr, trapnr);
		/* Return 0 because we don't handle the fault. */
		return 0;
	}
	/* NOKPROBE_SYMBOL() is also available */
	NOKPROBE_SYMBOL(handler_fault);
	int k_m1_symbol(void)
	{
		printk("for test:%s:%d\n",__func__,__LINE__);	
		return 0;
	}
	EXPORT_SYMBOL(k_m1_symbol);


 Makefile文件

.. code-block:: c
	:caption: k_m2.c
	:emphasize-lines: 4,5
	:linenos:

	obj-m := k_m.o
	k_m-m := k_m1.o k_m2.o
		KDIR:=/lib/modules/$(shell uname -r)/build
		PWD:=$(shell pwd)
	default:
		$(MAKE) -C $(KDIR) M=$(PWD) modules
	clean:
		$(MAKE) -C $(KDIR) M=$(PWD) clean

重点步骤

- k_m1.c/k_m2.c --> k_m1.o/k_m2.o:


.. code-block:: c
	:caption: k_m2.c
	:emphasize-lines: 4,5
	:linenos:
	
	gcc -Wp,-MMD,/root/for_work/rootkit/demo1/k_mod/.k_m1.o.d -nostdinc -isystem /usr/lib/gcc/x86_64-linux-gnu/10/include -I./arch/x86/include -I./arch/x86/include/generated  -I./include -I./arch/x86/include/uapi -I./arch/x86/include/generated/uapi -I./include/uapi -I./include/generated/uapi -include ./include/linux/kconfig.h -include ./include/linux/compiler_types.h -D__KERNEL__ -fmacro-prefix-map=./= -Wall -Wundef -Werror=strict-prototypes -Wno-trigraphs -fno-strict-aliasing -fno-common -fshort-wchar -fno-PIE -Werror=implicit-function-declaration -Werror=implicit-int -Werror=return-type -Wno-format-security -std=gnu89 -mno-sse -mno-mmx -mno-sse2 -mno-3dnow -mno-avx -m64 -falign-jumps=1 -falign-loops=1 -mno-80387 -mno-fp-ret-in-387 -mpreferred-stack-boundary=3 -mskip-rax-setup -mtune=generic -mno-red-zone -mcmodel=kernel -DCONFIG_X86_X32_ABI -Wno-sign-compare -fno-asynchronous-unwind-tables -mindirect-branch=thunk-extern -mindirect-branch-register -fno-jump-tables -fno-delete-null-pointer-checks -Wno-frame-address -Wno-format-truncation -Wno-format-overflow -Wno-address-of-packed-member -O2 -fno-allow-store-data-races -Wframe-larger-than=2048 -fstack-protector-strong -Wno-unused-but-set-variable -Wimplicit-fallthrough -Wno-unused-const-variable -g -pg -mrecord-mcount -mfentry -DCC_USING_FENTRY -Wdeclaration-after-statement -Wvla -Wno-pointer-sign -Wno-stringop-truncation -Wno-zero-length-bounds -Wno-array-bounds -Wno-stringop-overflow -Wno-restrict -Wno-maybe-uninitialized -fno-strict-overflow -fno-stack-check -fconserve-stack -Werror=date-time -Werror=incompatible-pointer-types -Werror=designated-init -fcf-protection=none -Wno-packed-not-aligned  -DMODULE  -DKBUILD_BASENAME='"k_m1"' -DKBUILD_MODNAME='"k_m"' -c -o /root/for_work/rootkit/demo1/k_mod/k_m1.o /root/for_work/rootkit/demo1/k_mod/k_m1.c 

  
  参数: 参考gcc


 - k_m1.o + k_m2.o --> k_m.o: ld -m elf_x86_64   -r -o k_m.o k_m1.o k_m2.o

  其默认链接脚本(与应用软件一致,可通过ld -ve)

  参数:

  - -m elf_x86_64：
  - -r：Generate relocatable output
  - -o：

- k_m.o --> Module.symvers + k_m.mod.c:输入文件:k_m.o,Module.symvers;输出文件:/path/k_mod/Module.symvers, k_m.mod.c

  modpost.c 代码分析(根据指令:scripts/mod/modpost -m -o /path/k_mod/Module.symvers -e -i Module.symvers -T  -)进行

  Module.symvers:


.. code-block:: c
	:caption: k_m2.c
	:emphasize-lines: 4,5
	:linenos:
	
  	/k_mod/Module.symvers := sed 's/ko$$/o/' /root/for_work/rootkit/demo1/k_mod/modules.order | scripts/mod/modpost -m    -o /root/for_work/rootkit/demo1/k_mod/Module.symvers -e -i Module.symvers   -T -

  
  等价于:


.. code-block:: c
	:caption: k_m2.c
	:emphasize-lines: 4,5
	:linenos:
	
  	#sed 's/ko$/o/' modules.order | scripts/mod/modpost -m    -o /root/for_work/rootkit/demo1/k_mod/Module.symvers -e -i Module.symvers   -T -

  
  看modules.order内容:

.. code-block:: c
	:caption: k_m2.c
	:emphasize-lines: 4,5
	:linenos:
	
  	#cat modules.order
  		/root/for_work/rootkit/demo1/k_mod/k_m.ko
  	#sed 's/ko$/o/' modules.order
  		/root/for_work/rootkit/demo1/k_mod/k_m.o

  
  指令最终为:

.. code-block:: c
	:caption: k_m2.c
	:emphasize-lines: 4,5
	:linenos:
	
	/root/for_work/rootkit/demo1/k_mod/k_m.o | scripts/mod/modpost -m    -o /root/for_work/rootkit/demo1/k_mod/Module.symvers -e -i Module.symvers   -T -

  
  参数描述:
  
  1. -m: modversions = 1;
  2. -o: dump_write = /path/k_mod/Module.symvers;(写入)
  3. -e: external_module = 1;
  4. -i: (*dump_read_iter)->file = srctree/Module.symvers;(读取)
  5. -T: files_source = -;(读取)
  6. 另外"/root/for_work/rootkit/demo1/k_mod/k_m.o"从stdin传递到应用程序modpost.
  
  代码分析(modpost.c):
  
  重要数据结构分析:
  
.. code-block:: c
	:caption: 符号结构
	:emphasize-lines: 4,5
	:linenos:
	
  	struct buffer {
  		char *p;
  		int pos;
  		int size;
  	};
  	struct symbol {
  		struct symbol *next;
  		struct module *module;
  		unsigned int crc;
  		int crc_valid;
  		char *namespace;
  		unsigned int weak:1;
  		unsigned int is_static:1;  /* 1 if symbol is not global */
  		enum export  export;       /* Type of export */
  		char name[];
  	};
  	enum export {
  		export_plain,      export_unused,     export_gpl,
  		export_unused_gpl, export_gpl_future, export_unknown
  	};
  	struct module {
  		struct module *next;
  		int gpl_compatible;
  		struct symbol *unres;
  		int from_dump;  /* 1 if module was loaded from *.symvers */
  		int is_vmlinux;
  		int seen;
  		int has_init;
  		int has_cleanup;
  		struct buffer dev_table_buf;
  		char	     srcversion[25];
  		// Missing namespace dependencies
  		struct namespace_list *missing_namespaces;
  		// Actual imported namespaces
  		struct namespace_list *imported_namespaces;
  		char name[];
  	};
      
  	struct elf_info {
  		size_t size;
  		Elf_Ehdr     *hdr;
  		Elf_Shdr     *sechdrs;
  		Elf_Sym      *symtab_start;
  		Elf_Sym      *symtab_stop;
  		Elf_Section  export_sec;
  		Elf_Section  export_unused_sec;
  		Elf_Section  export_gpl_sec;
  		Elf_Section  export_unused_gpl_sec;
  		Elf_Section  export_gpl_future_sec;
  		char         *strtab;
  		char	     *modinfo;
  		unsigned int modinfo_len;
  
  		/* support for 32bit section numbers */
  	
  		unsigned int num_sections; /* max_secindex + 1 */
  		unsigned int secindex_strings;
  		/* if Nth symbol table entry has .st_shndx = SHN_XINDEX,
  	 	* take shndx from symtab_shndx_start[N] instead */
  		Elf32_Word   *symtab_shndx_start;
  		Elf32_Word   *symtab_shndx_stop;
  	};

  
流程图:

.. image::../../img/modpost.svg
   :align: center

重要过程描述:
  
  - 过程分析：
  
    1. 读取kernelsrc/Module.symvers;
    2. 读取并path/k_mod/k_m.o，并对其文件中的节进行解析存储;
    3. 根据第2步对符号进行校验（调用符号是否可用，借助第1步读取到的信息）；
    4. 根据1,2两步生成的关于外部符号信息生成/path/k_mod/Module.symvers文件：
    5. 如果有其他内核模块编译需要用到当前模块导出的符号，则需要第4步生成的文件。
  
  - k_m.mod.c：
  
  
.. code-block:: c
	:caption: k_m.mod.c
	:emphasize-lines: 4,5
	:linenos:
	
    	#include <linux/module.h>
    	#define INCLUDE_VERMAGIC
    	#include <linux/build-salt.h>
    	#include <linux/vermagic.h>
    	#include <linux/compiler.h>
    
    	BUILD_SALT;
    
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
    	__used __section("__versions") = {//解析调用的内核函数
    		{ 0x9463ffe0, "module_layout" },
    		{ 0x7f7b1cfd, "unregister_kprobe" },
    		{ 0x8c7cb666, "commit_creds" },
    		{ 0x9d447e54, "register_kprobe" },
    		{ 0xb2e20e99, "param_ops_string" },
    		{ 0x2b9c46f8, "__get_task_comm" },
    		{ 0xb44b338d, "current_task" },
    		{ 0xc5850110, "printk" },
    		{ 0x2b372d93, "prepare_creds" },
    		{ 0xbdfb6dbb, "__fentry__" },
    	};
    
    	MODULE_INFO(depends, "");
    
  
    描述:
  
k_m.mod.c --> k_m.mod.o:
*************************
.. code-block:: c
	:caption: k_m.mod.c
	:emphasize-lines: 4,5
	:linenos:
	
  	gcc -Wp,-MMD,/root/for_work/rootkit/demo1/k_mod/.k_m.mod.o.d -nostdinc -isystem /usr/lib/gcc/x86_64-linux-gnu/10/include -I./arch/x86/include -I./arch/x86/include/generated  -I./include -I./arch/x86/include/uapi -I./arch/x86/include/generated/uapi -I./include/uapi -I./include/generated/uapi -include ./include/linux/kconfig.h -include ./include/linux/compiler_types.h -D__KERNEL__ -fmacro-prefix-map=./= -Wall -Wundef -Werror=strict-prototypes -Wno-trigraphs -fno-strict-aliasing -fno-common -fshort-wchar -fno-PIE -Werror=implicit-function-declaration -Werror=implicit-int -Werror=return-type -Wno-format-security -std=gnu89 -mno-sse -mno-mmx -mno-sse2 -mno-3dnow -mno-avx -m64 -falign-jumps=1 -falign-loops=1 -mno-80387 -mno-fp-ret-in-387 -mpreferred-stack-boundary=3 -mskip-rax-setup -mtune=generic -mno-red-zone -mcmodel=kernel -DCONFIG_X86_X32_ABI -Wno-sign-compare -fno-asynchronous-unwind-tables -mindirect-branch=thunk-extern -mindirect-branch-register -fno-jump-tables -fno-delete-null-pointer-checks -Wno-frame-address -Wno-format-truncation -Wno-format-overflow -Wno-address-of-packed-member -O2 -fno-allow-store-data-races -Wframe-larger-than=2048 -fstack-protector-strong -Wno-unused-but-set-variable -Wimplicit-fallthrough -Wno-unused-const-variable -g -pg -mrecord-mcount -mfentry -DCC_USING_FENTRY -Wdeclaration-after-statement -Wvla -Wno-pointer-sign -Wno-stringop-truncation -Wno-zero-length-bounds -Wno-array-bounds -Wno-stringop-overflow -Wno-restrict -Wno-maybe-uninitialized -fno-strict-overflow -fno-stack-check -fconserve-stack -Werror=date-time -Werror=incompatible-pointer-types -Werror=designated-init -fcf-protection=none -Wno-packed-not-aligned  -DMODULE  -DKBUILD_BASENAME='"k_m.mod"' -DKBUILD_MODNAME='"k_m"' -c -o /root/for_work/rootkit/demo1/k_mod/k_m.mod.o /root/for_work/rootkit/demo1/k_mod/k_m.mod.c


  参数描述可参考"gcc 手册"中描述.

k_m.o + k_m.mod.o --> k_m.ko
******************************

.. code-block:: c
	:caption: k_m.mod.c
	:emphasize-lines: 4,5
	:linenos:
	
	ld -r -m elf_x86_64 --build-id=sha1  -T scripts/module.lds -o k_m.ko k_m.o k_m.mod.o
	
   参数描述:

   - -r
   - -m elf_x86_64
   - --build-id=sha1
   - -T

   scripts/module.lds 代码：这部分主要针对符号进行处理。


.. code-block:: c
	:caption: k_m.mod.c
	:emphasize-lines: 4,5
	:linenos:
	
   	SECTIONS {
    	/DISCARD/ : {
     	*(.discard)
     	*(.discard.*)
    	}
    	__ksymtab 0 : { *(SORT(___ksymtab+*)) }
    	__ksymtab_gpl 0 : { *(SORT(___ksymtab_gpl+*)) }
    	__ksymtab_unused 0 : { *(SORT(___ksymtab_unused+*)) }
    	__ksymtab_unused_gpl 0 : { *(SORT(___ksymtab_unused_gpl+*)) }
    	__ksymtab_gpl_future 0 : { *(SORT(___ksymtab_gpl_future+*)) }
    	__kcrctab 0 : { *(SORT(___kcrctab+*)) }
    	__kcrctab_gpl 0 : { *(SORT(___kcrctab_gpl+*)) }
    	__kcrctab_unused 0 : { *(SORT(___kcrctab_unused+*)) }
    	__kcrctab_unused_gpl 0 : { *(SORT(___kcrctab_unused_gpl+*)) }
    	__kcrctab_gpl_future 0 : { *(SORT(___kcrctab_gpl_future+*)) }
    	.init_array 0 : ALIGN(8) { *(SORT(.init_array.*)) *(.init_array) }
    	__jump_table 0 : ALIGN(8) { KEEP(*(__jump_table)) }
   	}	


   注意：这主要针对导出符号（eg:EXPORT_SYMBOL),init_array(),__jump_table_节

k_m.ko二进制格式分析
*******************

.. code-block:: c
	:caption: k_m.mod.c
	:emphasize-lines: 4,5
	:linenos:
	
   	ELF 头：
     		Magic：  7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00 
     		类别:                              ELF64
     		数据:                              2 补码，小端序 (little endian)
     		Version:                           1 (current)
     		OS/ABI:                            UNIX - System V
     		ABI 版本:                          0
     		类型:                              REL (可重定位文件)
     		系统架构:                          Advanced Micro Devices X86-64
     		版本:                              0x1
     		入口点地址：              0x0
     		程序头起点：              0 (bytes into file)
     		Start of section headers:          256352 (bytes into file)
     		标志：             0x0
     		Size of this header:               64 (bytes)
     		Size of program headers:           0 (bytes)
     		Number of program headers:         0
     		Size of section headers:           64 (bytes)
     		Number of section headers:         56
     		Section header string table index: 55
   


总结
*********

到这里生成了完整的ko文件，下一章我们总结模块加载过程。
 
  
内核模块加载过程分析
""""""""""""""""""


.. image::../../img/module_load.svg
   :align: center

  
  
内核模块卸载
"""""""""""
.. image::../../img/module_unload.svg
   :align: center
   

内核模块中可调用的函数
""""""""""""""""""

总结
""""""""
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  

