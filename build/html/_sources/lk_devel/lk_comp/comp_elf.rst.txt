内核与ELF
--------

ELF文件格式理解
^^^^^^^^^^^^^^
不对规范进行说明，
不详细介绍，只介绍我们能用的，方便我们理解的，目的在于总结，在于一目了然的理解。

可执行文件格式更多的给程序加载软件一个程序解析指南。并不是运行时的内存拓扑，而是加载软件根据ELF文件来解析并放置二进制程序，生成内存镜像。

结合对ELF文件的理解，对编译过程理解，对链接过程理解，加上对架构级相关的物理地址、线性地址和虚拟地址的理解，才能更深入透彻地理解linux中的高级编程部分。

我们看内核头文件定义：readelf -h vmlinux

.. code-block:: c
	:caption: make 调用自定义函数
	:emphasize-lines: 4,5
	:linenos:
	
	ELF 头：
  	  Magic：  7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00 
  	  类别:                              ELF64
  	  数据:                              2 补码，小端序 (little endian)
  	  Version:                           1 (current)
    	  OS/ABI:                            UNIX - System V
  	  ABI 版本:                          0
    	  类型:                              EXEC (可执行文件)
  	  系统架构:                          Advanced Micro Devices X86-64
  	  版本:                              0x1
  	  入口点地址：              0x1000000
  	  程序头起点：              64 (bytes into file)
  	  Start of section headers:          636238104 (bytes into file)
  	  标志：             0x0
  	  Size of this header:               64 (bytes)
  	  Size of program headers:           56 (bytes)
  	  Number of program headers:         5
  	  Size of section headers:           64 (bytes)
  	  Number of section headers:         80
  	  Section header string table index: 79
  	  

下面总结时会动点对ELF64,EXEC，X86-64进行总结说明。
我们根据这个信息，看其拓扑。(/usr/include/elf.h)

ELF头：

.. code-block:: c
	:caption:  The ELF file header.  This appears at the start of every ELF file.
	:emphasize-lines: 4,5
	:linenos:
	typedef struct
	{
 	 unsigned char	e_ident[EI_NIDENT];	/* Magic number and other info */
 	 Elf64_Half	e_type;			/* Object file type */
 	 Elf64_Half	e_machine;		/* Architecture */
 	 Elf64_Word	e_version;		/* Object file version */
 	 Elf64_Addr	e_entry;		/* Entry point virtual address: 虚拟地址入口，注意在加载程序时要保证这个地址是确定的 */
	 Elf64_Off	e_phoff;		/* Program header table file offset：在文件中的偏移，与加载后程序布局无关 */
	 Elf64_Off	e_shoff;		/* Section header table file offset：在文件中的偏移，与加载后程序布局无关 */
	 Elf64_Word	e_flags;		/* Processor-specific flags */
 	 Elf64_Half	e_ehsize;		/* ELF header size in bytes */
 	 Elf64_Half	e_phentsize;		/* Program header table entry size */
 	 Elf64_Half	e_phnum;		/* Program header table entry count */
	 Elf64_Half	e_shentsize;		/* Section header table entry size */
	 Elf64_Half	e_shnum;		/* Section header table entry count */
	 Elf64_Half	e_shstrndx;		/* Section header string table index */
	} Elf64_Ehdr;

section:Section header.

.. code-block:: c
	:caption:  Section header.
	:emphasize-lines: 4,5
	:linenos:
	
	typedef struct
	{
 	 Elf64_Word	sh_name;		/* Section name (string tbl index) */
 	 Elf64_Word	sh_type;		/* Section type */
 	 Elf64_Xword	sh_flags;		/* Section flags */
 	 Elf64_Addr	sh_addr;		/* Section virtual addr at execution：执行时的虚拟地址，程序加载后，在内存中的虚拟地址，可以作为引用进行使用 */
 	 Elf64_Off	sh_offset;		/* Section file offset：在文件中的索引，与执行镜像无关 */
 	 Elf64_Xword	sh_size;		/* Section size in bytes */
 	 Elf64_Word	sh_link;		/* Link to another section */
 	 Elf64_Word	sh_info;		/* Additional section information */
 	 Elf64_Xword	sh_addralign;		/* Section alignment */
  	 Elf64_Xword	sh_entsize;		/* Entry size if section holds table */
	} Elf64_Shdr;

header:

.. code-block:: c
	:caption: Program segment header. 
	:emphasize-lines: 4,5
	:linenos:
	
	typedef struct
	{
	  Elf64_Word	p_type;			/* Segment type */
	  Elf64_Word	p_flags;		/* Segment flags */
 	  Elf64_Off	p_offset;		/* Segment file offset */
  	  Elf64_Addr	p_vaddr;		/* Segment virtual address：执行时的虚拟地址，程序加载后，在内存中的虚拟地址 */
 	  Elf64_Addr	p_paddr;		/* Segment physical address：除非特殊情况，平常可不予关注 */
 	  Elf64_Xword	p_filesz;		/* Segment size in file */
  	  Elf64_Xword	p_memsz;		/* Segment size in memory */
  	  Elf64_Xword	p_align;		/* Segment alignment */
	} Elf64_Phdr;
节是存储，可当结构，细粒度；
段是属性相同的节的集合；

ELF文件类型
^^^^^^^^^^^^
.. code-block:: c
	:caption: e_type
	:emphasize-lines: 4,5
	:linenos:

	#define ET_NONE		0		/* No file type */
	#define ET_REL		1		/* Relocatable file */
	#define ET_EXEC		2		/* Executable file */
	#define ET_DYN		3		/* Shared object file */
	#define ET_CORE		4		/* Core file */
	#define	ET_NUM		5		/* Number of defined types */
	#define ET_LOOS		0xfe00		/* OS-specific range start */
	#define ET_HIOS		0xfeff		/* OS-specific range end */
	#define ET_LOPROC	0xff00		/* Processor-specific range start */
	#define ET_HIPROC	0xffff		/* Processor-specific range end */

