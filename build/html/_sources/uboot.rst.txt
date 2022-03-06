uboot理解
^^^^^^^^^
uboot配置
"""""""""
- make menuconfig:根据参数进行各种配置，其配置与linux的内核配置思想完全一样。具体配置参数要配合目标硬件进行设置。
uboot烧写
"""""""""
1.启动系统进入uboot操作界面；

2.tftp方式烧写新的uboot;

3.重启

注意：如果uboot烧写失败或中途中断，uboot就成板砖了，此时只能将芯片拿下来通过烧写器进行uboot烧写。

内核，文件系统烧写
""""""""""""""""
vmlinuz和zImage和uImage(引用自：https://www.daimajiaoliu.com/daima/47949fbd1100406）

1.uboot经过编译直接生成的elf格式的可执行程序是u-boot，这个程序类似于windows下的exe格式，在操作系统下是可以直接执行的。但是这种格式不能用来烧录下载。我们用来烧录下载的是u-boot.bin，这个东西是由u-boot使用arm-linux-objcopy工具进行加工（主要目的是去掉一些无用的）得到的。这个u-boot.bin就叫镜像（image），镜像就是用来烧录到iNand中执行的。

2.linux内核经过编译后也会生成一个elf格式的可执行程序，叫vmlinux或vmlinuz，这个就是原始的未经任何处理加工的原版内核elf文件；嵌入式系统部署时烧录的一般不是这个vmlinuz/vmlinux，而是要用objcopy工具去制作成烧录镜像格式（就是u-boot.bin这种，但是内核没有.bin后缀），经过制作加工成烧录镜像的文件就叫Image（制作把78M大的精简成了7.5M，因此这个制作烧录镜像主要目的就是缩减大小，节省磁盘）。

3.原则上Image就可以直接被烧录到Flash上进行启动执行（类似于u-boot.bin），但是实际上并不是这么简单。实际上linux的作者们觉得Image还是太大了所以对Image进行了压缩，并且在image压缩后的文件的前端附加了一部分解压缩代码。构成了一个压缩格式的镜像就叫zImage。（因为当年Image大小刚好比一张软盘（软盘有2种，1.2M的和1.44MB两种）大，为了节省1张软盘的钱于是乎设计了这种压缩Image成zImage的技术）。

4.uboot为了启动linux内核，还发明了一种内核格式叫uImage。uImage是由zImage加工得到的，uboot中有一个工具，可以将zImage加工生成uImage。注意：uImage不关linux内核的事，linux内核只管生成zImage即可，然后uboot中的mkimage工具再去由zImage加工生成uImage来给uboot启动。这个加工过程其实就是在zImage前面加上64字节的uImage的头信息即可。

5.原则上uboot启动时应该给他uImage格式的内核镜像，但是实际上uboot中也可以支持zImage，是否支持就看x210_sd.h中是否定义了LINUX_ZIMAGE_MAGIC这个宏。所以大家可以看出：有些uboot是支持zImage启动的，有些则不支持。但是所有的uboot肯定都支持uImage启动。

- 内核烧写：如：

	.. code-block:: c
		:caption: tftp烧写内核
		:linenos:
	  	
	  	tftp addr uImage
  
- 文件系统烧写：

	.. code-block:: c
		:caption: tftp烧写文件系统
		:linenos:
	  	
	  	tftp xxx fs_qtopia.yaffs2
  
总结
"""""""
具体应用中根据特定板卡或芯片，厂商会提供相应的包含操作手册的软件套装，我们可以根据具体软件套装进行操作。

