
内核make目标分析
------------
除了我们常用的几个目标，还有很多深入内核开发需要掌握的目标。

.. code-block:: sh
   :caption: make 参数
   :emphasize-lines: 2
   :linenos:
	
	Cleaning targets:
	  clean		  - Remove most generated files but keep the config and
	                    enough build support to build external modules
	  mrproper	  - Remove all generated files + config + various backup files
	  distclean	  - mrproper + remove editor backup and patch files
	
	Configuration targets:
	  config	  - Update current config utilising a line-oriented program
	  nconfig         - Update current config utilising a ncurses menu based program
	  menuconfig	  - Update current config utilising a menu based program
	  xconfig	  - Update current config utilising a Qt based front-end
	  gconfig	  - Update current config utilising a GTK+ based front-end
	  oldconfig	  - Update current config utilising a provided .config as base
	  localmodconfig  - Update current config disabling modules not loaded
	                    except those preserved by LMC_KEEP environment variable
	  localyesconfig  - Update current config converting local mods to core
	                    except those preserved by LMC_KEEP environment variable
	  defconfig	  - New config with default from ARCH supplied defconfig
	  savedefconfig   - Save current config as ./defconfig (minimal config)
	  allnoconfig	  - New config where all options are answered with no
	  allyesconfig	  - New config where all options are accepted with yes
	  allmodconfig	  - New config selecting modules when possible
	  alldefconfig    - New config with all symbols set to default
	  randconfig	  - New config with random answer to all options
	  yes2modconfig	  - Change answers from yes to mod if possible
	  mod2yesconfig	  - Change answers from mod to yes if possible
	  listnewconfig   - List new options
	  helpnewconfig   - List new options and help text
	  olddefconfig	  - Same as oldconfig but sets new symbols to their
	                    default value without prompting
	  tinyconfig	  - Configure the tiniest possible kernel
	  testconfig	  - Run Kconfig unit tests (requires python3 and pytest)
	
	Other generic targets:
	  all		  - Build all targets marked with [*]
	* vmlinux	  - Build the bare kernel
	* modules	  - Build all modules
	  modules_install - Install all modules to INSTALL_MOD_PATH (default: /)
	  dir/            - Build all files in dir and below
	  dir/file.[ois]  - Build specified target only
	  dir/file.ll     - Build the LLVM assembly file
	                    (requires compiler support for LLVM assembly generation)
	  dir/file.lst    - Build specified mixed source/assembly target only
	                    (requires a recent binutils and recent build (System.map))
	  dir/file.ko     - Build module including final link
	  modules_prepare - Set up for building external modules
	  tags/TAGS	  - Generate tags file for editors
	  cscope	  - Generate cscope index
	  gtags           - Generate GNU GLOBAL index
	  kernelrelease	  - Output the release version string (use with make -s)
	  kernelversion	  - Output the version stored in Makefile (use with make -s)
	  image_name	  - Output the image name (use with make -s)
	  headers_install - Install sanitised kernel headers to INSTALL_HDR_PATH
	                    (default: ./usr)
	
	Static analysers:
	  checkstack      - Generate a list of stack hogs
	  versioncheck    - Sanity check on version.h usage
	  includecheck    - Check for duplicate included header files
	  export_report   - List the usages of all exported symbols
	  headerdep       - Detect inclusion cycles in headers
	  coccicheck      - Check with Coccinelle
	  clang-analyzer  - Check with clang static analyzer
	  clang-tidy      - Check with clang-tidy
	
	Tools:
	  nsdeps          - Generate missing symbol namespace dependencies
	
	Kernel selftest:
	  kselftest         - Build and run kernel selftest
	                      Build, install, and boot kernel before
	                      running kselftest on it
	                      Run as root for full coverage
	  kselftest-all     - Build kernel selftest
	  kselftest-install - Build and install kernel selftest
	  kselftest-clean   - Remove all generated kselftest files
	  kselftest-merge   - Merge all the config dependencies of
			      kselftest to existing .config.
	
	Userspace tools targets:
	  use "make tools/help"
	  or  "cd tools; make help"
	
	Kernel packaging:
	  rpm-pkg             - Build both source and binary RPM kernel packages
	  binrpm-pkg          - Build only the binary kernel RPM package
	  deb-pkg             - Build both source and binary deb kernel packages
	  bindeb-pkg          - Build only the binary kernel deb package
	  snap-pkg            - Build only the binary kernel snap package
	                        (will connect to external hosts)
	  dir-pkg             - Build the kernel as a plain directory structure
	  tar-pkg             - Build the kernel as an uncompressed tarball
	  targz-pkg           - Build the kernel as a gzip compressed tarball
	  tarbz2-pkg          - Build the kernel as a bzip2 compressed tarball
	  tarxz-pkg           - Build the kernel as a xz compressed tarball
	  perf-tar-src-pkg    - Build perf-5.14.16.tar source tarball
	  perf-targz-src-pkg  - Build perf-5.14.16.tar.gz source tarball
	  perf-tarbz2-src-pkg - Build perf-5.14.16.tar.bz2 source tarball
	  perf-tarxz-src-pkg  - Build perf-5.14.16.tar.xz source tarball
	
	Documentation targets:
	 Linux kernel internal documentation in different formats from ReST:
	  htmldocs        - HTML
	  latexdocs       - LaTeX
	  pdfdocs         - PDF
	  epubdocs        - EPUB
	  xmldocs         - XML
	  linkcheckdocs   - check for broken external links
	                    (will connect to external hosts)
	  refcheckdocs    - check for references to non-existing files under
	                    Documentation
	  cleandocs       - clean all generated files
	
	  make SPHINXDIRS="s1 s2" [target] Generate only docs of folder s1, s2
	  valid values for SPHINXDIRS are: PCI RCU accounting admin-guide arm arm64 block bpf cdrom core-api cpu-freq crypto dev-tools devicetree doc-guide driver-api fault-injection fb filesystems firmware-guide fpga gpu hid hwmon i2c ia64 ide iio infiniband input isdn kbuild kernel-hacking leds livepatch locking m68k maintainer mhi mips misc-devices netlabel networking nios2 openrisc parisc pcmcia power powerpc process riscv s390 scheduler scsi security sh sound sparc spi staging target timers trace translations usb userspace-api virt vm w1 watchdog x86 xtensa
	
	  make SPHINX_CONF={conf-file} [target] use *additional* sphinx-build
	  configuration. This is e.g. useful to build with nit-picking config.
	
	  Default location for the generated documents is Documentation/output
	
	Architecture specific targets (x86):
	* bzImage		- Compressed kernel image (arch/x86/boot/bzImage)
	  install		- Install kernel using (your) ~/bin/installkernel or
				  (distribution) /sbin/installkernel or install to 
				  $(INSTALL_PATH) and run lilo
	
	  fdimage		- Create 1.4MB boot floppy image (arch/x86/boot/fdimage)
	  fdimage144		- Create 1.4MB boot floppy image (arch/x86/boot/fdimage)
	  fdimage288		- Create 2.8MB boot floppy image (arch/x86/boot/fdimage)
	  hdimage		- Create a BIOS/EFI hard disk image (arch/x86/boot/hdimage)
	  isoimage		- Create a boot CD-ROM image (arch/x86/boot/image.iso)
				  bzdisk/fdimage*/hdimage/isoimage also accept:
				  FDARGS="..."  arguments for the booted kernel
	                  	  FDINITRD=file initrd for the booted kernel
	
	  kvm_guest.config	- Enable Kconfig items for running this kernel as a KVM guest
	  xen.config		- Enable Kconfig items for running this kernel as a Xen guest
	
	  i386_defconfig              - Build for i386
	  x86_64_defconfig            - Build for x86_64
	
	  make V=0|1 [targets] 0 => quiet build (default), 1 => verbose build
	  make V=2   [targets] 2 => give reason for rebuild of target
	  make O=dir [targets] Locate all output files in "dir", including .config
	  make C=1   [targets] Check re-compiled c source with $CHECK
	                       (sparse by default)
	  make C=2   [targets] Force check of all c source with $CHECK
	  make RECORDMCOUNT_WARN=1 [targets] Warn about ignored mcount sections
	  make W=n   [targets] Enable extra build checks, n=1,2,3 where
			1: warnings which may be relevant and do not occur too often
			2: warnings which occur quite often but may still be relevant
			3: more obscure warnings, can most likely be ignored
			Multiple levels can be combined with W=12 or W=123
	
	Execute "make" or "make all" to build all targets marked with [*] 

