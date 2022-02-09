linux 加载ELF 内核文件：kexec
---------------------------------------
kexec应用场景
^^^^^^^^^^^^
demo分析：linux-source-5.14/tools/testing/selftests/kexec

.. code-block:: c
   :caption: test_kexec_load.sh
   :emphasize-lines: 4,5
   :linenos:

	#!/bin/sh
	# SPDX-License-Identifier: GPL-2.0
	#
	# Prevent loading a kernel image via the kexec_load syscall when
	# signatures are required.  (Dependent on CONFIG_IMA_ARCH_POLICY.)

	TEST="$0"
	. ./kexec_common_lib.sh

	# kexec requires root privileges
	require_root_privileges

	# get the kernel config
	get_kconfig

	kconfig_enabled "CONFIG_KEXEC=y" "kexec_load is enabled"
	if [ $? -eq 0 ]; then
		log_skip "kexec_load is not enabled"
	fi

	kconfig_enabled "CONFIG_IMA_APPRAISE=y" "IMA enabled"
	ima_appraise=$?

	kconfig_enabled "CONFIG_IMA_ARCH_POLICY=y" \
		"IMA architecture specific policy enabled"
	arch_policy=$?

	get_secureboot_mode
	secureboot=$?

	# kexec_load should fail in secure boot mode and CONFIG_IMA_ARCH_POLICY enabled
	kexec --load $KERNEL_IMAGE > /dev/null 2>&1
	if [ $? -eq 0 ]; then
		kexec --unload
		if [ $secureboot -eq 1 ] && [ $arch_policy -eq 1 ]; then
			log_fail "kexec_load succeeded"
		elif [ $ima_appraise -eq 0 -o $arch_policy -eq 0 ]; then
			log_info "Either IMA or the IMA arch policy is not enabled"
		fi
		log_pass "kexec_load succeeded"
	else
		if [ $secureboot -eq 1 ] && [ $arch_policy -eq 1 ] ; then
			log_pass "kexec_load failed"
		else
			log_fail "kexec_load failed"
		fi
	fi

这里kexec对应git://git.kernel.org/pub/scm/utils/kernel/kexec/kexec-tools.git

澄清自己的一个错误：加载的这个内核就是一个普通内核，不需要额外定制。直接分析原理。


