linux KCSAN分析
--------------------------
.. code-block:: c
   :caption: struct_task --> mm
   :emphasize-lines: 4,5
   :linenos:
   
 The Kernel Concurrency Sanitizer (KCSAN)  
   
内核文档：https://www.kernel.org/doc/html/latest/dev-tools/kcsan.html
 Kernel Concurrency Sanitizer (KCSAN) 是一种动态竞争检测器，它依赖于编译时检测，并使用基于观察点的采样方法来检测竞争。KCSAN 的主要目的是检测数据竞争。

GCC 和 Clang 都支持 KCSAN。对于 GCC，我们需要 11 或更高版本，而对于 Clang，我们也需要 11 或更高版本。

需要通过设置以下内核设置来使能KCSAN：


.. code-block:: c
   :caption: struct_task --> mm
   :emphasize-lines: 4,5
   :linenos:
   
   CONFIG_KCSAN = y 及lib/Kconfig.kcsan相关选项。
   
错误报告
^^^^^^^^

常见报告类型


.. code-block:: c
   :caption: struct_task --> mm
   :emphasize-lines: 4,5
   :linenos:
   
   ==================================================================
   BUG: KCSAN: data-race in test_kernel_read / test_kernel_write
   
   write to 0xffffffffc009a628 of 8 bytes by task 487 on cpu 0:
   test_kernel_write+0x1d/0x30
   access_thread+0x89/0xd0
   kthread+0x23e/0x260
   ret_from_fork+0x22/0x30

   read to 0xffffffffc009a628 of 8 bytes by task 488 on cpu 6:
   test_kernel_read+0x10/0x20
   access_thread+0x89/0xd0
   kthread+0x23e/0x260
   ret_from_fork+0x22/0x30

   value changed: 0x00000000000009a6 -> 0x00000000000009b2

   Reported by Kernel Concurrency Sanitizer on:
   CPU: 6 PID: 488 Comm: access_thread Not tainted 5.12.0-rc2+ #1
   Hardware name: QEMU Standard PC (i440FX + PIIX, 1996), BIOS 1.14.0-2 04/01/2014
   ==================================================================


报告的标题提供了比赛中涉及的功能的简短摘要。紧随其后的是参与数据竞争的 2 个线程的访问类型和堆栈跟踪。如果 KCSAN 也观察到值变化，则观察到的旧值和新值分别显示在“值变化”行上。


另一种不太常见的数据竞争报告类型如下所示：

.. code-block:: c
   :caption: struct_task --> mm
   :emphasize-lines: 4,5
   :linenos:
   
   ==================================================================
   BUG: KCSAN: data-race in test_kernel_rmw_array+0x71/0xd0

   race at unknown origin, with read to 0xffffffffc009bdb0 of 8 bytes by task 515 on cpu 2:
   test_kernel_rmw_array+0x71/0xd0
   access_thread+0x89/0xd0
   kthread+0x23e/0x260
   ret_from_fork+0x22/0x30

   value changed: 0x0000000000002328 -> 0x0000000000002329

   Reported by Kernel Concurrency Sanitizer on:
   CPU: 2 PID: 515 Comm: access_thread Not tainted 5.12.0-rc2+ #1
   Hardware name: QEMU Standard PC (i440FX + PIIX, 1996), BIOS 1.14.0-2 04/01/2014
   ==================================================================


此报告是在无法确定其他竞赛线程的情况下生成的，但由于监视的内存位置的数据值已更改而推断出竞赛。这些报告始终显示“值已更改”行。这种类型的报告的一个常见原因是在比赛线程中缺少检测，但也可能由于例如 DMA 访问而发生。CONFIG_KCSAN_REPORT_RACE_UNKNOWN_ORIGIN=y只有在默认情况下启用此类报告时才会显示此类报告。

选择性分析
^^^^^^^^^

可能需要禁用特定访问、函数、编译单元或整个子系统的数据竞争检测。对于静态黑名单，可以使用以下选项：

- KCSAN 理解data_race(expr)注释，它告诉 KCSAN 由于访问expr而导致的任何数据竞争都应该被忽略，并且在遇到数据竞争时所产生的行为被认为是安全的。有关详细信息，请参阅 LKMM 中的“标记共享内存访问”。