更深入描述：

   make driver/tty/sysrq.i  //预编译
   make driver/tty/sysrq.s  //产生汇编代码
   make driver/tty/sysrq.o // 产生对象文件
   
- 清除目标：
	a. clean	-删除大多数产生的文件，保持config和构建外部模块需要的支持文件不变。	
	b. mrproper	-删除所有产生的文件、config文件和各种备份文件。
	c. distclean	- 其除实现mrproper功能外删除编辑器备份和补丁文件。

- 目标配置：


- 其他通用目标：
	a. all		  - 构建b和c中构建的目标
	b. * vmlinux	  - 构建裸内核
	c. * modules	  - 构建所有模块
	e. modules_install - 安装所有内核模块到路径 INSTALL_MOD_PATH (default: /)
	f. dir/            - 构建目录dir下的所有文件
	g. dir/file.[ois]  - 构建指定的目标
		   1. make driver/tty/sysrq.i  //预编译
		   2. make driver/tty/sysrq.s  //产生汇编代码
		   3. make driver/tty/sysrq.o // 产生对象文件
	h. dir/file.ll     - 构建LLVM汇编文件（需要编译器支持LLVM汇编功能）
	
	i. dir/file.lst    - 构建指定的源码/汇编代码混合目标(需要最近的 binutils 和最近的构建 (System.map))
	j. dir/file.ko     - 构建模块，包括最终链接
	k. modules_prepare - 设置用于构建外部模块环境
	l. tags/TAGS	  - 为编辑器生成标签文件 （如何理解？）
	m. cscope	  - 产生 cscope 索引
	n. gtags           - 产生 GNU 全局索引
	o. kernelrelease	  - 输出发布版本字符串 ( use with make -s)
	p. kernelversion	  - 输出Makefile保存的 version  (use with make -s)
	q. image_name	  - 输出镜像名称 (use with make -s)
	r. headers_install - 安装经过清理的内核头文件到路径INSTALL_HDR_PATH(default: ./usr)


