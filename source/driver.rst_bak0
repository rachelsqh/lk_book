设备驱动管理框架
^^^^^^^^^^^^^^^^^^
ACPI协议
"""""""""









开源固件
"""""""""

ARM Device Tree起源于OpenFirmware (OF)，在过去的Linux中，arch/arm/plat-xxx和arch/arm/mach-xxx中充斥着大量的垃圾代码，相当多数的代码只是在描述板级细节，而这些板级细节对于内核来讲，不过是垃圾，如板上的platform设备、resource、i2c_board_info、spi_board_info以及各种硬件的platform_data。为了改变这种局面，Linux社区的大牛们参考了PowerPC等体系架构中使用的Flattened Device Tree（FDT），也采用了Device Tree结构，许多硬件的细节可以直接透过它传递给Linux，而不再需要在kernel中进行大量的冗余编码。

    Device Tree是一种描述硬件的数据结构，由一系列被命名的结点（node）和属性（property）组成，而结点本身可包含子结点。所谓属性，其实就是成对出现的name和value。在Device Tree中，可描述的信息包括（原先这些信息大多被hard code到kernel中）：CPU的数量和类别，内存基地址和大小，总线和桥，外设连接，中断控制器和中断使用情况，GPIO控制器和GPIO使用情况，Clock控制器和Clock使用情况。 通常由.dts文件以文本方式对系统设备树进行描述，经过Device Tree Compiler(dtc)将dts文件转换成二进制文件binary device tree blob(dtb)，.dtb文件可由Linux内核解析，有了device tree就可以在不改动Linux内核的情况下，对不同的平台实现无差异的支持，只需更换相应的dts文件，即可满足，当然这样会增加内核的体积。
    
https://e-mailky.github.io/2016-12-06-dts-introduce
    
    




设备树
"""""""
dts格式，
dtb,
Makefile.dtbinst
install命令：
复制文件（通常是编译过的）到指定的目标位置。
-D：创建除最后一个以外的 DEST 的所有前导组件，或 --target-directory 的所有组件，然后将 SOURCE 复制到 DEST

$@  表示目标文件
$^  表示所有的依赖文件
$<  表示第一个依赖文件
$?  表示比目标还要新的依赖文件列表


quiet_cmd_dtb_install = INSTALL $@
      cmd_dtb_install = install -D $< $@

所以：
install -D $(boj)/%.dtb $(dst)/%.dtb
   
   
   
dtbinst := -f $(srctree)/scripts/Makefile.dtbinst obj   


Makefile:

# ---------------------------------------------------------------------------
# Devicetree files

ifneq ($(wildcard $(srctree)/arch/$(SRCARCH)/boot/dts/),)
dtstree := arch/$(SRCARCH)/boot/dts
endif

ifneq ($(dtstree),)

%.dtb: include/config/kernel.release scripts_dtc
	$(Q)$(MAKE) $(build)=$(dtstree) $(dtstree)/$@

%.dtbo: include/config/kernel.release scripts_dtc
	$(Q)$(MAKE) $(build)=$(dtstree) $(dtstree)/$@

PHONY += dtbs dtbs_install dtbs_check
dtbs: include/config/kernel.release scripts_dtc
	$(Q)$(MAKE) $(build)=$(dtstree)

ifneq ($(filter dtbs_check, $(MAKECMDGOALS)),)
export CHECK_DTBS=y
dtbs: dt_binding_check
endif

dtbs_check: dtbs

dtbs_install:
	$(Q)$(MAKE) $(dtbinst)=$(dtstree) dst=$(INSTALL_DTBS_PATH)

ifdef CONFIG_OF_EARLY_FLATTREE
all: dtbs
endif

endif

PHONY += scripts_dtc
scripts_dtc: scripts_basic
	$(Q)$(MAKE) $(build)=scripts/dtc

ifneq ($(filter dt_binding_check, $(MAKECMDGOALS)),)
export CHECK_DT_BINDING=y
endif

PHONY += dt_binding_check
dt_binding_check: scripts_dtc
	$(Q)$(MAKE) $(build)=Documentation/devicetree/bindings

# ---------------------------------------------------------------------------

设备树的一般操作方式是：开发人员根据开发需求编写dts文件，然后使用dtc将dts编译成dtb文件。

dts文件是文本格式的文件，而dtb是二进制文件，在linux启动时被加载到内存中，接下来我们需要来分析设备树dtb文件的格式。

dtb作为二进制文件被加载到内存中，然后由内核读取并进行解析，如果对dtb文件的格式不了解，那么在看设备树解析相关的内核代码时将会寸步难行，而阅读源代码才是了解设备树最好的方式，所以，如果需要更透彻的了解设备树解析的细节，第一步就是需要了解设备树的格式。

vmlinux.s:
__dtb_start = .; KEEP(*(.dtb.init.rodata)) __dtb_end = .;

我们看.dtb.init.rodata节：


script/Makefile.lib:332

# DTC
# ---------------------------------------------------------------------------
DTC ?= $(objtree)/scripts/dtc/dtc
DTC_FLAGS += -Wno-interrupt_provider

# Disable noisy checks by default
ifeq ($(findstring 1,$(KBUILD_EXTRA_WARN)),)
DTC_FLAGS += -Wno-unit_address_vs_reg \
	-Wno-unit_address_format \
	-Wno-avoid_unnecessary_addr_size \
	-Wno-alias_paths \
	-Wno-graph_child_address \
	-Wno-simple_bus_reg \
	-Wno-unique_unit_address \
	-Wno-pci_device_reg
endif

ifneq ($(findstring 2,$(KBUILD_EXTRA_WARN)),)
DTC_FLAGS += -Wnode_name_chars_strict \
	-Wproperty_name_chars_strict \
	-Winterrupt_provider
endif

DTC_FLAGS += $(DTC_FLAGS_$(basetarget))

# Set -@ if the target is a base DTB that overlay is applied onto
DTC_FLAGS += $(if $(filter $(patsubst $(obj)/%,%,$@), $(base-dtb-y)), -@)

# Generate an assembly file to wrap the output of the device tree compiler
quiet_cmd_dt_S_dtb= DTB     $@
cmd_dt_S_dtb=						\
{							\
	echo '\#include <asm-generic/vmlinux.lds.h>'; 	\
	echo '.section .dtb.init.rodata,"a"';		\
	echo '.balign STRUCT_ALIGNMENT';		\
	echo '.global __dtb_$(subst -,_,$(*F))_begin';	\
	echo '__dtb_$(subst -,_,$(*F))_begin:';		\
	echo '.incbin "$<" ';				\
	echo '__dtb_$(subst -,_,$(*F))_end:';		\
	echo '.global __dtb_$(subst -,_,$(*F))_end';	\
	echo '.balign STRUCT_ALIGNMENT'; 		\
} > $@

$(obj)/%.dtb.S: $(obj)/%.dtb FORCE
	$(call if_changed,dt_S_dtb)

quiet_cmd_dtc = DTC     $@
cmd_dtc = $(HOSTCC) -E $(dtc_cpp_flags) -x assembler-with-cpp -o $(dtc-tmp) $< ; \
	$(DTC) -o $@ -b 0 \
		$(addprefix -i,$(dir $<) $(DTC_INCLUDE)) $(DTC_FLAGS) \
		-d $(depfile).dtc.tmp $(dtc-tmp) ; \
	cat $(depfile).pre.tmp $(depfile).dtc.tmp > $(depfile)

$(obj)/%.dtb: $(src)/%.dts $(DTC) FORCE
	$(call if_changed_dep,dtc)

$(obj)/%.dtbo: $(src)/%.dts $(DTC) FORCE
	$(call if_changed_dep,dtc)

quiet_cmd_fdtoverlay = DTOVL   $@
      cmd_fdtoverlay = $(objtree)/scripts/dtc/fdtoverlay -o $@ -i $(real-prereqs)

$(multi-dtb-y): FORCE
	$(call if_changed,fdtoverlay)
$(call multi_depend, $(multi-dtb-y), .dtb, -dtbs)

DT_CHECKER ?= dt-validate
DT_CHECKER_FLAGS ?= $(if $(DT_SCHEMA_FILES),,-m)
DT_BINDING_DIR := Documentation/devicetree/bindings
# DT_TMP_SCHEMA may be overridden from Documentation/devicetree/bindings/Makefile
DT_TMP_SCHEMA ?= $(objtree)/$(DT_BINDING_DIR)/processed-schema.json

quiet_cmd_dtb_check =	CHECK   $@
      cmd_dtb_check =	$(DT_CHECKER) $(DT_CHECKER_FLAGS) -u $(srctree)/$(DT_BINDING_DIR) -p $(DT_TMP_SCHEMA) $@

define rule_dtc
	$(call cmd_and_fixdep,dtc)
	$(call cmd,dtb_check)
endef

$(obj)/%.dt.yaml: $(src)/%.dts $(DTC) $(DT_TMP_SCHEMA) FORCE
	$(call if_changed_rule,dtc)

dtc-tmp = $(subst $(comma),_,$(dot-target).dts.tmp)


我们看
quiet_cmd_dt_S_dtb= DTB     $@
cmd_dt_S_dtb=						\
{							\
	echo '\#include <asm-generic/vmlinux.lds.h>'; 	\
	echo '.section .dtb.init.rodata,"a"';		\
	echo '.balign STRUCT_ALIGNMENT';		\
	echo '.global __dtb_$(subst -,_,$(*F))_begin';	\
	echo '__dtb_$(subst -,_,$(*F))_begin:';		\
	echo '.incbin "$<" ';				\
	echo '__dtb_$(subst -,_,$(*F))_end:';		\
	echo '.global __dtb_$(subst -,_,$(*F))_end';	\
	echo '.balign STRUCT_ALIGNMENT'; 		\
} > $@

看指令执行处：

$(obj)/%.dtb.S: $(obj)/%.dtb FORCE
	$(call if_changed,dt_S_dtb)
	
所以这个节包含的是二进制文件：
$< ==> $(obj)/%.dtb


Makefile里的subst

　　用法是$(subst FROM,TO,TEXT),即将TEXT中的东西从FROM变为TO

　　Makefile中的字符串处理函数

　　格式：

　　$(subst <from>;,<to>;,<text>;)

　　名称：字符串替换函数——subst。

　　功能：把字串<text>;中的<from>;字符串替换成<to>;。

　　返回：函数返回被替换过后的字符串。

　　示例：

　　$(subst a,the,There is a big tree)，

　　把“There is a big tree”中的“a”替换成“the”，返回结果是“There is the big tree”。
   
 
 
所以：

__dtb_$(subst -,_,$(*F))_begin:
等价于：
*F:文件：

也就是说，

quiet_cmd_dt_S_dtb= DTB     $@
cmd_dt_S_dtb=						\
{							\
	echo '\#include <asm-generic/vmlinux.lds.h>'; 	\
	echo '.section .dtb.init.rodata,"a"';		\
	echo '.balign STRUCT_ALIGNMENT';		\
	echo '.global __dtb_$(subst -,_,$(*F))_begin';	\
	echo '__dtb_$(subst -,_,$(*F))_begin:';		\
	echo '.incbin "$<" ';				\
	echo '__dtb_$(subst -,_,$(*F))_end:';		\
	echo '.global __dtb_$(subst -,_,$(*F))_end';	\
	echo '.balign STRUCT_ALIGNMENT'; 		\
} > $@

看指令执行处：

$(obj)/%.dtb.S: $(obj)/%.dtb FORCE
	$(call if_changed,dt_S_dtb)

假设有一个te-st.dtb,会产生一个te-st.dtb.S,内容

#include <asm-generic/vmlinux.lds.h> 	
.section .dtb.init.rodata,"a"		
.balign STRUCT_ALIGNMENT		
.global __dtb_te_st_begin	
__dtb_te_st_begin:		
.incbin "$(obj)/te-st.dtb" 			
__dtb_te_st_end:	
.global __dtb_te_st_end	
.balign STRUCT_ALIGNMENT 		
 
也就是说：
内核映像vmlinux中的__dtb_start = .; KEEP(*(.dtb.init.rodata)) __dtb_end = .;节中包含dtb二进制文件。


dts --> dtb/dtbo --> dtb.S --> 内核镜像中；


dts -> dtb:

dtc:device tree compiler:设备树编译器： dtc [options] <input file>

dts:device tree source

dtb:device tree blob







-------------------------------

但是我还是简单说一下。。。dt主要由两种文件组成，分别是xx.dts和xx.dtsi，其中只有xx.dts文件才能生成对应的dtb/dtbo，dtsi文件是用来include的。
也就是说，一个dtb/dtbo文件中包含了

生成这个dtb/dtbo的dts文件内容
这个dts文件中include的dtsi文件内容
被include的dtsi文件中引用的其它dtsi文件内容
至于这里的include(引用)，其实在生成dtb时你可以简单的理解为复制粘贴，也就是把那个文件的内容替换到include的位置（（
还有一个非常关键的点，关系到dtbo的原理，那就是dt之间是可以互相覆盖的
比如1.dtsi引用了2.dtsi，那么1.dtsi就可以在include的下方重写2.dtsi中的节点

总结一下，编译dtb/dtbo的过程实际上先是一个合并+递归include的过程，其中谁距离dts文件越近，就具有越高的覆盖优先级，可以覆盖越多的节点而更难被别人覆盖

dtb是device tree binary的简称
binary，顾名思义，就是可以被bootloader直接读取执行的内容
它们在开机启动在早期阶段由bootloader解码，传递给内核，从而帮助内核完成启动过程

在较老的平台上（msm-3.18 / msm-4.4)，device tree只存在于boot分区中， 可以通过在Makefile中指定dtb-y += <名称>.dtb来编译对应的dtb文件（其中名称是指源dts的名称，也就是<名称>.dts）。这些文件将会被与内核的编译产物Image.xx连接，最终生成Image.xx-dtb，常见的有Image-dtb Image.gz-dtb Image.lz4-dtb等，而这个过程由CONFIG_BUILD_ARM64_APPENDED_DTB_IMAGE控制。在这个选项被关闭后，编译也会生成dtb文件，但不会主动连接至内核镜像。
dtbo
dtbo是device tree binary overlay的简称
在msm-4.9平台上，dtbo横空出世（准确来说是出厂搭载安卓9的要求）。device tree被拆分到了两个地方，一个是boot分区中的老位置，另一个则是dtbo分区。谷歌做这件事的初衷在于：希望分离芯片厂商和手机厂商的修改，芯片厂商只修改内核中的dtb，而手机厂商只修改dtbo分区，这样能够井井有条（（但是事实是手机厂商也还在改内核的dtb草
因此，就初衷而言，我们已经可以看出dtb和dtbo分区之间的关系

