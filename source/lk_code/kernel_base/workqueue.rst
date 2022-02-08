linux 工作队列: workqueue
--------------------------
.. code-block:: c
   :caption: struct_task --> mm
   :emphasize-lines: 4,5
   :linenos:

使用共享工作池的通用异步执行

这是通用的异步执行机制。 在流程上下文中执行的工作项。 工作池是共享的并自动管理。 每个 CPU 有两个工作池（一个用于正常工作项目，另一个用于高优先级工作项目）和一些额外的工作队列池，它们不绑定到任何特定的 CPU——这些后备池的数量是动态的。

工作队列应用场景
^^^^^^^^^^^^^^^^^

工作队列实现原理
^^^^^^^^^^^^^^^^^
内核线程:kworker 


具体参考
^^^^^^^^
https://www.kernel.org/doc/html/latest/core-api/workqueue.html
   
   
