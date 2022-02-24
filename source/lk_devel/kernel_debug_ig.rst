kernel debug 实验记录
--------------------
host
^^^^^^
os: kali
hw:x86 8 core
vm:
  vma 3.2.0
 
guest
^^^^^^
os: kali
hw: x86 2 core
mem: 2048 M

kernel: 5.14

vma虚拟机基本操作
^^^^^^^^^^^^^^^
https://virt-manager.org/

virt-manager应用程序是一个桌面用户界面，用于通过 libvirt 管理虚拟机。它主要针对KVM虚拟机，但也管理Xen和LXC（Linux 容器）。它提供了正在运行的域、它们的实时性能和资源利用率统计信息的摘要视图。向导可以创建新域，以及配置和调整域的资源分配和虚拟硬件。嵌入式 VNC 和 SPICE 客户端查看器向来宾域提供完整的图形控制台。
关于 virt-manager 的配套工具
virt-install是一个命令行工具，它提供了一种将操作系统配置到虚拟机中的简单方法。

virt-viewer是一个轻量级的 UI 界面，用于与虚拟客户操作系统的图形显示进行交互。它可以显示 VNC 或 SPICE，并使用 libvirt 查找图形连接详细信息。

virt-clone是一个用于克隆现有非活动客户的命令行工具。它复制磁盘映像，并使用指向复制磁盘的新名称、UUID 和 MAC 地址定义配置。

virt-xml是一个命令行工具，用于使用 virt-install 的命令行选项轻松编辑 libvirt 域 XML。

virt-bootstrap是一个命令行工具，提供了一种简单的方法来为基于 libvirt 的容器设置根文件系统。

虚拟机xml设置
""""""""""""
https://libvirt.org/formatdomain.html




虚拟机与宿主机间信息共享
"""""""""""""""""""""


.. code-block:: c
	:caption: host与guest共享文件
	:emphasize-lines: 4,5
	:linenos:
	
       <filesystem type="mount" accessmode="passthrough">
         <source dir="/usr/src"/>
         <target dir="/usr/src"/>
       </filesystem>
    
 
3，开启虚拟机，在虚拟机上执行：


.. code-block:: c
	:caption: guest加载共享文件
	:emphasize-lines: 4,5
	:linenos:
	
	# mount /usr/src /usr/src -t 9p -o trans=virtio   
    
    
不同虚拟机镜像格式转换
"""""""""""""""""""
- VMDK–>qcow2:

  qemu-img convert -f vmdk -O qcow2 SLES11SP1-single.vmdk SLES11SP1-single.img

- qcow2–>raw:

  qemu-img convert -O qcow2 image-raw.raw image-raw-converted.qcow

- raw-> qcow2:

  qemu-img convert -f raw -O qcow2 2fuel2.img  2fuel2.qcow2                    

- 其他格式转换。

磁盘扩容
^^^^^^^^
https://blog.51cto.com/u_11101184/3136512


内核函数跟踪
^^^^^^^^^^^^^
内核配置
""""""""

跟踪demo
^^^^^^^^^




