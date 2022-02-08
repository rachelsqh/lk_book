linux rcu原理分析
--------------------------

   
   从RCU实现原理上理解一下。

.. code-block:: c
   :caption: struct_task --> mm
   :emphasize-lines: 4,5
   :linenos:
   
	void __init rcu_init(void)
	{
		open_softirq(RCU_SOFTIRQ, rcu_process_callbacks);
		rcu_early_boot_tests();
	}


也就是说这就是用软中断实现的一个周期性任务。


.. code-block:: c
   :caption: struct_task --> mm
   :emphasize-lines: 4,5
   :linenos:
   
	/* Invoke the RCU callbacks whose grace period has elapsed.  */
	static __latent_entropy void rcu_process_callbacks(struct softirq_action *unused)
	{
		struct rcu_head *next, *list;
		unsigned long flags;

		/* Move the ready-to-invoke callbacks to a local list. */
		local_irq_save(flags);
		if (rcu_ctrlblk.donetail == &rcu_ctrlblk.rcucblist) {
			/* No callbacks ready, so just leave. */
			local_irq_restore(flags);
			return;
		}
		list = rcu_ctrlblk.rcucblist;
		rcu_ctrlblk.rcucblist = *rcu_ctrlblk.donetail;
		*rcu_ctrlblk.donetail = NULL;
		if (rcu_ctrlblk.curtail == rcu_ctrlblk.donetail)
			rcu_ctrlblk.curtail = &rcu_ctrlblk.rcucblist;
		rcu_ctrlblk.donetail = &rcu_ctrlblk.rcucblist;
		local_irq_restore(flags);

		/* Invoke the callbacks on the local list. */
		while (list) {
			next = list->next;
			prefetch(next);
			debug_rcu_head_unqueue(list);
			local_bh_disable();
			rcu_reclaim_tiny(list);
			local_bh_enable();
			list = next;
		}
	}


这是一个理解起来非常容易的一个过程。下面我们阐述其应用场景。

应用场景：



运行原理图：
