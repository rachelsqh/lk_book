linux 加载ELF 内核文件：kexec
---------------------------------------


系统调用
^^^^^^^^^

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
