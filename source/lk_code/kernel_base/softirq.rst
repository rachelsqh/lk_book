linux 软中断：softirq
-----------------------

   
软中断：在硬件中断处理程序结束时调用的的句柄

.. code-block:: c
   :caption: struct_task --> mm
   :emphasize-lines: 4,5
   :linenos:
   
	/* PLEASE, avoid to allocate new softirqs, if you need not _really_ high
   	frequency threaded job scheduling. For almost all the purposes
   	tasklets are more than enough. F.e. all serial device BHs et
   	al. should be converted to tasklets, not to softirqs.
 	*/
	/* 所有的软中断向量，数字越小优先级越高 */
	enum
	{
		HI_SOFTIRQ=0,
		TIMER_SOFTIRQ,
		NET_TX_SOFTIRQ,
		NET_RX_SOFTIRQ,
		BLOCK_SOFTIRQ,
		IRQ_POLL_SOFTIRQ,
		TASKLET_SOFTIRQ,
		SCHED_SOFTIRQ,
		HRTIMER_SOFTIRQ,
		RCU_SOFTIRQ,    /* Preferable RCU should always be the last softirq */

		NR_SOFTIRQS
	};

	struct softirq_action
	{
		void	(*action)(struct softirq_action *);
	};
	static struct softirq_action softirq_vec[NR_SOFTIRQS] __cacheline_aligned_in_smp;

	DEFINE_PER_CPU(struct task_struct *, ksoftirqd);

	const char * const softirq_to_name[NR_SOFTIRQS] = {
		"HI", "TIMER", "NET_TX", "NET_RX", "BLOCK", "IRQ_POLL",
		"TASKLET", "SCHED", "HRTIMER", "RCU"
	};


初始化流程：start_kernel-->softirq_init() -->early_initcall(spawn_ksoftirqd)

可以理解为所有软中断通过softirq_action进行组织，数组的每个成员指向处理某类事件的处理函数。函数在不同的CPU上是可重入的。每个CPU上运行的软中断处理线程负责处理本CPU上产生的事件。处理完挂起的事件后内核线程就调度出去。在特定时机唤醒内核线程。内核线程继续检查是否有挂起的事件，周而复始。这个时机在下文中进一步解释。唤醒则参考wakeup_softirqd函数。可考虑比较不同版本内核处理方式上的差异。此时关注内核线程优先级问题。

static struct softirq_action softirq_vec[NR_SOFTIRQS] __cacheline_aligned_in_smp;

DEFINE_PER_CPU(struct task_struct *, ksoftirqd);






1.softirq_init()

.. code-block:: c
   :caption: struct_task --> mm
   :emphasize-lines: 4,5
   :linenos:

	/* Tasklets --- multithreaded analogue of BHs.

   	This API is deprecated. Please consider using threaded IRQs instead:
   	https://lore.kernel.org/lkml/20200716081538.2sivhkj4hcyrusem@linutronix.de

   	Main feature differing them of generic softirqs: tasklet
   	is running only on one CPU simultaneously.//与通用软中断不同的主要特点：tasklet只同时在一个 CPU 上运行。

   	Main feature differing them of BHs: different tasklets
   	may be run simultaneously on different CPUs.//与 BH 不同的主要特征：不同的 tasklet
   	可以在不同的 CPU 上同时运行。

   	Properties:
   	* If tasklet_schedule() is called, then tasklet is guaranteed
    	 to be executed on some cpu at least once after this.
   	* If the tasklet is already scheduled, but its execution is still not
   	  started, it will be executed only once.
   	* If this tasklet is already running on another CPU (or schedule is called
    	 from tasklet itself), it is rescheduled for later.
   	* Tasklet is strictly serialized wrt itself, but not
    	 wrt another tasklets. If client needs some intertask synchronization,
    	 he makes it with spinlocks.//Tasklet 是严格序列化的 wrt 本身，但不是 wrt 另一个 tasklets。如果客户端需要一些任务间同步，他会使用自旋锁来实现。
	Tasklet shì yángé xùliè huà de wrt 
 	*/

	struct tasklet_struct
	{
		struct tasklet_struct *next;
		unsigned long state;
		atomic_t count;
		bool use_callback;
		union {
			void (*func)(unsigned long data);
			void (*callback)(struct tasklet_struct *t);
		};
		unsigned long data;
	};

	/*
	 * Tasklets
	 */
	struct tasklet_head {//tasklet_struct组织方式
		struct tasklet_struct *head;
		struct tasklet_struct **tail;
	};

	static DEFINE_PER_CPU(struct tasklet_head, tasklet_vec);
	static DEFINE_PER_CPU(struct tasklet_head, tasklet_hi_vec);

	void __init softirq_init(void)
	{
		int cpu;

		for_each_possible_cpu(cpu) { 
			per_cpu(tasklet_vec, cpu).tail =
				&per_cpu(tasklet_vec, cpu).head;
			per_cpu(tasklet_hi_vec, cpu).tail = &per_cpu(tasklet_hi_vec, cpu).head;
		}

		open_softirq(TASKLET_SOFTIRQ, tasklet_action);// softirq_vec[TASKLET_SOFTIRQ] = tasklet_action;初始化tasklet_action,具体操作参考下文描述；
		open_softirq(HI_SOFTIRQ, tasklet_hi_action); // softirq_vec[HI_SOFTIRQ] = tasklet_hi_action;初始化tasklet_hi_action,具体操作参考下文描述；


	}


