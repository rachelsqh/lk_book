linux 沙箱:seccomp
--------------------------


内核代码：kernel/seccomp.c

这定义了一个简单但可靠的安全计算工具，有两种实现方式：

1. 使用允许的系统调用的固定列表：SECCOMP_MODE_STRICT, 进程只能访问read,write,_exit,sigreturn系统调用。（如何手动扩充）；
2. 允许用户以 Berkeley Packet Filters/Linux Socket Filters 的形式定义系统调用过滤器。SECCOM_MODE_FILTER，通过设置bpf规则，来过滤和检查系统调用号，和系统调用参数，来决定对进程访问系统调用的处理

对系统调用的应用进行限制。

提供一个系统调用seccomp（）来实现沙箱功能。

seccomp系统调用
^^^^^^^^^^^^^^
.. code-block:: c
	:caption: struct kset_uevent_ops
	:emphasize-lines: 4,5
	:linenos:

	/* Common entry point for both prctl and syscall. */
	static long do_seccomp(unsigned int op, unsigned int flags,
		       void __user *uargs)
	{
		switch (op) {
		case SECCOMP_SET_MODE_STRICT://固定系统调用
			if (flags != 0 || uargs != NULL)
				return -EINVAL;
			return seccomp_set_mode_strict();
		case SECCOMP_SET_MODE_FILTER: //设置系统调用过滤条件
			return seccomp_set_mode_filter(flags, uargs);
		case SECCOMP_GET_ACTION_AVAIL: //获取可用的系统调用列表
			if (flags != 0)
				return -EINVAL;

			return seccomp_get_action_avail(uargs);
		case SECCOMP_GET_NOTIF_SIZES: //获取。。。。。
			if (flags != 0)
				return -EINVAL;

			return seccomp_get_notif_sizes(uargs);
		default:
			return -EINVAL;
	}
   
- 固定系统调用：
  

应用场景：

- systemd
- container

prctl

内核示例程序
^^^^^^^^^^^^^
sample/seccomp/




















   
