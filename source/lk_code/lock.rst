linux 内核竞争处理
^^^^^^^^^^^^^^^^^^
内核中的锁与应用的锁功能类似，都是为了保护某个内核数据结构或者内存区域在多个执行路径时不被破坏，确保数据的一致性。linux 内核一方面为应用进程提供系统调用接口，代表用户进程运行在进程上下文；一方面要相应硬件中断，也就是运行在中断上下文。内核在进程上下文与中断上下文中来回切换，执行相应的任务请求，自然就会产生数据的并发访问，产生竞争条件；另为当前的硬件系统大多数为多核CPU，多CPU同时访问数据也会产生竞争。


linux 内核提供了一系列锁定原语，可以分为三种类型：
- 睡眠锁；
- CPU 本地锁；
- 自旋锁。

描述锁类型和其嵌套规则，包括PREEMPT_RT下的使用规则。


信号量在创建时需要设置一个初始值，表示同时可以有几个任务可以访问该信号量保护的共享资源，初始值为1 就变成互斥锁（Mutex），即同时只能有一个任务可以访问信号量保护的共享资源。

信号量与互斥锁：





睡眠锁
"""""""
只能在可抢占进程上下文中使用睡眠锁。除非没有其他选择，负责不要在其他上下文中获取休眠锁。
- mutex:互斥锁，同一时间只能有一个任务持有互斥锁，而且只有这个任务可以对互斥锁进行解锁。互斥锁不能进行递归锁定或解锁。一个互斥锁对象必须通过其API初始化，而不能使用memset或复制初始化。一个任务在持有互斥锁的时候是不能结束的。互斥锁所使用的内存区域是不能被释放的。使用中的互斥锁是不能被重新初始化的。并且互斥锁不能用于中断上下文。
    互斥锁比当前的内核信号量选项更快，并且更加紧凑。
    调用try_xx避免睡眠，如果直接调用mutex_lock()而不能理解获得锁，会导致睡眠等待（获得信号返回？）
  - 实现原理:
  include linux/mutex.h
/*
 * Simple, straightforward mutexes with strict semantics:
 *
 * - only one task can hold the mutex at a time
 * - only the owner can unlock the mutex
 * - multiple unlocks are not permitted
 * - recursive locking is not permitted
 * - a mutex object must be initialized via the API
 * - a mutex object must not be initialized via memset or copying
 * - task may not exit with mutex held
 * - memory areas where held locks reside must not be freed
 * - held mutexes must not be reinitialized
 * - mutexes may not be used in hardware or software interrupt
 *   contexts such as tasklets and timers
 *
 * These semantics are fully enforced when DEBUG_MUTEXES is
 * enabled. Furthermore, besides enforcing the above rules, the mutex
 * debugging code also implements a number of additional features
 * that make lock debugging easier and faster:
 *
 * - uses symbolic names of mutexes, whenever they are printed in debug output
 * - point-of-acquire tracking, symbolic lookup of function names
 * - list of all locks held in the system, printout of them
 * - owner tracking
 * - detects self-recursing locks and prints out all relevant info
 * - detects multi-task circular deadlocks and prints out all affected
 *   locks and tasks (and only those tasks)
 */
struct mutex {
	atomic_long_t		owner;/* 未加锁是0,加锁后是持有锁的进程id 
	
					* @owner: contains: 'struct task_struct *' to the current lock owner,
 					* NULL means not owned. Since task_struct pointers are aligned at
 					* at least L1_CACHE_BYTES, we have low bits to store extra state.
 					*
 					* Bit0 indicates a non-empty waiter list; unlock must issue a wakeup.
					 * Bit1 indicates unlock needs to hand the lock to the top-waiter
 					* Bit2 indicates handoff has been done and we're waiting for pickup.
 					
					#define MUTEX_FLAG_WAITERS	0x01
					#define MUTEX_FLAG_HANDOFF	0x02
					#define MUTEX_FLAG_PICKUP	0x04

					#define MUTEX_FLAGS		0x07
	
					*/
	spinlock_t		wait_lock;/* 等待获取互斥锁中使用的自旋锁。在获取互斥锁的过程中，操作会在自旋锁的保护中进行。初始化为为锁定。  */
#ifdef CONFIG_MUTEX_SPIN_ON_OWNER
	struct optimistic_spin_queue osq; /* Spinner MCS lock */
#endif
	struct list_head	wait_list;/* 等待锁的进程列表 */
#ifdef CONFIG_DEBUG_MUTEXES
	void			*magic;
#endif
#ifdef CONFIG_DEBUG_LOCK_ALLOC
	struct lockdep_map	dep_map;
#endif
};
  - 场景
  - api
  	1. mutex_init
  		
  		/**
 * mutex_init - initialize the mutex
 * @mutex: the mutex to be initialized
 *
 * Initialize the mutex to unlocked state.
 *
 * It is not allowed to initialize an already locked mutex.
 */
