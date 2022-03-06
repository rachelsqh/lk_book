.. Rachel's E-book documentation master file, created by
   sphinx-quickstart on Sun Jan  9 16:38:00 2022.
   You can adapt this file completely to your liking, but it should at least
   contain the root `toctree` directive.

linux 内核跟踪调试:trace
--------------------------
我们对架构的理解是弱点。trace如何组织？以什么为纲领？
trace就是以debugfs为依托利用kprobe,uprobe,ftrace,bpf,perf event等提供的功能为用户提供一个统一的调试界面，方便对内核和应用程序的跟踪、调试和性能测试（这个待定）。

kernel/trace/trace.c
^^^^^^^^^^^^^^^^^^^^^^^
基于环形缓冲区的函数跟踪器

/*
 * On boot up, the ring buffer is set to the minimum size, so that
 * we do not waste memory on systems that are not using tracing.
 */
bool ring_buffer_expanded;

/*
 * We need to change this state when a selftest is running.
 * A selftest will lurk into the ring-buffer to count the
 * entries inserted during the selftest although some concurrent
 * insertions into the ring-buffer such as trace_printk could occurred
 * at the same time, giving false positive or negative results.
 */
static bool __read_mostly tracing_selftest_running;

/*
 * If boot-time tracing including tracers/events via kernel cmdline
 * is running, we do not want to run SELFTEST.
 */
bool __read_mostly tracing_selftest_disabled;




结构
/* Pipe tracepoints to printk */
struct trace_iterator *tracepoint_print_iter;


int register_ftrace_export(struct trace_export *export)
int unregister_ftrace_export(struct trace_export *export)






















































