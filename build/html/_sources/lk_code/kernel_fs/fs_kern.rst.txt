内存文件系统
----------
这部分主要实现内核与用户空间的信息交换。
我们主要利用其提供的接口实现信息传递目的。

目的：总结对内存文件系统编程调用

kernfs/sysfs/procfs/debugfs/configfs综述

config SYSFS
	......
	select KERNFS
	
config CONFIGFS
	......
	select CONFIGFS
	
procfs
^^^^^^^^
proc的实现方法与其他方法是不相同的，需要注意。
导出符号：
""""""""

EXPORT_SYMBOL(proc_symlink)：
EXPORT_SYMBOL_GPL(_proc_mkdir)：
EXPORT_SYMBOL_GPL(proc_mkdir_data)：
EXPORT_SYMBOL(proc_mkdir_mode)：
EXPORT_SYMBOL(proc_mkdir)：
EXPORT_SYMBOL(proc_create_mount_point)：
EXPORT_SYMBOL(proc_create_data);
EXPORT_SYMBOL(proc_create);
EXPORT_SYMBOL(proc_create_seq_private);
EXPORT_SYMBOL(proc_create_single_data);
EXPORT_SYMBOL(proc_set_size);
EXPORT_SYMBOL(proc_set_user);
EXPORT_SYMBOL(remove_proc_entry);
EXPORT_SYMBOL(remove_proc_subtree);
EXPORT_SYMBOL_GPL(proc_get_parent_data);
EXPORT_SYMBOL(proc_remove);
EXPORT_SYMBOL(PDE_DATA);
EXPORT_SYMBOL_GPL(proc_create_net_data);
EXPORT_SYMBOL_GPL(proc_create_net_data_write);
EXPORT_SYMBOL_GPL(proc_create_net_single);
EXPORT_SYMBOL_GPL(proc_create_net_single_write);
EXPORT_SYMBOL(sysctl_vals);
EXPORT_SYMBOL(register_sysctl);
EXPORT_SYMBOL(register_sysctl_paths);
EXPORT_SYMBOL(register_sysctl_table);
EXPORT_SYMBOL(unregister_sysctl_table);
EXPORT_SYMBOL_GPL(register_oldmem_pfn_is_ram);
EXPORT_SYMBOL_GPL(unregister_oldmem_pfn_is_ram);
EXPORT_SYMBOL(vmcore_add_device_dump);


proc_create:需要初始化的操作句柄为：

proc文件系统初始化
""""""""""""""""


从系统调用到 proc_ops方法调用
"""""""""""""""""""""""""""""

重要目录应用 
"""""""""""""
注意，这个实现方式的方法实现最应该弄清楚


kcore
*********

sysctl
************

iomem
*************


总结
"""""""
以前疑问，现在总结：	

kernfs
^^^^^^^^
描述kernfs的实现机制及应用场景。





sysfs
^^^^^^^




debugfs
^^^^^^^^^^^










