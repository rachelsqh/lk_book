yocto uboot与内核模块、内核开发总结
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
- 文档手册：https://docs.yoctoproject.org/

以qemuarm64为例进行说明
uboot相关指令
*************
- uboot编译、部署指令：

.. code-block:: c
	:caption: u-boot编译、部署
	:linenos:
		
	$ bitbake -c compile -f u-boot
	$ bitbake -c deploy -f u-boot        //部署编译生成的u-boot镜像到deploy
	
- uboot配置：

.. code-block:: c
	:caption: u-boot配置
	:linenos:
		
	bitbake -c menuconfig u-boot
	
- 编译指令

.. code-block:: c
	:caption: u-boot编译
	:linenos:
		
	bitbake u-boot
	
kernel相关指令
**************

.. code-block:: c
	:caption: 编译、部署、配置内核
	:linenos:	
	
	bitbake -c compile -f virtual/kernel       //编译内核
	bitbake -c deploy -f virtual/kernel        //部署内核镜像到deploy目录
	bitbake -c menuconfig virtual/kernel 
	
linux 内核模块编译：
*******************
- 基本流程：
  1.新建layer;
  2.添加内核模块代码，创建对应的bb文件；
  3.田间layer;
  4.bitbake *.bb;
- 实际操作：
1.新建layer：

.. code-block:: c
	:caption: 新建layer:k_mod_sample
	:linenos:	
	
	$bitbake-layer create-layer k_mod_sample
	$tree k_mod_sample
	k_mod_sample/ 
	|-- COPYING.MIT
	|-- README
	|-- conf
	|   `-- layer.conf
	`-- recipes-example
	    `-- example
		`-- example_0.1.bb

	3 directories, 4 files
	
2.放入内核代码，创建对应bb文件：
  a. 放入内核代码

	.. code-block:: c
		:caption: 代码文件放置
		:linenos:
				
		$cd k_mod_sample
		$rm -rf recipes-example
		$mkdir recipes-ksample
		$cd recipes-ksample
		$mkdir ksample
		$cd ksample
		$cp -rf /usr/src/k_mod_sample files
		$cp poky/meta-skeleton/recipes-kernel/hello-mod/hello-mod_0.1.bb ksample_0.1.bb
		$cp poky/meta-skeleton/recipes-kernel/hello-mod/files/COPYING files/
		$cp poky/meta-skeleton/recipes-kernel/hello-mod/files/Makefile files/
		
  b. 修改files/Makefile文件

	.. code-block:: c
		:caption: 修改后的Makefile
		:linenos:
				  
		  $cat files/Makefile:
		   
		     obj-m := k_mod_sample.o
		     SRC := $(shell pwd)
		     all:
			     $(MAKE) -C $(KERNEL_SRC) M=$(SRC) modules
		     modules_install:
			     $(MAKE) -C $(KERNEL_SRC) M=$(SRC) modules_install
		     clean:
		     	     rm -f *.o *~ core .depend .*.cmd *.ko *.mod.c
			     rm -f Module.markers Module.symvers modules.order
			     rm -rf .tmp_versions Modules.symvers
			     
  c. 修改files/ksample_0.1.bb后内容为：
   
	.. code-block:: c
		:caption: 修改后的ksample_0.1.bb
		:linenos:   	
			
	        $cat files/ksample_0.1.bb
	        SUMMARY = "Example of how to build an external Linux kernel module"
	        DESCRIPTION = "${SUMMARY}"
	        LICENSE = "GPLv2"
	        LIC_FILES_CHKSUM = "file://COPYING;md5=12f884d2ae1ff87c09e5b7ccc2c4ca7e"


	        inherit module
	        SRC_URI = "file://k_mod_sample.c \
		         file://Makefile \
		        file://COPYING \
		       "
	        S = "${WORKDIR}"

	        # The inherit of module.bbclass will automatically name module packages with
	        # "kernel-module-" prefix as required by the oe-core build environment.

	        RPROVIDES_${PN} += "kernel-module-k_mod_sample"

	        KERNEL_MODULE_AUTOLOAD += "k_mod_sample"
	        
3. 添加层：
   
.. code-block:: c
	:caption: 添加层k_mod_sample
	:linenos:   	
	
	$bitbake-layer add-layer k_mod_sample
	
4. 编译模块：

   
.. code-block:: c
	:caption: 编译
	:linenos:   	
	
	$ bitbake ksample
        $ find -name k_mod_sample.ko
        $ ./tmp/work/qemuarm64-poky-linux/ksample/0.1-r0/k_mod_sample.ko
        
linux 内核开发
***************
- 参考：https://docs.yoctoproject.org/kernel-dev/common.html

 















