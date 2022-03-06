.. Rachel's E-book documentation master file, created by
   sphinx-quickstart on Sun Jan  9 16:38:00 2022.
   You can adapt this file completely to your liking, but it should at least
   contain the root `toctree` directive.

linux 底层指令替换原理
--------------------------
   
/*
 * Alternative inline assembly for SMP.
 *
 * The LOCK_PREFIX macro defined here replaces the LOCK and
 * LOCK_PREFIX macros used everywhere in the source tree.
 *
 * SMP alternatives use the same data structures as the other
 * alternatives and the X86_FEATURE_UP flag to indicate the case of a
 * UP system running a SMP kernel.  The existing apply_alternatives()
 * works fine for patching a SMP kernel for UP.
 *
 * The SMP alternative tables can be kept after boot and contain both
 * UP and SMP versions of the instructions to allow switching back to
 * SMP at runtime, when hotplugging in a new CPU, which is especially
 * useful in virtualized environments.
 *
 * The very common lock prefix is handled as special case in a
 * separate table which is a pure address list without replacement ptr
 * and size information.  That keeps the table sizes small.
 */
 
 基础结构
 ^^^^^^^^^
 
.. code-block:: c
	:caption: kobject_uevent
	:emphasize-lines: 4,5
	:linenos:
	
 struct alt_instr {
	s32 instr_offset;	/* original instruction */
	s32 repl_offset;	/* offset to replacement instruction */
	u16 cpuid;		/* cpuid bit set for replacement */
	u8  instrlen;		/* length of original instruction */
	u8  replacementlen;	/* length of new instruction */
} __packed;


API分析
^^^^^^
arch/x86/kernel/alternative.c



.. code-block:: c
	:caption: kobject_uevent
	:emphasize-lines: 4,5
	:linenos:

static const unsigned char x86nops[] =
{
	BYTES_NOP1,
	BYTES_NOP2,
	BYTES_NOP3,
	BYTES_NOP4,
	BYTES_NOP5,
	BYTES_NOP6,
	BYTES_NOP7,
	BYTES_NOP8,
};

const unsigned char * const x86_nops[ASM_NOP_MAX+1] =
{
	NULL,
	x86nops,
	x86nops + 1,
	x86nops + 1 + 2,
	x86nops + 1 + 2 + 3,
	x86nops + 1 + 2 + 3 + 4,
	x86nops + 1 + 2 + 3 + 4 + 5,
	x86nops + 1 + 2 + 3 + 4 + 5 + 6,
	x86nops + 1 + 2 + 3 + 4 + 5 + 6 + 7,
};




extern struct alt_instr __alt_instructions[], __alt_instructions_end[];
extern s32 __smp_locks[], __smp_locks_end[];




/*
 * Replace instructions with better alternatives for this CPU type. This runs
 * before SMP is initialized to avoid SMP problems with self modifying code.
 * This implies that asymmetric systems where APs have less capabilities than
 * the boot processor are not handled. Tough. Make sure you disable such
 * features by hand.
 *
 * Marked "noinline" to cause control flow change and thus insn cache
 * to refetch changed I$ lines.
 */
void __init_or_module noinline apply_alternatives(struct alt_instr *start,
						  struct alt_instr *end)





struct smp_alt_module {
	/* what is this ??? */
	struct module	*mod;
	char		*name;

	/* ptrs to lock prefixes */
	const s32	*locks;
	const s32	*locks_end;

	/* .text segment, needed to avoid patching init code ;) */
	u8		*text;
	u8		*text_end;

	struct list_head next;
};
static LIST_HEAD(smp_alt_modules);
static bool uniproc_patched = false;	/* protected by text_mutex */

void __init_or_module alternatives_smp_module_add(struct module *mod,
						  char *name,
						  void *locks, void *locks_end,






void __init_or_module alternatives_smp_module_del(struct module *mod)


void alternatives_enable_smp(void)



/*
 * Return 1 if the address range is reserved for SMP-alternatives.
 * Must hold text_mutex.
 */
int alternatives_text_reserved(void *start, void *end)


void __init alternative_instructions(void)


/**
 * text_poke_early - Update instructions on a live kernel at boot time
 * @addr: address to modify
 * @opcode: source of the copy
 * @len: length to copy
 *
 * When you use this code to patch more than one byte of an instruction
 * you need to make sure that other CPUs cannot execute this code in parallel.
 * Also no thread must be currently preempted in the middle of these
 * instructions. And on the local CPU you need to be protected against NMI or
 * MCE handlers seeing an inconsistent instruction while you patch.
 */
void __init_or_module text_poke_early(void *addr, const void *opcode,
				      size_t len)



/**
 * text_poke - Update instructions on a live kernel
 * @addr: address to modify
 * @opcode: source of the copy
 * @len: length to copy
 *
 * Only atomic text poke/set should be allowed when not doing early patching.
 * It means the size must be writable atomically and the address must be aligned
 * in a way that permits an atomic write. It also makes sure we fit on a single
 * page.
 *
 * Note that the caller must ensure that if the modified code is part of a
 * module, the module would not be removed during poking. This can be achieved
 * by registering a module notifier, and ordering module removal and patching
 * trough a mutex.
 */
void *text_poke(void *addr, const void *opcode, size_t len)





/**
 * text_poke_kgdb - Update instructions on a live kernel by kgdb
 * @addr: address to modify
 * @opcode: source of the copy
 * @len: length to copy
 *
 * Only atomic text poke/set should be allowed when not doing early patching.
 * It means the size must be writable atomically and the address must be aligned
 * in a way that permits an atomic write. It also makes sure we fit on a single
 * page.
 *
 * Context: should only be used by kgdb, which ensures no other core is running,
 *	    despite the fact it does not hold the text_mutex.
 */
void *text_poke_kgdb(void *addr, const void *opcode, size_t len)






void text_poke_finish(void)



void __ref text_poke_queue(void *addr, const void *opcode, size_t len, const void *emulate)


/**
 * text_poke_bp() -- update instructions on live kernel on SMP
 * @addr:	address to patch
 * @opcode:	opcode of new instruction
 * @len:	length to copy
 * @emulate:	instruction to be emulated
 *
 * Update a single instruction with the vector in the stack, avoiding
 * dynamically allocated memory. This function should be used when it is
 * not possible to allocate memory.
 */
void __ref text_poke_bp(void *addr, const void *opcode, size_t len, const void *emulate)














































