linux 内核竞争处理
^^^^^^^^^^^^^^^^^^
https://www.kernel.org/doc/html/latest/locking/locktypes.html


内核提供了多种锁定原语，可分为三类：

- 睡眠锁

只能在抢占式任务上下文中获取休眠锁。尽管实现允许来自其他上下文的 try_lock()，但有必要仔细评估 unlock() 以及 try_lock() 的安全性。此外，还需要评估这些原语的调试版本。简而言之，除非没有其他选择，否则不要从其他上下文获取休眠锁。

睡锁类型：
  - mutex
  - rt_mutex
  互斥锁，RT 互斥体是支持优先级继承 (PI) 的互斥体。

由于抢占和中断禁用部分，PI 对非 PREEMPT_RT 内核有限制。

PI 显然不能抢占禁用抢占或禁用中断的代码区域，即使在 PREEMPT_RT 内核上也是如此。相反，PREEMPT_RT 内核在可抢占任务上下文中执行大多数此类代码区域，尤其是中断处理程序和软中断。这种转换允许通过 RT 互斥锁实现 spinlock_t 和 rwlock_t。
  
  
  
  - semaphore
  semaphore 是一个计数信号量实现。信号量通常用于序列化和等待，但新的用例应该使用单独的序列化和等待机制，例如互斥锁和完成。
  
  - rw_semaphore
  rw_semaphore 是一个多读单写锁机制。在非 PREEMPT_RT 内核上，实现是公平的，从而防止写入器饥饿。rw_semaphore 默认符合严格的所有者语义，但存在允许读者进行非所有者释放的特殊用途接口。这些接口独立于内核配置工作。

rw_semaphore 和 PREEMPT_RT
PREEMPT_RT 内核将 rw_semaphore 映射到一个单独的基于 rt_mutex 的实现，从而改变了公平性：

因为 rw_semaphore writer 不能将其优先级授予多个 reader，所以抢占的低优先级 reader 将继续持有其锁，因此即使是高优先级 writers 也会挨饿。相反，因为读者可以将他们的优先级授予写者，所以被抢占的低优先级写者的优先级将被提升，直到它释放锁，从而防止写者饿死读者
  
  - ww_mutex：
  
  - percpu_rw_semaphore
在 PREEMPT_RT 内核上，这些锁类型被转换为休眠锁：
  - local_lock
  
  
  - spinlock_t
  - rwlock_t





- 基于CPU的锁
  - local_lock
  在非 PREEMPT_RT 内核上，local_lock 函数是抢占和中断禁用原语的包装器。与其他锁定机制相反，禁用抢占或中断是纯 CPU 本地并发控制机制，不适合 CPU 间并发控制。

- 自旋锁
  - raw_spinlock_t
  - bit spinlocks
在非 PREEMPT_RT 内核上，这些锁类型也是旋转锁：
  - spinlock_t
  - rwlock_t
旋转锁隐式禁用抢占，锁定/解锁功能可以具有应用进一步保护的后缀：
  - _bh()：禁用/启用下半部分（软中断）
  - _irq()：禁用/启用中断
  - _irqsave/restore()：保存和禁用/恢复中断禁用状态


。。。。。。


总结：

