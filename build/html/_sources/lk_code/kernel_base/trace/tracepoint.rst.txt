linux内核跟踪点
--------------

本文档介绍 Linux 内核跟踪点及其使用。它提供了如何在内核中插入跟踪点并将探测函数连接到它们的示例，并提供了一些探测函数的示例。

应用场景
^^^^^^^^^^^
放置在代码中的跟踪点提供了一个挂钩来调用您可以在运行时提供的函数（探针）。跟踪点可以是“开启”（探针连接到它）或“关闭”（没有连接探针）。当跟踪点“关闭”时，它没有任何影响，除了添加微小的时间损失（检查分支的条件）和空间损失（在检测函数的末尾为函数调用添加几个字节并添加数据结构在一个单独的部分）。当跟踪点“打开”时，每次执行跟踪点时都会在调用者的执行上下文中调用您提供的函数。当提供的函数结束执行时，它返回给调用者（从跟踪点站点继续）。可以在代码中的重要位置放置跟踪点。它们是轻量级的钩子，可以传递任意数量的参数，这些参数的原型在头文件中的跟踪点声明中描述。

原理描述
^^^^^^^^^^^
跟踪点需要两个元素：
- 跟踪点定义，放置在头文件中。

- 跟踪点语句，在 C 代码中。

为了使用跟踪点，您应该包含 linux/tracepoint.h。

在 include/trace/events/subsys.h 中：


#undef TRACE_SYSTEM
#define TRACE_SYSTEM subsys

#if !defined(_TRACE_SUBSYS_H) || defined(TRACE_HEADER_MULTI_READ)
#define _TRACE_SUBSYS_H

#include <linux/tracepoint.h>

DECLARE_TRACE(subsys_eventname,
        TP_PROTO(int firstarg, struct task_struct *p),
        TP_ARGS(firstarg, p));

#endif /* _TRACE_SUBSYS_H */

/* This part must be outside protection */
#include <trace/define_trace.h>



在 subsys/file.c 中（必须添加跟踪语句）：

#include <trace/events/subsys.h>

#define CREATE_TRACE_POINTS
DEFINE_TRACE(subsys_eventname);

void somefct(void)
{
        ...
        trace_subsys_eventname(arg, task);
        ...
}


- subsys_eventname 是您的事件唯一的标识符

	- subsys 是您的子系统的名称。

	- eventname 是要跟踪的事件的名称。

- TP_PROTO(int firstarg, struct task_struct *p)是此跟踪点调用的函数的原型。

- TP_ARGS(firstarg, p)是参数名称，与原型中的相同。

- 如果在多个源文件中使用标题，#define CREATE_TRACE_POINTS 应该只出现在一个源文件中。

将函数（探针）连接到跟踪点是通过 register_trace_subsys_eventname() 为特定跟踪点提供探针（要调用的函数）来完成的。删除探针是通过 unregister_trace_subsys_eventname(); 它将移除探头。

tracepoint_synchronize_unregister() 必须在模块退出函数结束之前调用，以确保没有调用者使用探针。这一点，以及在探测调用周围禁用抢占的事实，确保探测移除和模块卸载是安全的。

跟踪点机制支持插入同一跟踪点的多个实例，但必须在整个内核中对给定的跟踪点名称进行单一定义，以确保不会发生类型冲突。跟踪点的名称修改是使用原型完成的，以确保键入正确。编译器在注册站点验证探针类型的正确性。跟踪点可以放在内联函数、内联静态函数、展开循环以及常规函数中。

此处建议命名方案“subsys_event”作为旨在限制冲突的约定。跟踪点名称对内核来说是全局的：无论它们是在核心内核映像中还是在模块中，它们都被认为是相同的。

如果必须在内核模块中使用跟踪点，可以使用 EXPORT_TRACEPOINT_SYMBOL_GPL() 或 EXPORT_TRACEPOINT_SYMBOL() 来导出定义的跟踪点。

如果您需要为跟踪点参数做一些工作，并且该工作仅用于跟踪点，则可以使用以下内容将该工作封装在 if 语句中：


if (trace_foo_bar_enabled()) {
        int i;
        int tot = 0;

        for (i = 0; i < count; i++)
                tot += calculate_nuggets();

        trace_foo_bar(tot);
}

所有 trace_<tracepoint>() 调用都定义了一个匹配的 trace_<tracepoint>_enabled() 函数，如果启用了跟踪点，则返回 true，否则返回 false。trace_<tracepoint>() 应该始终在 if (trace_<tracepoint>_enabled()) 的块内，以防止启用跟踪点和看到检查之间的竞争。

使用 trace_<tracepoint>_enabled() 的好处是它使用了 tracepoint 的 static_key 来允许使用跳转标签来实现 if 语句并避免条件分支。


注意：便利宏 TRACE_EVENT 提供了另一种定义跟踪点的方法。查看http://lwn.net/Articles/379903、 http://lwn.net/Articles/381064和http://lwn.net/Articles/383362 以获取更详细的系列文章。



如果您需要从头文件调用跟踪点，不建议直接调用或使用 trace_<tracepoint>_enabled() 函数调用，因为如果文件中包含头文件，则头文件中的跟踪点可能会产生副作用设置了 CREATE_TRACE_POINTS 以及 trace_<tracepoint>() 不是那么小，如果被其他内联函数使用，可能会使内核膨胀。相反，包括 tracepoint-defs.h 并使用 tracepoint_enabled()。

在 C 文件中：

void do_trace_foo_bar_wrapper(args)
{
        trace_foo_bar(args);
}


在头文件中：


DECLARE_TRACEPOINT(foo_bar);

static inline void some_inline_function()
{
        [..]
        if (tracepoint_enabled(foo_bar))
                do_trace_foo_bar_wrapper(args);
        [..]
}


总结：
^^^^^^^
静态插入点流程：
































