
内存管理概述
----------
物理内存管理buddy
进程内存管理 <---> buddy;

内核内存管理 <--> slab <---> buddy;

疑问：内核内存管理内核分配的内存是否是已经映射好的？还是跟进程内存一样可以随便一页？是否会出现。。。
内核空间的映射已经完全映射好了，还是要现成的映射，如果要现成映射，页表如何构建

内核内存（针对kmalloc)初始化：setup_arch -> init_mem_mapping->init_memory_mapping -> kernel_physical_mapping_init提前映射。kmalloc就是在其上分配
疑问：x86下应该vmalloc直接对应kmalloc(只是留下空间？）kmalloc 后buddy中的页被标准为已经被占用，其他包括页表应该没变化。


其实内存就这么会事情，换另一个角度，那iomap呢？

ioremap() + vmalloc ==> struct vm_struct:列表 pgd_offset_k(address):pgd_offset():也就是说虚拟地址对应内核部分。用vm_struct 列表组织

X86下 缺页中断对内核地址的处理是对内核内存管理的一个提纲。为什么X86_64下缺页中断内核内存部分没有vmalloc部分的处理了，为什么呢？因为这部分确定是映射的，不会产生缺页中断了？那init_task.mm如何与当前进程同步？

struct vmap_area{//VM_ALLOC_START,VM_ALLOC_END区域中对应的虚拟空间单元
	va_start,va_end;}
	
struct vm_struct {} 物理单元
vmap_area_root中
map_vm_area对分配的页面进行了映射，map_vm_area-->vmap_page_range-->vmap_page_range_noflush。

32位下，vmalloc在pg_fault时，完成所有同步，其实不管谁去访问，那个地址，那是一个内核地址，只是因为映射不在其中而已，但页表是在地址空间的？所有只是针对init.pgd就够了？v

x86:CONFIG_DYNAMIC_MEMORY_LAYOUT:

------------------------------
X86-64,如果不使能CONFIG_DYNAMIC_MEMORY_LAYOUT，则处理方式与kmalloc一致。

--------------------

KASLR 
-----------
VMAP_STACK();


iomem相关总结
-------------
内存统计信息
--------------------------
.. code-block:: c
   :caption: struct_task --> mm
   :emphasize-lines: 4,5
   :linenos:
   
   
mmap来理解vma及硬件页关联：
--------------------------
.. code-block:: c
   :caption: struct_task --> mm
   :emphasize-lines: 4,5
   :linenos:
   
   
内存性能测试方法
--------------------------
.. code-block:: c
   :caption: struct_task --> mm
   :emphasize-lines: 4,5
   :linenos:
   
   
物理页面管理机制：slab/slub/...
--------------------------
.. code-block:: c
   :caption: struct_task --> mm
   :emphasize-lines: 4,5
   :linenos:
   
   
内存安全机制：
--------------------------
.. code-block:: c
   :caption: struct_task --> mm
   :emphasize-lines: 4,5
   :linenos:
   
   
swap:交换内存：
--------------------------
.. code-block:: c
   :caption: struct_task --> mm
   :emphasize-lines: 4,5
   :linenos:
   
   
其他内存相关：
--------------------------
.. code-block:: c
   :caption: struct_task --> mm
   :emphasize-lines: 4,5
   :linenos:
   
   