- ET_REL: .ko
- ET_EXEC: vmlinux
- ET_DYN: .so文件，demo:test:DYN (Position-Independent Executable file)
- ET_CORE: 

ELF在vmlinux中的应用
^^^^^^^^^^^^^^^^^^^

编译过程中，对符号的处理
""""""""""""""""""""

内核加载过程总结
"""""""""""""""









.ko 加载过程分析
^^^^^^^^^^^^^^














ET_EXEC/ET_DYN 加载过程分析（exec)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
elf_an.c
"""""""""
.. code-block:: c
	:caption: e_type
	:emphasize-lines: 4,5
	:linenos:
	
	#include <stdio.h>
	void main(void)
	{
		printf("%s:%d\n",__func__,__LINE__);
	}

编译
"""""""
.. code-block:: c
	:caption: e_type
	:emphasize-lines: 4,5
	:linenos:
	
	gcc -o elf_an elf_an.c 

查看elf_an文件头
""""""""""""""""
.. code-block:: c
	:caption: e_type
	:emphasize-lines: 4,5
	:linenos:
	
	$ readelf -h elf_an
	ELF 头：
	  Magic：  7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00 
	  类别:                              ELF64
	  数据:                              2 补码，小端序 (little endian)
	  Version:                           1 (current)
	  OS/ABI:                            UNIX - System V
	  ABI 版本:                          0
	  类型:                              DYN (Position-Independent Executable file)
	  系统架构:                          Advanced Micro Devices X86-64
	  版本:                              0x1
	  入口点地址：              0x1050
	  程序头起点：              64 (bytes into file)
	  Start of section headers:          14176 (bytes into file)
	  标志：             0x0
	  Size of this header:               64 (bytes)
	  Size of program headers:           56 (bytes)
	  Number of program headers:         13
	  Size of section headers:           64 (bytes)
	  Number of section headers:         31
	  Section header string table index: 30

查看其依赖库
""""""""""

.. code-block:: c
	:caption: e_type
	:emphasize-lines: 4,5
	:linenos:
	
	$ldd elf_an
	
		linux-vdso.so.1 (0x00007ffcf3f1c000)
        	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f94bb649000)
        	/lib64/ld-linux-x86-64.so.2 (0x00007f94bb84a000)

	 
现在我们跟踪其系统调用
""""""""""""""""""

