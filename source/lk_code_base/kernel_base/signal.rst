linux 信号:signal
--------------------------

信号在进程中的实现
^^^^^^^^^^^^^^^^
.. code-block:: c
   :caption: 信号在进程中的表示
   :emphasize-lines: 4,5
   :linenos:
   
	struct task_struct {
	......
		/* Signal handlers: */
		struct signal_struct		*signal;
		struct sighand_struct __rcu		*sighand;
		sigset_t			blocked;
		sigset_t			real_blocked;
		/* Restored if set_restore_sigmask() was used: */
		sigset_t			saved_sigmask;
		struct sigpending		pending;
		unsigned long			sas_ss_sp;
		size_t				sas_ss_size;
		unsigned int			sas_ss_flags;

		struct callback_head		*task_works;
	.....
	}


信号处理
^^^^^^^^^
我们以系统调用kill为例进行说明：
kill --> group_send_sig_info() --> send_signal() --> __send_signal()


TIF_SIGPENDING

信号状态图
^^^^^^^^^


信号处理时机
^^^^^^^^^^^^


这部分整理，不要继续重复基础部分，仅按当前框架完善。