那么问题来了，谁的优先级更高呢？假如一个东西同时出现在dtb与dtbo中，谁会覆盖谁呢？

肯定是dtbo覆盖kernel dtb啊，不然它凭什么叫overlay…(((不过我并没有去验证（懒
在Makefile中，我们可以看到包含dtbo分区的设备的dt编译逻辑，和上方的旧平台有些许不同
我们可以通过dtbo-y += <名称>.dtbo来编译dtbo文件（和上方的dtb一样，名称来自于源文件<名称>.dts）
但是，同时我们需要指定dtbo的base，也就是这个叠加层是基于哪个dtb进行叠加覆盖的
<名称>.dtbo-base := <名称2>.dtb
在这样配置之后，编译内核时，编译系统将会编译对应的dtbo和dtb，并将dtb打包进入内核（前提是开启CONFIG_BUILD_ARM64_APPENDED_DTB_IMAGE），但是dtbo将会留在原处。厂商在编译系统时，dtbo文件是由编译系统的其他部分（非内核）处理并打包成为dtbo分区，生成dtbo镜像。
但是，我们依然可以在单跑内核时生成dtbo镜像。
我们需要摘下以下几个提交

dtb中装有芯片级配置，比如gpu频率表，这就是为什么gpu超频卡刷包里面是个dtb文件的原因（用来替换kernel dtb）
dtbo中装有厂商级的配置，比如屏幕、相机等，这就是为什么超刷新率改的是dtbo分区
具体，你可以去溯源，只需要追随着dts文件的include，就可以知道它们里面到底装了些什么。

以kona-v2.1（骁龙865）为例


dtb和dtbo文件是同一种东西

编译出来的dtb和dtbo文件的编码格式是完全一致的，它们仅仅只是后缀不一样
------------------------------------

1. linux设备树中DTS、 DTC和DTB的关系
(1) DTS：.dts文件是设备树的源文件。由于一个SoC可能对应多个设备，这些.dst文件可能包含很多共同的部分，共同的部分一般被提炼为一个 .dtsi 文件，这个文件相当于C语言的头文件。
(2) DTC：DTC是将.dts编译为.dtb的工具，相当于gcc。
(3) DTB：.dtb文件是 .dts 被 DTC 编译后的二进制格式的设备树文件，它可以被linux内核解析。


-------------------

DTS语法

和 C 语言一样，设备树也支持头文件，设备树的头文件扩展名为 .dtsi；同时也可以像C 语言一样包含 .h头文件；例如：（代码来源 linux-4.15/arch/arm/boot/dts/s3c2416.dtsi）

注：.dtsi 文件一般用于描述 SOC 的内部外设信息，比如 CPU 架构、主频、外设寄存器地址范围，比如 UART、 IIC 等等。

设备节点

在设备树中节点命名格式如下：

node-name@unit-address

node-name：是设备节点的名称，为ASCII字符串，节点名字应该能够清晰的描述出节点的功能，比如“uart1”就表示这个节点是UART1外设；unit-address：一般表示设备的地址或寄存器首地址，如果某个节点没有地址或者寄存器的话 “unit-address” 可以不要；注：根节点没有node-name 或者 unit-address，它被定义为 /。

设备节点的例子如下图：
......

在上图中：cpu 和 ethernet依靠不同的unit-address 分辨不同的CPU；可见，node-name相同的情况下，可以通过不同的unit-address定义不同的设备节点。
-------

2.2.1 设备节点的标准属性
2.2.1.1 compatible 属性
compatible 属性也叫做 “兼容性” 属性，这是非常重要的一个属性！compatible 属性的值是一个字符串列表， compatible 属性用于将设备和驱动绑定起来。字符串列表用于选择设备所要使用的驱动程序。compatible 属性值的推荐格式：
"manufacturer,model"

① manufacturer : 表示厂商；
② model : 一般是模块对应的驱动名字。
例如：
compatible = "fsl,mpc8641", "ns16550";


上面的compatible有两个属性，分别是 "fsl,mpc8641" 和 "ns16550"；其中 "fsl,mpc8641" 的厂商是 fsl；设备首先会使用第一个属性值在 Linux 内核里面查找，看看能不能找到与之匹配的驱动文件；

如果没找到，就使用第二个属性值查找，以此类推，直到查到到对应的驱动程序 或者 查找完整个 Linux 内核也没有对应的驱动程序为止。


注：一般驱动程序文件都会有一个 OF 匹配表，此 OF 匹配表保存着一些 compatible 值，如果设备节点的 compatible 属性值和 OF 匹配表中的任何一个值相等，那么就表示设备可以使用这个驱动。


2.2.1.2 model 属性
model 属性值也是一个字符串，一般 model 属性描述设备模块信息，比如名字什么的，例如：model = "Samsung S3C2416 SoC";

2.2.1.3 phandle 属性
phandle属性为devicetree中唯一的节点指定一个数字标识符，节点中的phandle属性，它的取值必须是唯一的(不要跟其他的phandle值一样)，例如：

pic@10000000 {
    phandle = <1>;
    interrupt-controller;
};
another-device-node {
    interrupt-parent = <1>;   // 使用phandle值为1来引用上述节点
};


注：DTS中的大多数设备树将不包含显式的phandle属性，当DTS被编译成二进制DTB格式时，DTC工具会自动插入phandle属性。

2.2.1.4 status 属性
status 属性看名字就知道是和设备状态有关的， status 属性值也是字符串，字符串是设备的状态信息，可选的状态如下表所示：



status值	描述
“okay”	表明设备是可操作的。
“disabled”	表明设备当前是不可操作的，但是在未来可以变为可操作的，比如热插拔设备插入以后。至于 disabled 的具体含义还要看设备的绑定文档。
“fail”	表明设备不可操作，设备检测到了一系列的错误，而且设备也不大可能变得可操作。
“fail-sss”	含义和“fail”相同，后面的 sss 部分是检测到的错误内容

2.2.1.5 #address-cells 和 #size-cells
#address-cells 和 #size-cells的值都是无符号 32 位整型，可以用在任何拥有子节点的设备中，用于描述子节点的地址信息。#address-cells 属性值决定了子节点 reg 属性中地址信息所占用的字长(32 位)， #size-cells 属性值决定了子节点 reg 属性中长度信息所占的字长(32 位)。#address-cells 和 #size-cells 表明了子节点应该如何编写 reg 属性值，一般 reg 属性都是和地址有关的内容，和地址相关的信息有两种：起始地址和地址长度，reg 属性的格式一为：

reg = <address1 length1 address2 length2 address3 length3……>


例如一个64位的处理器：

soc {
    #address-cells = <2>;
    #size-cells = <1>;
    serial {
        compatible = "xxx";
        reg = <0x4600 0x5000 0x100>;  /*地址信息是：0x00004600 00005000,长度信息是：0x100*/
        };
};


2.2.1.6 reg 属性
reg 属性的值一般是 (address， length) 对，reg 属性一般用于描述设备地址空间资源信息，一般都是某个外设的寄存器地址范围信息。

例如：一个设备有两个寄存器块，一个的地址是0x3000，占据32字节；另一个的地址是0xFE00，占据256字节，表示如下：

reg = <0x3000 0x20 0xFE00 0x100>;

2.2.1.7 ranges 属性
ranges属性值可以为空或者按照 (child-bus-address,parent-bus-address,length) 格式编写的数字矩阵， ranges 是一个地址映射/转换表， ranges 属性每个项目由子地址、父地址和地址空间长度这三部分组成：

child-bus-address：子总线地址空间的物理地址，由父节点的 #address-cells 确定此物理地址所占用的字长。
parent-bus-address：父总线地址空间的物理地址，同样由父节点的 #address-cells 确定此物理地址所占用的字长。
length：子地址空间的长度，由父节点的 #size-cells 确定此地址长度所占用的字长。

soc {
    compatible = "simple-bus";
    #address-cells = <1>;
    #size-cells = <1>;
    ranges = <0x0 0xe0000000 0x00100000>;
    serial {
        device_type = "serial";
        compatible = "ns16550";
        reg = <0x4600 0x100>;
        clock-frequency = <0>;
        interrupts = <0xA 0x8>;
        interrupt-parent = <&ipic>;
        };
};


节点 soc 定义的 ranges 属性，值为 <0x0 0xe0000000 0x00100000>，此属性值指定了一个 1024KB(0x00100000) 的地址范围，子地址空间的物理起始地址为 0x0，父地址空间的物理起始地址为 0xe0000000。

serial 是串口设备节点，

reg 属性定义了 serial 设备寄存器的起始地址为 0x4600，寄存器长度为 0x100。

经过地址转换， serial 设备可以从 0xe0004600 开始进行读写操作，0xe0004600=0x4600+0xe0000000。

2.2.1.8 name 属性
name 属性值为字符串， name 属性用于记录节点名字， name 属性已经被弃用，不推荐使用name 属性，一些老的设备树文件可能会使用此属性。

2.2.1.9 device_type 属性
device_type 属性值为字符串， IEEE 1275 会用到此属性，用于描述设备的 FCode，但是设备树没有 FCode，所以此属性也被抛弃了。此属性只能用于 cpu 节点或者 memory 节点。

memory@30000000 {
    device_type = "memory";
    reg =  <0x30000000 0x4000000>;
};

2.2.2 根节点
每个设备树文件只有一个根节点，其他所有的设备节点都是它的子节点，它的路径是 /。根节点有以下属性：

属性	属性值类型	描述
#address-cells	< u32 >	在它的子节点的reg属性中, 使用多少个u32整数来描述地址(address)
model	< string >	用于标识系统板卡(例如smdk2440开发板)，推荐的格式是“manufacturer,model-number”
compatible	< stringlist >	定义一系列的字符串, 用来指定内核中哪个machinedesc可以支持本设备
例如：compatible = "samsung,smdk2440","samsung,s3c24xx" ,内核会优先寻找支持smdk2440的machinedesc结构体，如果找不到才会继续寻找支持s3c24xx的machine_desc结构体(优先选择第一项，然后才是第二项，第三项……)

2.2.3 特殊节点
2.2.3.1 /aliases 子节点
aliases 节点的主要功能就是定义别名，定义别名的目的就是为了方便访问节点。

例如：定义 flexcan1 和 flexcan2 的别名是 can0 和 can1。

aliases {
    can0 = &flexcan1;
    can1 = &flexcan2;
};
2.2.3.2 /memory 子节点
所有设备树都需要一个memory设备节点，它描述了系统的物理内存布局。如果系统有多个内存块，可以创建多个memory节点，或者可以在单个memory节点的reg属性中指定这些地址范围和内存空间大小。

例如：一个64位的系统有两块内存空间：RAM1：起始地址是0x0，地址空间是 0x80000000；RAM2：起始地址是0x10000000，地址空间也是0x80000000；同时根节点下的 #address-cells = <2>和#size-cells = <2>，这个memory节点描述为：

memory@0 {
    device_type = "memory";
    reg = <0x00000000 0x00000000 0x00000000 0x80000000
           0x00000000 0x10000000 0x00000000 0x80000000>;
};
或者：

memory@0 {
    device_type = "memory";
    reg = <0x00000000 0x00000000 0x00000000 0x80000000>;
};
memory@10000000 {
    device_type = "memory";
    reg = <0x00000000 0x10000000 0x00000000 0x80000000>;
};
2.2.3.3 /chosen 子节点
chosen 并不是一个真实的设备， chosen 节点主要是为了 uboot 向 Linux 内核传递数据，重点是 bootargs 参数。例如：

chosen {
    bootargs = "root=/dev/nfs rw nfsroot=192.168.1.1 console=ttyS0,115200";
};
2.2.3.4 /cpus 和 /cpus/cpu* 子节点
cpus节点下有1个或多个cpu子节点, cpu子节点中用reg属性用来标明自己是哪一个cpu，所以 /cpus 中有以下2个属性:

#address-cells   // 在它的子节点的reg属性中, 使用多少个u32整数来描述地址(address)

#size-cells      // 在它的子节点的reg属性中, 使用多少个u32整数来描述大小(size)
                 // 必须设置为0
例如：

cpus {
    #address-cells = <1>;
    #size-cells = <0>;
    cpu@0 {
        device_type = "cpu";
        reg = <0>;
        cache-unified;
        cache-size = <0x8000>; // L1, 32KB
        cache-block-size = <32>;
        timebase-frequency = <82500000>; // 82.5 MHz
        next-level-cache = <&L2_0>; // phandle to L2
        L2_0:l2-cache {
            compatible = "cache";
            cache-unified;
            cache-size = <0x40000>; // 256 KB
            cache-sets = <1024>;
            cache-block-size = <32>;
            cache-level = <2>;
            next-level-cache = <&L3>; // phandle to L3
            L3:l3-cache {
                compatible = "cache";
                cache-unified;
                cache-size = <0x40000>; // 256 KB
                cache-sets = <0x400>; // 1024
                cache-block-size = <32>;
                cache-level = <3>;
                };
            };
        };
    cpu@1 {
        device_type = "cpu";
        reg = <1>;
        cache-unified;
        cache-block-size = <32>;
        cache-size = <0x8000>; // L1, 32KB
        timebase-frequency = <82500000>; // 82.5 MHzclock-frequency = <825000000>; // 825 MHz
        cache-level = <2>;
        next-level-cache = <&L2_1>; // phandle to L2
        L2_1:l2-cache {
            compatible = "cache";
            cache-unified;
            cache-size = <0x40000>; // 256 KB
            cache-sets = <0x400>; // 1024
            cache-line-size = <32>; // 32 bytes
            next-level-cache = <&L3>; // phandle to L3
            };
        };
};
2.2.3.5 引用其他节点
2.2.3.5.1 phandle
节点中的phandle属性, 它的取值必须是唯一的(不要跟其他的phandle值一样)

pic@10000000 {
    phandle = <1>;
    interrupt-controller;
};

another-device-node {
    interrupt-parent = <1>;   // 使用phandle值为1来引用上述节点
};
2.2.3.5.2 label
PIC: pic@10000000 {
    interrupt-controller;
};

another-device-node {
    interrupt-parent = <&PIC>;   // 使用label来引用上述节点, 
                                 // 使用lable时实际上也是使用phandle来引用, 
                                 // 在编译dts文件为dtb文件时, 编译器dtc会在dtb中插入phandle属性
};
2.2.4 DTB格式
.dtb文件是 .dts 被 DTC 编译后的二进制格式的设备树文件，它的文件布局如下：

。。。。。。。。。。。。。。


从上图可以看出，DTB文件主要包含四部分内容：struct ftdheader、memory reservation block、structure block、strings block；

① struct ftdheader：用来表明各个分部的偏移地址，整个文件的大小，版本号等；
② memory reservation block：在设备树中使用/memreserve/ 定义的保留内存信息；
③ structure block：保存节点的信息，节点的结构；
④ strings block：保存属性的名字，单独作为字符串保存；
struct ftd_header结构体的定义如下：


struct fdt_header {
    uint32_t magic; /*它的值为0xd00dfeed，以大端模式保存*/
    uint32_t totalsize; /*整个DTB文件的大小*/
    uint32_t off_dt_struct; /*structure block的偏移地址*/
    uint32_t off_dt_strings; /*strings block的偏移地址*/
    uint32_t off_mem_rsvmap; /*memory reservation block的偏移地址*/
    uint32_t version; /*设备树版本信息*/
    uint32_t last_comp_version; /*向后兼容的最低设备树版本信息*/
    uint32_t boot_cpuid_phys; /*CPU ID*/
    uint32_t size_dt_strings; /*strings block的大小*/
    uint32_t size_dt_struct; /*structure block的大小*/
};

fdtreserveentry结构体如下：

struct fdt_reserve_entry {
    uint64_t address;  /*64bit 的地址*/
    uint64_t size;    /*保留的内存空间的大小*/
};
该结构体用于表示memreserve的起始地址和内存空间的大小，它紧跟在struct ftdheader结构体后面。

例如：/memreserve/ 0x33000000 0x10000，fdtreserve_entry 结构体的成员 address = 0x33000000，size = 0x10000。

structure block是用于描述设备树节点的结构，保存着节点的信息、节点的结构，它有5种标记类型:

① FDTBEGINNODE (0x00000001)：表示节点的开始，它的后面紧跟的是节点的名字；
② FDTENDNODE (0x00000002)：表示节点的结束；
③ FDTPROP (0x00000003) ：表示开始描述节点里面的一个属性，在FDTPROP后面紧跟一个结构体如下所示:
struct {
    uint32_t len;       /*表示属性值的长度*/
    uint32_t nameoff;   /*属性的名字在string block的偏移*/
} 
注：上面的这个结构体后紧跟着是属性值，属性的名字保存在字符串块（Strings block）中。

④ FDT_END (0x00000009)：表示structure block的结束。
单个节点在structure block的存储格式如下图如所示：(注：子节点的存储格式也是一样)


总结：

(1) DTB文件可以分为四个部分:struct ftdheader、memory reservation block、structure block、strings block；
(2) 最开始的为struct ftdheader，包含其它三个部分的偏移地址；
(3) memory reservation block记录保留内存信息；
(4) structure block保存节点的信息，节点的结构；
(5) strings block保存属性的名字，将属性名字单独作为字符串保存；

2.2.5 DTB文件分析
编译生成dtb文件的源设备树jz2440.dts文件如下：

// SPDX-License-Identifier: GPL-2.0
/*
 * SAMSUNG SMDK2440 board device tree source
 *
 * Copyright (c) 2018 weidongshan@qq.com
 * dtc -I dtb -O dts -o jz2440.dts jz2440.dtb
 */

#define S3C2410_GPA(_nr)    ((0<<16) + (_nr))
#define S3C2410_GPB(_nr)    ((1<<16) + (_nr))
#define S3C2410_GPC(_nr)    ((2<<16) + (_nr))
#define S3C2410_GPD(_nr)    ((3<<16) + (_nr))
#define S3C2410_GPE(_nr)    ((4<<16) + (_nr))
#define S3C2410_GPF(_nr)    ((5<<16) + (_nr))
#define S3C2410_GPG(_nr)    ((6<<16) + (_nr))
#define S3C2410_GPH(_nr)    ((7<<16) + (_nr))
#define S3C2410_GPJ(_nr)    ((8<<16) + (_nr))
#define S3C2410_GPK(_nr)    ((9<<16) + (_nr))
#define S3C2410_GPL(_nr)    ((10<<16) + (_nr))
#define S3C2410_GPM(_nr)    ((11<<16) + (_nr))

/dts-v1/;

/ {
    model = "SMDK2440";
    compatible = "samsung,smdk2440";
    #address-cells = <1>;
    #size-cells = <1>;

    memory@30000000 {
        device_type = "memory";
        reg =  <0x30000000 0x4000000>;
    };

    chosen {
        bootargs = "console=ttySAC0,115200 rw root=/dev/mtdblock4 rootfstype=yaffs2";
    };

    led {
        compatible = "jz2440_led";
        reg = <S3C2410_GPF(5) 1>;
    };
};


jz2440.dtb 文件的内容如下：


。。。。。。。。。。。。。。

接下来我们对应上图的编号逐一分析，其中编号①~⑩表示的是fdtheader 结构体的成员信息：

① 对应 magic，表示设备树魔数，固定为0xd00dfeed；
② 对应 totalsize，表示整个设备设dtb文件的大小，从上图可知0x000001B9正好是dtb文件的大小441B；
③ 对应 offdtstruct，表示structure block的偏移地址，为 0x00000038；
④ 对应offdtstrings，表示 strings block的偏移地址，为 0x00000174；
⑤ 对应 offmemrsvmap;，表示memory reservation block的偏移地址，为 0x00000028；
⑥ 对应 version ，设备树版本的版本号为0x11；
⑦ 对应 lastcompversion，向下兼容版本号0x10；
⑧ 对应 bootcpuidphys，在多核处理器中用于启动的主cpu的物理id，为0x0；
⑨ 对应 sizedtstrings，strings block的大小为 0x45；
⑩ 对应 sizedtstruct，structure block的大小为 0x0000013C；
⑪~⑫ 对应结构体 fdtreserve_entry ，它所在的地址为0x28，jz2440.dts 设备树文件没有设置 /memreserve/，所以address = 0x0，size = 0x0；
⑬ 所处的地址是0x38，它处在structure block中，0x00000001表示的是设备节点的开始；
⑭ 接着紧跟的是设备节点的名字，这里是根节点，所以为0x00000000；
⑮ 0x00000003 表示的是开始描述设备节点的一个属性；
⑯ 表示这个属性值的长度为0x09；
⑰ 表示这个属性的名字在strings block的偏移量是0，找到strings block的地址0x0174的地方，可知这个属性的名字是model；
⑱ 这个model属性的值是"SMDK2440"，加上字符串的结束符NULL，刚好9个字节；


2.2.6 DTB文件结构图
(1) dtb 文件的结构图如下：
。。。。。。。。。。。。

Linux设备树语法规范 (2) 设备节点的结构图如下：

----------------
DeviceTree 内核 API
truct device_node * of_find_all_nodes( struct device_node  *prev )
获取全局列表中的下一个节点

参数

struct device_node *prev
of_node_put()将在其上调用上一个节点或开始迭代的 NULL

返回

引用计数递增的节点指针， of_node_put()完成后使用它。

struct device_node * of_get_cpu_node( int  cpu , unsigned int  *thread )
获取与给定逻辑 CPU 关联的设备节点

参数

int cpu
需要设备节点的 CPU 编号（逻辑索引）

unsigned int *thread
如果不为 NULL，则返回物理内核中的本地线程号

描述

此函数的主要目的是检索给定逻辑 CPU 索引的设备节点。它应该用于初始化 cpu 设备中的 of_node。一旦 cpu 设备中的 of_node 被填充，所有进一步的引用都可以使用它。

CPU 逻辑到物理索引映射是特定于体系结构的，并且是在引导辅助内核之前构建的。此函数使用arch_match_cpu_phys_id，它可以被体系结构特定的实现覆盖。

返回

逻辑 cpu 的节点指针，引用计数递增， of_node_put()完成后使用它。如果未找到，则返回 NULL。

struct device_node {
	const char *name;
	phandle phandle;
	const char *full_name;
	struct fwnode_handle fwnode;

	struct	property *properties;
	struct	property *deadprops;	/* removed properties */
	struct	device_node *parent;
	struct	device_node *child;
	struct	device_node *sibling;
#if defined(CONFIG_OF_KOBJ)
	struct	kobject kobj;
#endif
	unsigned long _flags;
	void	*data;
#if defined(CONFIG_SPARC)
	unsigned int unique_id;
	struct of_irq_controller *irq_trans;
#endif
};





linux/of_fdt.h:35:
extern char __dtb_start[];
extern char __dtb_end[];

----------------------------------
在linux中，开放固件设备树和扁平设备树有什么区别
在linux中，开放固件设备树和扁平设备树有什么区别。如何识别linux内核使用的是哪个Device tree OF DT或FDT。

设备树：open fireware：DTB

APCI:APCI设备表FDT？

ARM的核心代码仍然保存在arch/arm目录下
ARM SoC core architecture code保存在arch/arm目录下
ARM SOC的周边外设模块的驱动保存在drivers目录下
ARM SOC的特定代码在arch/arm/mach-xxx目录下
ARM SOC board specific的代码被移除，由DeviceTree机制来负责传递硬件拓扑和硬件资源信息。
本质上，Device Tree改变了原来用hardcode方式将HW 配置信息嵌入到内核代码的方法，改用bootloader传递一个DB的形式。

如果我们认为kernel是一个black box，那么其输入参数应该包括：

识别platform的信息
runtime的配置参数
设备的拓扑结构以及特性
对于嵌入式系统，在系统启动阶段，bootloader会加载内核并将控制权转交给内核，此外， 还需要把上述的三个参数信息传递给kernel，以便kernel可以有较大的灵活性。在linux kernel中，Device Tree的设计目标就是如此。

DTS的加载过程
如果要使用Device Tree，首先用户要了解自己的硬件配置和系统运行参数，并把这些信息组织成Device Tree source file。 通过DTC（Device Tree Compiler），可以将这些适合人类阅读的Device Tree source file变成适合机器处理的 Device Tree binary file（有一个更好听的名字，DTB，device tree blob）。在系统启动的时候，boot program （例如：firmware、bootloader）可以将保存在flash中的DTB copy到内存（当然也可以通过其他方式， 例如可以通过bootloader的交互式命令加载DTB，或者firmware可以探测到device的信息，组织成DTB保存在内存中）， 并把DTB的起始地址传递给client program（例如OS kernel，bootloader或者其他特殊功能的程序）。 对于计算机系统（computer system），一般是firmware->bootloader->OS，对于嵌入式系统，一般是bootloader->OS。




DTS的描述信息
Device Tree由一系列被命名的结点（node）和属性（property）组成，而结点本身可包含子结点。所谓属性， 其实就是成对出现的name和value。在Device Tree中，可描述的信息包括（原先这些信息大多被hard code到kernel中）：

CPU的数量和类别
内存基地址和大小
总线和桥
外设连接
中断控制器和中断使用情况
GPIO控制器和GPIO使用情况
Clock控制器和Clock使用情况
它基本上就是画一棵电路板上CPU、总线、设备组成的树，Bootloader会将这棵树传递给内核，然后内核可以识别这棵树， 并根据它展开出Linux内核中的platform_device、i2c_client、spi_device等设备，而这些设备用到的内存、IRQ等资源， 也被传递给了内核，内核会将这些资源绑定给展开的相应的设备。

是否Device Tree要描述系统中的所有硬件信息？答案是否定的。基本上，那些可以动态探测到的设备是不需要描述的， 例如USB device。不过对于SOC上的usb hostcontroller，它是无法动态识别的，需要在device tree中描述。同样的道理， 在computersystem中，PCI device可以被动态探测到，不需要在device tree中描述，但是PCI bridge如果不能被探测，那么就需要描述之。

.dts文件是一种ASCII 文本格式的Device Tree描述，此文本格式非常人性化，适合人类的阅读习惯。 基本上，在ARM Linux在，一个.dts文件对应一个ARM的machine，一般放置在内核的arch/arm/boot/dts/目录。 由于一个SoC可能对应多个machine（一个SoC可以对应多个产品和电路板），势必这些.dts文件需包含许多共同的部分， Linux内核为了简化，把SoC公用的部分或者多个machine共同的部分一般提炼为.dtsi，类似于C语言的头文件。 其他的machine对应的.dts就include这个.dtsi。譬如，对于RK3288而言， rk3288.dtsi就被rk3288-chrome.dts所引用， rk3288-chrome.dts有如下一行：#include“rk3288.dtsi”, 对于rtd1195,在 rtd-119x-nas.dts中就包含了/include/ ”rtd-119x.dtsi” 当然，和C语言的头文件类似，.dtsi也可以include其他的.dtsi，譬如几乎所有的ARM SoC的.dtsi都引用了skeleton.dtsi，即#include”skeleton.dtsi“ 或者 /include/ “skeleton.dtsi”

正常情况下所有的dts文件以及dtsi文件都含有一个根节点”/”,这样include之后就会造成有很多个根节点? 按理说 device tree既然是一个树，那么其只能有一个根节点，所有其他的节点都是派生于根节点的child node. 其实Device Tree Compiler会对DTS的node进行合并，最终生成的DTB中只有一个 root node.

device tree的基本单元是node。这些node被组织成树状结构，除了root node，每个node都只有一个parent。 一个device tree文件中只能有一个root node。每个node中包含了若干的property/value来描述该node的一些特性。 每个node用节点名字（node name）标识，节点名字的格式是node-name@unit-address。如果该node没有reg属性（后面会描述这个property）， 那么该节点名字中必须不能包括@和unit-address。unit-address的具体格式是和设备挂在那个bus上相关。例如对于cpu， 其unit-address就是从0开始编址，以此加一。而具体的设备，例如以太网控制器，其unit-address就是寄存器地址。root node的node name是确定的，必须是“/”。 在一个树状结构的device tree中，如何引用一个node呢？要想唯一指定一个node必须使用full path，例如/node-name-1/node-name-2/node-name-N。 

DTS的组成结构
/ {  
    node1 {  
        a-string-property = "A string";  
        a-string-list-property = "first string", "second string";  
        a-byte-data-property = [0x01 0x23 0x34 0x56];  
        child-node1 {  
            first-child-property;  
            second-child-property = <1>;  
            a-string-property = "Hello, world";  
        };  
        child-node2 {  
        };  
    };  
    node2 {  
        an-empty-property;  
        a-cell-property = <1 2 3 4>; /* each number (cell) is a uint32 */  
        child-node1 {  
        };  
    };  
};
上述.dts文件并没有什么真实的用途，但它基本表征了一个Device Tree源文件的结构：

1个root结点”/”；

root结点下面含一系列子结点，本例中为”node1”和 “node2”；

结点”node1”下又含有一系列子结点，本例中为”child-node1”和 “child-node2”；

各结点都有一系列属性。这些属性可能为空，如”an-empty-property”；可能为字符串，如”a-string-property”； 可能为字符串数组，如”a-string-list-property”；可能为Cells（由u32整数组成），如”second-child-property”， 可能为二进制数，如”a-byte-data-property”。

下面以一个最简单的machine为例来看如何写一个.dts文件。假设此machine的配置如下：

1个双核ARM Cortex-A9 32位处理器；

ARM的local bus上的内存映射区域分布了2个串口（分别位于0x101F1000 和 0x101F2000）、 GPIO控制器（位于0x101F3000）、SPI控制器（位于0x10115000）、中断控制器（位于0x10140000）和一个external bus桥；

External bus桥上又连接了SMC SMC91111 Ethernet（位于0x10100000）、I2C控制器（位于0x10160000）、64MB NOR Flash（位于0x30000000）；

External bus桥上连接的I2C控制器所对应的I2C总线上又连接了Maxim DS1338实时钟（I2C地址为0x58）

其对应的.dts文件为：

/ {  
    compatible = "acme,coyotes-revenge";  
    #address-cells = <1>;  
    #size-cells = <1>;  
    interrupt-parent = <&intc>;  
  
    cpus {  
        #address-cells = <1>;  
        #size-cells = <0>;  
        cpu@0 {  
            compatible = "arm,cortex-a9";  
            reg = <0>;  
        };  
        cpu@1 {  
            compatible = "arm,cortex-a9";  
            reg = <1>;  
        };  
    };  
  
    serial@101f0000 {  
        compatible = "arm,pl011";  
        reg = <0x101f0000 0x1000 >;  
        interrupts = < 1 0 >;  
    };  
  
    serial@101f2000 {  
        compatible = "arm,pl011";  
        reg = <0x101f2000 0x1000 >;  
        interrupts = < 2 0 >;  
    };  
  
    
 
gpio@101f3000 {  
        compatible = "arm,pl061";  
        reg = <0x101f3000 0x1000  
               0x101f4000 0x0010>;  
        interrupts = < 3 0 >;  
    };  
  
    intc: interrupt-controller@10140000 {  
        compatible = "arm,pl190";  
        reg = <0x10140000 0x1000 >;  
        interrupt-controller;  
        #interrupt-cells = <2>;  
    };  
  
    spi@10115000 {  
        compatible = "arm,pl022";  
        reg = <0x10115000 0x1000 >;  
        interrupts = < 4 0 >;  
    };  
  
 
 
external-bus {  
        #address-cells = <2>  
        #size-cells = <1>;  
        ranges = <0 0  0x10100000   0x10000     // Chipselect 1, Ethernet  
                          1 0  0x10160000   0x10000     // Chipselect 2, i2c controller  
                          2 0  0x30000000   0x1000000>; // Chipselect 3, NOR Flash  
  
        ethernet@0,0 {  
            compatible = "smc,smc91c111";  
            reg = <0 0 0x1000>;  
            interrupts = < 5 2 >;  
        };  
  
        i2c@1,0 {  
            compatible = "acme,a1234-i2c-bus";  
            #address-cells = <1>;  
            #size-cells = <0>;  
            reg = <1 0 0x1000>;  
            rtc@58 {  
                compatible = "maxim,ds1338";  
                reg = <58>;  
                interrupts = < 7 3 >;  
            };  
        };  
  
        flash@2,0 {  
            compatible = "samsung,k8f1315ebm", "cfi-flash";  
            reg = <2 0 0x4000000>;  
        };  
    };  

};
上述.dts文件中,root结点”/”的compatible 属性compatible = “acme,coyotes-revenge”;定义了系统的名称， 它的组织形式为：,。Linux内核透过root结点"/"的compatible 属性即可判断它启动的是什么machine。

在.dts文件的每个设备，都有一个compatible属性，compatible属性用户驱动和设备的绑定。compatible 属性是一个字符串的列表， 列表中的第一个字符串表征了结点代表的确切设备，形式为”,"，其后的字符串表征可兼容的其他设备。可以说前面的是特指， 后面的则涵盖更广的范围。

如在arch/arm/boot/dts/vexpress-v2m.dtsi中的Flash结点：

flash@0,00000000 {  
     compatible = "arm,vexpress-flash", "cfi-flash";  
     reg = <0 0x00000000 0x04000000>,  
     <1 0x00000000 0x04000000>;  
     bank-width = <4>;  
 };
compatible属性的第2个字符串”cfi-flash”明显比第1个字符串”arm,vexpress-flash”涵盖的范围更广。 接下来root结点”/”的cpus子结点下面又包含2个cpu子结点，描述了此machine上的2个CPU，并且二者的compatible 属性为”arm,cortex-a9”。

注意

**cpus和cpus的2个cpu子结点的命名，它们遵循的组织形式为：[@]，<>中的内容是必选项， []中的则为可选项。name是一个ASCII字符串，用于描述结点对应的设备类型，如3com Ethernet适配器对应的结点name宜为ethernet， 而不是3com509。如果一个结点描述的设备有地址，则应该给出@unit-address。多个相同类型设备结点的name可以一样， 只要unit-address不同即可，如本例中含有cpu@0、cpu@1以及serial@101f0000与serial@101f2000这样的同名结点。 设备的unit-address地址也经常在其对应结点的reg属性中给出。 reg的组织形式为reg = <address1 length1 [address2 length2][address3 length3] ... >， 其中的每一组addresslength表明了设备使用的一个地址范围。address为1个或多个32位的整型（即cell），而length则为cell的列表或者为空 （若#size-cells = 0）。address和length字段是可变长的，父结点的#address-cells和#size-cells分别决定了子结点的reg属性的address和length字段的长度。**

在本例中，root结点的#address-cells = <1>;和#size-cells =<1>;决定了serial、gpio、spi等结点的address和length字段的长度分别为1。 cpus 结点的#address-cells= <1>;和#size-cells =<0>;决定了2个cpu子结点的address为1，而length为空， 于是形成了2个cpu的reg =<0>;和reg =<1>;。external-bus结点的#address-cells= <2>和#size-cells =<1>; 决定了其下的ethernet、i2c、flash的reg字段形如reg = <0 00x1000>;、reg = <1 00x1000>;和reg = <2 00x4000000>;。 其中，address字段长度为0，开始的第一个cell（0、1、2）是对应的片选，第2个cell（0，0，0）是相对该片选的基地址， 第3个cell（0x1000、0x1000、0x4000000）为length。特别要留意的是i2c结点中定义的 #address-cells = <1>;和#size-cells =<0>; 又作用到了I2C总线上连接的RTC，它的address字段为0x58，是设备的I2C地址。

root结点的子结点描述的是CPU的视图，因此root子结点的address区域就直接位于CPU的memory区域。但是， 经过总线桥后的address往往需要经过转换才能对应的CPU的memory映射。external-bus的ranges属性定义了经过external-bus桥后的地址范围如何映射到CPU的memory区域。

ranges = <0 0  0x10100000   0x10000         // Chipselect 1, Ethernet  
          1 0  0x10160000   0x10000         // Chipselect 2, i2c controller  
          2 0  0x30000000   0x1000000>;     // Chipselect 3, NOR Flash
ranges是地址转换表，其中的每个项目是一个子地址、父地址以及在子地址空间的大小的映射。映射表中的子地址、 父地址分别采用子地址空间的#address-cells和父地址空间的#address-cells大小。对于本例而言，子地址空间的#address-cells为2， 父地址空间的#address-cells值为1，因此0 0  0x10100000   0x10000的前2个cell为external-bus后片选0上偏移0， 第3个cell表示external-bus后片选0上偏移0的地址空间被映射到CPU的0x10100000位置，第4个cell表示映射的大小为0x10000。ranges的后面2个项目的含义可以类推。

Device Tree中还可以中断连接信息，对于中断控制器而言，它提供如下属性：

interrupt-controller – 这个属性为空，中断控制器应该加上此属性表明自己的身份；

#interrupt-cells – 与#address-cells 和 #size-cells相似，它表明连接此中断控制器的设备的interrupts属性的cell大小。在整个Device Tree中，与中断相关的属性还包括：

interrupt-parent - 设备结点透过它来指定它所依附的中断控制器的phandle，当结点没有指定interrupt-parent时，则从父级结点继承。对于本例而言，root结点指定了interrupt-parent= <&intc>;其对应于intc: interrupt-controller@10140000，而root结点的子结点并未指定interrupt-parent，因此它们都继承了intc，即位于0x10140000的中断控制器。

interrupts - 用到了中断的设备结点透过它指定中断号、触发方法等，具体这个属性含有多少个cell，由它依附的中断控制器结点的#interrupt-cells属性决定。而具体每个cell又是什么含义，一般由驱动的实现决定，而且也会在Device Tree的binding文档中说明

譬如，对于ARM GIC中断控制器而言，#interrupt-cells为3，它3个cell的具体含义

kernel/Documentation/devicetree/bindings/arm/gic.txt就有如下文字说明：


PPI(Private peripheral interrupt) SPI(Shared peripheral interrupt)

一个设备还可能用到多个中断号。对于ARM GIC而言，若某设备使用了SPI的168、169号2个中断，而言都是高电平触发， 则该设备结点的interrupts属性可定义为：interrupts =<0 168 4>, <0 169 4>;

dts引起BSP和driver的变更
没有使用dts之前的BSP和driver

DTC (device tree compiler)
将.dts编译为.dtb的工具。DTC的源代码位于内核的scripts/dtc目录，在Linux内核使能了Device Tree的情况下， 编译内核的时候主机工具dtc会被编译出来，对应scripts/dtc/Makefile中的“hostprogs-y := dtc”这一hostprogs编译target。 在Linux内核的arch/arm/boot/dts/Makefile中，描述了当某种SoC被选中后，哪些.dtb文件会被编译出来，如与VEXPRESS对应的.dtb包括：

dtb-$(CONFIG_ARCH_VEXPRESS) += vexpress-v2p-ca5s.dtb \
         vexpress-v2p-ca9.dtb \
         vexpress-v2p-ca15-tc1.dtb \
         vexpress-v2p-ca15_a7.dtb \
         xenvm-4.2.dtb
在Linux下，我们可以单独编译Device Tree文件。当我们在Linux内核下运行make dtbs时，若我们之前选择了ARCH_VEXPRESS， 上述.dtb都会由对应的.dts编译出来。因为arch/arm/Makefile中含有一个dtbs编译target项目。

linux 内核在启动的时候会解析设备树中的各个节点信息，并在 /proc/device-tree/ 目录下根据节点名字创建不同的文件夹或文件。如图 2 所示，可以看到创建的目录层级结构和 dts 中指定的层级结构是完全对应的。

驱动框架：


设备节点中对应的节点信息已经被内核构造成struct platform_device。


驱动描述：

static 




--------------------------------
看与c的交接处：
include/linux/of_fdt.h

extern char __dtb_start[];
extern char __dtb_end[];


现在看加载部分：

early_init_dt_scan(__dtb_start);

------------------
include/asm-generic/vmlinux.lds.h:
#define KERNEL_DTB()
	STRUCT_ALIGN();
	__dtb_start = .;
	KEEP(*(.dtb.init.rodata))
	__dtb_end = .;


DeviceTree 内核 API
*******************


  

ARM资源管理
"""""""""""













X86资源管理
""""""""""""

内核资源扫描与自举
""""""""""""""""""

设备组织架构
"""""""""""""










ISA设备驱动
************

1. 资源划分：
   内存 0~0xFFFFF;//1M空间；readb/w/l;//ioremap;
   IO  0 ～ 0xffff; 64k; // outb/w,inb/w;
2. 资源获取：

   mem：request_mem_region;
   io:request_region;

设备注册：

platform_driver_register/platform_driver_unregister;
platform_device_register/platform_device_unregister;
 

顺序：platform_device_register...



-----------------------------------
i2c驱动：

#define PCA9555_ADDR	(0x20)
#define	PCA9555_CHIP0	(PCA9555_ADDR | 0X0)
#define	PCA9555_CHIP1   (PCA9555_ADDR | 0X01)
#define	PCA9555_CHIP2	(PCA9555_ADDR | 0X02)
#endif

static struct i2c_board_info __initdata byt_i2c_devices[] = {
	{	I2C_BOARD_INFO("pca9555-0",		PCA9555_CHIP0),	},
	{	I2C_BOARD_INFO("pca9555-1",		PCA9555_CHIP1),	},
	{	I2C_BOARD_INFO("pca9555-2",		PCA9555_CHIP2),	},
};

。。。。。。

	if(0x0f41 == id->device){
		i2c_new_device(adap, &byt_i2c_devices[0]);
		i2c_new_device(adap, &byt_i2c_devices[1]);
		i2c_new_device(adap, &byt_i2c_devices[2]);
	}
	
从设备：	
static const struct i2c_device_id pca953x_id[] = {

	{ "pca9555-0", 16 | PCA953X_TYPE | PCA_INT, },
	{ "pca9555-1", 16 | PCA953X_TYPE | PCA_INT, },
	{ "pca9555-2", 16 | PCA953X_TYPE | PCA_INT, },
。。。。。。
}

MODULE_DEVICE_TABLE(i2c, pca953x_id);



static const struct of_device_id pca953x_dt_ids[] = {
	{ .compatible = "nxp,pca9505", },
	{ .compatible = "nxp,pca9534", },
	{ .compatible = "nxp,pca9535", },
	{ .compatible = "nxp,pca9536", },
	{ .compatible = "nxp,pca9537", },
	{ .compatible = "nxp,pca9538", },
	{ .compatible = "nxp,pca9539", },
	{ .compatible = "nxp,pca9554", },
	{ .compatible = "nxp,pca9555", },
	{ .compatible = "nxp,pca9556", },
	{ .compatible = "nxp,pca9557", },
	{ .compatible = "nxp,pca9574", },
	{ .compatible = "nxp,pca9575", },
	{ .compatible = "nxp,pca9698", },

	{ .compatible = "maxim,max7310", },
	{ .compatible = "maxim,max7312", },
	{ .compatible = "maxim,max7313", },
	{ .compatible = "maxim,max7315", },

	{ .compatible = "ti,pca6107", },
	{ .compatible = "ti,tca6408", },
	{ .compatible = "ti,tca6416", },
	{ .compatible = "ti,tca6424", },

	{ .compatible = "exar,xra1202", },
	{ }
};

MODULE_DEVICE_TABLE(of, pca953x_dt_ids);

static struct i2c_driver pca953x_driver = {
	.driver = {
		.name	= "pca953x",
		.of_match_table = pca953x_dt_ids,
	},
	.probe		= pca953x_probe,
	.remove		= pca953x_remove,
	.id_table	= pca953x_id,
};

。。。。。。
static int __init pca953x_init(void)
{
	return i2c_add_driver(&pca953x_driver);
}
。。。。。。




------------------------------------------------

设备结构：
/**
 * struct device_driver - The basic device driver structure
 * @name:	Name of the device driver.
 * @bus:	The bus which the device of this driver belongs to.
 * @owner:	The module owner.
 * @mod_name:	Used for built-in modules.
 * @suppress_bind_attrs: Disables bind/unbind via sysfs.
 * @probe_type:	Type of the probe (synchronous or asynchronous) to use.
 * @of_match_table: The open firmware table.
 * @acpi_match_table: The ACPI match table.
 * @probe:	Called to query the existence of a specific device,
 *		whether this driver can work with it, and bind the driver
 *		to a specific device.
 * @sync_state:	Called to sync device state to software state after all the
 *		state tracking consumers linked to this device (present at
 *		the time of late_initcall) have successfully bound to a
 *		driver. If the device has no consumers, this function will
 *		be called at late_initcall_sync level. If the device has
 *		consumers that are never bound to a driver, this function
 *		will never get called until they do.
 * @remove:	Called when the device is removed from the system to
 *		unbind a device from this driver.
 * @shutdown:	Called at shut-down time to quiesce the device.
 * @suspend:	Called to put the device to sleep mode. Usually to a
 *		low power state.
 * @resume:	Called to bring a device from sleep mode.
 * @groups:	Default attributes that get created by the driver core
 *		automatically.
 * @dev_groups:	Additional attributes attached to device instance once
 *		it is bound to the driver.
 * @pm:		Power management operations of the device which matched
 *		this driver.
 * @coredump:	Called when sysfs entry is written to. The device driver
 *		is expected to call the dev_coredump API resulting in a
 *		uevent.
 * @p:		Driver core's private data, no one other than the driver
 *		core can touch this.
 *
 * The device driver-model tracks all of the drivers known to the system.
 * The main reason for this tracking is to enable the driver core to match
 * up drivers with new devices. Once drivers are known objects within the
 * system, however, a number of other things become possible. Device drivers
 * can export information and configuration variables that are independent
 * of any specific device.
 */
struct device_driver {
	const char		*name;
	struct bus_type		*bus;

	struct module		*owner;
	const char		*mod_name;	/* used for built-in modules */

	bool suppress_bind_attrs;	/* disables bind/unbind via sysfs */
	enum probe_type probe_type;

	const struct of_device_id	*of_match_table;//匹配的of设备表
	const struct acpi_device_id	*acpi_match_table; //匹配的acpi 设备表

	int (*probe) (struct device *dev);
	void (*sync_state)(struct device *dev);
	int (*remove) (struct device *dev);
	void (*shutdown) (struct device *dev);
	int (*suspend) (struct device *dev, pm_message_t state);
	int (*resume) (struct device *dev);
	const struct attribute_group **groups;
	const struct attribute_group **dev_groups;

	const struct dev_pm_ops *pm;
	void (*coredump) (struct device *dev);

	struct driver_private *p;
};




/*
 * Struct used for matching a device
 */
struct of_device_id {
	char	name[32];
	char	type[32];
	char	compatible[128];
	const void *data;
};



struct acpi_device_id {
	__u8 id[ACPI_ID_LEN];
	kernel_ulong_t driver_data;
	__u32 cls;
	__u32 cls_msk;
};

/**
 * struct device - The basic device structure
 * @parent:	The device's "parent" device, the device to which it is attached.
 * 		In most cases, a parent device is some sort of bus or host
 * 		controller. If parent is NULL, the device, is a top-level device,
 * 		which is not usually what you want.
 * @p:		Holds the private data of the driver core portions of the device.
 * 		See the comment of the struct device_private for detail.
 * @kobj:	A top-level, abstract class from which other classes are derived.
 * @init_name:	Initial name of the device.
 * @type:	The type of device.
 * 		This identifies the device type and carries type-specific
 * 		information.
 * @mutex:	Mutex to synchronize calls to its driver.
 * @lockdep_mutex: An optional debug lock that a subsystem can use as a
 * 		peer lock to gain localized lockdep coverage of the device_lock.
 * @bus:	Type of bus device is on.
 * @driver:	Which driver has allocated this
 * @platform_data: Platform data specific to the device.
 * 		Example: For devices on custom boards, as typical of embedded
 * 		and SOC based hardware, Linux often uses platform_data to point
 * 		to board-specific structures describing devices and how they
 * 		are wired.  That can include what ports are available, chip
 * 		variants, which GPIO pins act in what additional roles, and so
 * 		on.  This shrinks the "Board Support Packages" (BSPs) and
 * 		minimizes board-specific #ifdefs in drivers.
 * @driver_data: Private pointer for driver specific info.
 * @links:	Links to suppliers and consumers of this device.
 * @power:	For device power management.
 *		See Documentation/driver-api/pm/devices.rst for details.
 * @pm_domain:	Provide callbacks that are executed during system suspend,
 * 		hibernation, system resume and during runtime PM transitions
 * 		along with subsystem-level and driver-level callbacks.
 * @em_pd:	device's energy model performance domain
 * @pins:	For device pin management.
 *		See Documentation/driver-api/pin-control.rst for details.
 * @msi_lock:	Lock to protect MSI mask cache and mask register
 * @msi_list:	Hosts MSI descriptors
 * @msi_domain: The generic MSI domain this device is using.
 * @numa_node:	NUMA node this device is close to.
 * @dma_ops:    DMA mapping operations for this device.
 * @dma_mask:	Dma mask (if dma'ble device).
 * @coherent_dma_mask: Like dma_mask, but for alloc_coherent mapping as not all
 * 		hardware supports 64-bit addresses for consistent allocations
 * 		such descriptors.
 * @bus_dma_limit: Limit of an upstream bridge or bus which imposes a smaller
 *		DMA limit than the device itself supports.
 * @dma_range_map: map for DMA memory ranges relative to that of RAM
 * @dma_parms:	A low level driver may set these to teach IOMMU code about
 * 		segment limitations.
 * @dma_pools:	Dma pools (if dma'ble device).
 * @dma_mem:	Internal for coherent mem override.
 * @cma_area:	Contiguous memory area for dma allocations
 * @archdata:	For arch-specific additions.
 * @of_node:	Associated device tree node.
 * @fwnode:	Associated device node supplied by platform firmware.
 * @devt:	For creating the sysfs "dev".
 * @id:		device instance
 * @devres_lock: Spinlock to protect the resource of the device.
 * @devres_head: The resources list of the device.
 * @knode_class: The node used to add the device to the class list.
 * @class:	The class of the device.
 * @groups:	Optional attribute groups.
 * @release:	Callback to free the device after all references have
 * 		gone away. This should be set by the allocator of the
 * 		device (i.e. the bus driver that discovered the device).
 * @iommu_group: IOMMU group the device belongs to.
 * @iommu:	Per device generic IOMMU runtime data
 * @removable:  Whether the device can be removed from the system. This
 *              should be set by the subsystem / bus driver that discovered
 *              the device.
 *
 * @offline_disabled: If set, the device is permanently online.
 * @offline:	Set after successful invocation of bus type's .offline().
 * @of_node_reused: Set if the device-tree node is shared with an ancestor
 *              device.
 * @state_synced: The hardware state of this device has been synced to match
 *		  the software state of this device by calling the driver/bus
 *		  sync_state() callback.
 * @can_match:	The device has matched with a driver at least once or it is in
 *		a bus (like AMBA) which can't check for matching drivers until
 *		other devices probe successfully.
 * @dma_coherent: this particular device is dma coherent, even if the
 *		architecture supports non-coherent devices.
 * @dma_ops_bypass: If set to %true then the dma_ops are bypassed for the
 *		streaming DMA operations (->map_* / ->unmap_* / ->sync_*),
 *		and optionall (if the coherent mask is large enough) also
 *		for dma allocations.  This flag is managed by the dma ops
 *		instance from ->dma_supported.
 *
 * At the lowest level, every device in a Linux system is represented by an
 * instance of struct device. The device structure contains the information
 * that the device model core needs to model the system. Most subsystems,
 * however, track additional information about the devices they host. As a
 * result, it is rare for devices to be represented by bare device structures;
 * instead, that structure, like kobject structures, is usually embedded within
 * a higher-level representation of the device.
 */
struct device {
	struct kobject kobj;
	struct device		*parent;

	struct device_private	*p;

	const char		*init_name; /* initial name of the device */
	const struct device_type *type;

	struct bus_type	*bus;		/* type of bus device is on */
	struct device_driver *driver;	/* which driver has allocated this
					   device */
	void		*platform_data;	/* Platform specific data, device
					   core doesn't touch it */
	void		*driver_data;	/* Driver data, set and get with
					   dev_set_drvdata/dev_get_drvdata */
#ifdef CONFIG_PROVE_LOCKING
	struct mutex		lockdep_mutex;
#endif
	struct mutex		mutex;	/* mutex to synchronize calls to
					 * its driver.
					 */

	struct dev_links_info	links;
	struct dev_pm_info	power;
	struct dev_pm_domain	*pm_domain;

#ifdef CONFIG_ENERGY_MODEL
	struct em_perf_domain	*em_pd;
#endif

#ifdef CONFIG_GENERIC_MSI_IRQ_DOMAIN
	struct irq_domain	*msi_domain;
#endif
#ifdef CONFIG_PINCTRL
	struct dev_pin_info	*pins;
#endif
#ifdef CONFIG_GENERIC_MSI_IRQ
	raw_spinlock_t		msi_lock;
	struct list_head	msi_list;
#endif
#ifdef CONFIG_DMA_OPS
	const struct dma_map_ops *dma_ops;
#endif
	u64		*dma_mask;	/* dma mask (if dma'able device) */
	u64		coherent_dma_mask;/* Like dma_mask, but for
					     alloc_coherent mappings as
					     not all hardware supports
					     64 bit addresses for consistent
					     allocations such descriptors. */
	u64		bus_dma_limit;	/* upstream dma constraint */
	const struct bus_dma_region *dma_range_map;

	struct device_dma_parameters *dma_parms;

	struct list_head	dma_pools;	/* dma pools (if dma'ble) */

#ifdef CONFIG_DMA_DECLARE_COHERENT
	struct dma_coherent_mem	*dma_mem; /* internal for coherent mem
					     override */
#endif
#ifdef CONFIG_DMA_CMA
	struct cma *cma_area;		/* contiguous memory area for dma
					   allocations */
#endif
	/* arch specific additions */
	struct dev_archdata	archdata;

	struct device_node	*of_node; /* associated device tree node */
	struct fwnode_handle	*fwnode; /* firmware device node */

#ifdef CONFIG_NUMA
	int		numa_node;	/* NUMA node this device is close to */
#endif
	dev_t			devt;	/* dev_t, creates the sysfs "dev" */
	u32			id;	/* device instance */

	spinlock_t		devres_lock;
	struct list_head	devres_head;

	struct class		*class;
	const struct attribute_group **groups;	/* optional groups */

	void	(*release)(struct device *dev);
	struct iommu_group	*iommu_group;
	struct dev_iommu	*iommu;

	enum device_removable	removable;

	bool			offline_disabled:1;
	bool			offline:1;
	bool			of_node_reused:1;
	bool			state_synced:1;
	bool			can_match:1;
#if defined(CONFIG_ARCH_HAS_SYNC_DMA_FOR_DEVICE) || \
    defined(CONFIG_ARCH_HAS_SYNC_DMA_FOR_CPU) || \
    defined(CONFIG_ARCH_HAS_SYNC_DMA_FOR_CPU_ALL)
	bool			dma_coherent:1;
#endif
#ifdef CONFIG_DMA_OPS_BYPASS
	bool			dma_ops_bypass : 1;
#endif
};


设备树
************


struct device_node {
	const char *name;
	phandle phandle;
	const char *full_name;
	struct fwnode_handle fwnode;

	struct	property *properties;
	struct	property *deadprops;	/* removed properties */
	struct	device_node *parent;
	struct	device_node *child;
	struct	device_node *sibling;
#if defined(CONFIG_OF_KOBJ)
	struct	kobject kobj;
#endif
	unsigned long _flags;
	void	*data;
#if defined(CONFIG_SPARC)
	unsigned int unique_id;
	struct of_irq_controller *irq_trans;
#endif
};

unflatten_device_tree
这一部分就进入了设备树的解析部分：

void __init unflatten_device_tree(void)
{
    __unflatten_device_tree(initial_boot_params, NULL, &of_root,early_init_dt_alloc_memory_arch, false);  —————— part1

    of_alias_scan(early_init_dt_alloc_memory_arch);                  —————— part2
    ...
}





LIST_HEAD(aliases_lookup);

struct device_node *of_root;
EXPORT_SYMBOL(of_root);
struct device_node *of_chosen;
EXPORT_SYMBOL(of_chosen);
struct device_node *of_aliases;
struct device_node *of_stdout;
static const char *of_stdout_options;




of_chosen和of_aliases都是struct device_node型的全局数据。


程序跟踪到这里，设备树由dtb二进制文件经过解析为每个节点生成一个struct device_node结构体的过程基本上就清晰了，我们再进行一下总结，首先看看struct device_node结构：

struct device_node {
    const char *name;
    const char *type;
    phandle phandle;
    const char *full_name;
    ...
    struct	property *properties;//资源列表？
    struct	property *deadprops;	/* removed properties */
    struct	device_node *parent;
    struct	device_node *child;
    struct	device_node *sibling;
    struct	kobject kobj;
    unsigned long _flags;
    void	*data;
    ...
};

.name属性：设备节点中的name属性转换而来。
.type属性：由设备节点中的device_type转换而来。
.phandle属性：有设备节点中的"phandle"和"linux,phandle"属性转换而来，特殊的还可能由"ibm,phandle"属性转换而来。
full_name:这个指针指向整个结构体的结尾位置，在结尾位置存储着这个结构体对应设备树节点的unit_name，意味着一个struct device_node结构体占内存空间为sizeof(struct device_node)+strlen(unit_name)+字节对齐。
.properties这是一个设备树节点的属性链表，属性可能有很多种，比如："interrupts","timer"，"hwmods"等等。
.parent,.child,.sibling:与当前属性链表节点相关节点，所以相关链表节点构成整个device_node的属性节点。
.kobj：用于在/sys目录下生成相应用户文件。
这就是设备树子节点到struct device_node的转换，为了能更直观地看出设备树节点到struct device_node的转换过程，博主特意制作了一张脑图：


dtb --> strut device_node --> struct platform_device




设备树的产生就是为了替代driver中过多的platform_device部分的静态定义，将硬件资源抽象出来，由系统统一解析，这样就可以避免各驱动中对硬件资源大量的重复定义，这样一来，几乎可以肯定的是，设备树中的节点最终目标是转换成platform device结构，在驱动开发时就只需要添加相应的platform driver部分进行匹配即可。

首先，对于所有的device_node，如果要转换成platform_device，必须满足以下条件：

一般情况下，只对设备树中根的子节点进行转换，也就是子节点的子节点并不处理。但是存在一种特殊情况，就是当某个根子节点的compatible属性为"simple-bus"、"simple-mfd"、"isa"、"arm,amba-bus"时，当前节点中的子节点将会被转换成platform_device节点。

节点中必须有compatible属性。

如果是device_node转换成platform device，这个转换过程又是怎么样的呢？

在设备树中，这一类资源通常通过reg属性来描述，中断则通过interrupts来描述，所以，设备树中的reg和interrupts资源将会被转换成platform_device内的struct resources资源。


那么，设备树中其他属性是怎么转换的呢？答案是：不需要转换，在platform_device中有一个成员struct device dev，这个dev中又有一个指针成员struct device_node *of_node,linux的做法就是将这个of_node指针直接指向由设备树转换而来的device_node结构。

例如，有这么一个struct platform_device* of_test.我们可以直接通过of_test->dev.of_node来访问设备树中的信息.


struct platform_device {
	const char	*name;
	int		id;
	bool		id_auto;
	struct device	dev;
	u64		platform_dma_mask;
	struct device_dma_parameters dma_parms;
	u32		num_resources;
	struct resource	*resource;

	const struct platform_device_id	*id_entry;
	char *driver_override; /* Driver name to force a match */

	/* MFD cell pointer */
	struct mfd_cell *mfd_cell;

	/* arch specific additions */
	struct pdev_archdata	archdata;
};


struct platform_device --> struct device dev --> struct device_node of_node

of_platform_default_populate_init(),它被调用的方式是这样一个声明：


明确：维护的是struct device链：


of_platform_default_populate()调用了of_platform_populate()。

需要注意的是，在调用of_platform_populate()时传入了参数of_default_bus_match_table[]，这个table是一个静态数组，这个静态数组中定义了一系列的compatible属性："simple-bus"、"simple-mfd"、"isa"、"arm,amba-bus"。

按照我们上文中的描述，当某个根节点下的一级子节点的compatible属性为这些属性其中之一时，它的一级子节点也将由device_node转换成platform_device.

of_device_add --> device_add;


struct platform_device *of_device_alloc(struct device_node *np,const char *bus_id,struct device *parent)
{
    //统计reg属性的数量
    while (of_address_to_resource(np, num_reg, &temp_res) == 0)
	    num_reg++;
    //统计中断irq属性的数量
    num_irq = of_irq_count(np);
    //根据num_irq和num_reg的数量申请相应的struct resource内存空间。
    if (num_irq || num_reg) {
        res = kzalloc(sizeof(*res) * (num_irq + num_reg), GFP_KERNEL);
        if (!res) {
            platform_device_put(dev);
            return NULL;
        }
        //设置platform_device中的num_resources成员
        dev->num_resources = num_reg + num_irq;
        //设置platform_device中的resource成员
        dev->resource = res;

        //将device_node中的reg属性转换成platform_device中的struct resource成员。  
        for (i = 0; i < num_reg; i++, res++) {
            rc = of_address_to_resource(np, i, res);
            WARN_ON(rc);
        }
        //将device_node中的irq属性转换成platform_device中的struct resource成员。 
        if (of_irq_to_resource_table(np, res, num_irq) != num_irq)
            pr_debug("not all legacy IRQ resources mapped for %s\n",
                np->name);
    }
    //将platform_device的dev.of_node成员指针指向device_node。  
    dev->dev.of_node = of_node_get(np);
    //将platform_device的dev.fwnode成员指针指向device_node的fwnode成员。
    dev->dev.fwnode = &np->fwnode;
    //设备parent为platform_bus
    dev->dev.parent = parent ? : &platform_bus;

}
首先，函数先统计设备树中reg属性和中断irq属性的个数，然后分别为它们申请内存空间，链入到platform_device中的struct resources成员中。除了设备树中"reg"和"interrupt"属性之外，还有可选的"reg-names"和"interrupt-names"这些io中断资源相关的设备树节点属性也在这里被转换。

将相应的设备树节点生成的device_node节点链入到platform_device的dev.of_node中。


将当前platform_device中的struct device成员注册到系统device中，并为其在用户空间创建相应的访问节点。


dtb -> struct device_node --> struct platform_device->dev.of_node
             ^						     |
             |_______________________________________________|

struct platform_device 将资源解析为struct resource列表。


ACPI FDT
**********

acpi_init

struct acpi_device --> struct device --> struct device_driver

acpi 设备表 --> struct acpi_device

extern struct acpi_device *acpi_root;

#define ACPI_BUS_CLASS			"system_bus"
#define ACPI_BUS_HID			"LNXSYBUS"
#define ACPI_BUS_DEVICE_NAME		"System Bus"

#define ACPI_IS_ROOT_DEVICE(device)    (!(device)->parent)

#define INVALID_ACPI_HANDLE	((acpi_handle)empty_zero_page)

static const char *dummy_hid = "device";

static LIST_HEAD(acpi_dep_list);
static DEFINE_MUTEX(acpi_dep_list_lock);
LIST_HEAD(acpi_bus_id_list);
static DEFINE_MUTEX(acpi_scan_lock);
static LIST_HEAD(acpi_scan_handlers_list);
DEFINE_MUTEX(acpi_device_lock);
LIST_HEAD(acpi_wakeup_device_list);
static DEFINE_MUTEX(acpi_hp_context_lock);


scan.c:
extern struct acpi_device *acpi_root;




PCI设备枚举
*************


------------------------------------------------------------------------------------------------






























linux设备架构：
"""""""""""""

驱动的绑定
**********

- 驱动绑定：设备与控制它的设备驱动程序相关联的过程。绑定动作一般由总线驱动来实现。
- 总线：总线类型的结构包含系统中总线类型相同的设备列表，当调用device_register时将设备数据插入这个列表。总线结构中还包含一个驱动程序列表，当驱动程序调用driver_register函数时，将驱动插入驱动程序类表末尾，这两种事件会触发设备与驱动程序的绑定：
  - device_register:查找匹配的驱动，通过设备ID进行匹配。这个ID的格式和语义依赖总线进行实现。由总线驱动程序提供回调函数实现ID的匹配比较，成功返回1,否则为0：int match(struct device * dev, struct device_driver * drv);如果找到了匹配的设备驱动，将 设备的驱动指针只想匹配的驱动结构，并调用驱动程序的probe回调函数。驱动程序对硬件状态进行检查，并对其工作状态进行初始化。
  - driver_register原理同上。

- 设备类：完成探测后设备注册所属的类。设备驱动只能属于一个类，在驱动结构的devclass中进行设置。在执行类中的register_dev回调函数时调用devclass_add_device来枚举类中的设备，并最终将设备注册为某个类。

- 驱动：当一个驱动匹配到一个设备，设备的结构会插入到驱动结构中的设备列表中。一个设备只能有一个驱动程序，如果已经匹配了要跳过。
- sysfs文件：在总线的“设备”目录中创建一个符号链接，该链接指向物理层次结构中的设备目录。在驱动程序的“设备”目录中创建了一个符号链接，该链接指向物理层次结构中的设备目录。设备的目录在类的目录中创建。在该目录中创建一个符号链接，指向设备在 sysfs 树中的物理位置。可以在设备的物理目录中创建指向其类目录或类的顶级目录的符号链接（尽管这还没有完成）。也可以创建一个指向其驱动程序的目录。

- 移除：
  - 当设备移除时，对应引用数会变成0。当引用计数为0时，驱动调用remove回调函数。将设备结构从驱动结构的设备列表中去除，并将驱动的应用计数减1.同时删除两者之间的符号链接。
  - 当驱动移除时，将对驱动结构的设备列表进行迭代处理，并针对每个设备调用驱动程序的remove回调函数。将设备结构从驱动结构的设备列表中删除并删除两者间的符号文件。


总线类型
*********
内核为每类如PCI,USB等设备声明一个静态的总线类型的对象。必须指定名字，可选择性初始化其match回调函数。

struct bus_type pci_bus_type = {
       .name = "pci",
       .match        = pci_bus_match,
};

extern struct bus_type pci_bus_type;作为全局变量包含在头文件中，供驱动程序使用。

- 注册：总线驱动初始化时，调用bus_register来初始化总线对象中的剩余域并将总线对象插入总线类型的全局列表中（struct bus_type的结构列表）。总线对象注册后，总线驱动就可以使用其中的字段。
- 回调：match:匹配驱动程序和设备。匹配引用的ID结构特定于总线实现。驱动程序通常会生命它们支持的设备的设备ID组，并保存在驱动结构中。
- 设备和驱动程序列表 ：分别是 struct devices 和 struct device_drivers 的列表。总线驱动可以自由使用这些列表。使用前可能主要转换为特定的类型（这句话不是很准确？）.
  两种列表的迭代函数：
  - int bus_for_each_dev(struct bus_type * bus, struct device * start,
              	     void * data,
                     int (*fn)(struct device *, void *));

  - int bus_for_each_drv(struct bus_type * bus, struct device_driver * start,
                     void * data, int (*fn)(struct device_driver *, void *));


  这两个函数遍历相应的列表，并为列表中的每个设备或驱动程序调用回调。所有列表访问都是通过获取总线的锁（当前读取）来同步的。列表中每个对象的引用计数在调用回调之前递增；在获得下一个对象后递减。调用回调时不持有锁。
  
- sysfs

有一个名为“bus”的顶级目录。每条总线在总线目录中都有一个目录，以及两个默认目录：

/sys/bus/pci/
|-- devices
`-- drivers

在总线上注册的驱动程序会在总线的驱动程序目录中获得一个目录：

/sys/bus/pci/
|-- devices
`-- drivers
    |-- Intel ICH
    |-- Intel ICH Joystick
    |-- agpgart
    `-- e100
    
在该类型的总线上发现的每个设备都会在总线的设备目录中获得指向物理层次结构中设备目录的符号链接：

/sys/bus/pci/
|-- devices
|   |-- 00:00.0 -> ../../../root/pci0/00:00.0
|   |-- 00:01.0 -> ../../../root/pci0/00:01.0
|   `-- 00:02.0 -> ../../../root/pci0/00:02.0
`-- drivers

- 导出属性

struct bus_attribute {
      struct attribute        attr;
      ssize_t (*show)(struct bus_type *, char * buf);
      ssize_t (*store)(struct bus_type *, const char * buf, size_t count);
};


总线驱动程序可以使用与设备的 DEVICE_ATTR_RW 宏类似的 BUS_ATTR_RW 宏导出属性。例如，这样的定义：

static BUS_ATTR_RW(debug);等价于 static bus_attribute bus_attr_debug;


然后可以使用以下命令在总线的 sysfs 目录中添加和删除属性：

int bus_create_file(struct bus_type *, struct bus_attribute *);
void bus_remove_file(struct bus_type *, struct bus_attribute *);


设备驱动程序设计模式
*****************
常见的设计模式

1.状态容器;
  虽然内核包含一些设备驱动程序，它们假设它们只会在某个系统（单例）上被探测（）一次，但通常假设驱动程序绑定到的设备将出现在多个实例中。这意味着 probe() 函数和所有回调都需要可重入。最常见的实现方式是使用状态容器设计模式。它通常有这种形式：
  
struct foo {
    spinlock_t lock; /* Example member */
    (...)
};

static int foo_probe(...)
{
    struct foo *foo;

    foo = devm_kzalloc(dev, sizeof(*foo), GFP_KERNEL);
    if (!foo)
        return -ENOMEM;
    spin_lock_init(&foo->lock);
    (...)
}
  这将在每次调用 probe() 时在内存中创建一个 struct foo 实例。这是设备驱动程序实例的状态容器。当然，必须始终将此状态实例传递给所有需要访问状态及其成员的函数。
  例如，如果驱动程序正在注册一个中断处理程序，您将传递一个指向 struct foo 的指针，如下所示：
 
static irqreturn_t foo_handler(int irq, void *arg)
{
    struct foo *foo = arg;
    (...)
}

static int foo_probe(...)
{
    struct foo *foo;

    (...)
    ret = request_irq(irq, foo_handler, 0, "foo", foo);
}
  
  
  
2.container_of();

添加一个卸载的工作：

struct foo {
    spinlock_t lock;
    struct workqueue_struct *wq;
    struct work_struct offload;
    (...)
};

static void foo_work(struct work_struct *work)
{
    struct foo *foo = container_of(work, struct foo, offload);

    (...)
}

static irqreturn_t foo_handler(int irq, void *arg)
{
    struct foo *foo = arg;

    queue_work(foo->wq, &foo->offload);
    (...)
}

static int foo_probe(...)
{
    struct foo *foo;

    foo->wq = create_singlethread_workqueue("foo-wq");
    INIT_WORK(&foo->offload, foo_work);
    (...)
}

对于 hrtimer 或类似的东西，设计模式是相同的，它们将返回一个参数，该参数是指向回调中结构成员的指针。

container_of() 是在 <linux/kernel.h> 中定义的宏,container_of() 所做的是使用标准 C 中的 offsetof() 宏通过简单的减法从指向成员的指针中获取指向包含结构的指针，这允许类似于面向对象的行为。请注意，包含的成员不能是指针，而是要使其正常工作的实际成员。避免了以这种方式使用指向 struct foo * 实例的全局指针，同时仍将传递给工作函数的参数数量保持为单个指针。

总结：


基本设备结构:struct device
**************************

- 编程接口：
  - int device_register(struct device * dev);执行这个函数时，总线驱动发现设备并将设备注册到核心
  - 总线需要初始化以下域：
    - parent;
    - name;
    - bus_id;
    - bus;
  - 当设备的引用计数变为 0 时，设备将从内核中移除。可以使用以下命令调整引用计数：
    struct device * get_device(struct device * dev);
    void put_device(struct device * dev);
    get_device()如果引用还不是 0（如果它已经在被删除的过程中），将返回一个指向传递给它的struct device结构指针。
  - 驱动可以通过以下方式访问设备结构中的锁：
     void lock_device(struct device * dev);
     void unlock_device(struct device * dev);

  - 属性：
    struct device_attribute {
      struct attribute        attr;
      ssize_t (*show)(struct device *dev, struct device_attribute *attr,
                      char *buf);
      ssize_t (*store)(struct device *dev, struct device_attribute *attr,
                       const char *buf, size_t count);
      };

      设备的属性可以由设备驱动程序通过 sysfs 导出。

      正如关于 kobjects、ksets 和 ktypes 的所有你不想知道的内容中所解释的，必须在生成 KOBJ_ADD uevent 之前创建设备属性。实现这一点的唯一方法是定义一个属性组。

      使用名为 DEVICE_ATTR 的宏声明属性：#define DEVICE_ATTR(name,mode,show,store)
      eg:
      static DEVICE_ATTR(type, 0444, type_show, NULL);
      static DEVICE_ATTR(power, 0644, power_show, power_store);

      针对模式值的宏：
      static DEVICE_ATTR_RO(type);
      static DEVICE_ATTR_RW(power);
      这声明了 struct device_attribute 类型的两个结构，其名称分别为“dev_attr_type”和“dev_attr_power”。这两个属性可以按如下方式组织成一个组：

      static struct attribute *dev_attrs[] = {
         &dev_attr_type.attr,
         &dev_attr_power.attr,
         NULL,
      };

      static struct attribute_group dev_group = {
         .attrs = dev_attrs,
      };

      static const struct attribute_group *dev_groups[] = {
         &dev_group,
         NULL,
      };
      辅助宏可用于单个组的常见情况，因此可以使用以下两种结构声明：

      ATTRIBUTE_GROUPS(dev);

      然后可以通过在调用之前设置组指针来将这个组数组与设备相关联：struct device  device_register()

	dev->groups = dev_groups;
	device_register(dev);

      该device_register()函数将使用“组”指针创建设备属性，并且该device_unregister()函数将使用该指针删除设备属性。

      虽然内核允许device_create_file()并且 device_remove_file()可以随时在设备上调用，但用户空间对何时创建属性有严格的期望。当一个新设备在内核中注册时，会生成一个 uevent 来通知用户空间（如 udev）有一个新设备可用。如果在注册设备后添加属性，则用户空间不会收到通知，用户空间将不知道新属性。

      这对于需要在驱动程序探测时为设备发布附加属性的设备驱动程序很重要。如果设备驱动程序只是调用device_create_file()传递给它的设备结构，那么用户空间将永远不会收到新属性的通知。
      
devres-管理设备资源
******************


设备驱动程序: struct device_driver
*********************************
- 分配：设备驱动程序是静态分配的结构。尽管驱动程序支持的系统中可能有多个设备，但 struct device_driver 将驱动程序表示为一个整体（而不是特定的设备实例）。
- 初始化：
- 声明：
- 注册：      
- 转换总线驱动程序：
- 访问：
- sysfs:
- 回调：
- 属性：
-       
      
linux 内核设备模型
*****************


平台设备和驱动程序
*****************
   有关平台总线的驱动程序模型接口，请参见 <linux/platform_device.h>：platform_device 和 platform_driver。这种伪总线用于连接具有最少基础设施的总线上的设备，例如用于在许多片上系统处理器上集成外围设备的设备，或一些“传统”PC 互连；而不是像 PCI 或 USB 这样的大型总线指定的。  
      
- 平台设备：平台设备是通常在系统中显示为自治实体的设备。这包括传统的基于端口的设备和外设总线的主机桥，以及集成到片上系统平台的大多数控制器。它们通常的共同点是从 CPU 总线直接寻址。极少情况下，platform_device 会通过其他某种总线的段连接；但它的寄存器仍然是可直接寻址的。平台设备有一个名称，用于驱动程序绑定，以及一个资源列表，例如地址和 IRQ：     
      
      struct platform_device {
      const char      *name;
      u32             id;
      struct device   dev;
      u32             num_resources;
      struct resource *resource;
	};
- 平台驱动程序：    平台驱动程序遵循标准驱动程序模型约定，其中发现/枚举在驱动程序之外处理，并且驱动程序提供probe() 和remove() 方法。它们使用标准约定支持电源管理和关机通知：
      
    struct platform_driver {
      int (*probe)(struct platform_device *);
      int (*remove)(struct platform_device *);
      void (*shutdown)(struct platform_device *);
      int (*suspend)(struct platform_device *, pm_message_t state);
      int (*suspend_late)(struct platform_device *, pm_message_t state);
      int (*resume_early)(struct platform_device *);
      int (*resume)(struct platform_device *);
      struct device_driver driver;
    };  
      
    请注意，probe() 通常应该验证指定的设备硬件是否确实存在；有时平台设置代码无法确定。探测可以使用设备资源，包括时钟和设备 platform_data。
    - 平台驱动程序以正常方式注册自己：int platform_driver_register(struct platform_driver *drv);
    或者，在已知设备不可热插拔的常见情况下，probe() 例程可以位于 init 部分中，以减少驱动程序的运行时内存占用：
    int platform_driver_probe(struct platform_driver *drv,
                  int (*probe)(struct platform_device *))
    - 内核模块可以由多个平台驱动程序组成。平台核心提供帮助程序来注册和注销一系列驱动程序：
    int __platform_register_drivers(struct platform_driver * const *drivers,
                              unsigned int count, struct module *owner);
     void platform_unregister_drivers(struct platform_driver * const *drivers,
                                 unsigned int count);

- 设备枚举：通常，特定于平台（通常是特定于板）的设置代码将注册平台设备：
  int platform_device_register(struct platform_device *pdev);
  int platform_add_devices(struct platform_device **pdevs, int ndev);
     
   一般规则是只注册那些实际存在的设备，但在某些情况下可能会注册额外的设备。例如，内核可能被配置为与可能未安装在所有板上的外部网络适配器一起使用，或者同样与某些板可能无法连接到任何外围设备的集成控制器一起使用。在某些情况下，引导固件将导出描述在给定板上填充的设备的表。如果没有这些表，系统设置代码设置正确设备的唯一方法通常是为特定目标板构建内核。这种特定于板的内核在嵌入式和定制系统开发中很常见。

在许多情况下，与平台设备相关的内存和 IRQ 资源不足以让设备的驱动程序工作。板设置代码通常会使用设备的 platform_data 字段提供附加信息以保存附加信息。嵌入式系统经常需要一个或多个用于平台设备的时钟，这些时钟通常会保持关闭，直到它们被主动需要（以节省电力）。系统设置还将这些时钟与设备相关联，以便对 clk_get(&pdev->dev, clock_name) 的调用根据需要返回它们。   
      
- 旧版驱动程序：设备探测




- 设备命名和驱动绑定

platform_device.dev.bus_id 是设备的规范名称。它由两个组件构成：
  - platform_device.name ...也用于驱动匹配。
  - platform_device.id ... 设备实例编号，否则“-1”表示只有一个。

这些是串联的，所以name/id“serial”/0表示bus_id“serial.0”，“serial/3”表示bus_id“serial.3”；两者都将使用名为“serial”的平台驱动程序。而“my_rtc”/-1 将是 bus_id “my_rtc”（无实例 ID）并使用名为“my_rtc”的平台驱动程序。

驱动程序绑定由驱动程序核心自动执行，在找到设备和驱动程序之间的匹配后调用驱动程序探针（）。如果probe() 成功，则驱动程序和设备照常绑定。有三种不同的方法可以找到这样的匹配：

  - 每当注册设备时，都会检查该总线的驱动程序是否匹配。平台设备应在系统引导期间尽早注册。

  - 当使用 platform_driver_register() 注册驱动程序时，将检查该总线上的所有未绑定设备是否匹配。驱动程序通常在引导期间稍后注册，或者通过模块加载进行注册。

  - 使用 platform_driver_probe() 注册驱动程序的工作方式与使用 platform_driver_register() 类似，但如果其他设备注册，则以后不会探测该驱动程序。（没关系，因为此接口仅适用于非热插拔设备。）



开源固件和设备树
""""""""""""""
“Open Firmware Device Tree”，或简称 Devicetree (DT)，是一种用于描述硬件的数据结构和语言。更具体地说，它是对操作系统可读的硬件的描述，因此操作系统不需要对机器的细节进行硬编码。

在结构上，DT 是一棵树，或具有命名节点的无环图，节点可能具有封装任意数据的任意数量的命名属性。还存在一种机制来创建从一个节点到自然树结构之外的另一个节点的任意链接。

从概念上讲，定义了一组通用的使用约定，称为“绑定”，用于描述数据应如何出现在树中以描述典型的硬件特征，包括数据总线、中断线、GPIO 连接和外围设备。

尽可能使用现有绑定来描述硬件以最大限度地利用现有支持代码，但由于属性和节点名称只是文本字符串，因此很容易扩展现有绑定或通过定义新节点和属性来创建新绑定。但是，要小心，不要先对已经存在的内容做一些功课就创建新的绑定。目前出现了两种不同的、不兼容的 i2c 总线绑定，因为新绑定是在没有首先调查现有系统中如何枚举 i2c 设备的情况下创建的。


DT 最初由 Open Firmware 创建，作为将数据从 Open Firmware 传递到客户端程序（如操作系统）的通信方法的一部分。操作系统使用设备树在运行时发现硬件的拓扑结构，从而支持大多数可用硬件而无需硬编码信息（假设驱动程序可用于所有设备）。

由于 Open Firmware 通常用于 PowerPC 和 SPARC 平台，因此 Linux 对这些架构的支持长期以来一直使用设备树。

2005 年，当 PowerPC Linux 开始进行重大清理并合并 32 位和 64 位支持时，决定要求所有 powerpc 平台都支持 DT，无论它们是否使用 Open Firmware。为此，创建了一个称为扁平设备树 (FDT) 的 DT 表示，它可以作为二进制 blob 传递给内核，而无需真正的开放固件实现。U-Boot、kexec 和其他引导加载程序已修改为支持传递设备树二进制文件 (dtb) 并在引导时修改 dtb。arch/powerpc/boot/*DT也被添加到 PowerPC 引导包装器（

一段时间后，FDT 基础设施被推广到所有架构都可以使用。在撰写本文时，6 个主线架构（arm、microblaze、mips、powerpc、sparc 和 x86）和 1 个主线外（nios）具有一定程度的 DT 支持。


设备树规范：
***********
https://www.devicetree.org/specifications/

要理解的最重要的一点是，DT 只是一个描述硬件的数据结构。它没有什么神奇之处，它也不会神奇地让所有硬件配置问题都消失。它所做的是提供一种语言，用于将硬件配置与 Linux 内核（或任何其他操作系统）中的板卡和设备驱动程序支持分离。使用它可以让电路板和设备支持成为数据驱动的；根据传递到内核的数据而不是每台机器的硬编码选择来做出设置决策。

理想情况下，数据驱动的平台设置应该会减少代码重复，并更容易使用单个内核映像支持广泛的硬件。

Linux 将 DT 数据用于三个主要目的：

1. platform identification,
首先，内核将使用 DT 中的数据来识别特定的机器。在一个完美的世界中，特定平台对内核应该无关紧要，因为所有平台细节都将由设备树以一致且可靠的方式完美描述。虽然硬件并不完美，因此内核必须在早期引导期间识别机器，以便它有机会运行特定于机器的修复程序。

在大多数情况下，机器身份无关紧要，内核将根据机器的核心 CPU 或 SoC 选择设置代码。例如，在 ARM 上，arch/arm/kernel/setup.c 中的 setup_arch() 将调用 arch/arm/kernel/devtree.c 中的 setup_machine_fdt() 搜索 machine_desc 表并选择与设备树数据最匹配的 machine_desc . 它通过查看根设备树节点中的“兼容”属性来确定最佳匹配，并将其与 struct machine_desc 中的 dt_compat 列表（如果您在 arch/arm/include/asm/mach/arch.h 中定义）进行比较'很好奇）。

'compatible' 属性包含一个排序的字符串列表，以机器的确切名称开头，后跟一个可选的与其兼容的板列表，从最兼容到最不兼容。例如，TI BeagleBoard 及其继任者 BeagleBoard xM 板的根兼容属性可能分别如下所示：
compatible = "ti,omap3-beagleboard", "ti,omap3450", "ti,omap3";
compatible = "ti,omap3-beagleboard-xm", "ti,omap3450", "ti,omap3";

在“ti,omap3-beagleboard-xm”指定确切型号的地方，它还声称它与 OMAP 3450 SoC 以及一般的 omap3 系列 SoC 兼容。您会注意到该列表从最具体（确切的板）到最不具体的（SoC 系列）排序。

精明的读者可能会指出，Beagle xM 还可以声称与原始 Beagle 板兼容。然而，应该注意在董事会层面这样做，因为从一个董事会到另一个董事会通常会有很大程度的变化，即使在同一产品线中也是如此，而且当一个董事会声称时，很难确定到底是什么意思与另一个兼容。对于顶级，最好谨慎行事，不要声称一个板与另一个板兼容。值得注意的例外是当一个板是另一个板的载体时，例如连接到载体板上的 CPU 模块。

关于兼容值的另一个说明。兼容属性中使用的任何字符串都必须记录它所指示的内容。在 Documentation/devicetree/bindings 中添加兼容字符串的文档。

同样在 ARM 上，对于每个 machine_desc，内核会查看是否有任何 dt_compat 列表条目出现在 compatible 属性中。如果有，那么 machine_desc 就是驱动机器的候选者。在搜索整个 machine_desc 表后，setup_machine_fdt() 根据每个 machine_desc 匹配的 compatible 属性中的哪个条目返回“最兼容”的 machine_desc。如果没有找到匹配的 machine_desc，则返回 NULL。

该方案背后的原因是观察到，在大多数情况下，如果它们都使用相同的 SoC 或相同的 SoC 系列，单个 machine_desc 可以支持大量板。但是，总会有一些例外情况，即特定板需要特殊设置代码，这在一般情况下无用。特殊情况可以通过在通用设置代码中明确检查有问题的板来处理，但是如果不止几个情况，这样做很快就会变得丑陋和/或无法维护。

相反，兼容列表允许通用 machine_desc 通过在 dt_compat 列表中指定“不太兼容”的值来为广泛的通用板集提供支持。在上面的示例中，通用板支持可以声称与“ti,omap3”或“ti,omap3450”兼容。如果在早期启动期间在原始 beagleboard 上发现需要特殊解决方法代码的错误，则可以添加一个新的 machine_desc 来实现解决方法并且仅匹配“ti,omap3-beagleboard”。

PowerPC 使用稍微不同的方案，它从每个 machine_desc 调用 .probe() 挂钩，并使用第一个返回 TRUE 的挂钩。但是，这种方法没有考虑兼容列表的优先级，并且可能应该避免用于新架构支持。





2. runtime configuration, and

在大多数情况下，DT 将是从固件向内核传递数据的唯一方法，因此也用于传递运行时和配置数据，例如内核参数字符串和 initrd 映像的位置。

大部分数据都包含在 /chosen 节点中，当引导 Linux 时，它看起来像这样：

chosen {
        bootargs = "console=ttyS0,115200 loglevel=8";
        initrd-start = <0xc8000000>;
        initrd-end = <0xc8200000>;
};


bootargs 属性包含内核参数，而 initrd-* 属性定义 initrd blob 的地址和大小。请注意，initrd-end 是 initrd 映像之后的第一个地址，因此这与结构资源的通常语义不匹配。选择的节点还可以可选地包含任意数量的特定于平台的配置数据的附加属性。

在早期启动期间，架构设置代码调用 of_scan_flat_dt() 多次使用不同的辅助回调来解析设备树数据，然后再设置分页。of_scan_flat_dt() 代码扫描设备树并使用帮助程序提取早期启动期间所需的信息。通常， early_init_dt_scan_chosen() 帮助器用于解析所选节点，包括内核参数， early_init_dt_scan_root() 用于初始化 DT 地址空间模型， early_init_dt_scan_memory() 用于确定可用 RAM 的大小和位置。

在 ARM 上，函数 setup_machine_fdt() 负责在选择支持板子的正确 machine_desc 后对设备树进行早期扫描。







3. device population.


识别出板子，解析出早期的配置数据后，就可以正常进行内核初始化了。在此过程的某个时刻，调用 unflatten_device_tree() 将数据转换为更有效的运行时表示。这也是调用特定于机器的设置挂钩的时候，例如 ARM 上的 machine_desc .init_early()、.init_irq() 和 .init_machine() 挂钩。本节的其余部分使用来自 ARM 实现的示例，但所有架构在使用 DT 时都会做几乎相同的事情。

从名称可以猜到，.init_early() 用于需要在引导过程早期执行的任何特定于机器的设置，而 .init_irq() 用于设置中断处理。使用 DT 不会实质性地改变这些函数中的任何一个的行为。如果提供了 DT，则 .init_early() 和 .init_irq() 都可以调用任何 DT 查询函数（在 include/linux/of*.h 中的 of_*）来获取有关平台的其他数据。

DT 上下文中最有趣的钩子是 .init_machine() ，它主要负责用有关平台的数据填充 Linux 设备模型。从历史上看，这已在嵌入式平台上实现，方法是在板支持 .c 文件中定义一组静态时钟结构、platform_devices 和其他数据，并将其整体注册到 .init_machine() 中。当使用 DT 时，不再为每个平台硬编码静态设备，而是可以通过解析 DT 并动态分配设备结构来获得设备列表。

最简单的情况是 .init_machine() 只负责注册一个平台设备块。platform_device 是 Linux 用于硬件无法检测到的内存或 I/O 映射设备以及“复合”或“虚拟”设备的概念（稍后将详细介绍）。虽然 DT 没有“平台设备”术语，但平台设备大致对应于树根处的设备节点和简单内存映射总线节点的子节点。

现在是展示示例的好时机。以下是 NVIDIA Tegra 板的设备树的一部分：

/{
      compatible = "nvidia,harmony", "nvidia,tegra20";
      #address-cells = <1>;
      #size-cells = <1>;
      interrupt-parent = <&intc>;

      chosen { };
      aliases { };

      memory {
              device_type = "memory";
              reg = <0x00000000 0x40000000>;
      };

      soc {
              compatible = "nvidia,tegra20-soc", "simple-bus";
              #address-cells = <1>;
              #size-cells = <1>;
              ranges;

              intc: interrupt-controller@50041000 {
                      compatible = "nvidia,tegra20-gic";
                      interrupt-controller;
                      #interrupt-cells = <1>;
                      reg = <0x50041000 0x1000>, < 0x50040100 0x0100 >;
              };

              serial@70006300 {
                      compatible = "nvidia,tegra20-uart";
                      reg = <0x70006300 0x100>;
                      interrupts = <122>;
              };

              i2s1: i2s@70002800 {
                      compatible = "nvidia,tegra20-i2s";
                      reg = <0x70002800 0x100>;
                      interrupts = <77>;
                      codec = <&wm8903>;
              };

              i2c@7000c000 {
                      compatible = "nvidia,tegra20-i2c";
                      #address-cells = <1>;
                      #size-cells = <0>;
                      reg = <0x7000c000 0x100>;
                      interrupts = <70>;

                      wm8903: codec@1a {
                              compatible = "wlf,wm8903";
                              reg = <0x1a>;
                              interrupts = <347>;
                      };
              };
      };

      sound {
              compatible = "nvidia,harmony-sound";
              i2s-controller = <&i2s1>;
              i2s-codec = <&wm8903>;
      };
};



在 .init_machine() 时，Tegra 板支持代码将需要查看此 DT 并决定为哪些节点创建 platform_devices。但是，查看树，并不能立即看出每个节点代表什么样的设备，或者即使一个节点根本代表一个设备。/chosen、/aliases 和 /memory 节点是不描述设备的信息节点（尽管可以说内存可以被视为设备）。/soc 节点的子节点是内存映射设备，但codec @ 1a是 i2c 设备，sound 节点代表的不是设备，而是其他设备如何连接在一起创建音频子系统。我知道每个设备是什么，因为我熟悉电路板设计，但是内核如何知道如何处理每个节点？

诀窍是内核从树的根开始并寻找具有“兼容”属性的节点。首先，通常假设任何具有“兼容”属性的节点都代表某种设备，其次，可以假设树根处的任何节点要么直接连接到处理器总线，要么是无法以任何其他方式描述的杂项系统设备。对于这些节点中的每一个，Linux 分配并注册一个 platform_device，而后者又可能绑定到一个 platform_driver。

为什么对这些节点使用 platform_device 是一个安全的假设？好吧，对于 Linux 对设备建模的方式，几乎所有 bus_types 都假定它的设备是总线控制器的子设备。例如，每个 i2c_client 都是 i2c_master 的子代。每个 spi_device 都是 SPI 总线的子设备。对于 USB、PCI、MDIO 等也是如此。在 DT 中也可以找到相同的层次结构，其中 I2C 设备节点仅作为 I2C 总线节点的子节点出现。同样适用于 SPI、MDIO、USB 等。唯一不需要特定类型父设备的设备是 platform_devices（和 amba_devices，但稍后会详细介绍），它们将愉快地存在于 Linux /sys/devices 的基础上树。因此，如果 DT 节点位于树的根部，那么它确实可能最好注册为 platform_device。

Linux 板支持代码调用 of_platform_populate(NULL, NULL, NULL, NULL) 以开始在树的根部发现设备。参数都是 NULL，因为从树的根开始时，不需要提供起始节点（第一个 NULL）、父节点（最后一个 NULL），而且我们还没有使用匹配表. 对于只需要注册设备的板子，.init_machine() 可以完全为空，除了 调用。struct deviceof_platform_populate()

在 Tegra 示例中，这说明了 /soc 和 /sound 节点，但是 SoC 节点的子节点呢？他们不应该也注册为平台设备吗？对于 Linux DT 支持，一般行为是由父设备驱动程序在驱动程序 .probe() 时间注册子设备。因此，i2c 总线设备驱动程序将为每个子节点注册一个 i2c_client，SPI 总线驱动程序将注册其 spi_device 子节点，其他总线类型也是如此。根据该模型，可以编写一个绑定到 SoC 节点并简单地为其每个子节点注册 platform_devices 的驱动程序。板卡支持代码将分配和注册 SoC 设备，（理论上）SoC 设备驱动程序可以绑定到 SoC 设备，并为 /soc/interrupt-controller、/soc/serial、/soc/i2s 和 /soc 注册 platform_devices /i2c 在其 . 探针（）钩子。容易，对吧？

实际上，将某些平台设备的子设备注册为更多平台设备是一种常见模式，设备树支持代码反映了这一点，并使上述示例更简单。to 的第二个参数of_platform_populate()是一个 of_device_id 表，任何与该表中的条目匹配的节点也将注册其子节点。在 Tegra 案例中，代码可能如下所示：


static void __init harmony_init_machine(void)
{
      /* ... */
      of_platform_populate(NULL, of_default_bus_match_table, NULL, NULL);
}




“simple-bus”在 Devicetree 规范中被定义为一个属性，意思是一个简单的内存映射总线，因此of_platform_populate()可以编写代码来假设简单总线兼容的节点将始终被遍历。但是，我们将它作为参数传递，以便板支持代码始终可以覆盖默认行为。

【添加i2c/spi/etc子设备需要补充讨论】

- 附录 A：AMBA 设备
ARM Primecells 是一种附加到 ARM AMBA 总线的设备，其中包括对硬件检测和电源管理的一些支持。在 Linux 中，struct amba_device 和 amba_bus_type 用于表示 Primecell 设备。然而，棘手的一点是，并非 AMBA 总线上的所有设备都是 Primecell，对于 Linux，amba_device 和 platform_device 实例通常是同一总线段的兄弟。

使用 DT 时，这会产生问题，of_platform_populate() 因为它必须决定是否将每个节点注册为 platform_device 或 amba_device。不幸的是，这使设备创建模型稍微复杂了一点，但结果证明该解决方案并没有太大的侵入性。如果节点与“arm,amba-primecell”兼容， of_platform_populate()则将其注册为 amba_device 而不是 platform_device。


ARM架构需要来一遍！！！！！！！！！！！！！！！！！！！



开源固件设备树单元测试
*******************

DeviceTree 内核 API
********************

设备树变更集
***********

Devicetree 动态解析器​​说明
***********************

设备树覆盖注释
*************


设备树 (DT) ABI
***************

用于设计和编写 Devicetree 绑定的注意事项
***********************************

在 json-schema 中编写 Devicetree 绑定
***********************************


提交 Devicetree (DT) 绑定补丁
*****************************




ACPI设备树
""""""""""""""

ACPI 设备树 - ACPI 命名空间的表示

https://www.kernel.org/doc/html/latest/firmware-guide/acpi/namespace.html



PC总线设备
""""""""""""

PCI 驱动程序
*************
https://www.kernel.org/doc/html/latest/PCI/pci.html

PCI-E端口总线驱动
****************


PCI-E I/O虚拟化
****************





MSI驱动指南
**************



PCI设备资源 sysfs接口
*******************



PCI主桥的ACPI配置
*****************





PCI错误恢复
************



PCI Express 高级错误报告驱动程序指南
*********************************


PCI 端点框架
*************





PCI中断
**************

PCI资源遍历
*************






