- 静态分析器：

	a. checkstack      - 生成堆栈占用列表
	b. versioncheck    - version.h 使用情况的健全性检查
	c. includecheck    - 检查重复的包含头文件
	d. export_report   - 列出所有导出符号的用法
	e. headerdep       - 检测标头中的包含周期
	f. coccicheck      - Coccinelle 检查
	g. clang-analyzer  - 用 clang 静态分析器检查
	h. clang-tidy      - 用 clang-tidy 检查
	




- 工具：
	a. nsdeps          - 生成缺少的符号命名空间依赖项

- 内核自测：
	a. kselftest         - 构建和运行内核 selftest 在运行 kselftest 之前构建、安装和引导内核 以 root 身份运行以获得全面覆盖
	b. kselftest-all     - 构建内核自检
	c. kselftest-install - 构建和安装内核自检
	d. kselftest-clean   - 删除所有生成的 kselftest 文件
	e. kselftest-merge   - 将 kselftest 的所有配置依赖项合并到现有的 .config 中。

- 用户空间工具目标：
	a. use "make tools/help"
	b. "cd tools; make help"


- 内核打包：
	a. rpm-pkg             - 构建源代码和二进制 RPM 内核包
	b. binrpm-pkg          - 仅构建二进制内核 RPM 包
	c. deb-pkg             - 构建源代码和二进制 deb 内核包
	d. bindeb-pkg          - 仅构建二进制内核 deb 包
	e. snap-pkg            - 仅构建二进制内核快照包（将连接到外部主机）
	f. dir-pkg             - 将内核构建为普通目录结构
	g. tar-pkg             - 将内核构建为未压缩的 tarball
	h. targz-pkg           - 将内核构建为 gzip 压缩 tarball
	i. tarbz2-pkg          - 将内核构建为 bgzip2 压缩 tarball
	j. tarxz-pkg           - 将内核构建为 xz 压缩 tarball
	k. perf-tar-src-pkg    - 构建 perf-5.14.16.tar 源 tarball
	l. perf-targz-src-pkg  - 构建 perf-5.14.16.tar.gz 源 tarball
	m. perf-tarbz2-src-pkg - 构建 perf-5.14.16.tar.bz2 源 tarball
	n. perf-tarxz-src-pkg  - 构建 perf-5.14.16.tar.xz 源 tarball
	