2. spawn_ksoftirqd()：在内核初始化初期为每一个CPU新建内核线程ksoftirqd


.. code-block:: c
   :caption: struct_task --> mm
   :emphasize-lines: 4,5
   :linenos:

	static struct smp_hotplug_thread softirq_threads = {
		.store			= &ksoftirqd,//每个CPU存储struct task指针
		.thread_should_run	= ksoftirqd_should_run, //判断句柄
		.thread_fn		= run_ksoftirqd,//处理每个CPU上的软中断
		.thread_comm		= "ksoftirqd/%u",//每个CPU运行的处理软中断的内核线程名字格式
	};

	static __init int spawn_ksoftirqd(void)
	{
		cpuhp_setup_state_nocalls(CPUHP_SOFTIRQ_DEAD, "softirq:dead", NULL,
				  takeover_tasklets); //cpu:CPUHP_SOFTIRQ_DEAD状态回调函数。
		BUG_ON(smpboot_register_percpu_thread(&softirq_threads));//

		return 0;
	}
	early_initcall(spawn_ksoftirqd);//新建内核线程时机


我们看正常运行系统负责处理软中断的内核线程（当前硬件：八核）：

.. code-block:: c
   :caption: 每个CPU运行一个处理软中断的线程
   :emphasize-lines: 4,5
   :linenos:

	root@rachel:/usr/src/linux-source-5.14/kernel# ps -aux|grep ksoft
	root          12  0.0  0.0      0     0 ?        S     2021   0:21 [ksoftirqd/0]
	root          18  0.0  0.0      0     0 ?        S     2021   0:03 [ksoftirqd/1]
	root          23  0.0  0.0      0     0 ?        S     2021   0:00 [ksoftirqd/2]
	root          28  0.0  0.0      0     0 ?        S     2021   0:00 [ksoftirqd/3]
	root          33  0.0  0.0      0     0 ?        S     2021   0:01 [ksoftirqd/4]
	root          38  0.0  0.0      0     0 ?        S     2021   0:01 [ksoftirqd/5]
	root          43  0.0  0.0      0     0 ?        S     2021   7:52 [ksoftirqd/6]
	root          48  0.0  0.0      0     0 ?        S     2021   0:02 [ksoftirqd/7]


到目前为止，初始化就完成了，我们看其运行周期：

软中断运行点：


.. code-block:: c
   :caption: ksoftirqd线程唤醒时机
   :emphasize-lines: 4,5
   :linenos:
	
	/*
 	* we cannot loop indefinitely here to avoid userspace starvation,
 	* but we also don't want to introduce a worst case 1/HZ latency
 	* to the pending events, so lets the scheduler to balance
 	* the softirq load for us.
 	*/
	static void wakeup_softirqd(void)
	{
		/* Interrupts are disabled: no need to stop preemption */
		struct task_struct *tsk = __this_cpu_read(ksoftirqd);

		if (tsk)
			wake_up_process(tsk);
	}



__do_softirq --> wakeup_softirqd:具体唤醒时间点:（上状态图)

- 具体场景：irq_exit_rcu(void) -->\_\_irq_exit_rcu -->invoke_softirq() -->wakeup_softirqd / __do_softirq_

- 具体场景：irq_exit(void) -->\_\_irq_exit_rcu -->invoke_softirq() -->wakeup_softirqd / __do_softirq_

- 具体场景：raise_softirq_irqoff --> wakeup_softirqd

- 具体场景：raise_softirq --> raise_softirq_irqoff --> wakeup_softirqd

- 具体场景： __local_bh_enable_ip --> wakeup_softirqd


