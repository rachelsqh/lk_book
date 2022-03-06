内核测试
-------
kUnit(KUnit-Linux Kernel Unit Testing)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
是一个完全在内核中的系统，用于“白盒”测试：因为测试代码是内核的一部分，它可以访问不暴露给用户空间的内部结构和函数。

因此，KUnit 测试最好针对内核的小而独立的部分编写，这些部分可以单独进行测试。这与“单元”测试的概念非常吻合。
例如，KUnit 测试可能会测试单个内核函数（甚至是通过函数的单个代码路径，例如错误处理案例），而不是整个特性。
这也使得 KUnit 测试的构建和运行速度非常快，允许它们作为开发过程的一部分频繁运行。



kselftest(Linux Kernel Selftests)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
主要是在用户空间中实现的，测试是普通的用户空间脚本或程序。
这使得编写更复杂的测试或需要更多地操纵整个系统状态的测试（例如，生成进程等）变得更容易。但是，不能直接从 kselftest 调用内核函数。这意味着只有以某种方式（例如通过系统调用、设备、文件系统等）暴露给用户空间的内核功能才能使用 kselftest 进行测试。为了解决这个问题，一些测试包括一个配套的内核模块，它公开了更多的信息或功能。但是，如果测试大部分或完全在内核中运行，KUnit 可能是更合适的工具。

因此，kselftest 非常适合测试整个功能，因为这些将向用户空间公开一个接口，该接口可以测试，但不能测试实现细节。这与“系统”或“端到端”测试非常吻合。
例如，所有新的系统调用都应该伴随着 kselftest 测试。


代码覆盖性测试工具
^^^^^^^^^^^^^^^^
gcov
"""""
kcov
"""""

动态分析工具
^^^^^^^^^^^^^
内核还支持许多动态分析工具，当它们出现在正在运行的内核中时，它们会尝试检测问题类别。这些通常会寻找不同类别的错误，例如无效的内存访问、并发问题（例如数据竞争）或其他未定义的行为（例如整数溢出）。


这些工具倾向于将内核作为一个整体进行测试，而不是像 kselftest 或 KUnit 测试那样“通过”。通过在启用这些工具的内核上运行测试，它们可以与 KUnit 或 kselftest 结合使用：然后您可以确保在测试期间不会发生这些错误。
其中一些工具与 KUnit 或 kselftest 集成，如果检测到问题，将自动失败测试。

工具示例
""""""""
- kmemleak:检测可能的内存泄漏.https://www.kernel.org/doc/html/latest/dev-tools/kmemleak.html
- KASAN: 检测无效的内存访问，例如越界和释放后使用错误.https://www.kernel.org/doc/html/latest/dev-tools/kasan.html
- UBSAN 检测 C 标准未定义的行为，例如整数溢出。https://www.kernel.org/doc/html/latest/dev-tools/ubsan.html
- KCSAN 检测数据竞争。请参阅内核并发清理程序 (KCSAN)https://www.kernel.org/doc/html/latest/dev-tools/kcsan.html
- KFENCE 是一种内存问题的低开销检测器，它比 KASAN 快得多，并且可以在生产中使用。参见内核电子围栏 (KFENCE)https://www.kernel.org/doc/html/latest/dev-tools/kfence.html
- lockdep 是一个锁定正确性验证器。请参阅 运行时锁定正确性验证器.https://www.kernel.org/doc/html/latest/locking/lockdep-design.html
- 内核中还有其他几个调试工具，其中许多可以在 lib/Kconfig.debug 中找到.


linux 内核跟踪
^^^^^^^^^^^^^^^

/sys/kernel/debug


ftrace:/sys/kernel/debug/tracing
""""""""""""""""""""""""""""""""

