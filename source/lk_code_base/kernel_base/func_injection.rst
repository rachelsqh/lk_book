linux fail_function分析:基于函数的错误注入
----------------------------------------
.. code-block:: c
   :caption: struct_task --> mm
   :emphasize-lines: 4,5
   :linenos:

描述：

针对内核函数的开始处进行注入。所以kprobe就是有这种功能在。哎呀真理了个啥呀，就提供了一个框架，都怀疑是我自己写的了，先放着，下一遍确认是否是代码树的问题。

底层原理
^^^^^^^  
   register_kprobe(&attr->kp);
   
浅显理解：debugfs文件夹：

   底层逻辑：设置时调用register_kprobe 在相关函数入口点注册hook.底层原理没有什么特别需要注意的。
   只有两个技术点：

- debugfs相关操作；
- kprobe


操作方法
^^^^^^^^^^
放下，貌似没有