- 文档目标

	从ReST构建内核文档:
	a. htmldocs        - HTML
	b. latexdocs       - LaTeX
	c. pdfdocs         - PDF
	d. epubdocs        - EPUB
	e. xmldocs         - XML
	f. linkcheckdocs   - check for broken external links
	                    (will connect to external hosts)
	g. refcheckdocs    - check for references to non-existing files under
	                    Documentation
	h. cleandocs       - clean all generated files


- 架构相关目标 (x86):

	a. * bzImage		- 压缩后的内核镜像 (arch/x86/boot/bzImage)
	b. install		- Install kernel using (your) ~/bin/installkernel or
				  (distribution) /sbin/installkernel or install to 
				  $(INSTALL_PATH) and run lilo
	
	c. fdimage		- Create 1.4MB boot floppy image (arch/x86/boot/fdimage)
	d. fdimage144		- Create 1.4MB boot floppy image (arch/x86/boot/fdimage)
	e. fdimage288		- Create 2.8MB boot floppy image (arch/x86/boot/fdimage)
	f. hdimage		- Create a BIOS/EFI hard disk image (arch/x86/boot/hdimage)
	g. isoimage		- Create a boot CD-ROM image (arch/x86/boot/image.iso)
				  bzdisk/fdimage*/hdimage/isoimage also accept:
				  FDARGS="..."  arguments for the booted kernel
	                  	  FDINITRD=file initrd for the booted kernel
	
	h. kvm_guest.config	- 启用 Kconfig 项以将此内核作为 KVM 客体运行
	i. xen.config		- 启用 Kconfig 项以将此内核作为 Xen 客体运行
	
	j. i386_defconfig              - Build for i386
	k. x86_64_defconfig            - Build for x86_64
	
	l. make V=0|1 [targets] 0 => quiet build (default), 1 => verbose build
	m. make V=2   [targets] 2 => 给出重建目标的理由
	n. make O=dir [targets] Locate all output files in "dir", including .config
	o. make C=1   [targets] 使用 $CHECK 检查重新编译的 c 源代码 (sparse by default)
	p. make C=2   [targets] 使用 $CHECK 强制检查所有 c 源代码
	q. make RECORDMCOUNT_WARN=1 [targets] 警告忽略的 mcount 部分
	r. make W=n   [targets] 启用额外的构建检查, n=1,2,3 where
			1. 可能相关且不经常出现的警告
			2. 经常出现但可能仍然相关的警告
			3. 更模糊的警告，很可能会被忽略
			多个级别可以结合 W=12 or W=123
	
	执行"make"或"make all"来构建所有加[*]的目标。