ftrace配置说明总结：/sys/kernel/debug/tracing/README
""""""""""""""""""""""""""""""""""""""""""""""""""""


原文：
********

.. code-block:: c
   :caption: /sys/kernel/debug/tracing/README
   :emphasize-lines: 4,5
   :linenos:

    tracing mini-HOWTO:

	# echo 0 > tracing_on : quick way to disable tracing
	# echo 1 > tracing_on : quick way to re-enable tracing

 	Important files:
 	 	trace			- The static contents of the buffer
				 	 To clear the buffer write into this file: echo > trace
  	 	trace_pipe		- A consuming read to see the contents of the buffer
  	 	current_tracer		- function and latency tracers
  	 	available_tracers	- list of configured tracers for current_tracer
  	 	error_log		- error log for failed commands (that support it)
  	 	buffer_size_kb		- view and modify size of per cpu buffer
  	 	buffer_total_size_kb  	- view total size of all cpu buffers

	trace_clock		-change the clock used to order events
       		local:   	Per cpu clock but may not be synced across CPUs
      		global:   	Synced across CPUs but slows tracing down.
     		counter:   	Not a clock, but just an increment
      		uptime:   	Jiffy counter from time of boot
        	perf:   	Same clock that perf events use
     		x86-tsc:   	TSC cycle counter

  	timestamp_mode	-view the mode used to timestamp events
       		delta:   Delta difference against a buffer-wide timestamp
    		absolute:   Absolute (standalone) timestamp

  	trace_marker		- Writes into this file writes into the kernel buffer

  	trace_marker_raw		- Writes into this file writes binary data into the kernel buffer
  	tracing_cpumask			- Limit which CPUs to trace
  	instances		- Make sub-buffers with: mkdir instances/foo
				  Remove sub-buffer with rmdir
  	trace_options		- Set format or modify how tracing happens
				  Disable an option by prefixing 'no' to the
				  option name
  	saved_cmdlines_size	- echo command number in here to store comm-pid list

  	available_filter_functions - list of functions that can be filtered on
  	set_ftrace_filter	- echo function name in here to only trace these
				  functions
	     accepts: func_full_name or glob-matching-pattern
	     modules: Can select a group via module
	     Format: :mod:<module-name>
	     example: echo :mod:ext3 > set_ftrace_filter
	     triggers: a command to perform when function is hit
	     Format: <function>:<trigger>[:count]
	     trigger: traceon, traceoff
		      enable_event:<system>:<event>
		      disable_event:<system>:<event>
		      stacktrace
		      snapshot
		      dump
		      cpudump
	     example: echo do_fault:traceoff > set_ftrace_filter
	              echo do_trap:traceoff:3 > set_ftrace_filter
	     The first one will disable tracing every time do_fault is hit
	     The second will disable tracing at most 3 times when do_trap is hit
	       The first time do trap is hit and it disables tracing, the
	       counter will decrement to 2. If tracing is already disabled,
	       the counter will not decrement. It only decrements when the
	       trigger did work
	     To remove trigger without count:
	       echo '!<function>:<trigger> > set_ftrace_filter
	     To remove trigger with a count:
	       echo '!<function>:<trigger>:0 > set_ftrace_filter
  
	set_ftrace_notrace	- echo function name in here to never trace.
	    accepts: func_full_name, *func_end, func_begin*, *func_middle*
	    modules: Can select a group via module command :mod:
	    	Does not accept triggers
  
  	set_ftrace_pid		- Write pid(s) to only function trace those pids
			 	   (function)
  	set_ftrace_notrace_pid		- Write pid(s) to not function trace those pids
				    (function)
  	set_graph_function	- Trace the nested calls of a function (function_graph)
  	set_graph_notrace	- Do not trace the nested calls of a function (function_graph)
  	max_graph_depth		- Trace a limited depth of nested calls (0 is unlimited)

  	snapshot		- Like 'trace' but shows the content of the static
				  snapshot buffer. Read the contents for more
				  information
  	stack_trace		- Shows the max stack trace when active
  	stack_max_size	- Shows current max stack size that was traced
				  Write into this file to reset the max size (trigger a
				  new trace)
 	stack_trace_filter	- Like set_ftrace_filter but limits what stack_trace
				  traces
  	dynamic_events		- Create/append/remove/show the generic dynamic events
				  Write into this file to define/undefine new trace events.
  	kprobe_events		- Create/append/remove/show the kernel dynamic events
				  Write into this file to define/undefine new trace events.
  	uprobe_events		- Create/append/remove/show the userspace dynamic events
				  Write into this file to define/undefine new trace events.
		accepts: event-definitions (one definition per line)
	   	Format: p[:[<group>/]<event>] <place> [<args>]
	       	    r[maxactive][:[<group>/]<event>] <place> [<args>]
	        	   -:[<group>/]<event>
	    	place: [<module>:]<symbol>[+<offset>]|<memaddr>
		place (kretprobe): [<module>:]<symbol>[+<offset>]%return|<memaddr>
   		place (uprobe): <path>:<offset>[%return][(ref_ctr_offset)]
	     	args: <name>=fetcharg[:type]
	 	fetcharg: %<register>, @<address>, @<symbol>[+|-<offset>],
	 	          $stack<index>, $stack, $retval, $comm, $arg<N>,
		           +|-[u]<offset>(<fetcharg>), \imm-value, \"imm-string"
	     	type: s8/16/32/64, u8/16/32/64, x8/16/32/64, string, symbol,
	        	   b<bit-width>@<bit-offset>/<container-size>, ustring,
	       	    <type>\[<array-size>\]
  	
  	events/		- Directory containing all trace event subsystems:
      		enable		- Write 0/1 to enable/disable tracing of all events
  	events/<system>/	- Directory containing all trace events for <system>:
      		enable		- Write 0/1 to enable/disable tracing of all <system>
				  events
      		filter		- If set, only events passing filter are traced
  	events/<system>/<event>/	- Directory containing control files for <event>:
      		enable		- Write 0/1 to enable/disable tracing of <event>
      		filter		- If set, only events passing filter are traced
     		trigger		- If set, a command to perform when event is hit
	    		Format: <trigger>[:count][if <filter>]
	   		trigger: traceon, traceoff
	            		enable_event:<system>:<event>
	            		disable_event:<system>:<event>
		    		stacktrace
		    		snapshot
	   		example: echo traceoff > events/block/block_unplug/trigger
	            		echo traceoff:3 > events/block/block_unplug/trigger
	            		echo 'enable_event:kmem:kmalloc:3 if nr_rq > 1' > \
	                  		events/block/block_unplug/trigger
	   		The first disables tracing every time block_unplug is hit.
	   		The second disables tracing the first 3 times block_unplug is hit.
	   		The third enables the kmalloc event the first 3 times block_unplug
	     		is hit and has value of greater than 1 for the 'nr_rq' event field.
	   		Like function triggers, the counter is only decremented if it
	    		 enabled or disabled tracing.
	   		To remove a trigger without a count:
	     		  echo '!<trigger> > <system>/<event>/trigger
	   		To remove a trigger with a count:
	     		  echo '!<trigger>:0 > <system>/<event>/trigger
	   		Filters can be ignored when removing a trigger.

中文：
追踪迷你HOWTO：

# echo 0 > tracking_on : 禁用跟踪的快速方法

# echo 1 > tracking_on : 重新启用跟踪的快速方法

重要文件：
trace - 缓冲区的静态内容

清除缓冲区写入此文件：echo > trace

trace_pipe - 用于查看缓冲区内容的消耗性读取

current_tracer - 函数和延迟跟踪器

available_tracers - current_tracer 的已配置跟踪器列表

error_log - 失败命令的错误日志（支持它）

buffer_size_kb - 查看和修改每个 cpu 缓冲区的大小

buffer_total_size_kb - 查看所有 cpu 缓冲区的总大小

trace_clock - 更改用于排序事件的时钟

本地：每个 cpu 时钟，但可能不会跨 CPU 同步

全局：跨 CPU 同步，但会减慢跟踪速度。

计数器：不是时钟，只是一个增量

正常运行时间：从启动时间开始的 Jiffy 计数器

perf：与 perf 事件使用的时钟相同

x86-tsc：TSC 周期计数器

timestamp_mode -查看用于时间戳事件的模式

delta：与缓冲区范围时间戳的差异

absolute：绝对（独立）时间戳

trace_marker - 写入此文件写入内核缓冲区

trace_marker_raw - 写入此文件将二进制数据写入内核缓冲区

tracking_cpumask - 限制要跟踪的 CPU

实例 - 使用： mkdir instances/foo 创建子缓冲区

使用 rmdir 删除子缓冲区

trace_options - 设置格式或修改跟踪的发生方式

通过在前面加上“no”来禁用选项

选项名称

saved_cmdlines_size - 在此处回显命令号以存储 comm-pid 列表

available_filter_functions - 可以过滤的函数列表

set_ftrace_filter - 在此处回显函数名称以仅跟踪这些

职能

接受：func_full_name 或 glob-matching-pattern

模块：可以通过模块选择一个组

格式：:mod:<模块名称>

示例：echo :mod:ext3 > set_ftrace_filter

triggers：触发函数时执行的命令

格式：<函数>:<触发器>[:count]

触发器：traceon，traceoff

enable_event:<系统>:<事件>

disable_event:<系统>:<事件>

堆栈跟踪

快照

倾倒

cpu转储

示例：echo do_fault:traceoff > set_ftrace_filter

echo do_trap:traceoff:3 > set_ftrace_filter

第一个将在每次 do_fault 被命中时禁用跟踪

当 do_trap 被击中时，第二个将禁用跟踪最多 3 次

第一次 do 陷阱被击中并禁用跟踪，

计数器将递减到 2。如果跟踪已被禁用，

计数器不会递减。只有当

触发器确实有效

要删除触发器而不计数：

echo '!<function>:<trigger> > set_ftrace_filter

要删除带有计数的触发器：

echo '!<function>:<trigger>:0 > set_ftrace_filter

set_ftrace_notrace - 在此处回显函数名称以从不跟踪。

接受：func_full_name、func_end、func_begin、func_middle

模块：可以通过模块命令选择一个组：mod：

不接受触发器

set_ftrace_pid - 写入 pid(s) 以仅跟踪这些 pid

（功能）

set_ftrace_notrace_pid - 写入 pid(s) 以不跟踪这些 pid

（功能）

set_graph_function - 跟踪函数的嵌套调用 (function_graph)

set_graph_notrace - 不跟踪函数的嵌套调用 (function_graph)

max_graph_depth - 跟踪嵌套调用的有限深度（0 是无限的）

快照 - 类似于“跟踪”，但显示静态内容

快照缓冲区。阅读内容了解更多

信息

stack_trace - 活动时显示最大堆栈跟踪

stack_max_size - 显示当前跟踪的最大堆栈大小

写入此文件以重置最大大小（触发

新的痕迹）

stack_trace_filter - 与 set_ftrace_filter 类似，但限制了 stack_trace 的内容

痕迹

dynamic_events - 创建/附加/删除/显示通用动态事件

写入此文件以定义/取消定义新的跟踪事件。

kprobe_events - 创建/附加/删除/显示内核动态事件

写入此文件以定义/取消定义新的跟踪事件。

uprobe_events - 创建/附加/删除/显示用户空间动态事件

写入此文件以定义/取消定义新的跟踪事件。

接受：事件定义（每行一个定义）

格式：p[:[<group>/]<event>] <place> [<args>]

r[maxactive][:[<group>/]<event>] <place> [<args>]

-:[<组>/]<事件>

地点：[<模块>:]<符号>[+<偏移>]|<memaddr>

位置（kretprobe）：[<module>:]<symbol>[+<offset>]%return|<memaddr>

地点（uprobe）：<path>:<offset>[%return][(ref_ctr_offset)]

参数：<名称>=fetcharg[:type]

fetcharg: %<register>, @<address>, @<symbol>[+|-<offset>],

$stack<index>, $stack, $retval, $comm, $arg<N>,

+|-[u]<offset>(<fetcharg>), imm-value, "imm-string"

类型：s8/16/32/64、u8/16/32/64、x8/16/32/64、字符串、符号、

b<bit-width>@<bit-offset>/<container-size>, ustring,

<类型>[<数组大小>]

events/ - 包含所有跟踪事件子系统的目录：

enable - 写入 0/1 以启用/禁用所有事件的跟踪

events/<system>/ - 包含 <system> 的所有跟踪事件的目录：

enable - 写入 0/1 以启用/禁用所有 <system> 的跟踪

事件

filter - 如果设置，则仅跟踪通过过滤器的事件

events/<system>/<event>/ - 包含 <event> 控制文件的目录：

enable - 写入 0/1 以启用/禁用 <event> 的跟踪

filter - 如果设置，则仅跟踪通过过滤器的事件

trigger - 如果设置，当事件被触发时执行的命令

格式：<trigger>[:count][if <filter>]

触发器：traceon，traceoff

enable_event:<系统>:<事件>

disable_event:<系统>:<事件>

堆栈跟踪

快照

示例：回显跟踪 > 事件/块/block_unplug/触发器

echo traceoff:3 > 事件/块/block_unplug/触发器

echo 'enable_event:kmem:kmalloc:3 if nr_rq > 1' >

事件/块/block_unplug/触发器

第一个在每次 block_unplug 被命中时禁用跟踪。

第二个禁用跟踪前 3 次 block_unplug 被命中。

第三个启用kmalloc事件前3次block_unplug

被击中并且“nr_rq”事件字段的值大于 1。

与函数触发器一样，计数器仅在

启用或禁用跟踪。

要删除没有计数的触发器：

echo '!<trigger> > <system>/<event>/trigger

要删除带有计数的触发器：

echo '!<trigger>:0 > <system>/<event>/trigger

删除触发器时可以忽略过滤器。



ftrace实现框架及原理分析
""""""""""""""""""""""""

参考demo
"""""""""

   
   
内核补丁
^^^^^^^^^^^^

https://www.kernel.org/doc/html/latest/dev-tools/checkpatch.html

linux 内核性能测试
^^^^^^^^^^^^^^^^^
.. code-block:: c
   :caption: struct_task --> mm
   :emphasize-lines: 4,5
   :linenos:
   
   
linux 内核调试编程
^^^^^^^^^^^^^^^^^

内核模块调试
^^^^^^^^^^^^^
linux 内核测试协议
^^^^^^^^^^^^^^^^^


linux 实时分析工具：rtla
^^^^^^^^^^^^^^^^^^^^^^^^
