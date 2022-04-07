vmalloc分析
^^^^^^^^^^^^^^
kmalloc返回的内存是物理的，连续的，更适合于类似设备驱动的程序来使用。 但vmalloc能使用更多的资源，因为vmalloc还可以处理交换空间。 kmalloc对应于kfree，可以分配连续的物理内存； vmalloc对应于vfree，分配连续的虚拟内存，但是物理上不一定连续。

vmalloc --> alloc_page -->更新到页表：
进行页表录页表项的设置， mm_struct的pgd->p4d->pud->pmd->pte逐层填充, 如果页表项涉及变化通过`arch_sync_kernel_mappings(start, end);进行更新

vmalloc使用SLUB使用伙伴系统的申请的是其结构信息, 物理页面是通过伙伴系统申请, 并完成映射。

在最近的版本中， 已经将vmalloc缺页中断处理移除， 通过XXX_alloc_track跟踪页目录及页表项变化，通过 arch_sync_kernel_mappings在申请和释放阶段同步通知所有页表
参见:x86/mm: remove vmalloc faulting

X86-64 处理如下，将全局pgd同步到每个建立映射的物理页面.



在Linux中，伙伴系统（buddy system）是以页为单位管理和分配内存。但是现实的需求却以字节为单位，假如我们需要申请20Bytes，总不能分配一页吧！那岂不是严重浪费内存。那么该如何分配呢？slab分配器就应运而生了，专为小内存分配而生。slab分配器分配内存以Byte为单位。但是slab分配器并没有脱离伙伴系统，而是基于伙伴系统分配的大内存进一步细分成小内存分配。

前段时间学习了下slab分配器工作原理。因为自己本身是做手机的，发现现在好像都在使用slub分配器，想想还是再研究一下slub的工作原理。之前看了代码，感觉挺多数据结构和成员的。成员的意思是什么？数据结构之间的关系是什么？不知道你是否感觉云里雾里。既然代码阅读起来晦涩难懂，如果有精美的配图，不知是否有助于阁下理解slub的来龙去脉呢？我想表达的意思就是文章图多，图多，图多。我们只说原理，尽量不看代码。因为所有代码中包含的内容我都会用图来说明。你感兴趣绝对有助于你看代码。

说明：slub是slab中的一种，slab也是slab中的一种。有时候用slab来统称slab, slub和slob。slab, slub和slob仅仅是分配内存策略不同。本篇文章中说的是slub分配器工作的原理。但是针对分配器管理的内存，下文统称为slab缓存池。所以文章中slub和slab会混用，表示同一个意思。

注：文章代码分析基于linux-4.15.0-rc3。 图片有点走形，请单独点开图片查看。


　kmalloc() 申请的内存位于物理内存映射区域，而且在物理上也是连续的，它们与真实的物理地址只有一个固定的偏移，因为存在较简单的转换关系，所以对申请的内存大小有限制，不能超过128KB。 
　　 
较常用的 flags（分配内存的方法）：

GFP_ATOMIC —— 分配内存的过程是一个原子过程，分配内存的过程不会被（高优先级进程或中断）打断；
GFP_KERNEL —— 正常分配内存；
GFP_DMA —— 给 DMA 控制器分配内存，需要使用该标志（DMA要求分配虚拟地址和物理地址连续）。
flags 的参考用法： 
　|– 进程上下文，可以睡眠　　　　　GFP_KERNEL 
　|– 进程上下文，不可以睡眠　　　　GFP_ATOMIC 
　|　　|– 中断处理程序　　　　　　　GFP_ATOMIC 
　|　　|– 软中断　　　　　　　　　　GFP_ATOMIC 
　|　　|– Tasklet　　　　　　　　　GFP_ATOMIC 
　|– 用于DMA的内存，可以睡眠　　　GFP_DMA | GFP_KERNEL 
　|– 用于DMA的内存，不可以睡眠　　GFP_DMA |GFP_ATOMIC 
　　 
对应的内存释放函数为：

void kfree(const void *objp);
 

kzalloc()
　　kzalloc() 函数与 kmalloc() 非常相似，参数及返回值是一样的，可以说是前者是后者的一个变种，因为 kzalloc() 实际上只是额外附加了 __GFP_ZERO 标志。所以它除了申请内核内存外，还会对申请到的内存内容清零。

/** * kzalloc - allocate memory. The memory is set to zero. * @size: how many bytes of memory are required. * @flags: the type of memory to allocate (see kmalloc). */
static inline void *kzalloc(size_t size, gfp_t flags)
{
    return kmalloc(size, flags | __GFP_ZERO);
}
kzalloc() 对应的内存释放函数也是 kfree()。

 

vmalloc()
函数原型：

void *vmalloc(unsigned long size);
　　vmalloc() 函数则会在虚拟内存空间给出一块连续的内存区，但这片连续的虚拟内存在物理内存中并不一定连续。由于 vmalloc() 没有保证申请到的是连续的物理内存，因此对申请的内存大小没有限制，如果需要申请较大的内存空间就需要用此函数了。

对应的内存释放函数为：

void vfree(const void *addr);
注意：vmalloc() 和 vfree() 可以睡眠，因此不能从中断上下文调用。 

 

总结
kmalloc()、kzalloc()、vmalloc() 的共同特点是：

用于申请内核空间的内存；
内存以字节为单位进行分配；
所分配的内存虚拟地址上连续；
kmalloc()、kzalloc()、vmalloc() 的区别是：

kzalloc 是强制清零的 kmalloc 操作；（以下描述不区分 kmalloc 和 kzalloc）
kmalloc 分配的内存大小有限制（128KB），而 vmalloc 没有限制；
kmalloc 可以保证分配的内存物理地址是连续的，但是 vmalloc 不能保证；
kmalloc 分配内存的过程可以是原子过程（使用 GFP_ATOMIC），而 vmalloc 分配内存时则可能产生阻塞；
kmalloc 分配内存的开销小，因此 kmalloc 比 vmalloc 要快；
一般情况下，内存只有在要被 DMA 访问的时候才需要物理上连续，但为了性能上的考虑，内核中一般使用 kmalloc()，而只有在需要获得大块内存时才使用 vmalloc()。例如，当模块被动态加载到内核当中时，就把模块装载到由 vmalloc() 分配的内存上。

















需要有一种手段可以分配非连续的页映射到一段连续的虚拟地址上. 这就是vmalloc的目的,接着就是32TB的vmalloc空间,以VMALLOC_START(64TB+1TB)开始,VMALLOC_END(64TB+1TB+32TB)结尾.非连续的页


/* bits in flags of vmalloc's vm_struct below */
#define VM_IOREMAP		0x00000001	/* ioremap() and friends */
#define VM_ALLOC		0x00000002	/* vmalloc() */
#define VM_MAP			0x00000004	/* vmap()ed pages */
#define VM_USERMAP		0x00000008	/* suitable for remap_vmalloc_range */
#define VM_DMA_COHERENT		0x00000010	/* dma_alloc_coherent */
#define VM_UNINITIALIZED	0x00000020	/* vm_struct is not fully initialized */
#define VM_NO_GUARD		0x00000040      /* don't add guard page */
#define VM_KASAN		0x00000080      /* has allocated kasan shadow memory */
#define VM_FLUSH_RESET_PERMS	0x00000100	/* reset direct map and flush TLB on unmap, can't be freed in atomic context */
#define VM_MAP_PUT_PAGES	0x00000200	/* put pages and free array in vfree */
#define VM_NO_HUGE_VMAP		0x00000400	/* force PAGE_SIZE pte mapping */

/*
 * VM_KASAN is used slightly differently depending on CONFIG_KASAN_VMALLOC.
 *
 * If IS_ENABLED(CONFIG_KASAN_VMALLOC), VM_KASAN is set on a vm_struct after
 * shadow memory has been mapped. It's used to handle allocation errors so that
 * we don't try to poison shadow on free if it was never allocated.
 *
 * Otherwise, VM_KASAN is set for kasan_module_alloc() allocations and used to
 * determine which allocations need the module shadow freed.
 */

/* bits [20..32] reserved for arch specific ioremap internals */

/*
 * Maximum alignment for ioremap() regions.
 * Can be overridden by arch-specific value.
 */
#ifndef IOREMAP_MAX_ORDER
#define IOREMAP_MAX_ORDER	(7 + PAGE_SHIFT)	/* 128 pages */
#endif

struct vm_struct {/* 主要数据结构 */
	struct vm_struct	*next;
	void			*addr;
	unsigned long		size;
	unsigned long		flags;
	struct page		**pages;
#ifdef CONFIG_HAVE_ARCH_HUGE_VMALLOC
	unsigned int		page_order;
#endif
	unsigned int		nr_pages;
	phys_addr_t		phys_addr;
	const void		*caller;
};

struct vmap_area {
	unsigned long va_start;
	unsigned long va_end;

	struct rb_node rb_node;         /* address sorted rbtree */
	struct list_head list;          /* address sorted list */

	/*
	 * The following two variables can be packed, because
	 * a vmap_area object can be either:
	 *    1) in "free" tree (root is vmap_area_root)
	 *    2) or "busy" tree (root is free_vmap_area_root)
	 */
	union {
		unsigned long subtree_max_size; /* in "free" tree */
		struct vm_struct *vm;           /* in "busy" tree */
	};
};

