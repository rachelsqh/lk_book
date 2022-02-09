内存在进程中的表示
---------------------
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

我们看struct mm_struct 结构：

.. literalinclude:: ./mm_struct.txt
   :caption: struct mm_struct 定义
   :language: c
   :emphasize-lines: 5,7-12
   :linenos:
   :name: 进程地址空间
   
.. literalinclude:: ./vma.txt
   :caption: struct vm_area_struct 定义
   :language: c
   :emphasize-lines: 5,7-12
   :linenos:
   :name: 内存区域定义
   
以X86为例，看vma与物理页面的关系   
 mm_struct --> pgd_t * pgd;
 
 现在我们看新建进程时对内存的处理：（fork --> kernel_clone)
 
.. literalinclude:: ./fork_1.txt
   :caption: fork --> kernel_clone()实现
   :language: c
   :emphasize-lines: 5,7-12
   :linenos:
   :name: fork主函数实现
   
copy_process --> copy_mm()函数实现

.. literalinclude:: ./copy_mm.txt
   :caption: copy_mm()实现
   :language: c
   :emphasize-lines: 5,7-12
   :linenos:
   :name: 空间复制
   
我们假设为通过函数产生的第一个进程，则 current_task == init_task,定义如下：

.. literalinclude:: ./init_task.txt
   :caption: init_task.mm初始化
   :language: c
   :emphasize-lines: 5,7-12
   :linenos:
   :name: init_task
   

因为内核线程不需要切换CR3寄存器，我们现在看第一个应用进程的产生，内核进程运行 run_init_process()

.. literalinclude:: ./run_init_process.txt
   :caption: 第一个应用进程
   :language: c
   :emphasize-lines: 5,7-12
   :linenos:
   :name: eg:sbin
   
   
    run_init_process --> kernel_execve --> alloc_bprm(fd, filename) --> mm_alloc(),所有的都在这里了
    
    
.. literalinclude:: ./mm_alloc.txt
   :caption: mm_alloc原理
   :language: c
   :emphasize-lines: 5,7-12
   :linenos:
   :name: 整个内存空间的初始化
   

现在我们看进程切换时，内存的处理方法：

.. literalinclude:: ./context_switch.txt
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




   
   



















   
   
