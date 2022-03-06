内存在进程中的表示
---------------------
描述平台：
- kernel:5.14
- hw:x86

概述
^^^^^
- 内核页表：swapper_pg_dir:内核页表目录（注意，这只代表地址空间，剩下的硬件都可以分配：这个要更深入理解一下，vmalloc的理解很重要）

- 应用页表：task->mm:注意，只有应用进程有struct mm_struct 结构，其中的mm->pgd指向进程的页表目录，在应用进程加载应用软件时（执行exec系统调用）为进程分配struct mm_struct结构，并在为mm_struct 结构分配也表目录（pgd成员）时首先复制内核也表到pgd(swapper_pg_dir),然后加载应用软件时分配应用空间并为其建立也表映射（pgd)。也就是说所有进程的页表目录都包含相同的内核映射。针对内核线程并没有单独的mm结构，当切换到一个内核线程时，不需要切换内存空间（x86中CR3寄存器切换），直接使用上一个进程的CR3的值（此时能调用call helper吗？）。

- 更明确mm 和 active_mm成员概念及应用场景：

- vmalloc 这部分的处理方式：


结构描述
^^^^^^^^^^
- 内存成员

.. code-block:: c
   :caption: struct_task --> mm
   :emphasize-lines: 4,5
   :linenos:
   
   struct task_struct {
   ......
   	struct mm_struct		*mm;
	struct mm_struct		*active_mm;
   ......
   }

- 我们看struct mm_struct 结构：

.. literalinclude:: ./code/mm_struct.txt
   :caption: struct mm_struct 定义
   :language: c
   :emphasize-lines: 5,7-12
   :linenos:
   :name: 进程地址空间
 
- struct vm_area_struct 结构描述
  
.. literalinclude:: ./code/vma.txt
   :caption: struct vm_area_struct 定义
   :language: c
   :emphasize-lines: 5,7-12
   :linenos:
   :name: 内存区域定义

struct vm_area_struct代表一块连续虚拟地址空间映射。
  
以X86为例，看vma与物理页面的关系   
 mm_struct --> pgd_t * pgd;
 
 现在我们看新建进程时对内存的处理：（fork --> kernel_clone)
 
.. literalinclude:: ./code/fork_1.txt
   :caption: fork --> kernel_clone()实现
   :language: c
   :emphasize-lines: 5,7-12
   :linenos:
   :name: fork主函数实现
   
copy_process --> copy_mm()函数实现

.. literalinclude:: ./code/copy_mm.txt
   :caption: copy_mm()实现
   :language: c
   :emphasize-lines: 5,7-12
   :linenos:
   :name: 空间复制
   
我们假设为通过函数产生的第一个进程，则 current_task == init_task,定义如下：

.. literalinclude:: ./code/init_task.txt
   :caption: init_task.mm初始化
   :language: c
   :emphasize-lines: 5,7-12
   :linenos:
   :name: init_task
   

因为内核线程不需要切换CR3寄存器，我们现在看第一个应用进程的产生，内核进程运行 run_init_process()

.. literalinclude:: ./code/run_init_process.txt
   :caption: 第一个应用进程
   :language: c
   :emphasize-lines: 5,7-12
   :linenos:
   :name: eg:sbin
   
   
    run_init_process --> kernel_execve --> alloc_bprm(fd, filename) --> mm_alloc(),所有的都在这里了
    
    
.. literalinclude:: ./code/mm_alloc.txt
   :caption: mm_alloc原理
   :language: c
   :emphasize-lines: 5,7-12
   :linenos:
   :name: 整个内存空间的初始化
   

现在我们看进程切换时，内存的处理方法：

.. literalinclude:: ./code/context_switch.txt
   :caption: 进程切换中：switch_mm
   :language: c
   :emphasize-lines: 5,7-12
   :linenos:
   :name: 内存空间切换
   


   
我们以X86平台为例，做个总结：
	1. 进程切换时，如果目标进程为内核进程，不需要切换CR3；若为应用进程，则需要切换CR3；
	2. 第一个进程，mm == NULL,active_mm == init_mm；
	3. 第一个应用进程，通过run_init_process --> kernel_execve --> alloc_bprm() --> mmalloc():实现task_struct -> mm_struct *mm的分配，并拷贝swapper_pg_dir到mm->pgd,这样内核空间也表初始化完成；
	4. 应用进程：父进程运行fork() --> copy_mm(),为子进程分配struct task_struct 结构，并将task->mm指向父进程的task->mm；子进程运行execxxx() --> kernel_execve() --> alloc_bprm() -->mmalloc()真正拥有自己的task->mm,并拷贝swapper_pg_dir到mm->pgd,这样内核空间也表初始化完成.




   
   



















   
   
