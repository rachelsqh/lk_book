all:
	dot -Tsvg kset_kobject_ra.dot -o kset_kobject_ra.svg
	dot -Tsvg kobject_struct.dot -o kobject_struct.svg
	dot -Tsvg kobject_uevent.dot -o kobject_uevent.svg
	dot -Tsvg syscall.dot -o syscall.svg
	dot -Tsvg notifier_reboot.dot -o notifier_reboot.svg
	dot -Tsvg module_make.dot -o module_make.svg
	dot -Tsvg modpost.dot -o modpost.svg
	dot -Tsvg softirq.dot -o softirq.svg
	dot -Tsvg linux-kernel-defence-map.dot -o linux-kernel-defence-map.svg
	dot -Tsvg lsm.dot -o lsm.svg
	dot -Tsvg kprobe_register.dot -o kprobe_register.svg
	dot -Tsvg fs/vfs.dot -o fs/vfs.svg
	dot -Tsvg fs/ps_fs.dot -o fs/ps_fs.svg
	#dot -Tsvg fs/fs_fun.dot -o fs/fs_fun.svg
	dot -Tsvg kernel_mem_map.dot -o kernel_mem_map.svg
	dot -Tsvg kernel_boot.dot -o kernel_boot.svg
clean:
	rm -rf *.svg
	rm -rf fs/*.svg
	