为某类软中断初始化处理句柄：

open_softirq(NET_TX_SOFTIRQ, net_tx_action);

.. code-block:: c
   :caption: struct_task --> mm
   :emphasize-lines: 4,5
   :linenos:
	void open_softirq(int nr, void (*action)(struct softirq_action *))
	{
		softirq_vec[nr].action = action;
	}


每个类型的软中断有一个hook，怎么处理，怎么组织这类事件则在这个hook中处理。

我们以net_tx_action为例，看下其代码：




.. code-block:: c
   :caption: net_tx_action分析
   :emphasize-lines: 4,5
   :linenos:

	static __latent_entropy void net_tx_action(struct softirq_action *h)
	{
		struct softnet_data *sd = this_cpu_ptr(&softnet_data);

		if (sd->completion_queue) {
			struct sk_buff *clist;

			local_irq_disable();
			clist = sd->completion_queue;
			sd->completion_queue = NULL;
			local_irq_enable();

			while (clist) {
				struct sk_buff *skb = clist;

				clist = clist->next;

				WARN_ON(refcount_read(&skb->users));
				if (likely(get_kfree_skb_cb(skb)->reason == SKB_REASON_CONSUMED))
					trace_consume_skb(skb);
				else
					trace_kfree_skb(skb, net_tx_action);

				if (skb->fclone != SKB_FCLONE_UNAVAILABLE)
					__kfree_skb(skb);
				else
					__kfree_skb_defer(skb);
			}
		}

		if (sd->output_queue) {
			struct Qdisc *head;

			local_irq_disable();
			head = sd->output_queue;
			sd->output_queue = NULL;
			sd->output_queue_tailp = &sd->output_queue;
			local_irq_enable();

			rcu_read_lock();

			while (head) {
				struct Qdisc *q = head;
				spinlock_t *root_lock = NULL;

				head = head->next_sched;

				/* We need to make sure head->next_sched is read
				 * before clearing __QDISC_STATE_SCHED
				 */
				smp_mb__before_atomic();

				if (!(q->flags & TCQ_F_NOLOCK)) {
					root_lock = qdisc_lock(q);
					spin_lock(root_lock);
				} else if (unlikely(test_bit(__QDISC_STATE_DEACTIVATED,
						     &q->state))) {
					/* There is a synchronize_net() between
					 * STATE_DEACTIVATED flag being set and
					 * qdisc_reset()/some_qdisc_is_busy() in
					 * dev_deactivate(), so we can safely bail out
					 * early here to avoid data race between
					 * qdisc_deactivate() and some_qdisc_is_busy()
					 * for lockless qdisc.
					 */
					clear_bit(__QDISC_STATE_SCHED, &q->state);
					continue;
				}

				clear_bit(__QDISC_STATE_SCHED, &q->state);
				qdisc_run(q);
				if (root_lock)
					spin_unlock(root_lock);
			}

			rcu_read_unlock();
		}

		xfrm_dev_backlog(sd);
	}


句柄注册(上状态图)
^^^^^^^^^^^^^^^^

- kernel/softirq.c:open_softirq(TASKLET_SOFTIRQ, tasklet_action);// softirq_vec[TASKLET_SOFTIRQ] = tasklet_action;

- kernel/softirq.c:open_softirq(HI_SOFTIRQ, tasklet_hi_action); // softirq_vec[HI_SOFTIRQ] = tasklet_hi_action;
- kernel/time/timer.c:2024:      open_softirq(TIMER_SOFTIRQ, run_timer_softirq);
- kernel/time/hrtimer.c:2165:    open_softirq(HRTIMER_SOFTIRQ, hrtimer_run_softirq);
- kernel/rcu/tiny.c:222: open_softirq(RCU_SOFTIRQ, rcu_process_callbacks) + kernel/rcu/tree.c:4757:                open_softirq(RCU_SOFTIRQ, rcu_core_si);
- kernel/sched/fair.c:11578:     open_softirq(SCHED_SOFTIRQ, run_rebalance_domains);
- net/core/dev.c:11718:       open_softirq(NET_TX_SOFTIRQ, net_tx_action);
- net/core/dev.c:11719:       open_softirq(NET_RX_SOFTIRQ, net_rx_action);
- block/blk-mq.c:4018:    open_softirq(BLOCK_SOFTIRQ, blk_done_softirq);
- lib/irq_poll.c:210:     open_softirq(IRQ_POLL_SOFTIRQ, irq_poll_softirq);

性能分析与总结
^^^^^^^^^^^^