- 禁用整个函数的数据竞争检测可以通过使用函数属性来完成__no_kcsan：

.. code-block:: c
   :caption: struct_task --> mm
   :emphasize-lines: 4,5
   :linenos:

    __no_kcsan
    void foo(void) {
    ...
要动态限制生成报告的函数，请参阅 DebugFS 接口黑名单/白名单功能。
- 要禁用特定编译单元的数据竞争检测，请添加 Makefile：

.. code-block:: c
   :caption: struct_task --> mm
   :emphasize-lines: 4,5
   :linenos:

   KCSAN_SANITIZE_file.o := n

- 要禁用 a 中列出的所有编译单元的数据竞争检测 Makefile，请添加到相应的Makefile：


.. code-block:: c
   :caption: struct_task --> mm
   :emphasize-lines: 4,5
   :linenos:
   
   KCSAN_SANITIZE := n
   

此外，可以根据偏好告诉 KCSAN 显示或隐藏整个数据竞争类别。这些可以通过以下 Kconfig 选项进行更改：

- CONFIG_KCSAN_REPORT_VALUE_CHANGE_ONLY：如果启用并且通过观察点观察到冲突写入，但观察到内存位置的数据值保持不变，则不报告数据竞争。

- CONFIG_KCSAN_ASSUME_PLAIN_WRITES_ATOMIC：假设默认情况下，不超过字大小的纯对齐写入是原子的。假设此类写入不受不安全编译器优化的影响，从而导致数据竞争。该选项会导致 KCSAN 不报告数据竞争，因为冲突是唯一的普通访问对齐写入到字长。

- CONFIG_KCSAN_PERMISSIVE：启用额外的许可规则以忽略某些类别的常见数据竞争。与上述不同的是，规则更复杂，涉及值更改模式、访问类型和地址。此选项取决于CONFIG_KCSAN_REPORT_VALUE_CHANGE_ONLY=y. 有关详细信息，请参阅kernel/kcsan/permissive.h. 建议只关注来自特定子系统而不是整个内核的报告的测试人员和维护人员禁用此选项。

要使用尽可能严格的规则，请选择CONFIG_KCSAN_STRICT=y，它将 KCSAN 配置为尽可能遵循 Linux 内核内存一致性模型 (LKMM)。


DebugFS 接口
^^^^^^^^^^^^
该文件/sys/kernel/debug/kcsan提供以下接口：

- 读取会/sys/kernel/debug/kcsan返回各种运行时统计信息。

- 写入on或off允许/sys/kernel/debug/kcsan分别打开或关闭 KCSAN。

- 写入添加 !some_func_name到报告过滤器列表中，该列表（默认情况下）将报告数据争用列入黑名单，其中顶部堆栈帧之一是列表中的函数。/sys/kernel/debug/kcsansome_func_name

- 写入blacklist或whitelist更改/sys/kernel/debug/kcsan 报告过滤行为。例如，黑名单功能可用于消除频繁发生的数据竞争；白名单功能可以帮助修复和测试修复。


调优性能
^^^^^^^^

影响 KCSAN 的整体性能和错误检测能力的核心参数作为内核命令行参数公开，其默认值也可以通过相应的 Kconfig 选项进行更改。

- kcsan.skip_watch( CONFIG_KCSAN_SKIP_WATCH)：在设置另一个观察点之前要跳过的每个 CPU 内存操作的数量。更频繁地设置观察点将导致观察比赛的可能性增加。该参数对整体系统性能和竞态检测能力的影响最为显着。

- kcsan.udelay_task( CONFIG_KCSAN_UDELAY_TASK)：对于任务，设置观察点后停止执行的微秒延迟。较大的值会导致我们可以观察到竞争增加的窗口。

- kcsan.udelay_interrupt( CONFIG_KCSAN_UDELAY_INTERRUPT)：对于中断，设置观察点后停止执行的微秒延迟。中断具有更严格的延迟要求，并且它们的延迟通常应该小于为任务选择的延迟。


它们可以在运行时通过/sys/module/kcsan/parameters/.


下一轮结合实验继续：


