.. code-block:: c
	:caption: e_type
	:emphasize-lines: 4,5
	:linenos:

	$ strace ./elf_an 
	execve("./elf_an", ["./elf_an"], 0x7ffd353803d0 /* 60 vars */) = 0
	brk(NULL)                               = 0x55c52ee9f000
	access("/etc/ld.so.preload", R_OK)      = -1 ENOENT (没有那个文件或目录)
	openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
	newfstatat(3, "", {st_mode=S_IFREG|0644, st_size=198967, ...}, AT_EMPTY_PATH) = 0
	mmap(NULL, 198967, PROT_READ, MAP_PRIVATE, 3, 0) = 0x7f94c812f000
	close(3)                                = 0
	openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libc.so.6", O_RDONLY|O_CLOEXEC) = 3
	read(3, "\177ELF\2\1\1\3\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0000y\2\0\0\0\0\0"..., 832) = 832
	pread64(3, "\6\0\0\0\4\0\0\0@\0\0\0\0\0\0\0@\0\0\0\0\0\0\0@\0\0\0\0\0\0\0"..., 784, 64) = 784
	pread64(3, "\4\0\0\0\20\0\0\0\5\0\0\0GNU\0\2\200\0\300\4\0\0\0\1\0\0\0\0\0\0\0", 32, 848) = 32
	pread64(3, "\4\0\0\0\24\0\0\0\3\0\0\0GNU\0\320\276\243\212\v\307^\t\263h8\371\266h\r\350"..., 68, 880) = 68
	newfstatat(3, "", {st_mode=S_IFREG|0755, st_size=1835120, ...}, AT_EMPTY_PATH) = 0
	mmap(NULL, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f94c812d000
	pread64(3, "\6\0\0\0\4\0\0\0@\0\0\0\0\0\0\0@\0\0\0\0\0\0\0@\0\0\0\0\0\0\0"..., 784, 64) = 784
	mmap(NULL, 1868664, PROT_READ, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7f94c7f64000
	mprotect(0x7f94c7f8a000, 1654784, PROT_NONE) = 0
	mmap(0x7f94c7f8a000, 1343488, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x26000) = 0x7f94c7f8a000
	mmap(0x7f94c80d2000, 307200, PROT_READ, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x16e000) = 0x7f94c80d2000
	mmap(0x7f94c811e000, 24576, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x1b9000) = 0x7f94c811e000
	mmap(0x7f94c8124000, 33656, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_ANONYMOUS, -1, 0) = 0x7f94c8124000
	close(3)                                = 0
	mmap(NULL, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f94c7f62000
	arch_prctl(ARCH_SET_FS, 0x7f94c812e580) = 0
	mprotect(0x7f94c811e000, 12288, PROT_READ) = 0
	mprotect(0x55c52d442000, 4096, PROT_READ) = 0
	mprotect(0x7f94c818f000, 8192, PROT_READ) = 0
	munmap(0x7f94c812f000, 198967)          = 0
	newfstatat(1, "", {st_mode=S_IFCHR|0600, st_rdev=makedev(0x88, 0x9), ...}, AT_EMPTY_PATH) = 0
	brk(NULL)                               = 0x55c52ee9f000
	brk(0x55c52eec0000)                     = 0x55c52eec0000
	write(1, "main:4\n", 7main:4
	)                 = 7
	exit_group(7)                           = ?
	+++ exited with 7 +++

过程如下：
1. execve("./elf_an", ["./elf_an"], 0x7ffd353803d0 /* 60 vars */) = 0：
    a. 可执行文件： ./elf_an;
    b. 参数argv： ["./elf_an"],如果运行如下指令"./elf_an test",则此处的argv参数为：execve("./elf_an", ["./elf_an", "test"], 0x7ffe9ea5acf8 /* 25 vars */) = 0
    c. envp: 环境变量指针
    

2. brk(NULL)：

3. ld.so.preload:

   .. code-block:: c
	:caption: e_type
	:emphasize-lines: 4,5
	:linenos:
	
	access("/etc/ld.so.preload", R_OK) 
	
3. ld.so.cache:

  .. code-block:: c
	:caption: 全映射：长度：198967,地址指针： 0x7f94c812f000
	:emphasize-lines: 4,5
	:linenos:
	
	openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
	newfstatat(3, "", {st_mode=S_IFREG|0644, st_size=198967, ...}, AT_EMPTY_PATH) = 0
	mmap(NULL, 198967, PROT_READ, MAP_PRIVATE, 3, 0) = 0x7f94c812f000
	close(3)                                = 0
	
4. libc.so.6：

  .. code-block:: c
	:caption: 全映射：长度
	:emphasize-lines: 4,5
	:linenos:
	
	openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libc.so.6", O_RDONLY|O_CLOEXEC) = 3
	read(3, "\177ELF\2\1\1\3\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0000y\2\0\0\0\0\0"..., 832) = 832
	pread64(3, "\6\0\0\0\4\0\0\0@\0\0\0\0\0\0\0@\0\0\0\0\0\0\0@\0\0\0\0\0\0\0"..., 784, 64) = 784
	pread64(3, "\4\0\0\0\20\0\0\0\5\0\0\0GNU\0\2\200\0\300\4\0\0\0\1\0\0\0\0\0\0\0", 32, 848) = 32
	pread64(3, "\4\0\0\0\24\0\0\0\3\0\0\0GNU\0\320\276\243\212\v\307^\t\263h8\371\266h\r\350"..., 68, 880) = 68
	newfstatat(3, "", {st_mode=S_IFREG|0755, st_size=1835120, ...}, AT_EMPTY_PATH) = 0
	mmap(NULL, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f94c812d000
	pread64(3, "\6\0\0\0\4\0\0\0@\0\0\0\0\0\0\0@\0\0\0\0\0\0\0@\0\0\0\0\0\0\0"..., 784, 64) = 784
	mmap(NULL, 1868664, PROT_READ, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7f94c7f64000
	mprotect(0x7f94c7f8a000, 1654784, PROT_NONE) = 0
	mmap(0x7f94c7f8a000, 1343488, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x26000) = 0x7f94c7f8a000
	mmap(0x7f94c80d2000, 307200, PROT_READ, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x16e000) = 0x7f94c80d2000
	mmap(0x7f94c811e000, 24576, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x1b9000) = 0x7f94c811e000
	mmap(0x7f94c8124000, 33656, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_ANONYMOUS, -1, 0) = 0x7f94c8124000
	close(3)

5. 其他：

     .. code-block:: c
	:caption: 全映射：长度
	:emphasize-lines: 4,5
	:linenos:
	
	mmap(NULL, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f94c7f62000
	arch_prctl(ARCH_SET_FS, 0x7f94c812e580) = 0
	mprotect(0x7f94c811e000, 12288, PROT_READ) = 0
	mprotect(0x55c52d442000, 4096, PROT_READ) = 0
	mprotect(0x7f94c818f000, 8192, PROT_READ) = 0
	munmap(0x7f94c812f000, 198967)          = 0
	newfstatat(1, "", {st_mode=S_IFCHR|0600, st_rdev=makedev(0x88, 0x9), ...}, AT_EMPTY_PATH) = 0
	brk(NULL)                               = 0x55c52ee9f000
	brk(0x55c52eec0000)                     = 0x55c52eec0000
	write(1, "main:4\n", 7main:4
	)                 = 7
	exit_group(7)                           = ?
	+++ exited with 7 +++	



库函数跟踪：
""""""""""


.. code-block:: c
	:caption: e_type
	:emphasize-lines: 4,5
	:linenos:
	
	$lstrace ./elf_an
	printf("%s:%d\n", "main", 4main:4)                                                = 7
	+++ exited (status 7) +++
	
下面我们跟踪exec系统调用内核执行路径
"""""""""""""""""""""""""""""""

内核路径跟踪
**********
路径跟踪参考：https://github.com/rachelsqh/lk_kernel/tree/main/elf_an_1

代码分析
*******
借助上一步的内核路径，仅针对ELF文件加载函数：


.. code-block:: c
	:caption: e_type
	:emphasize-lines: 4,5
	:linenos:
	
	load_elf_binary() {
	
		load_elf_phdrs() {
		
		914 - 4144


elf_an elf文件头信息：

.. code-block:: c
	:caption: e_type
	:emphasize-lines: 4,5
	:linenos:
	
	ELF 头：
  	Magic：  7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00 
  	类别:                              ELF64
 	 数据:                              2 补码，小端序 (little endian)
  	Version:                           1 (current)
  	OS/ABI:                            UNIX - System V
  	ABI 版本:                          0
  	类型:                              DYN (Position-Independent Executable file)
  	系统架构:                          Advanced Micro Devices X86-64
  	版本:                              0x1
  	入口点地址：              0x1050
  	程序头起点：              64 (bytes into file)
  	Start of section headers:          14176 (bytes into file)
  	标志：             0x0
  	Size of this header:               64 (bytes)
  	Size of program headers:           56 (bytes)
  	Number of program headers:         13
  	Size of section headers:           64 (bytes)
  	Number of section headers:         31
  	Section header string table index: 30
	......
	
	程序头：
  	Type           Offset             VirtAddr           PhysAddr
     	               FileSiz            MemSiz              Flags  Align
  	PHDR           0x0000000000000040 0x0000000000000040 0x0000000000000040
      	               0x00000000000002d8 0x00000000000002d8  R      0x8
        INTERP         0x0000000000000318 0x0000000000000318 0x0000000000000318
                       0x000000000000001c 0x000000000000001c  R      0x1
      			[Requesting program interpreter: /lib64/ld-linux-x86-64.so.2]
  	LOAD           0x0000000000000000 0x0000000000000000 0x0000000000000000
                 	0x0000000000000600 0x0000000000000600  R      0x1000
  	LOAD           0x0000000000001000 0x0000000000001000 0x0000000000001000
                 	0x00000000000001dd 0x00000000000001dd  R E    0x1000
  	LOAD           0x0000000000002000 0x0000000000002000 0x0000000000002000
                 	0x0000000000000158 0x0000000000000158  R      0x1000
  	LOAD           0x0000000000002de8 0x0000000000003de8 0x0000000000003de8
                 	0x0000000000000248 0x0000000000000250  RW     0x1000
  	DYNAMIC        0x0000000000002df8 0x0000000000003df8 0x0000000000003df8
                 	0x00000000000001e0 0x00000000000001e0  RW     0x8
  	NOTE           0x0000000000000338 0x0000000000000338 0x0000000000000338
                 	0x0000000000000020 0x0000000000000020  R      0x8
  	NOTE           0x0000000000000358 0x0000000000000358 0x0000000000000358
                 	0x0000000000000044 0x0000000000000044  R      0x4
  	GNU_PROPERTY   0x0000000000000338 0x0000000000000338 0x0000000000000338
                 	0x0000000000000020 0x0000000000000020  R      0x8
  	GNU_EH_FRAME   0x0000000000002010 0x0000000000002010 0x0000000000002010
                 	0x000000000000003c 0x000000000000003c  R      0x4
  	GNU_STACK      0x0000000000000000 0x0000000000000000 0x0000000000000000
                 	0x0000000000000000 0x0000000000000000  RW     0x10
  	GNU_RELRO      0x0000000000002de8 0x0000000000003de8 0x0000000000003de8
                 	0x0000000000000218 0x0000000000000218  R      0x1

加载完后： nmaps:

.. code-block:: c
	:caption: e_type
	:emphasize-lines: 4,5
	:linenos:
	
	55add343f000-55add3440000 r--p 00000000 103:05 27404226                  /root/for_work/jekyll-theme/sphinx-book/lk_kernel_demo/lk_kernel/elf_an_1/elf_an
	55add3440000-55add3441000 r-xp 00001000 103:05 27404226                  /root/for_work/jekyll-theme/sphinx-book/lk_kernel_demo/lk_kernel/elf_an_1/elf_an
	55add3441000-55add3442000 r--p 00002000 103:05 27404226                  /root/for_work/jekyll-theme/sphinx-book/lk_kernel_demo/lk_kernel/elf_an_1/elf_an
	55add3442000-55add3443000 r--p 00002000 103:05 27404226                  /root/for_work/jekyll-theme/sphinx-book/lk_kernel_demo/lk_kernel/elf_an_1/elf_an
	55add3443000-55add3444000 rw-p 00003000 103:05 27404226                  /root/for_work/jekyll-theme/sphinx-book/lk_kernel_demo/lk_kernel/elf_an_1/elf_an
	55add3558000-55add3579000 rw-p 00000000 00:00 0                          [heap]
	7f8d7f8b7000-7f8d7f8b9000 rw-p 00000000 00:00 0 
	7f8d7f8b9000-7f8d7f8df000 r--p 00000000 103:05 30673355                  /usr/lib/x86_64-linux-gnu/libc-2.33.so
	7f8d7f8df000-7f8d7fa27000 r-xp 00026000 103:05 30673355                  /usr/lib/x86_64-linux-gnu/libc-2.33.so
	7f8d7fa27000-7f8d7fa72000 r--p 0016e000 103:05 30673355                  /usr/lib/x86_64-linux-gnu/libc-2.33.so
	7f8d7fa72000-7f8d7fa73000 ---p 001b9000 103:05 30673355                  /usr/lib/x86_64-linux-gnu/libc-2.33.so
	7f8d7fa73000-7f8d7fa76000 r--p 001b9000 103:05 30673355                  /usr/lib/x86_64-linux-gnu/libc-2.33.so
	7f8d7fa76000-7f8d7fa79000 rw-p 001bc000 103:05 30673355                  /usr/lib/x86_64-linux-gnu/libc-2.33.so
	7f8d7fa79000-7f8d7fa84000 rw-p 00000000 00:00 0 
	7f8d7fab5000-7f8d7fab6000 r--p 00000000 103:05 30672509                  /usr/lib/x86_64-linux-gnu/ld-2.33.so
	7f8d7fab6000-7f8d7fada000 r-xp 00001000 103:05 30672509                  /usr/lib/x86_64-linux-gnu/ld-2.33.so
	7f8d7fada000-7f8d7fae4000 r--p 00025000 103:05 30672509                  /usr/lib/x86_64-linux-gnu/ld-2.33.so
	7f8d7fae4000-7f8d7fae6000 r--p 0002e000 103:05 30672509                  /usr/lib/x86_64-linux-gnu/ld-2.33.so
	7f8d7fae6000-7f8d7fae8000 rw-p 00030000 103:05 30672509                  /usr/lib/x86_64-linux-gnu/ld-2.33.so
	7ffce0116000-7ffce0137000 rw-p 00000000 00:00 0                          [stack]
	7ffce0144000-7ffce0148000 r--p 00000000 00:00 0                          [vvar]
	7ffce0148000-7ffce014a000 r-xp 00000000 00:00 0                          [vdso]

我们直接啃代码：

.. code-block:: c
	:caption: e_type
	:emphasize-lines: 4,5
	:linenos:
	
	/*
 	  * This structure is used to hold the arguments that are used when loading binaries.
 	*/
	struct linux_binprm {
	#ifdef CONFIG_MMU
		struct vm_area_struct *vma;//这是一个链
		unsigned long vma_pages;
	#else
	# define MAX_ARG_PAGES	32
		struct page *page[MAX_ARG_PAGES];//arg
	#endif
		struct mm_struct *mm; // ==> task->mm,所以内核线程这个成员是NULL，就是这么联系起来的
		unsigned long p; /* current top of mem :栈指针：*/
		unsigned long argmin; /* rlimit marker for copy_strings(): */
		unsigned int
			/* Should an execfd be passed to userspace? : nmap 吗？这个应用场景*/
			have_execfd:1,

			/* Use the creds of a script (see binfmt_misc) ：应用场景？*/
			execfd_creds:1,
			/*
		 	* Set by bprm_creds_for_exec hook to indicate a
		 	* privilege-gaining exec has happened. Used to set
		 	* AT_SECURE auxv for glibc.
		 	*/
			secureexec:1, /* 如何理解 */
			/*
		 	* Set when errors can no longer be returned to the
		 	* original userspace.
		 	*/
			point_of_no_return:1;  /* 如何理解   */
	#ifdef __alpha__
		unsigned int taso:1;
	#endif
		struct file *executable; /* Executable to pass to the interpreter ：这是个指针吗？*/
		struct file *interpreter; /* 这是解析吗？ */
		struct file *file;
		struct cred *cred;	/* new credentials */
		int unsafe;		/* how unsafe this exec is (mask of LSM_UNSAFE_*) */
		unsigned int per_clear;	/* bits to clear in current->personality */
		int argc, envc;   /* 变量数 */
		const char *filename;	/* Name of binary as seen by procps ：proc路径？*/
		const char *interp;	/* Name of the binary really executed. Most
				   of the time same as filename, but could be
				   different for binfmt_{misc,script} */
		const char *fdpath;	/* generated filename for execveat： 内容是什么？ */
		unsigned interp_flags;
		int execfd;		/* File descriptor of the executable */
		unsigned long loader, exec;

		struct rlimit rlim_stack; /* Saved RLIMIT_STACK used during exec. ：栈大小限制吗*/

		char buf[BINPRM_BUF_SIZE]; /* 这存储什么？ */
	} __randomize_layout;
	


.. code-block:: c
	:caption: execv代码分析。
	:emphasize-lines: 4,5
	:linenos:

	bprm = alloc_bprm(fd,filename): 分配struct mm_struct
		bprm = kzalloc();
		bprm->interp = bprm->filename;
		bprm_mm_init(); /* 注意栈的设置是临时的，需要在setup_arg_pages中更新 */
			bprm->m = mm_alloc();
			bprm->rlim_stack = current->signal->rlim[RLIMIT_STACK];
			__bprm_mm_init()
				bprm->vma = vm_area_alloc();
				vma_set_anonymous(vma);
				vma->vm_end = STACK_TOP_MAX; // TASK_SIZE_MAX,最大地址空间 - PAGE_SIZE
				vma->vm_start = vma->end - PAGE_SIZE; //栈占用一个页面。
				vma->vm_page_prot = vm_get_page_prot(vma->vm_flags);
				insert_vm_struct(mm,vma);
				mm->stack_vm = mm->total_vm = 1;
				bprm->p = vma->vm_end - sizeof(void *); /* 所以有一个指针的空 */
		
    	bprm -> argc = count();
    	bprm -> envc = count();
    	bprm -> argmin = bprm_stack_limits();
    
   	 bprm->p -= len(bprm->filename);
  	  bprm->p -= bprm->envc;
  	  bprm->p -= bprm->argc;
    
   	 bprm_execve();
  	  	prepare_bprm_creds();
 	   		bprm->cred = prepare_exec_creds();
    	
  	  	check_unsafe_exec();
    	
  	  	current->in_execve = 1;
    	
   	 	file = do_open_execat();
    			file = do_filp_open();
    		
    			S_ISREG(file_inode(file)->i_mode) 或 path_noexec(&file->f_path);判断，不满足就结束了。
    		
    			deny_write_access(file)
    				atomic_dec_unless_positive(inode->i_writecount)
    		
    			fsnotify_open(file);
    		
    		sched_exec();// 此处是调度均衡的很好时机，因为当前进程有最小的有效内存和cache
    	
    		bprm->file;

		security_bprm_creds_for_exec();// call_int_hook();
    	
    		exec_binprm(bprm);
    		
    			old_pid = current->pid;
    			old_vpid = task_pid_nr_ns();
    		
    			for(depth = 0 ;; depth++) { //可执行文件 + 解释器，循环加载，这是外层逻辑。
    			
    				search_binary_handler(bprm); //这是核心部分
    				
    					need_retry = IS_ENABLED(CONFIG_MODULES);
    				
    					prepare_binprm(bprm); //填充bprm结构，读文件开始的BINPRM_BUF_SIZE个字节。256
    						kernel_read(bprm->file,bprm->buf,BINPRM_BUF_SIZE,&pos); // pos = 0
    				
    					security_bprm_check(bprm);
    				
    				     retry:
    				 	
    			 		list_for_each_entry(fmt,&formats,lh) { // struct linux_binfmt *fmt; 
    			 	
    			 			try_module_get();
    			 	
    			 			retval = fmt->load_binary(bprm); // 继续核心，以ELF为例
    			 			    load_elf_binary()
    			 			    	executable_stack = EXSTACK_DEFAULT;
    			 		    		elf_ex = bprm->buf;
    			 		    		arch_state = INIT_ARCH_ELF_STATE;
    			 		    	
    			 		    		memcmp(elf_ex->e_ident,ELFMAG,SELFMAG); //判断魔数
    			 		    	
    			 		    		elf_ex->e_type == ET_EXEC || elf_ex->e_type == ET_DYN; //只处理这两种类型
    			 		    	
    			 		    		elf_check_arch(elf_ex); // 架构匹配性校验
    			 		    	
    			 		    		elf_check_fdpic(elf_ex);// 这个怎么理解
    			 		    	
    			 		    		bprm->file->f_op->mmap; // 确认是否实现函数
    			 		    	
    			 		    		elf_phdata= load_elf_phdrs(elf,ex,bprm->file);//加载文件头
    			 		
    			 			
    			 				for(i = 0;i < elf_ex->e_phnum;i++,elf_ppnt++) { // 按程序段进行解析,第一步解析解析程序,即：PT_INTERP
    			 				
    			 					elf_ppnt->p_type == PT_GNU_PROPERTY ==> elf_property_phdata = elf_ppnt;
    			 						continue;
    			 				
    			 					elf_ppnt->p_type != PT_INTERP;  
    			 						continue;
    			 				
    			 					elf_interpreter = kmalloc(elf_ppnt->p_filesz,GFP_KERNEL);	 // 为段分配足够长度的空间
    			 				
    			 					elf_read(bprm->file,elf_interpreter,elf_ppnt->p_filesz,elf_ppnt->p_offset); //加载段，针对解析器，此处存储解析器路径。
    			 				
    			 					interpreter = open_exec(elf_interpreter); //解析器文件句柄
    			 				
    			 					kfree(elf_interpreter); // 文件已经打开，这个存储路径的就没必要了
    			 				
    			 				
    			 					would_dump(bprm,interpreter); //
    			 				
    			 					elf_read(interpreter,interp_elf_ex,sizeof(*interp_elf_ex),0); // 读解析器的文件头
    			 				
    			 					break; //只有一个解析器。
    			 				}
    			 			
    			 				elf_ppnt = elf_phdata;
    			 			
    			 				for(i = 0;i < elf_ex->e_phnum;i++,elf_ppnt++) //可执行文件段的遍历
    			 					elf_ppnt->p_type:
    			 						PT_GNU_STACK:
    			 							p_flags:
    			 								if PF_X:
    			 								   executable_stack = EXSTACK_ENABLE_X;
    			 								 else
    			 								   executable_stack = EXSTACK_DISABLE_X;
    			 					 	PT_LOPROC ... PT_HIPROC:
    			 					 		arch_elf_pt_proc(elf_ex,elf_ppnt,bprm->file,false,&arch_state);
    			 					 
    			 			
    			 				if(interpreter) {
    			 			
    			 					memcmp(interp_elf_ex->e_ident,ELFMAG,SELFMAG); // 解释器函数魔数比较
    			 				
    			 					elf_check_arch(interp_elf_ex);
    			 				
    			 					elf_check_fdpic(interp_elf_ex);
    			 				
    			 					interp_elf_phdata = load_elf_phdrs(interp_elf_ex,interpreter); // 解释器文件的文件头
    			 				
    			 					elf_property_phdata = NULL;	
    			 				
    			 					for(i = 0;i < interp_elf_ex->e_phnum;i++,elf_ppnt++) //解释器文件段的遍历	 
    			 				
    			 						elf_ppnt->p_type:
    			 							PT_GNU_PROPERTY:
    			 								elf_property_phdata = elf_ppnt;
    			 								break;
    			 							
    			 							PT_LOPPROC ... PT_HIPROC:
    			 								arch_elf_pt_proc(interp_elf_ex,elf_ppnt,interpreter,true,&arch_state);
    			 							
    			 				} //end if (interpreter)
    			 			
    			 				parse_elf_properties(interpreter ? : bprm->file,elf_property_phdata,&arch_state);				  
    			 				
    			 				
    			 				
    			 				arch_check_elf(elf_ex,!!interpreter,interp_elf_ex,&arch_state);// 此处允许架构级代码hook
    			 			
    			 			
    			 				begin_new_exec(bprm);//
    			 			
    			 				SET_PERSONALITY2(*elf_ex,&arch_state); //
    			 			
    			 				elf_read_implies_exec();
    			 			
    			 				current->personality;//这个成员功能，如何处理
    			 			
    			 				setup_new_exec(bprm);
    			 			
    			 				setup_arg_pages(bprm,randomize_stack_top(STACK_TOP),executable_stack); //重要函数
    			 			
    			 			
    			 				elf_bss = 0;
    			 			
    			 				elf_brk = 0;
    			 			
    			 				start_code = ~0UL;
    			 			
    			 				end_code = 0;
    			 				start_data = 0;
    			 				end_data = 0;
    			 			
    			 				for(i = 0;elf_ppnt = elf_phdata;i < elf_ex->e_phnum; i++,elf_ppnt++)// 处理PT_LOAD类型段
    			 					elf_prot = make_prot(elf_ppnt->p_flags,&arch_state,!!interpreter,false);
    			 					elf_flags = MAP_PRIVATE |MAP_DENYWRITE;
    			 				
    			 					vaddr = elf_ppnt->p_vaddr;
    			 				
    			 					ET_EXEC || ET_DYN:
    			 				
    			 					elf_map(bprm->file,load_bias + vaddr,elf_ppnt,elf_prot,elf_flags,total_size);
    			 				
    			 					.......
    			 			
    			 			
    			 				e_entry = elf_ex->e_entry + load_bias;
    			 			
    			 				set_brk();
    			 			
    			 				if(interpreter) { // 有加载器时,elf_entry = interp_entry;
    			 			
    			 					elf_entry = load_elf_interp();
    			 			
    			 					interp_load_addr = elf_entry;
    			 					elf_entry += interp_elf_ex->e_entry;
    			 				
    			 					reloc_func_desc = interp_load_addr;
    			 					allow_write_access(interpreter);
    			 					fput(interpreter);
    			 				
    			 					kfree(interp_elf_ex);
    			 					kfree(interp_elf_phdata);
    			 				
    			 			
    			 				} else { //没有解释器
    			 			
    			 					elf_entry = e_entry;//可执行文件入口处作为程序执行的起点
    			 				
    			 				}
    			 			
    			 				kfree(elf_phdata);
    			 				set_binfmt();// mm->binfmt = new:struct linux_binfmt *new,空间与应用挂接
    			 			
    			 				create_elf_tables(bprm,elf_ex,load_addr,interp_load_addr,e_entry);//如果有加载器，interp_load_addr: 应用程序入口，e_entry,
    			 				
    			 					k_platform = ELF_PLATFORM; //utsname()->machine
    			 					p = bprm->p;
    			 					arch_align_stack(p);//
    			 				
    			 					k_platform/u_platform;//这个如何理解
    			 				
    			 					k_rand_bytes/u_rand_bytes;//如何理解
    			 				
    			 					elf_info = mm->saved_auxv; // 构建ELF加载器信息。saved_auxv,mm:saved_auxv[AT_VECTOR_SIZE];这是数组，静态空间
    			 						#define NEW_AUX_ENT(id,val) \
    			 							do {\
    			 								*elf_info++ = id; \
    			 								*elf_info++ = val; \
    			 							} while (0)
    			 				
    			 					NEW_AUX_ENT(AT_HWCAP,ELF_HWCAP); // *elf_info++ = AT_HWCAP,*elf_info++ = ELF_HWCAP;
    			 					NEW_AUX_ENT(AT_PAGESZ,ELF_EXEC_PAGESIZE); // *elf_info++ = AT_PAGESZ,*elf_info++ = ELF_EXEC_PAGESIZE;
    			 					NEW_AUX_ENT(AT_CLKTCK,CLOCKS_PER_SEC); // *elf_info++ = ,
    			 					NEW_AUX_ENT(AT_PHDR, load_addr + exec->e_phoff);
								NEW_AUX_ENT(AT_PHENT, sizeof(struct elf_phdr));
								NEW_AUX_ENT(AT_PHNUM, exec->e_phnum);
								NEW_AUX_ENT(AT_BASE, interp_load_addr);
								if (bprm->interp_flags & BINPRM_FLAGS_PRESERVE_ARGV0)
									flags |= AT_FLAGS_PRESERVE_ARGV0;
								NEW_AUX_ENT(AT_FLAGS, flags);
								NEW_AUX_ENT(AT_ENTRY, e_entry);
								NEW_AUX_ENT(AT_UID, from_kuid_munged(cred->user_ns, cred->uid));
								NEW_AUX_ENT(AT_EUID, from_kuid_munged(cred->user_ns, cred->euid));
								NEW_AUX_ENT(AT_GID, from_kgid_munged(cred->user_ns, cred->gid));
								NEW_AUX_ENT(AT_EGID, from_kgid_munged(cred->user_ns, cred->egid));
								NEW_AUX_ENT(AT_SECURE, bprm->secureexec);
								NEW_AUX_ENT(AT_RANDOM, (elf_addr_t)(unsigned long)u_rand_bytes);
								#ifdef ELF_HWCAP2
									NEW_AUX_ENT(AT_HWCAP2, ELF_HWCAP2);
								#endif
								NEW_AUX_ENT(AT_EXECFN, bprm->exec);
								if (k_platform) {
									NEW_AUX_ENT(AT_PLATFORM,
			    						(elf_addr_t)(unsigned long)u_platform);
								}
								if (k_base_platform) {
									NEW_AUX_ENT(AT_BASE_PLATFORM,
			    						(elf_addr_t)(unsigned long)u_base_platform);
								}
								if (bprm->have_execfd) {
									NEW_AUX_ENT(AT_EXECFD, bprm->execfd);
								}
							
  				 				#undef NEW_AUX_ENT
								/* AT_NULL is zero; clear the rest too */
								memset(elf_info, 0, (char *)mm->saved_auxv + sizeof(mm->saved_auxv) - (char *)elf_info); //剩下的部分置零
	
								/* And advance past the AT_NULL entry.  */
								elf_info += 2;

								ei_index = elf_info - (elf_addr_t *)mm->saved_auxv;
								sp = STACK_ADD(p, ei_index);

								items = (argc + 1) + (envc + 1) + 1;
								bprm->p = STACK_ROUND(sp, items);	
    				 					
   			 					/* Point sp at the lowest address on the stack */
								#ifdef CONFIG_STACK_GROWSUP
									sp = (elf_addr_t __user *)bprm->p - items - ei_index;
									bprm->exec = (unsigned long)sp; /* XXX: PARISC HACK */
								#else
									sp = (elf_addr_t __user *)bprm->p;
								#endif
								
								
								/*
		 						* Grow the stack manually; some architectures have a limit on how
		 						* far ahead a user-space access may be in order to grow the stack.
		 						*/
	
								vma = find_extend_vma(mm, bprm->p);
	
								/* Now, let's put argc (and argv, envp if appropriate) on the stack，所以加载的时候没有放到栈上吗？回头再看一下，理解有误？ */
								if (put_user(argc, sp++))
									return -EFAULT;
	
								/* Populate list of argv pointers back to argv strings. */
								p = mm->arg_end = mm->arg_start;
								while (argc-- > 0) {
		
									put_user((elf_addr_t)p, sp++)
		
									len = strnlen_user((void __user *)p, MAX_ARG_STRLEN);
		
									p += len;
								}
								
								put_user(0, sp++)
								mm->arg_end = p;

								/* Populate list of envp pointers back to envp strings. */
								mm->env_end = mm->env_start = p;
								while (envc-- > 0) {
									put_user((elf_addr_t)p, sp++))
									strnlen_user((void __user *)p, MAX_ARG_STRLEN);
									p += len;
								}
								put_user(0, sp++)
	
								mm->env_end = p;
	
								/* Put the elf_info on the stack in the right place.  */
								copy_to_user(sp, mm->saved_auxv, ei_index * sizeof(elf_addr_t)) //saved_auxv也是压栈的，也就是说所有的信息都是压入应用栈
								
	
    				 			mm = current->mm;
    				 			mm->end_code = end_code;
    				 			mm->start_code = start_code;
    				 			mm->start_data = start_data;
    				 			mm->end_data = end_data;
    				 			mm->start_stack = bprm->p;
    			 			
    				 			current->flags & PF_RANDOMIZE:
    				 			
    				 			current->personality & MMAP_PAGE_ZERO
    			 			
    			 			
    				 			rets = current_pt_regs(); //获取内核栈栈顶的struct pt_regs结构
    				 				#define current_pt_regs() task_pt_regs(current)
    				 					#define task_pt_regs(current)	\
    				 						({ \
    				 							unsigned long __ptr = (unsigned long)task_stack_page(task);	\ // (void *)(task)->stack,内核栈
    				 							__ptr += THREAD_SIZE - TOP_OF_KERNEL_STACK_PADDING;	\ // 栈顶位置，有一个TOP_OF_KERNEL_STACK_PADDING空
    				 							((struct pt_regs *)__ptr) - 1;	\ // 往下 - sizeof(struct pt_regs *)处存放struct pt_regs.
    				 						})
    			 						
    			 						    			 			
    				 			ELF_PLAT_INIT(regs,reloc_func_desc);
    				 				#define ELF_PLAT_INIT(_r,load_addr)	elf_common_init(&current->thread,_r,0) //初始化pt_regs和thread_struct中的相关变量
    			 					    			 		
    				 			finalize_exec(bprm);
    				 				current->signal->rlim[RLIMIT_STACK] = bprm->rlim_stack;
    			 			
    				 			START_THREAD(elf_ex,regs,elf_entry,bprm->p); // 初始化应用环境，这些环境信息存储在内核栈的栈顶，在返回系统调用时会进行上下文切换，第一个由内核线程而来的应用程序时则会通过执行特定的函数进入应用程序执行。
    				 				#define START_THREAD(elf_ex,regs,elf_entry,start_stack)	start_thread(regs,elf_entry,start_stack)
    				 					void start_thread(struct pt_regs *regs,unsigned long new_ip,unsigned long new_sp)
    				 					{
    				 						start_thread_common(regs,new_ip,new_sp,__USER_CS,__USER_DS,0);
    				 					}
    				 					static void start_thread_common(struct pt_regs *regs, unsigned long new_ip,unsigned long new_sp,unsigned int _cs, 
    				 						unsigned int _ss, unsigned int _ds)
    				 					{
    				 						WARN_ON_ONCE(regs != current_pt_regs());

										if (static_cpu_has(X86_BUG_NULL_SEG)) {
											/* Loading zero below won't clear the base. */
											loadsegment(fs, __USER_DS);
											load_gs_index(__USER_DS);
										}

										loadsegment(fs, 0);
										loadsegment(es, _ds);
										loadsegment(ds, _ds);
										load_gs_index(0);

										regs->ip		= new_ip; //作为连续的地址，入口地址在应用空间，
										regs->sp		= new_sp;
										regs->cs		= _cs; //x86-64下主要控制返回应用时的权限
										regs->ss		= _ss;
										regs->flags		= X86_EFLAGS_IF;
									}
    			 					
    			 			
    			 			
    				 		//end load_elf_binary()
    				 		
    				 		put_binfmt();
    			 		
    				 	if need_retry //这个程序块的逻辑意义？
    				 		printable:0,1,2,3 如何理解？,已经找到了？，所以return retval//可打印字符，#define printable(c) (((c) == '\t') || ((c) == '\n') || (0x20 <= (c) && (c) <= 0x7e))
    				 		request_module();
    				 		need_retry = false;
    				 		goto retry;
    			 	
    			
    				if(!bprm->interpreter)//如果这个为空，也就结束了,也就是彻底加载完毕
					break;
				
			
				exec = bprm->file;
				bprm->file  = bprm->interpreter; //进一步加载解释器
				
				bprm->interpreter = NULL;
				
				allow_write_access(exec);
			}  //到这儿就加载完了
		
			audit_bprm(bprm);
			trace_sched_process_exec();
			ptrace_event();  // ptrace_notify / send_sig
			proc_exec_connector();
	
		current->fs->in_exec = 0; //这个指示是否在运行exec系统调用。
		current->in_execve = 0; //这两个参数意义在哪里？
	
		rseq_execve();  //这个注意，有深度
		acct_update_integrals();
		task_numa_free();

	    free_bprm(bprm);
    
    	putname(filename);		

  			
ELF文件加载过程总结：
^^^^^^^^^^^^^^^^^
			
1. linux_binprm 结构初始化；
2. 打开可执行文件，并读文件头；
3. 根据文件头读段表；
4. 判断是否存在PT_INTRP段，并从文件中读出段对应的加载器路径名；
5. 加载可执行文件的PT_LOAD属性的段；
6. 如果存在加载器，加载加载器中的PT_LOAD段；
7. 构建mm->saved_auxv;
8. 将传入的参数列表、环境变量列表及构建的mm->saved_auxv存入应用栈；
9. 如果有加载器，则将与加载器入口相关的参数存入内核栈的pt_regs变量中，否则使用应用程序的入口构建；
10. 完成加载并返回系统调用，从而进入应用程序执行，如果有加载器，则跳入加载器应用入口根据已有信息对可执行文件进行符号链接等进一步操作，最后加载器跳到应用软件入口进行执行；否则直接执行应用软件；

注意，这仅对可执行程序的加载进行粗略描述，具体实现中还涉及进程、内存、安全校验等放方面面的内容。具体的链接过程我们在GNU LD中进行描述。


总结
^^^^^^
如果要了解ELF细节，可结合ELF规范和系统ELF相关头文件中结构定义进行理解，可借助reaelf,objdump等工具进行分析。此处我们已经理解ELF文件格式，重点在Linux内核中ELF文件的加载上。






	

	
了版本
