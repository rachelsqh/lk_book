linux 资源管理: resource
--------------------------

资源管理数据结构
^^^^^^^^^^^^^^^^
针对物理地址空间的定义。

.. code-block:: c
   :caption: struct resource
   :emphasize-lines: 4,5
   :linenos:

	/*
 	* Resources are tree-like, allowing
 	* nesting etc..
	 */
	struct resource {
		resource_size_t start;
		resource_size_t end;
		const char *name; //名称
		unsigned long flags;//bus-specific bit(0:7) : resource type(8:12): ...参考头文件定义
		unsigned long desc; //IO资源描述：eg:IORES_DESC_CRASH_KERNEL/IORES_DESC_ACPI_TABLES/IORES_DESC_ACPI_NV_STORAGE .
		struct resource *parent, *sibling, *child; //树型组织方式
	};

struct resource 全局定义
^^^^^^^^^^^^^^^^^^^^^^^

- PCI IO

.. code-block:: c
   :caption: struct resource
   :emphasize-lines: 4,5
   :linenos:
   
	struct resource ioport_resource = {
		.name	= "PCI IO",
		.start	= 0,
		.end	= IO_SPACE_LIMIT,
		.flags	= IORESOURCE_IO,
	};
	EXPORT_SYMBOL(ioport_resource);

- PCI mem

.. code-block:: c
   :caption: struct resource
   :emphasize-lines: 4,5
   :linenos:
   
   	struct resource iomem_resource = {
		.name	= "PCI mem",
		.start	= 0,
		.end	= -1,
		.flags	= IORESOURCE_MEM,
	};
	EXPORT_SYMBOL(iomem_resource);
	
	
	
	
	
	
	
	
	
