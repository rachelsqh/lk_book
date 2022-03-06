bpf 内核辅助函数定义原理
---------------------
#define BPF_CALL_x(x, name, ...)					       \
	static __always_inline						       \
	u64 ____##name(__BPF_MAP(x, __BPF_DECL_ARGS, __BPF_V, __VA_ARGS__));   \
	typedef u64 (*btf_##name)(__BPF_MAP(x, __BPF_DECL_ARGS, __BPF_V, __VA_ARGS__)); \
	u64 name(__BPF_REG(x, __BPF_DECL_REGS, __BPF_N, __VA_ARGS__));	       \
	u64 name(__BPF_REG(x, __BPF_DECL_REGS, __BPF_N, __VA_ARGS__))	       \
	{								       \
		return ((btf_##name)____##name)(__BPF_MAP(x,__BPF_CAST,__BPF_N,__VA_ARGS__));\
	}								       \
	static __always_inline						       \
	u64 ____##name(__BPF_MAP(x, __BPF_DECL_ARGS, __BPF_V, __VA_ARGS__))

#define BPF_CALL_0(name, ...)	BPF_CALL_x(0, name, __VA_ARGS__)


eg:
BPF_CALL_2(bpf_map_lookup_elem, struct bpf_map *, map, void *, key)

所以其实是定义的内联函数。为什么内核模块编译的时候只能用这些函数？我的理解是不是有很大的问题？
BPF技术虽然强大，但是为了保证内核的处理安全和及时响应，内核对于BPF 技术也给予了诸多限制，如下是几个重点限制：

- eBPF 程序不能调用任意的内核参数，只限于内核模块中列出的 BPF Helper 函数，函数支持列表也随着内核的演进在不断增加
- eBPF程序不允许包含无法到达的指令，防止加载无效代码，延迟程序的终止
- eBPF 程序中循环次数限制且必须在有限时间内结束
- eBPF 堆栈大小被限制在 MAXBPFSTACK，截止到内核 Linux 5.8 版本，被设置为 512。目前没有计划增加这个限制，解决方法是改用 BPF Map，它的大小是无限的。
- eBPF 字节码大小最初被限制为 4096 条指令，截止到内核 Linux 5.8 版本， 当前已将放宽至 100 万指令（ BPF_COMPLEXITY_LIMIT_INSNS），对于无权限的BPF程序，仍然保留4096条限制 ( BPF_MAXINSNS )
更多相关信息可以查看这里。随着技术的发展和演进，限制也在逐步放宽或者提供了对应的解决方案。


如何实现函数限制？

demo:/usr/src/bpf_trace_kprobe_demo:

#include <linux/skbuff.h>
#include <linux/netdevice.h>
#include <uapi/linux/bpf.h>
#include <linux/version.h>
#include <bpf/bpf_helpers.h>
#include <bpf/bpf_tracing.h>

#define _(P)                                                                   \
	({                                                                     \
		typeof(P) val = 0;                                             \
		bpf_probe_read_kernel(&val, sizeof(val), &(P));                \
		val;                                                           \
	})

/* kprobe is NOT a stable ABI
 * kernel functions can be removed, renamed or completely change semantics.
 * Number of arguments and their positions can change, etc.
 * In such case this bpf+kprobe example will no longer be meaningful
 */
SEC("kprobe/__netif_receive_skb_core")
int bpf_prog1(struct pt_regs *ctx)
{
	/* attaches to kprobe __netif_receive_skb_core,
	 * looks for packets on loobpack device and prints them
	 */
	char devname[IFNAMSIZ];
	struct net_device *dev;
	struct sk_buff *skb;
	int len;

	/* non-portable! works for the given kernel only */
	bpf_probe_read_kernel(&skb, sizeof(skb), (void *)PT_REGS_PARM1(ctx));
	dev = _(skb->dev);
	len = _(skb->len);

	bpf_probe_read_kernel(devname, sizeof(devname), dev->name);

	if (devname[0] == 'l' && devname[1] == 'o') {
		char fmt[] = "skb %p len %d\n";
		/* using bpf_trace_printk() for DEBUG ONLY */
		bpf_trace_printk(fmt, sizeof(fmt), skb, len);
	}

	return 0;
}

char _license[] SEC("license") = "GPL";
u32 _version SEC("version") = LINUX_VERSION_CODE;
















生成的内核模块只是架构：readelf -a ..

root@rachel:/usr/src/bpf_trace_kprobe_demo# readelf -a trace_kprobe_kern.o
ELF 头：
  Magic：  7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00 
  类别:                              ELF64
  数据:                              2 补码，小端序 (little endian)
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI 版本:                          0
  类型:                              REL (可重定位文件)
  系统架构:                          Linux BPF
  版本:                              0x1
  入口点地址：              0x0
  程序头起点：              0 (bytes into file)
  Start of section headers:          616 (bytes into file)
  标志：             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           0 (bytes)
  Number of program headers:         0
  Size of section headers:           64 (bytes)
  Number of section headers:         9
  Section header string table index: 1



nm ....:
0000000000000000 T bpf_prog
0000000000000000 D _license
0000000000000000 T prog

另外一个demo:
root@rachel:/usr/src/bpf_kprobe_bac# nm trace_kprobe_kern.o
0000000000000000 D _license
0000000000000000 T prog

所以我们发现了什么？对的那个宏定义还是有什么不对的地方。


code:

#include <bpf/bpf_tracing.h>
#include "vmlinux.h"
#include <bpf/bpf_helpers.h>
#ifndef SEC
#define SEC(NAME) __attribute__((section(NAME),used))
#endif
SEC("kprobe/do_unlinkat")
int prog(struct pt_regs *ctx)
{

	char file_name[] = "comm = %s\n";
	char buf_comm[16];
	bpf_get_current_comm(buf_comm,sizeof(buf_comm));

	
	bpf_trace_printk(file_name,sizeof(file_name),buf_comm);
	return 0;
}

char _license[] SEC("license") = "GPL";













llvm-objdump -d trace_kprobe_kern.o

trace_kprobe_kern.o:    file format elf64-bpf


Disassembly of section kprobe/do_unlinkat:

0000000000000000 <prog>:
       0:       b7 01 00 00 73 0a 00 00 r1 = 2675
       1:       6b 1a f8 ff 00 00 00 00 *(u16 *)(r10 - 8) = r1
       2:       18 01 00 00 63 6f 6d 6d 00 00 00 00 20 3d 20 25 r1 = 2675205388142210915 ll
       4:       7b 1a f0 ff 00 00 00 00 *(u64 *)(r10 - 16) = r1
       5:       b7 01 00 00 00 00 00 00 r1 = 0
       6:       73 1a fa ff 00 00 00 00 *(u8 *)(r10 - 6) = r1
       7:       bf a6 00 00 00 00 00 00 r6 = r10
       8:       07 06 00 00 e0 ff ff ff r6 += -32
       9:       bf 61 00 00 00 00 00 00 r1 = r6
      10:       b7 02 00 00 10 00 00 00 r2 = 16
      11:       85 00 00 00 10 00 00 00 call 16
      12:       bf a1 00 00 00 00 00 00 r1 = r10
      13:       07 01 00 00 f0 ff ff ff r1 += -16
      14:       b7 02 00 00 0b 00 00 00 r2 = 11
      15:       bf 63 00 00 00 00 00 00 r3 = r6
      16:       85 00 00 00 06 00 00 00 call 6
      17:       b7 00 00 00 00 00 00 00 r0 = 0
      18:       95 00 00 00 00 00 00 00 exit

我们现在补充bpf汇编的语法：
原始的BPF又称之为class BPF(cBPF), BPF与eBPF类似于i386与amd64的关系, 最初的BPF只能用于套接字的过滤,内核源码树中tools/bpf/bpf_asm可以用于编写这种原始的BPF程序,

eBPF虚拟机
eBPF虚拟机是一个RISC指令, 带有寄存器的虚拟机, 内部有11个64位寄存器, 一个程序计数器(PC), 以及一个512字节的固定大小的栈. 9个通用寄存器可以读写, 一个是只能读的栈指针寄存器(SP), 以及一个隐含的程序计数器, 我们只能根据PC进行固定偏移的跳转. 虚拟机寄存器总是64位的(就算是32位物理机也是这样的), 并且支持32位子寄存器寻址(寄存器高32位自动设置为0)


r0: 保存函数调用和当前程序退出的返回值
r1~r5: 作为函数调用参数, 当程序开始运行时, r1包含一个指向context参数的指针
r6~r9: 在内核函数调用之间得到保留
r10: 只读的指向512字节栈的栈指针





注意内核地址与BPF地址对应。

先补充BPF汇编。后面补充虚拟机。


























