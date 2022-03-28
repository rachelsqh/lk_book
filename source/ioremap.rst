ioremap实现原理
^^^^^^^^^^^^^^^^^
/**
 * ioremap     -   map bus memory into CPU space
 * @phys_addr:    bus address of the memory
 * @size:      size of the resource to map
 *
 * ioremap performs a platform specific sequence of operations to
 * make bus memory CPU accessible via the readb/readw/readl/writeb/
 * writew/writel functions and the other mmio helpers. The returned
 * address is not guaranteed to be usable directly as a virtual
 * address.
 *
 * This version of ioremap ensures that the memory is marked uncachable
 * on the CPU as well as honouring existing caching rules from things like
 * the PCI bus. Note that there are other caches and buffers on many
 * busses. In particular driver authors should read up on PCI writes
 *
 * It's useful if some control registers are in such an area and
 * write combining or read caching is not desirable:
 *
 * Must be freed with iounmap.
 */
void __iomem *ioremap(resource_size_t phys_addr, unsigned long size)
{
	/*
	 * Ideally, this should be:
	 *	pat_enabled() ? _PAGE_CACHE_MODE_UC : _PAGE_CACHE_MODE_UC_MINUS;
	 *
	 * Till we fix all X drivers to use ioremap_wc(), we will use
	 * UC MINUS. Drivers that are certain they need or can already
	 * be converted over to strong UC can use ioremap_uc().
	 */
	enum page_cache_mode pcm = _PAGE_CACHE_MODE_UC_MINUS;

	return __ioremap_caller(phys_addr, size, pcm,
				__builtin_return_address(0), false);
}
EXPORT_SYMBOL(ioremap);


/*
 * Remap an arbitrary physical address space into the kernel virtual
 * address space. It transparently creates kernel huge I/O mapping when
 * the physical address is aligned by a huge page size (1GB or 2MB) and
 * the requested size is at least the huge page size.
 *
 * NOTE: MTRRs can override PAT memory types with a 4KB granularity.
 * Therefore, the mapping code falls back to use a smaller page toward 4KB
 * when a mapping range is covered by non-WB type of MTRRs.
 *
 * NOTE! We need to allow non-page-aligned mappings too: we will obviously
 * have to convert them into an offset in a page-aligned mapping, but the
 * caller shouldn't need to know that small detail.
 */
static void __iomem *
__ioremap_caller(resource_size_t phys_addr, unsigned long size,
		 enum page_cache_mode pcm, void *caller, bool encrypted)
{
	unsigned long offset, vaddr;
	resource_size_t last_addr;
	const resource_size_t unaligned_phys_addr = phys_addr;
	const unsigned long unaligned_size = size;
	struct ioremap_desc io_desc;
	struct vm_struct *area;
	enum page_cache_mode new_pcm;
	pgprot_t prot;
	int retval;
	void __iomem *ret_addr;

	/* Don't allow wraparound or zero size */
	last_addr = phys_addr + size - 1;
	if (!size || last_addr < phys_addr)
		return NULL;

	if (!phys_addr_valid(phys_addr)) {
		printk(KERN_WARNING "ioremap: invalid physical address %llx\n",
		       (unsigned long long)phys_addr);
		WARN_ON_ONCE(1);
		return NULL;
	}

	__ioremap_check_mem(phys_addr, size, &io_desc);

	/*
	 * Don't allow anybody to remap normal RAM that we're using..
	 */
	if (io_desc.flags & IORES_MAP_SYSTEM_RAM) {
		WARN_ONCE(1, "ioremap on RAM at %pa - %pa\n",
			  &phys_addr, &last_addr);
		return NULL;
	}

	/*
	 * Mappings have to be page-aligned
	 */
	offset = phys_addr & ~PAGE_MASK;
	phys_addr &= PHYSICAL_PAGE_MASK;
	size = PAGE_ALIGN(last_addr+1) - phys_addr;

	retval = memtype_reserve(phys_addr, (u64)phys_addr + size,
						pcm, &new_pcm);
	if (retval) {
		printk(KERN_ERR "ioremap memtype_reserve failed %d\n", retval);
		return NULL;
	}

	if (pcm != new_pcm) {
		if (!is_new_memtype_allowed(phys_addr, size, pcm, new_pcm)) {
			printk(KERN_ERR
		"ioremap error for 0x%llx-0x%llx, requested 0x%x, got 0x%x\n",
				(unsigned long long)phys_addr,
				(unsigned long long)(phys_addr + size),
				pcm, new_pcm);
			goto err_free_memtype;
		}
		pcm = new_pcm;
	}

	/*
	 * If the page being mapped is in memory and SEV is active then
	 * make sure the memory encryption attribute is enabled in the
	 * resulting mapping.
	 */
	prot = PAGE_KERNEL_IO;
	if ((io_desc.flags & IORES_MAP_ENCRYPTED) || encrypted)
		prot = pgprot_encrypted(prot);

	switch (pcm) {
	case _PAGE_CACHE_MODE_UC:
	default:
		prot = __pgprot(pgprot_val(prot) |
				cachemode2protval(_PAGE_CACHE_MODE_UC));
		break;
	case _PAGE_CACHE_MODE_UC_MINUS:
		prot = __pgprot(pgprot_val(prot) |
				cachemode2protval(_PAGE_CACHE_MODE_UC_MINUS));
		break;
	case _PAGE_CACHE_MODE_WC:
		prot = __pgprot(pgprot_val(prot) |
				cachemode2protval(_PAGE_CACHE_MODE_WC));
		break;
	case _PAGE_CACHE_MODE_WT:
		prot = __pgprot(pgprot_val(prot) |
				cachemode2protval(_PAGE_CACHE_MODE_WT));
		break;
	case _PAGE_CACHE_MODE_WB:
		break;
	}

	/*
	 * Ok, go for it..
	 */
	area = get_vm_area_caller(size, VM_IOREMAP, caller);
	if (!area)
		goto err_free_memtype;
	area->phys_addr = phys_addr;
	vaddr = (unsigned long) area->addr;/* 这是1对1映射 */

	if (memtype_kernel_map_sync(phys_addr, size, pcm))
		goto err_free_area;

	if (ioremap_page_range(vaddr, vaddr + size, phys_addr, prot))
		goto err_free_area;

	ret_addr = (void __iomem *) (vaddr + offset);
	mmiotrace_ioremap(unaligned_phys_addr, unaligned_size, ret_addr);

	/*
	 * Check if the request spans more than any BAR in the iomem resource
	 * tree.
	 */
	if (iomem_map_sanity_check(unaligned_phys_addr, unaligned_size))
		pr_warn("caller %pS mapping multiple BARs\n", caller);

	return ret_addr;
err_free_area:
	free_vm_area(area);
err_free_memtype:
	memtype_free(phys_addr, phys_addr + size);
	return NULL;
}

这部分与vmalloc的关系