#define mutex_init(mutex)						\
do {									\
	static struct lock_class_key __key;				\
									\
	__mutex_init((mutex), #mutex, &__key);				\
} while (0)

#ifdef CONFIG_DEBUG_LOCK_ALLOC
# define __DEP_MAP_MUTEX_INITIALIZER(lockname)			\
		, .dep_map = {					\
			.name = #lockname,			\
			.wait_type_inner = LD_WAIT_SLEEP,	\
		}
#else
# define __DEP_MAP_MUTEX_INITIALIZER(lockname)
#endif

#define __MUTEX_INITIALIZER(lockname) \
		{ .owner = ATOMIC_LONG_INIT(0) \
		, .wait_lock = __SPIN_LOCK_UNLOCKED(lockname.wait_lock) \
		, .wait_list = LIST_HEAD_INIT(lockname.wait_list) \
		__DEBUG_MUTEX_INITIALIZER(lockname) \
		__DEP_MAP_MUTEX_INITIALIZER(lockname) }

#define DEFINE_MUTEX(mutexname) \
	struct mutex mutexname = __MUTEX_INITIALIZER(mutexname)

extern void __mutex_init(struct mutex *lock, const char *name,
			 struct lock_class_key *key);
			 
void
__mutex_init(struct mutex *lock, const char *name, struct lock_class_key *key)
{
	atomic_long_set(&lock->owner, 0);/* 初始化为0：未加锁是0,加锁后是持有锁的进程id  */
	spin_lock_init(&lock->wait_lock);/* 初始化持有的自旋锁 */
	INIT_LIST_HEAD(&lock->wait_list); /* 初始化等待的进程列表 */
#ifdef CONFIG_MUTEX_SPIN_ON_OWNER
	osq_lock_init(&lock->osq);
#endif

	debug_mutex_init(lock, name, key);
}
EXPORT_SYMBOL(__mutex_init);	

void __sched mutex_lock(struct mutex *lock)
void __sched mutex_unlock(struct mutex *lock)
int  __sched mutex_lock/unlock_interruptible();/*信号可唤醒睡眠，返回值 */
			 



- rt_mutex:
  - 实现原理
  - 场景
  - api
- semaphore:
  - 实现原理：semaphore 是一个计数信号量实现。
  - 场景：信号量通常用于序列化和等待，但新的用例应该使用单独的序列化和等待机制，例如互斥锁和完成。
  - api
- rw_semaphore
  - 实现原理：是一个多读单写锁机制。
  - 场景：在非 PREEMPT_RT 内核上，实现是公平的，从而防止写入器饥饿。rw_semaphore 默认符合严格的所有者语义，但存在允许读者进行非所有者释放的特殊用途接口。这些接口独立于内核配置工作。
  - api：
- ww_mutex
  - 实现原理
  - 场景
  - api
- percpu_rw_semaphore
  - 实现原理
  - 场景
  - api


  - 场景
  - api
- spinlock_t
  - 实现原理:忙等待，原地等待方式等待资源访问空闲。自旋锁不应该被长时间的持有（消耗 CPU 资源）
  - 场景
  - api
  spinlock_t lock;         //定义 spinlock 变量
spin_lock_init(&lock);   //初始化 spinlock，或者直接使用 DEFINE_SPINLOCK 接口实现 定义+初始化
spin_lock(&lock);        //获取 spin_lock（也有给临界区加锁的说法），该接口会一直等待直到获取到锁
spin_lock(&unlock);      //解锁，被加锁的临界区只有在解锁后其它进程才可以进入。
  	
 不要嵌套使用 spinlock，这会直接导致死锁。
除了最基本的使用接口之外，还支持以下的接口以供在特殊情况下使用： 	
  	
  spin_lock_irq()      spin_unlock_irq()
spin_lock_irqsave()  spin_unlock_irqstore()
spin_lock_bh()       spin_unlock_bh()
spin_is_locked()	
  	
  	
  	
  	
  	
  	
  	local_irq_disable();
	spin_lock(&lock);
	
	完全等同于：
	
	spin_lock_irq(&lock);
	
	
  在分析 spinlock 之前，先考虑一个问题：如果让你来实现 linux 中的 spinlock，你会怎么做？

本质上来说，全局对象需要使用锁来保护的原因是并发的产生，如果是单一的执行流，不需要考虑使用锁来做数据保护，所以第一个需要思考的问题是：spinlock 的使用过程中会有哪些并发的产生？

内核的抢占
中断的抢占
下半部的抢占
SMP 架构中的并发


linux 默认支持内核抢占，这个特性使得当前进程的执行可以被其它内核进(线)程抢占执行，这种抢占并不是无时不刻都在进行的，而是存在一些抢占点，比如从中断处理函数返回到内核或用户空间，比如调用 preempt_enable 使能内核抢占，因此可能出现这样的情况：

内核进程 A 正在执行，并获取了一个 spinlock，此时发生了中断并跳转到中断服务程序中，在中断中一个更高优先级的进程 B 被唤醒，在中断返回时 B 会抢占 A 并执行，如果这时候 B 也请求同一个 spinlock，问题就来了：B 因为请求不到 spinlock 而一直自旋，而占用 spinlock 的 A 因为被 B 抢占而得不到执行，无法释放 spinlock。

因此，对于自旋锁而言，内核抢占是一个明显的风险点，作为最简单的处理，需要在使用自旋锁之前，就禁止内核抢占。

  
  
- rwlock_t
  - 实现原理:rwlock_t 是一个多读单写锁机制。
  - 场景
  - api

CPU本地锁
"""""""""""
- local_lock:
  - 实现原理
  - 场景
  - api
  
自旋锁
"""""""
- raw_spinlock_t
  - 实现原理
  - 场景
  - api
- bit spinlocks
  - 实现原理
  - 场景
  - api
在非PREEMPT_RT内核下，下面的锁也属于旋转锁：
- spinlock_t:
  - 实现原理
  - 场景
  - api
- rwlock_t:
  - 实现原理
  - 场景
  - api
旋转锁隐式禁用抢占，可通过添加后缀来锁定/解锁额外功能：
- _bh():禁用/启用下半部分（软中断）；
  - 实现原理
  - 场景
  - api
- _irq():禁用/启用中断
  - 实现原理
  - 场景
  - api
- _irqsave/restore():保存和禁用/回复中断，保存/恢复中断状态。
  - 实现原理
  - 场景
  - api


锁类型嵌套
"""""""""
- 最基本的规则是：
  - 相同锁类别（睡眠、CPU 本地、旋转）的锁类型可以任意嵌套，只要它们遵守通用锁排序规则以防止死锁。
  - 休眠锁类型不能嵌套在 CPU 本地和旋转锁类型中。
  - CPU 本地和旋转锁类型可以嵌套在睡眠锁类型中。
  - 旋转锁类型可以嵌套在所有锁类型中



  

在非 PREEMPT_RT 内核上，local_lock 函数是抢占和中断禁用原语的包装器。与其他锁定机制相反，禁用抢占或中断是纯 CPU 本地并发控制机制，不适合 CPU 间并发控制。



简单来说，内核中锁要做的事情就是确保临界区<critical section>始终只有一个执行路径，就是说在有锁保护的情况下，临界区的执行不会被其他执行路径中断; 接下来，就分别看一看内核中常用的几个锁保护机制<对应的代码实现在/kernel/locking/>：

semaphore(信号量）
Spin Lock(自旋锁）
Mutex(互斥锁）
Atomic(原子操作）


Semaphore(信号量）
semaphore是很常见的同步资源访问的方法，可以用于多个资源的访问控制;一般用于多个内核路径试图控制某个数据的并发访问，内核中对应的头文件在linux/semaphore.h:


struct semaphore {
  raw_spinlock_t		lock;
  unsigned int		count;
  struct list_head	wait_list;
};


上述中count就是需要同步访问的资源个数，一般在内核中都设置为1，即等同于互斥锁。semaphore有两个常用的操作方法：

down: 获取锁，读应可用的资源减少，通常获取锁会将阻塞所执行的路径，让任务进入等待状态; 内核实现了好几种方法供调用：
down: 如果锁已被持有，则执行任务会被阻塞，在内核中不推荐使用该方法
down_interruptible： 允许获取锁时被中断，在接收到中断信号后，返回中断错误-EINTR
down_killable: 执行任务阻塞时如果发生错误，则被中断返回-EINTR
down_trylock: 尝试获取锁，如果已被占用，则直接返回，该方法支持在中断上下文中使用
down_timeout： 设置一个超时时间，超过该等待时间未获取到锁则返回-ETIME
up: 释放锁，可以在任何执行路径执行该方法，即使未执行过down也可以进行锁的释放
互斥锁的使用与semaphore比较类似，具体的使用可以参考源码kernel/mutex.c.




spin lock(自旋锁)
spin lock是内核中最常用的同步方法，通常用于多个CPU执行路径尝试访问同一个内存数据时的同步并且执行任务不能休眠的场景。与其他如semaphore不同的是，spin lock不会让锁等待者进入休眠状态，而是执行一个简单的循环等待，如果此时锁被释放，则会尝试获取锁，这样就避免了上下文切换，从而提升效率。一般如果锁等待的时间如果超过系统上下文切换的时间，使用spin lock则会较少任务的等待时间，改善系统性能。

除此之外，在某些特殊的场景比如在中断上下文与内核执行路径上共享数据时，就不能使用如semaphore这类会时执行任务休眠的同步锁，因为内核一旦在处理中断时，发生进程调度，则可能发生中断无法被处理的情况。同样地，在处理中断时，也不能使用spin lock以防止类似的情况;因此，在中断处理上下文中，使用spin lock时要将本地中断禁止。另外，在可能发生内核抢占(kernel preemption)的时候，如果被抢占任务执有spin lock，就可能导致该锁一直未被释放。总结来说，使用spin lock要注意如下几个原则:

内核抢占应该被禁止，以防出现竞争条件
本地中断需要被禁止，防止中断无法处理的情况
持有锁的时间越短越好，避免引起性能问题
跟上述几个场景对应，Linux中的spin lock提供了好几个函数来实现不同场景下的同步<linux/spinlock.h>：

spin_lock: 获取锁，如果锁被持有了，则等待
spin_lock_bh： 获取锁，禁止了软中断/本地中断，但可以响应物理中断
spin_lock_irq： 获取锁时打开中断
spin_lock_irqsave： 获取锁时禁止本地中断
上述几个函数的实现都在kernel/locking/spinlock.c中，如果不希望获取锁失败时等待，则可以通过spin_trylock*来实现。


在PREEMPT_RT内核中，以下锁类型被转换为休眠锁：
- local_lock
  - 实现原理：local_lock 为通过禁用抢占或中断保护的关键部分提供命名范围。在非 PREEMPT_RT 内核上，local_lock 操作映射到抢占和中断禁用和启用原语：

	- local_lock(&llock)			preempt_disable()
	- local_unlock(&llock)			preempt_enable（）
	- local_lock_irq(&lock)			local_irq_disable()
	- local_unlock_irq(&llock)		local_irq_enable()
	- local_lock_irqsave(&llock)		local_irq_save()
	- local_unlock_irqrestore(&llock)		local_irq_restore()
   与常规原语相比，local_lock 的命名范围有两个优点：
   	- 锁名称允许静态分析，也是保护范围的清晰文档，而常规原语是无范围且不透明的。
   	- 如果启用了 lockdep，local_lock 将获得一个 lockmap，它允许验证保护的正确性。这可以检测从中断或软中断上下文调用使用 preempt_disable() 作为保护机制的函数的情况。除此之外，lockdep_assert_held(&llock) 与任何其他锁定原语一样工作。
   	- local_lock 和 PREEMPT_RT:PREEMPT_RT 内核将 local_lock 映射到每个 CPU 的 spinlock_t，从而改变语义：
   	  - 所有 spinlock_t 更改也适用于 local_lock。
	local_lock 应该在禁用抢占或中断是适当形式的并发控制以保护非 PREEMPT_RT 内核上的每个 CPU 数据结构的情况下使用。由于 PREEMPT_RT 特定的 spinlock_t 语义，local_lock 不适用于防止 PREEMPT_RT 内核上的抢占或中断。




内核文档参考：https://www.kernel.org/doc/html/latest/locking/locktypes.html