kexec --load $KERNEL_IMAGE，就对这个程序进行分析
"""""""""""""""""""""""""""""""""""""""""""""


.. code-block:: c
   :caption: kexec代码分析
   :emphasize-lines: 4,5
   :linenos:

	kesec:main

	--load --> OPT_LOAD
			has_opt_load = 1;
			do_load = 1;
			do_exec = 0;
			do_shutdown = 0;

	if (do_load &&
	    ((kexec_flags & KEXEC_ON_CRASH) ||
	     (kexec_file_flags & KEXEC_FILE_ON_CRASH)) &&
	    !is_crashkernel_mem_reserved()) {
		die("Memory for crashkernel is not reserved\n"
		    "Please reserve memory by passing"
		    "\"crashkernel=Y@X\" parameter to kernel\n"
		    "Then try to loading kdump kernel\n");
	}

	if (do_load && (kexec_flags & KEXEC_PRESERVE_CONTEXT) &&
	    mem_max == ULONG_MAX) {
		die("Please specify memory range used by kexeced kernel\n"
		    "to preserve the context of original kernel with \n"
		    "\"--mem-max\" parameter\n");
	}

	if (do_load && (kexec_flags & KEXEC_LIVE_UPDATE) &&
	    !xen_present()) {
		die("--load-live-update can only be used with xen\n");
	}
	
	
	......
	if (do_load && (result == 0)) {
		if (do_kexec_file_syscall) {
			result = do_kexec_file_load(fileind, argc, argv,
						 kexec_file_flags);
			if (result == EFALLBACK && do_kexec_fallback) {
				/* Reset getopt for fallback */
				opterr = 1;
				optind = 1;
				do_kexec_file_syscall = 0;
			}
		}
		if (!do_kexec_file_syscall)
			result = my_load(type, fileind, argc, argv, //最终到这儿
						kexec_flags, skip_checks, entry);
	}

	..........


现在我们看my_load()函数：


.. code-block:: c
   :caption: do_exec_load
   :emphasize-lines: 4,5
   :linenos:
   
	/*
 	*	Load the new kernel
 	*/
	static int my_load(const char *type, int fileind, int argc, char **argv,
		   unsigned long kexec_flags, int skip_checks, void *entry)
      {
	char *kernel;
	char *kernel_buf;
	off_t kernel_size;
	int i = 0;
	int result;
	struct kexec_info info;
	long native_arch;
	int guess_only = 0;

	memset(&info, 0, sizeof(info));
	info.kexec_flags = kexec_flags;
	info.skip_checks = skip_checks;

	result = 0;
	if (argc - fileind <= 0) {
		fprintf(stderr, "No kernel specified\n");
		usage();
		return -1;
	}
	kernel = argv[fileind];
	/* slurp in the input kernel */
	kernel_buf = slurp_decompress_file(kernel, &kernel_size); //解压内核并读如内存

	dbgprintf("kernel: %p kernel_size: %#llx\n",
		  kernel_buf, (unsigned long long)kernel_size);

	if (get_memory_ranges(&info.memory_range, &info.memory_ranges,
		info.kexec_flags) < 0 || info.memory_ranges == 0) {
		fprintf(stderr, "Could not get memory layout\n");
		return -1;
	}
	/* if a kernel type was specified, try to honor it */
	if (type) {
		for (i = 0; i < file_types; i++) {
			if (strcmp(type, file_type[i].name) == 0)
				break;
		}
		if (i == file_types) {
			fprintf(stderr, "Unsupported kernel type %s\n", type);
			return -1;
		} else {
			/* make sure our file is really of that type */
			if (file_type[i].probe(kernel_buf, kernel_size) < 0)
				guess_only = 1;
		}
	}
	if (!type || guess_only) {
		for (i = 0; i < file_types; i++) {
			if (file_type[i].probe(kernel_buf, kernel_size) == 0)
				break;
		}
		if (i == file_types) {
			fprintf(stderr, "Cannot determine the file type "
					"of %s\n", kernel);
			return -1;
		} else {
			if (guess_only) {
				fprintf(stderr, "Wrong file type %s, "
					"file matches type %s\n",
					type, file_type[i].name);
				return -1;
			}
		}
	}
	/* Figure out our native architecture before load */
	native_arch = physical_arch(); //
	if (native_arch < 0) {
		return -1;
	}
	info.kexec_flags |= native_arch;

	result = file_type[i].load(argc, argv, kernel_buf, kernel_size, &info);// 
	if (result < 0) {
		switch (result) {
		case ENOCRASHKERNEL:
			fprintf(stderr,
				"No crash kernel segment found in /proc/iomem\n"
				"Please check the crashkernel= boot parameter.\n");
			break;
		case EFAILED:
		default:
			fprintf(stderr, "Cannot load %s\n", kernel);
			break;
		}
		return result;
	}
	/* If we are not in native mode setup an appropriate trampoline */
	if (arch_compat_trampoline(&info) < 0) {
		return -1;
	}
	if (info.kexec_flags & KEXEC_PRESERVE_CONTEXT) {
		add_backup_segments(&info, mem_min, mem_max - mem_min + 1);
	}
	/* Verify all of the segments load to a valid location in memory */
	for (i = 0; i < info.nr_segments; i++) {
		if (!valid_memory_segment(&info, info.segment +i)) {
			fprintf(stderr, "Invalid memory segment %p - %p\n",
				info.segment[i].mem,
				((char *)info.segment[i].mem) + 
				info.segment[i].memsz);
			return -1;
		}
	}
	/* Sort the segments and verify we don't have overlaps */
	if (sort_segments(&info) < 0) {
		return -1;
	}
	/* if purgatory is loaded update it */
	update_purgatory(&info);
	if (entry)
		info.entry = entry;

	dbgprintf("kexec_load: entry = %p flags = 0x%lx\n",
		  info.entry, info.kexec_flags);
	if (kexec_debug)
		print_segments(stderr, &info);

	if (xen_present())
		result = xen_kexec_load(&info);
	else
		result = kexec_load(info.entry,
				    info.nr_segments, info.segment,
				    info.kexec_flags);
	if (result != 0) {
		/* The load failed, print some debugging information */
		fprintf(stderr, "kexec_load failed: %s\n", 
			strerror(errno));
		fprintf(stderr, "entry       = %p flags = 0x%lx\n", 
			info.entry, info.kexec_flags);
		print_segments(stderr, &info);
	}
	return result;
	}


现在又进入函数kexec_load():

.. code-block:: c
   :caption: do_exec_load
   :emphasize-lines: 4,5
   :linenos:
   
   static inline long kexec_load(void *entry, unsigned long nr_segments,
			struct kexec_segment *segments, unsigned long flags)
   {
	return (long) syscall(__NR_kexec_load, entry, nr_segments, segments, flags);
   }

内核加载流程图
""""""""""""

.. image:: ../img/kexec_load_flow.svg
   :align: center



系统调用
^^^^^^^^^
kexec_load
""""""""""""
kexec.c: kexec_load系统调用：只有root可以调用，分为三部分：
- 从当前地址空间加载新内核的通用部分，并非常小心地将数据放置在分配的页面中。
- 与内核交互并告诉所有设备关闭的通用部分。 阻止正在进行的 dmas，并将设备置于一致状态，以便以后的内核可以重新初始化它们。
- 包含系统调用号的机器特定部分，然后将映像复制到其最终目的地。 并在入口处跳转到内核镜像。
- kexec 不同步或卸载文件系统，所以如果你需要这样做，你需要自己做。


sys_kexec_load --> do_exec_load(entry,nr_segments,segments,flags)

.. code-block:: c
   :caption: do_exec_load
   :emphasize-lines: 4,5
   :linenos:
   
   static int do_kexec_load(unsigned long entry, unsigned long nr_segments,
		struct kexec_segment __user *segments, unsigned long flags)
	{	
	struct kimage **dest_image, *image;
	unsigned long i;
	int ret;

	if (flags & KEXEC_ON_CRASH) {
		dest_image = &kexec_crash_image;
		if (kexec_crash_image)
			arch_kexec_unprotect_crashkres();
	} else {
		dest_image = &kexec_image;
	}

	if (nr_segments == 0) {
		/* Uninstall image */
		kimage_free(xchg(dest_image, NULL));
		return 0;
	}
	if (flags & KEXEC_ON_CRASH) {
		/*
		 * Loading another kernel to switch to if this one
		 * crashes.  Free any current crash dump kernel before
		 * we corrupt it.
		 */
		kimage_free(xchg(&kexec_crash_image, NULL));
	}

	ret = kimage_alloc_init(&image, entry, nr_segments, segments, flags);
	if (ret)
		return ret;

	if (flags & KEXEC_PRESERVE_CONTEXT)
		image->preserve_context = 1;

	ret = machine_kexec_prepare(image);
	if (ret)
		goto out;

	/*
	 * Some architecture(like S390) may touch the crash memory before
	 * machine_kexec_prepare(), we must copy vmcoreinfo data after it.
	 */
	ret = kimage_crash_copy_vmcoreinfo(image);
	if (ret)
		goto out;

	for (i = 0; i < nr_segments; i++) {
		ret = kimage_load_segment(image, &image->segment[i]);
		if (ret)
			goto out;
	}

	kimage_terminate(image);

	ret = machine_kexec_post_load(image);
	if (ret)
		goto out;

	/* Install the new kernel and uninstall the old */
	image = xchg(dest_image, image);

	out:
		if ((flags & KEXEC_ON_CRASH) && kexec_crash_image)
			arch_kexec_protect_crashkres();

	kimage_free(image);
	return ret;
	}

流程图
"""""""""

.. image:: ../img/do_kexec_load.svg
   :align: center


kexec_file_load
^^^^^^^^^^^^^^^

.. code-block:: c
   :caption: kexec_file_load
   :emphasize-lines: 4,5
   :linenos:
   
   
流程图
""""""" 

.. image:: ../img/kexec_file_load.svg
   :align: center  



kexec_elf_load
^^^^^^^^^^^^^^^

.. code-block:: c
   :caption: kexec_elf_load
   :emphasize-lines: 4,5
   :linenos:
   



原理流程图：
"""""""""




总结
^^^^^^
- 应用场景及示意图
  - 应用场景
  - 示意图
- 示例代码及工具
  - 内核自带tool自测：
  - kexec-tools：














