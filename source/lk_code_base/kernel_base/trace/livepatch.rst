.. Rachel's E-book documentation master file, created by
   sphinx-quickstart on Sun Jan  9 16:38:00 2022.
   You can adapt this file completely to your liking, but it should at least
   contain the root `toctree` directive.

linux 实时补丁-livepatch
--------------------------

kernel:v5.10.13
二进制组织格式
linux内核代码流程，加载原理，冲突总结等。



livepatch
^^^^^^^^^^^

本文对内核实时补丁进行概要性描述。

1. 动机：在许多情况下，用户都不愿意重新引导系统。可能是因为他们的系统正在执行复杂的科学计算，或者在高峰使用期间处于高负载下。除了保持系统正常运行之外，用户还希望拥有一个稳定而安全的系统。 Livepatching通过允许重定向函数调用来为用户提供服务；这样，无需系统重新启动即可修复关键功能。

2. Kprobes,Ftrace,Livepatching：Linux内核中有多种与代码执行重定向直接相关的机制。即：内核探测，函数跟踪和实时修补：

   1. 内核探针是最通用的。可以通过放置断点指令来重定向代码;
   2. 函数跟踪器从靠近函数入口点的预定义位置调用代码。该位置是由编译器使用“ -pg” gcc选项生成的(空指令替换？）;
   3. Livepatching通常需要在修改函数参数或堆栈之前，在函数条目的最开始处重定向代码;

   这三种方法都需要在运行时修改现有代码。 因此，他们需要彼此了解，不要跨过彼此的脚趾。 通过使用动态ftrace框架作为基础可以解决大多数这些问题。 探测到函数条目时，会将Kprobe注册为ftrace处理程序，请参见CONFIG_KPROBES_ON_FTRACE。 此外，还可以通过自定义ftrace处理程序调用实时补丁的替代功能。 但是有一些限制，请参见下文。

3. 一致性模型:**有功能是有原因的**。 它们采用一些输入参数，获取或释放锁，读取，处理甚至以定义的方式写入某些数据，并具有返回值。 换句话说，每个功能都有定义的语义。 

   许多修复程序不会更改已修改函数的语义。 例如，它们添加NULL指针或边界检查，通过添加缺少的内存屏障来解决争用问题，或在关键部分周围添加一些锁定。 这些更改大多数都是自包含的，并且该功能以相同的方式向系统的其余部分显示。 在这种情况下，功能可能会一个接一个地独立更新。 

   但是，还有更复杂的修复程序。 例如，补丁可能会同时更改多个功能中的锁定顺序。 或者补丁可以交换一些临时结构的含义并更新所有相关功能。 在这种情况下，受影响的单元（线程，整个内核）需要同时开始使用所有新版本的功能。 而且，只有在安全的情况下，例如，在进行切换时，才必须进行切换。 当受影响的锁被释放或当前没有数据存储在修改后的结构中时。

   关于如何安全地应用功能的理论非常复杂。 目的是定义一个所谓的一致性模型。 它尝试定义可以使用新实现的条件，以使系统保持一致。 

   Livepatch的一致性模型是kGraft和kpatch的混合体：它使用kGraft的每任务一致性和syscall障碍切换以及kpatch的堆栈跟踪切换。 还有许多后备选项，使其非常灵活。 

   当认为任务可以安全切换时，将基于每个任务应用补丁。 启用修补程序后，livepatch会进入过渡状态，在此状态下任务将收敛到修补状态。 通常，此过渡状态可以在几秒钟内完成。 禁用修补程序时，发生相同的顺序，除了任务从修补状态收敛到未修补状态。 

   中断处理程序继承其中断的任务的修补状态。 分叉任务也是如此：子代继承父代的修补状态。 

   Livepatch使用几种补充方法来确定何时可以安全地修补任务:

   1. 第一种也是最有效的方法是对睡眠任务进行堆栈检查。 如果给定任务的堆栈上没有受影响的函数，则对该任务进行修补。 在大多数情况下，这将在第一次尝试时修补大多数或所有任务。 否则，它将继续定期尝试。 仅当体系结构具有可靠的堆栈（HAVE_RELIABLE_STACKTRACE）时，此选项才可用。 
   2. 如果需要，第二种方法是内核出口切换。 当任务从系统调用，用户空间IRQ或信号返回到用户空间时，将切换任务。 在以下情况下很有用：
      1. 修补在受影响的功能上处于休眠状态的受I / O约束的用户任务。 在这种情况下，您必须发送SIGSTOP和SIGCONT强制其退出内核并进行修补。 
      2. 修补受CPU约束的用户任务。 如果任务是CPU高度绑定的，则下次被IRQ中断时将对其进行修补。 
   3. 对于空闲的“交换器”任务，由于它们从未退出内核，因此它们在空闲循环中具有klp_update_patch_state（）调用，该调用使它们可以在CPU进入空闲状态之前进行修补。

   （请注意，目前还没有针对kthreads的方法。） 

   没有HAVE_RELIABLE_STACKTRACE的架构仅依靠第二种方法。 在此功能返回之前，很可能某些任务可能仍在使用该功能的旧版本运行。 在这种情况下，您将必须发出信号。 这尤其适用于kthreads。 他们可能不会被唤醒，需要被迫。 请参阅下面的详细信息。 

   除非我们能提出另一种修补kthread的方法，否则内核livepatching不会完全支持没有HAVE_RELIABLE_STACKTRACE的体系结构。 

   / sys / kernel / livepatch / <patch> / transition文件显示修补程序是否正在过渡。 在给定时间只能转换一个补丁。 如果有任何任务停留在初始补丁程序状态，则补丁程序可以无限期保持过渡状态。 

   通过在转换进行过程中将相反的值写入/ sys / kernel / livepatch / <patch> / enabled文件，可以撤消转换并有效取消转换。 然后，所有任务将尝试收敛回到原始修补程序状态。 

   还有一个/ proc / <pid> / patch_state文件，可用于确定哪些任务阻止了修补操作的完成。 如果正在执行补丁程序，则此文件显示0表示任务尚未打补丁，显示1表示任务已打补丁。 否则，如果没有补丁在过渡，则显示-1。 可以使用SIGSTOP和SIGCONT发出任何阻止转换的任务的信号，以强制它们更改其修补状态。 但是，这可能对系统有害。 向所有剩余的阻止任务发送虚假信号是更好的选择。 实际上没有传递适当的信号（信号暂挂结构中没有数据）。 任务被中断或唤醒，并被迫更改其修补状态。 伪信号每15秒自动发送一次。 

   管理员还可以通过/ sys / kernel / livepatch / <patch> / force属性影响过渡。 在此处写入1将清除所有任务的TIF_PATCH_PENDING标志，从而将任务强制为修补状态。 重要的提示！ force属性适用于由于阻塞任务而导致过渡卡住很长时间的情况。 管理员应收集所有必要的数据（即此类阻止任务的堆栈跟踪），并请求补丁分发者许可以强制过渡。 未经授权的使用可能会损坏系统。 这取决于修补程序的性质，哪些功能是（未）修补的，阻塞任务正在休眠的是哪些功能（/ proc / <pid> / stack在这里可能会有所帮助）。 使用强制功能时，永久禁用补丁模块的移除（rmmod）。 无法保证在此类模块中没有任何任务处于休眠状态。 如果补丁模块在循环中被禁用和启用，则意味着无限制的引用计数。 

   此外，使用武力还可能会影响实时补丁的未来应用，甚至会对系统造成更大的伤害。 管理员应首先考虑简单地取消过渡（请参见上文）。 如果使用强制，则应计划重新启动，并且不再应用任何实时补丁。 

   1. ​	向新架构添加一致性模型支持 :为了向新架构添加一致性模型支持，有以下几种选择：

      1. 添加CONFIG_HAVE_RELIABLE_STACKTRACE。 这意味着要移植objtool，对于非DWARF展开器，还应确保堆栈跟踪代码有一种方法可以检测堆栈上的中断。 

      2. 或者，确保每个kthread在安全的位置都有对klp_update_patch_state（）的调用。 Kthread通常处于无限循环中，该循环会重复执行某些操作。 切换kthread补丁程序状态的安全位置将在循环中的指定点，其中没有采取任何锁定，并且所有数据结构都处于定义良好的状态。 

         使用工作队列或kthread worker API时，该位置很清楚。 这些kthread在通用循环中处理独立的动作。 

         它与具有自定义循环的kthreads更复杂。 必须在逐案的基础上仔细选择安全位置。

         在那种情况下，没有pass_relize_stacktrace的拱门仍然能够使用一致性模型的非堆叠检查部分：

         1. 在跨越内核/用户空间边界时修补用户任务; 和 
         2. 在其指定的补丁点修补kthreads和空闲任务。 

         此选项不像选项1一样好，因为它需要发信号通知用户任务并唤醒kthreads来修补它们。 但对于那些没有可靠的堆栈迹线，它仍然是一个很好的备份选项。 

4. livepatch 模块

   LiveCatches使用内核模块分发，请参阅示例/ LivePatch / LivePatch-Sample.c。 

   该模块包括我们要替换的功能的新实现。 此外，它定义了一些描述原始和新实现之间关系的结构。 然后，有代码使内核在加载LivePatch模块时使用新代码开始。 此外，还有在删除LivePatch模块之前清理的代码。 所有这些都在下一节中的更多细节中解释。 

   1. 新函数

      新版本的功能通常只是从原始来源复制。 良好的做法是向名称添加前缀，以便它们可以与原始的前缀区分开，例如，它们可以区分开。 在回程中。 此外，它们也可以被声明为静态，因为它们不会直接调用，不需要全局可见性。 

      该修补程序仅包含真正修改的函数。 但他们可能希望从原始源文件中访问函数或数据，该文件只能是本地可访问的。 这可以通过生成的LivePatch模块中的特殊重定位部分来解决，请参阅LivePatch模块ELF格式以获取更多详细信息。 

   2. 元数据

      该补丁由几个结构描述了将信息拆分为三个级别：

      1. 为每个修补函数定义struct klp_func。 它描述了原始功能与新实现之间的关系。 

         该结构包括原始函数的名称，作为字符串。 在运行时通过Kallsyms找到函数地址。

         然后它包括新功能的地址。 它通过分配函数指针直接定义。 请注意，新功能通常在相同的源文件中定义。 作为可选参数，Kallsyms数据库中的符号位置可用于消除相同名称的函数。 这不是数据库中的绝对位置，而是只针对特定对象（vmlinux或内核模块）找到的顺序。 请注意，Kallsyms允许根据对象名称搜索符号。

      2. struct klp_object在同一对象中定义了一个修补函数数组（结构klp_func）。 该对象是vmlinux（null）或模块名称的位置。

         该结构有助于为每个对象组合在一起并处理每个对象的功能。 请注意，修补模块可能会在稍后加载而不是修补程序本身，并且只有在可用时才会修补相关功能。 

      3. struct klp_patch定义了一个修补的对象数组（struct klp_object）。该结构始终如一地处理所有修补的功能，并最终同步地处理所有修补的功能。 仅在找到所有修补符号时才会应用整个补丁。 唯一的例外是尚未加载的对象（内核模块）的符号。 

         有关如何在每次任务的基础上应用补丁如何应用程序的更多详细信息，请参阅“一致性模型”部分。 

5. Livepatch生命周期 

   Livepatching可以通过五个基本操作来描述：加载，启用，替换，禁用，删除。 

   替换和禁用操作的互斥互斥。 它们对给定的补丁具有相同的结果，但不适用于系统。

   1. 装载 

      唯一合理的方式是在加载LivePatch内核模块时启用修补程序。 为此，必须在module_init（）回调中调用klp_enable_patch（）。 有两个主要原因：

      首先，只有模块才能轻松访问相关的结构klp_patch。 

      其次，当修补程序无法启用时，错误代码可用于拒绝加载模块。 

   2. 启用 

      通过从module_init（）回调的klp_enable_patch（）通过调用klp_enable_patch（）启用LivePatch。 该系统将在此阶段开始使用修补功能的新实现。

      首先，根据修补函数的名称查找它们的地址。 应用“新功能”一节中提到的特殊重定位。 相关条目在/ sys / kernel / livepatch / <名称>下创建。 当上述任何操作失败时，补丁将被拒绝。

      其次，livepatch进入过渡状态，在此状态下任务正在收敛到修补状态。 如果是第一次修补原始函数，则会创建特定于函数的struct klp_ops并注册通用ftrace处理程序1。 / sys / kernel / livepatch / <name> / transition中的值“ 1”表示该阶段。 有关此过程的更多信息，请参见“一致性模型”部分

      最后，修补完所有任务后，“ transition”值将变为“ 0”。 

      请注意，功能可能会多次打补丁。 对于给定的函数，ftrace处理程序仅注册一次。 进一步的补丁程序仅将一个条目添加到struct klp_ops的列表中（请参见func_stack字段）。 正确的实现由ftrace处理程序选择，请参见“一致性模型”部分。      

      也就是说，强烈建议使用累积实时修补程序，因为它们有助于保持所有更改的一致性。 在这种情况下，可能仅在过渡期间对功能进行了两次修补。 

   3. 更换 :所有启用的修补程序都可能会被设置了.replace标志的累积修补程序替换。 一旦启用了新补丁并完成了“转换”，与替换的补丁相关联的所有功能（结构klp_func）就会从相应的结构klp_ops中删除。 同样当相关功能未被新补丁修改且func_stack列表为空时，ftrace处理程序也将取消注册，并释放struct klp_ops。 有关更多详细信息，请参见原子替换和累积补丁。

   4. 禁用:通过将“ 0”写入/ sys / kernel / livepatch / <name> / enabled，可能会禁用已启用的修补程序。首先，livepatch进入过渡状态，在此状态下任务正在收敛到未修补状态。 系统开始使用以前启用的补丁中的代码，甚至使用原始补丁中的代码。 / sys / kernel / livepatch / <name> / transition中的值“ 1”表示该阶段。 有关此过程的更多信息，请参见“一致性模型”部分。 其次，一旦所有任务均未打补丁，“ transition”值将变为“ 0”。 与待禁用补丁相关联的所有功能（struct klp_func）都从相应的struct klp_ops中删除。 当func_stack列表为空时，将取消注册ftrace处理程序，并释放struct klp_ops。 第三，sysfs接口被破坏。 

   5. 卸载：仅当没有用户使用该模块提供的功能时，才可以安全地卸下模块。 这就是强制功能永久禁用删除的原因。 仅当系统成功转换为新的补丁程序状态（已补丁/未补丁）而没有被强制执行时，才可以确保没有任务在旧代码中休眠或运行。 

6. sysfs:可在/ sys / kernel / livepatch下找到有关已注册补丁的信息。 可以通过在其中写入来启用和禁用补丁。 / sys / kernel / livepatch / <patch> / force属性允许管理员影响修补操作。 有关更多详细信息，请参见文档/ ABI / testing / sysfs-kernel-livepatch。 

7. 局限性 :当前的Livepatch实现有几个限制： 

   1. 只能修补可以跟踪的功能。 Livepatch基于动态ftrace。 特别是，无法修补实现ftrace或livepatch ftrace处理程序的函数。 否则，代码将陷入无限循环。 通过用“ notrace”标记有问题的功能可以防止潜在的错误。 
   2. 仅当动态ftrace位于函数的开头时，Livepatch才能可靠地工作。 在以任何方式修改堆栈或函数参数之前，都需要对函数进行重定向。 例如，livepatch要求在x86_64上使用-fentry gcc编译器选项。PPC端口是一种例外。 它使用相对寻址和TOC。 每个函数都必须先处理TOC并保存LR，然后才能调用ftrace处理程序。 此操作必须在返回时恢复。 幸运的是，通用ftrace代码具有相同的问题，所有这些都在ftrace级别上进行了处理。 
   3. 使用ftrace框架的Kretprobes与修补的函数冲突。kretprobes和livepatches都使用ftrace处理程序来修改返回地址。 第一个用户获胜。 当处理程序已被另一个使用时，探针或修补程序都会被拒绝。 
   4. 当代码重定向到新的实现时，原始函数中的Kprobes被忽略。 正在进行一项工作以添加关于这种情况的警告。

---------------

（取消）修补回调函数

Livepatch（un）patch-callbacks为livepatch模块提供了一种机制，该机制可在（未）修补内核对象时执行回调函数。 可以将它们视为一项强大功能，将实时修补功能扩展为： 

1. 安全更新全局数据 ;
2. 初始化和探测功能的“补丁” ;
3. 修补否则无法修补的代码（即汇编）

在大多数情况下，（un）patch回调将需要与内存屏障和内核同步原语（例如互斥锁/自旋锁，甚至stop_machine（））结合使用，以避免并发问题。 

1. 动机：回调不同于现有的内核功能：

   1. 禁用和重新启用修补程序时，模块初始化/退出代码无法运行。
   2. 模块通知程序无法阻止要修补的模块加载。

   回调是klp_object结构的一部分，其实现特定于该klp_object。 不论目标klp_object的当前状态如何，其他livepatch对象都可能会被打补丁，也可能不会被打补丁。

2. 回调类型 ：可以为以下实时补丁操作注册回调： 

   1. Pre-patch：在修补klp_object之前 ;
   2. Post-patch：在对klp_object进行修补并在所有任务中处于活动状态之后 ;
   3. Pre-unpatch：在未对klp_object进行修补之前（即，已修补的代码处于活动状态），用于清理修补后的回调资源 
   4. Post-unpatch：修补了klp_object之后，所有代码都已还原，并且没有任务正在运行修补的代码，用于清除修补前的回调资源 

3. How it works:

   每个回调都是可选的，省略一个并不排除指定其他任何回调。 但是，livepatching核心是对称地执行处理程序的：补丁前的回调有后释放的对应，而补丁后的回调有前释放的对应。 仅当执行了其对应的补丁程序回调时，才会执行unpatch回调程序。 典型的用例是将获取和配置资源的补丁处理程序与取消补丁处理程序配对，以拆除并释放相同的资源。 

   仅当加载了其主机klp_object时才执行回调。 对于内核内vmlinux目标，这意味着启用/禁用livepatch时将始终执行回调。 对于补丁程序目标内核模块，仅当目标模块已加载时才执行回调。 加载（取消）模块目标时，仅当启用livepatch模块时，才会执行其回调。

   补丁程序前的回调，如果已指定，则应返回状态码（0为成功，-ERRNO为错误）。 错误状态代码向livepatching内核指示对当前klp_object的修补是不安全的，并且将停止当前的修补请求。 （如果未提供补丁前回调，则认为过渡是安全的。）如果补丁前回调返回失败，则内核的模块加载器将：

   1. 如果在目标代码之后加载了实时补丁，则拒绝加载实时补丁。 或
   2. 如果已成功加载livepatch，则拒绝加载模块。

   如果由于pre_patch回调失败或任何其他原因而导致对象修补失败，则不会为给定的klp_object执行补丁后，补丁前或补丁后回调。如果补丁转换被反向，则将不运行预补丁处理程序（这遵循前面提到的对称性-仅在执行其相应的补丁后回调时才会发生补丁前回调）。 如果对象确实成功修补，但是由于某种原因（例如，如果另一个对象修补失败），修补过渡从未开始，则仅会调用解后的回调。

4. 用例 ：可以在samples / livepatch /目录中找到演示回调API的示例livepatch模块。 修改了这些样本以用于kselftests，可以在lib / livepatch目录中找到它们。 

5. 全局数据更新 ：补丁程序前的回调对更新全局变量很有用。 例如，75ff39ccc1bd（“ tcp：降低挑战性可预测性”）更改了全局sysctl，并修补了tcp_send_challenge_ack（）函数。 在这种情况下，如果我们超级偏执，那么在修补完成后使用修补程序后的回调对数据进行修补可能是有意义的，因此可以首先将tcp_send_challenge_ack（）更改为使用READ_ONCE读取sysctl_tcp_challenge_ack_limit。 

6. __init和探针功能补丁支持 ： 

   尽管__init和probe函数不能直接进行实时修补，但是可以通过修补前/修补后回调实现类似的更新。 提交48900cb6af42（“ virtio-net：删除NETIF_F_FRAGLIST”）更改了virtnet_probe（）初始化其驱动程序的net_device功能的方式。 补丁前/补丁后回调可以遍历所有此类设备，并对它们的hw_features值进行类似的更改。 （该值的客户端功能可能需要相应地更新。） 

   -------------

   # 原子替换和累积补丁 

   实时补丁之间可能存在依赖关系。 如果多个补丁程序需要对同一功能进行不同的更改，那么我们需要定义补丁程序的安装顺序。 而且，任何较新的livepatch的函数实现都必须在较旧的livepatch之上完成。 

   这可能会成为维护的噩梦。 尤其是当更多补丁以不同方式修改相同功能时。

   一个优雅的解决方案带有称为“原子替换”的功能。 它允许创建所谓的“累积补丁”。 它们包括所有较旧的实时补丁的所有所需更改，并在一个过渡中完全替换了它们。 

   用法 

   ```
   static struct klp_patch patch = {
           .mod = THIS_MODULE,
           .objs = objs,
           .replace = true,
   };
   ```

   然后，将所有进程迁移为仅使用新补丁中的代码。 过渡完成后，将自动禁用所有较旧的补丁程序。 

   Ftrace处理程序将从不再由新的累积修补程序修改的函数中透明删除。

   结果，实时补丁作者可能只维护一个累积补丁的来源。 在添加或删除各种修补程序或功能时，它有助于使修补程序保持一致。 

   转换完成后，用户只能保留系统上安装的最后一个修补程序。 它有助于清楚地看到实际使用的代码。 然后，livepatch可能会被视为修改内核行为的“正常”模块。 唯一的区别是可以在运行时更新它而不会破坏其功能。

## 特征 

原子替换允许：

- 在升级其他功能时，以原子方式还原先前补丁中的某些功能。
- 消除由于核心重定向对不再打补丁的功能造成的最终性能影响。 
- 减少用户对实时补丁之间的依赖关系的困惑。 

## 局限性：

- 操作完成后，没有直接的方法可以将其恢复并自动恢复被替换的补丁。 好的做法是在任何已发布的Livepatch中设置.replace标志。 然后，重新添加较旧的livepatch等效于降级到该补丁。 只要livepatches在（未）修补回调中或在module_init（）或module_exit（）函数中进行_not_额外的修改，这是安全的，请参见下文。 还要注意，只有在不强制过渡的情况下，才能删除并重新加载替换的补丁。 
- 仅执行_new_累积livepatch中的（未）补丁回调。 来自替换补丁的任何回调都将被忽略。 换句话说，累积修补程序负责执行适当替换任何较旧修补程序所必需的任何操作。 结果，用较旧的累积补丁替换较新的累积补丁可能很危险。 旧的实时修补程序可能未提供必要的回调。在某些情况下，这可能被视为限制。 但这使许多其他人的生活更加轻松。 只有新的累积性Livepatch知道添加/删除了哪些修复程序/功能，以及为平稳过渡需要采取哪些特殊措施。 无论如何，如果调用了所有已启用补丁的回调，则考虑各种回调的顺序及其交互将是一场噩梦。
- 阴影变量没有特殊处理。 Livepatch作者必须创建自己的规则，如何将它们从一个累积修补程序传递到另一个累积修补程序。 特别是它们不应该在module_exit（）函数中盲目地将其删除。 一个好的实践可能是在解压后回调中删除阴影变量。 仅当适当禁用livepatch时，才调用它。 

----------------

Livepatch模块Elf格式 
^^^^^^^^^^^^^^^^^^^^^

1.背景和动机 

以前，livepatch需要单独的体系结构特定代码来编写重定位。 但是，模块加载程序中已经存在用于写重定位的特定于arch的代码，因此前一种方法产生了冗余代码。 因此，livepatch无需复制代码并重新实现模块加载器已经可以执行的操作，而是利用模块加载器中的现有代码来执行所有特定于架构的重定位工作。 具体而言，livepatch重用模块加载器中的apply_relocate_add（）函数以写入重定位。 本文档中描述的补丁模块Elf格式使livepatch能够执行此操作。 希望这将使livepatch更容易移植到其他体系结构，并减少将livepatch移植到特定体系结构所需的特定于arch的代码量。

由于apply_relocate_add（）要求访问模块的节头表，符号表和重定位节索引，因此将为livepatch模块保留Elf信息（请参阅第5节）。 Livepatch管理其自己的重定位部分和符号，本文档中对此进行了描述。 根据glibc的定义，从OS特定范围中选择了用于标记livepatch符号和重定位部分的Elf常数。 

为什么livepatch需要编写自己的重定位？ 
"""""""""""""""""""""""""""""""""

典型的livepatch模块包含可引用未导出的全局符号和未包含的本地符号的功能的修补版本。不能照原样保留引用这些类型的符号的重定位，因为内核模块加载器无法解析它们，因此将拒绝livepatch模块。此外，我们无法应用影响补丁程序模块加载时尚未加载的模块的重定位（例如，补丁程序未加载的驱动程序）。以前，livepatch通过在生成的补丁模块Elf输出中嵌入特殊的“ dynrela”（动态rela）部分来解决此问题。使用这些dynrela部分，livepatch可以在考虑符号范围和符号所属模块的情况下解析符号，然后手动应用动态重定位。但是，此方法需要livepatch提供特定于拱的代码才能编写这些重定位。在新格式中，livepatch代替dynrela节管理其自己的SHT_RELA重定位节，并且relas引用的符号是特殊的livepatch符号（请参见第2和3节）。特定于拱的livepatch重定位代码被对apply_relocate_add（）的调用所代替。 

2.Livepatch modinfo字段 

Livepatch模块必须具有“ livepatch” modinfo属性。 有关如何完成的操作，请参阅samples / livepatch /中的示例livepatch模块。

用户可以通过使用“ modinfo”命令并查找“ livepatch”字段的存在来标识Livepatch模块。 内核模块加载程序还使用此字段来标识实时补丁模块。 

例子：Modinfo输出： 

```
% modinfo livepatch-meminfo.ko
filename:               livepatch-meminfo.ko
livepatch:              Y
license:                GPL
depends:
vermagic:               4.3.0+ SMP mod_unload

```

# 3.Livepatch重定位部分 

livepatch模块管理自己的Elf重定位部分，以在适当的时候将重定位应用于模块以及内核（vmlinux）。 例如，如果修补程序模块修补当前未加载的驱动程序，则livepatch将在加载后将相应的livepatch重定位部分应用于驱动程序。 

补丁模块中的每个“对象”（例如vmlinux或模块）都可以具有与其关联的多个livepatch重定位部分（例如，同一对象中多个功能的补丁）。 实时修补程序重定位部分与适用重定位的目标部分（通常是函数的文本部分）之间存在1-1对应关系。 livepatch模块也可能没有livepatch的重定位部分，例如在示例livepatch模块的情况下（请参见samples / livepatch）。 

由于Elf信息是为livepatch模块保留的（请参见第5节），因此只需将适当的段索引传递给apply_relocate_add（），即可应用livepatch重定位节，然后使用它访问relocation节并应用重定位。 

实时修补程序重定位部分中，rela引用的每个符号都是实时修补程序符号。 必须先解决这些问题，然后livepatch才能调用apply_relocate_add（）。 有关更多信息，请参见第3节。 

## 3.1Livepatch重定位部分格式 

Livepatch重定位部分必须标记为SHF_RELA_LIVEPATCH部分标志。 有关定义，请参见include / uapi / linux / elf.h。 模块加载器会识别此标志，并将避免在补丁模块加载时应用那些重定位部分。 这些部分还必须标有SHF_ALLOC，以便模块加载程序不会在模块加载时将其丢弃（即，它们将与其他SHF_ALLOC部分一起复制到内存中）。 

livepatch重定位部分的名称必须符合以下格式： 

```
.klp.rela.objname.section_name
^        ^^     ^ ^          ^
|________||_____| |__________|
   [A]      [B]        [C]
```

- **[A]**

  重定位节名称的前缀为字符串“ .klp.rela”。 

- **[B]**

  重定位部分所属的对象的名称（即“ vmlinux”或模块的名称）紧随前缀之后。 

- **[C]**

  此重定位部分适用于的部分的实际名称。 

## 例子： 

Livepatch重定位部分的名称： 

```
.klp.rela.ext4.text.ext4_attr_store
.klp.rela.vmlinux.text.cmdline_proc_show
```

修补vmlinux和模块9p，btrfs，ext4的修补程序模块的“ readelf –sections”输出：

```
Section Headers:
[Nr] Name                          Type                    Address          Off    Size   ES Flg Lk Inf Al
[ snip ]
[29] .klp.rela.9p.text.caches.show RELA                    0000000000000000 002d58 0000c0 18 AIo 64   9  8
[30] .klp.rela.btrfs.text.btrfs.feature.attr.show RELA     0000000000000000 002e18 000060 18 AIo 64  11  8
[ snip ]
[34] .klp.rela.ext4.text.ext4.attr.store RELA              0000000000000000 002fd8 0000d8 18 AIo 64  13  8
[35] .klp.rela.ext4.text.ext4.attr.show RELA               0000000000000000 0030b0 000150 18 AIo 64  15  8
[36] .klp.rela.vmlinux.text.cmdline.proc.show RELA         0000000000000000 003200 000018 18 AIo 64  17  8
[37] .klp.rela.vmlinux.text.meminfo.proc.show RELA         0000000000000000 003218 0000f0 18 AIo 64  19  8
[ snip ]                                       ^                                             ^
                                               |                                             |
                                              [*]                                           [*]


```



**[\*]**

Livepatch重定位部分是SHT_RELA部分，但具有一些特殊特征。 请注意，它们被标记为SHF_ALLOC（“ A”），以便在将模块加载到内存中时以及SHF_RELA_LIVEPATCH标志（对于操作系统特定，“ o”）都不会被丢弃。 

补丁模块的`readelf –relocs`输出： 

```
Relocation section '.klp.rela.btrfs.text.btrfs_feature_attr_show' at offset 0x2ba0 contains 4 entries:
    Offset             Info             Type               Symbol's Value  Symbol's Name + Addend
000000000000001f  0000005e00000002 R_X86_64_PC32          0000000000000000 .klp.sym.vmlinux.printk,0 - 4
0000000000000028  0000003d0000000b R_X86_64_32S           0000000000000000 .klp.sym.btrfs.btrfs_ktype,0 + 0
0000000000000036  0000003b00000002 R_X86_64_PC32          0000000000000000 .klp.sym.btrfs.can_modify_feature.isra.3,0 - 4
000000000000004c  0000004900000002 R_X86_64_PC32          0000000000000000 .klp.sym.vmlinux.snprintf,0 - 4
[ snip ]                                                                   ^
                                                                           |
                                                                          [*]

```

**[\*]**

重定位引用的每个符号都是实时补丁符号。

# 4.Livepatch符号

实时补丁符号是实时补丁重定位部分所引用的符号。 这些是从修补程序对象的新功能版本访问的符号，模块加载程序无法解析其地址（因为它们是本地符号或未导出的全局符号）。 由于模块加载器仅解析导出的符号，而并非导出新修补函数引用的每个符号，因此引入了livepatch符号。 在补丁模块加载时我们无法立即知道符号地址的情况下，也可以使用它们。 例如，当livepatch修补尚未加载的模块时，就是这种情况。 在这种情况下，只需在目标模块加载时就可以解析相关的实时补丁符号。 无论如何，对于任何livepatch重定位节，必须先解析该节引用的所有livepatch符号，然后livepatch才能为该reloc节调用apply_relocate_add（）。 

必须用SHN_LIVEPATCH标记Livepatch符号，以便模块加载器可以识别和忽略它们。 Livepatch模块将这些符号保留在其符号表中，并且可以通过module-> symtab来访问符号表。 

## 4.1Livepatch模块的符号表 

通常，通过module-> symtab（请参阅kernel / module.c中的layout_symtab（））可以使用模块符号表的精简副本（仅包含“核心”符号）。 对于livepatch模块，在模块加载时复制到内存中的符号表必须与编译补丁模块时生成的符号表完全相同。 这是因为每个livepatch重定位部分中的重定位均使用其符号索引引用其各自的符号，并且必须保留原始符号索引（以及symtab顺序），以便apply_relocate_add（）找到正确的符号。 

例如，从livepatch模块获取以下特定关系：

```
Relocation section '.klp.rela.btrfs.text.btrfs_feature_attr_show' at offset 0x2ba0 contains 4 entries:
    Offset             Info             Type               Symbol's Value  Symbol's Name + Addend
000000000000001f  0000005e00000002 R_X86_64_PC32          0000000000000000 .klp.sym.vmlinux.printk,0 - 4

This rela refers to the symbol '.klp.sym.vmlinux.printk,0', and the symbol index is encoded
in 'Info'. Here its symbol index is 0x5e, which is 94 in decimal, which refers to the
symbol index 94.
And in this patch module's corresponding symbol table, symbol index 94 refers to that very symbol:
[ snip ]
94: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT OS [0xff20] .klp.sym.vmlinux.printk,0
[ snip ]
```

## 4.2Livepatch符号格式 

Livepatch符号必须将其部分索引标记为SHN_LIVEPATCH，以便模块加载器可以识别它们，而不尝试解析它们。 有关实际定义，请参见include / uapi / linux / elf.h。 

Livepatch符号名称必须符合以下格式： 

```
.klp.sym.objname.symbol_name,sympos
^       ^^     ^ ^         ^ ^
|_______||_____| |_________| |
   [A]     [B]       [C]    [D]

```

**[A]**

符号名称以字符串“ .klp.sym”为前缀。

**[B]**

该符号所属的对象名称（即“ vmlinux”或模块名称）紧随前缀之后。 

**[C]**

符号的实际名称。 

**[D]**

符号在对象中的位置（根据kallsyms），用于区分同一对象内的重复符号。 符号位置以数字表示（0、1、2…）。 唯一符号的符号位置为0。 

例子： 

Livepatch符号名称： 

```
.klp.sym.vmlinux.snprintf,0.klp.sym.vmlinux.printk,0.klp.sym.btrfs.btrfs_ktype,0
```

补丁模块的`readelf –symbols`输出： 

```
Symbol table '.symtab' contains 127 entries:   Num:    Value          Size Type    Bind   Vis     Ndx         Name   [ snip ]    73: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT OS [0xff20] .klp.sym.vmlinux.snprintf,0    74: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT OS [0xff20] .klp.sym.vmlinux.capable,0    75: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT OS [0xff20] .klp.sym.vmlinux.find_next_bit,0    76: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT OS [0xff20] .klp.sym.vmlinux.si_swapinfo,0  [ snip ]                                               ^                                                         |                                                        [*]
```

**[\*]**

请注意，这些符号的“ Ndx”（部分索引）为SHN_LIVEPATCH（0xff20）。 “ OS”是指特定于OS的。 

# 5.符号表和Elf节访问 

可通过module-> symtab访问livepatch模块的符号表。 

由于apply_relocate_add（）需要访问模块的节头，符号表和重定位节的索引，因此Elf信息将保留给livepatch模块，并可由模块加载器通过module-> klp_info（klp_modinfo结构）进行访问。 当livepatch模块加载时，该结构由模块加载器填充。 其字段记录如下： 

```
struct klp_modinfo {        Elf_Ehdr hdr; /* Elf header */        Elf_Shdr *sechdrs; /* Section header table */        char *secstrings; /* String table for the section headers */        unsigned int symndx; /* The symbol table section index */};
```





-------------------------------

# 阴影变量 

影子变量是livepatch模块将其他“影子”数据与现有数据结构关联的一种简单方法。 影子数据与父数据结构分开分配，而父数据结构保持不变。 本文档中描述的影子变量API用于向/从其父代分配/添加和删除/释放影子变量。 

该实现引入了全局的内核内哈希表，该哈希表将指向父对象的指针与影子数据的数字标识符相关联。 数字标识符是一个简单的枚举，可用于描述阴影变量版本，类或类型等。更具体地说，父指针用作哈希表键，而数字ID随后过滤哈希表查询。 多个阴影变量可以附加到同一父对象，但是它们的数字标识符可以区分它们。 

# 1.简要的API摘要 

（请参阅livepatch / shadow.c中的完整API使用情况docbook注释。） 

哈希表引用所有影子变量。 这些引用通过<obj，id>对进行存储和检索。 

- klp_shadow变量数据结构封装了跟踪元数据和影子数据： 

  - meta-data
    - obj-指向父对象的指针 
    - id-数据标识符 
  - data []-存储阴影数据 

  重要的是要注意，默认情况下，klp_shadow_alloc（）和klp_shadow_get_or_alloc（）将变量清零。 当需要非零值时，它们还允许调用自定义构造函数。 呼叫者应提供所需的任何互斥。 

  请注意，该构造函数在klp_shadow_lock自旋锁下调用。 它允许执行分配新变量后只能执行一次的操作。 

- klp_shadow_get（）-检索阴影变量数据指针-在<obj，id>对中搜索哈希表 

- klp_shadow_alloc（）-分配并添加一个新的影子变量-在<obj，id>对中搜索哈希表 

  - 如果存在：
    - WARN and return NULL
  - 如果<obj，id>还不存在 
    - 分配一个新的阴影变量 
    - 使用自定义构造函数和数据（如果提供）初始化变量 
    - 将<obj，id>添加到全局哈希表 

- klp_shadow_get_or_alloc（）-获取现有的或分配新的影子变量-搜索<obj，id>对的哈希表 

  - 如果存在
    - 返回现有的阴影变量 
  - 如果<obj，id>还不存在 
    - 分配一个新的阴影变量 
    - 使用自定义构造函数和数据（如果提供）初始化变量 
    - 将<obj，id>对添加到全局哈希表 

- klp_shadow_free（）-分离并释放<obj，id>阴影变量-查找并从全局哈希表中删除<obj，id>引用 

  - 如果发现
    - 调用析构函数（如果已定义） 
    - 释放阴影变量 

- klp_shadow_free_all（）-分离并释放所有<，id>阴影变量-查找并从全局哈希表中删除任何<，id>引用 

  - 如果发现
    - 调用析构函数（如果已定义） 
    - 释放阴影变量

用例 
-------

（有关完整的工作演示，请参阅sample / livepatch /中的示例阴影变量livepatch模块。） 

对于以下用例示例，请考虑提交1d147bfa6429（“ mac80211：修复AP省电TX与唤醒竞争”），该示例向net / mac80211 / sta_info.h :: struct sta_info添加了自旋锁。 每个用例示例都可以视为此修补程序的独立livepatch实现。 

匹配父母的生命周期 
""""""""""""""""

如果经常创建和销毁父数据结构，则最简单的方法是将其影子变量的生存期与相同的分配和释放函数对齐。 在这种情况下，通常以某种方式分配，初始化和注册父数据结构。 然后可以将影子变量分配和设置视为父级初始化的一部分，并且应在父级“上线”之前完成（即，为此<obj，id>对发出任何影子变量get-API请求）。 

对于提交1d147bfa6429，在分配父sta_info结构时，分配ps_lock指针的影子副本，然后对其进行初始化： 

```
#define PS_LOCK 1struct sta_info *sta_info_alloc(struct ieee80211_sub_if_data *sdata,                                const u8 *addr, gfp_t gfp){      struct sta_info *sta;      spinlock_t *ps_lock;      /* Parent structure is created */      sta = kzalloc(sizeof(*sta) + hw->sta_data_size, gfp);      /* Attach a corresponding shadow variable, then initialize it */      ps_lock = klp_shadow_alloc(sta, PS_LOCK, sizeof(*ps_lock), gfp,                                 NULL, NULL);      if (!ps_lock)              goto shadow_fail;      spin_lock_init(ps_lock);      ...
```

当需要ps_lock时，查询影子变量API以检索特定结构sta_info的API ：： 

```
void ieee80211_sta_ps_deliver_wakeup(struct sta_info *sta){      spinlock_t *ps_lock;      /* sync with ieee80211_tx_h_unicast_ps_buf */      ps_lock = klp_shadow_get(sta, PS_LOCK);      if (ps_lock)              spin_lock(ps_lock);      ...
```

当父sta_info结构被释放时，首先释放shadow变量： 

```
void sta_info_free(struct ieee80211_local *local, struct sta_info *sta){      klp_shadow_free(sta, PS_LOCK, NULL);      kfree(sta);      ...
```

飞行中的父对象 

有时，将阴影变量与它们的父对象一起分配可能不方便或不可能。 或者，实时补丁修复可能只需要对父对象实例的子集使用阴影变量。 在这些情况下，可以使用klp_shadow_get_or_alloc（）调用将阴影变量附加到已在运行的父级中。

 对于提交1d147bfa6429，在ieee80211_sta_ps_deliver_wakeup（）内部分配阴影自旋锁的好地方： 

```
int ps_lock_shadow_ctor(void *obj, void *shadow_data, void *ctor_data){      spinlock_t *lock = shadow_data;      spin_lock_init(lock);      return 0;}#define PS_LOCK 1void ieee80211_sta_ps_deliver_wakeup(struct sta_info *sta){      spinlock_t *ps_lock;      /* sync with ieee80211_tx_h_unicast_ps_buf */      ps_lock = klp_shadow_get_or_alloc(sta, PS_LOCK,                      sizeof(*ps_lock), GFP_ATOMIC,                      ps_lock_shadow_ctor, NULL);      if (ps_lock)              spin_lock(ps_lock);      ...
```

此用法仅在需要时才会创建阴影变量，否则将使用已为此<obj，id>对创建的变量。 

像以前的用例一样，阴影自旋锁需要清理。 可以在释放其父对象之前释放影子变量，甚至在不再需要影子变量本身时也可以释放影子变量。 

**其他用例** 

影子变量也可以用作标志，指示数据结构是由新的实时修补代码分配的。 在这种情况下，shadow变量拥有的数据值无关紧要，它的存在暗示了如何处理父对象。

3.参考 

- https://github.com/dynup/kpatch

  livepatch实现基于影子变量的kpatch版本。 

- http://files.mkgnu.net/files/dynamos/doc/papers/dynamos_eurosys_07.pdf

  商品操作系统内核中非静态子系统的动态和自适应更新（Kritis Makris，Kyung Dong Ryu 2007）提出了一种称为“影子数据结构”的数据类型更新技术。 

  --------------

# 系统状态变更 

有些用户确实不愿意重新启动系统。 这带来了提供更多实时补丁并保持它们之间某种兼容性的需求。 

使用累积的实时补丁，维护更多的实时补丁要容易得多。 每个新的Livepatch都会完全替换任何较旧的Livepatch。 它可以保留，添加甚至删除修复程序。 由于原子替换功能，通常可以安全地将任何版本的livepatch替换为其他任何版本。 

问题可能来自影子变量和回调。 他们可能会更改系统行为或状态，以使返回过去使用较旧的livepatch或原始内核代码不再安全。 同样，任何新的实时修补程序都必须能够检测到已安装的实时修补程序已经进行了哪些更改。 

这是livepatch系统状态跟踪变得有用的地方。 它允许： 

- 存储操作和恢复系统状态所需的数据 
- 使用更改ID和版本定义实时补丁之间的兼容性 

1. # Livepatch系统状态API 

系统状态可以通过几个livepatch回调或新使用的代码进行修改。 此外，还必须能够找到已安装的Livepatches完成的更改。 每个修改后的状态由struct klp_state描述，请参见include / linux / livepatch.h。 每个livepatch定义了一个struct klp_states数组。 他们提到了Livepatch修改的所有状态。livepatch作者必须为每个struct klp_state定义以下两个字段： 

- id
  - 非零数字，用于标识受影响的系统状态。 
- 版本
  - 描述给定livepatch支持的系统状态更改变体的数字。 

可以使用两个功能来操纵状态： 

- *klp_get_state(patch, id)*
  - 获取与给定livepatch和状态ID关联的struct klp_state。 
- *klp_get_prev_state(id)*
  - 获取与给定功能ID和已安装的实时补丁关联的struct klp_state。

# 2. Livepatch兼容性 

系统状态版本用于防止加载不兼容的实时补丁。 启用livepatch时，将完成检查。 规则是： 

- 允许进行任何全新的系统状态修改。 
- 对于已修改的系统状态，允许使用相同或更高版本的系统状态修改。 
- 累积实时补丁必须处理已安装实时补丁中的所有系统状态修改。 
- 允许非累积性实时补丁接触已修改的系统状态。 

# 3.支持的方案 

Livepatches有其生命周期，并且对于系统状态更改也是如此。 每个兼容的livepatch必须支持以下方案： 

- 启用livepatch且尚未由要替换的livepatches修改状态后，请修改系统状态。 
- 当正在被替换的livepatch完成后，接管或更新系统状态修改。 
- 禁用Livepatch时，恢复原始状态。 
- 恢复转换后，恢复以前的状态。 可能是原始系统状态，也可能是由实时补丁所替换的状态修改。 
- 发生错误且无法启用livepatch时，请删除所有已进行的更改。 

# 4.预期用途 

系统状态通常通过livepatch回调进行修改。 每个回调的预期作用如下： 

*pre_patch()*

- 必要时分配状态->数据。 分配可能会失败，并且pre_patch（）是唯一可以停止加载livepatch的回调。 如果以前安装的实时修补程序已经提供了数据，则不需要分配。 
- 即使在转换完成之前，也要执行新代码所需的任何其他准备操作。 例如，初始化state-> data。 当整个系统都能够处理时，通常在post_patch（）中修改系统状态本身。 
- 清理自己的混乱，以防出现错误。 这可以通过自定义代码或显式调用post_unpatch（）来完成。 

*post_patch()*

- 当它们兼容时，从先前的实时补丁中复制state-> data。
- 进行实际的系统状态修改。 最终允许新代码使用它。
- 确保state-> data具有所有必要的信息。
- 当不再需要实时补丁时，释放状态->数据可代替实时补丁。 

*pre_unpatch()*

- 防止由livepatch添加的代码依赖于系统状态更改。
- 恢复系统状态修改

*post_unpatch()*

- 通过检查* klp_get_prev_state（）*来区分反向转换和livepatch禁用。 
- 如果转换反向，请恢复先前的系统状态。 这可能意味着什么都不做。
- 删除所有不再需要的设置或数据。 

-----------------

# 可靠的Stacktrace 

本文档概述了有关可靠堆栈跟踪的基本信息。 

1.简介 

内核livepatch一致性模型依赖于准确识别哪些功能可能处于活动状态，因此可能不安全进行修补。 标识哪些函数处于活动状态的一种方法是使用堆栈跟踪。

现有的stacktrace代码可能无法始终始终准确地显示具有实时状态的所有功能，而对于调试而言，尽力而为的方法是不可行的。 Livepatching依赖于体系结构来提供可靠的堆栈跟踪，从而确保它永远不会忽略跟踪中的任何实时功能。 

2.要求 

架构必须实现可靠的堆栈跟踪功能之一。 使用CONFIG_ARCH_STACKWALK的体系结构必须实现“ arch_stack_walk_reliable”，而其他体系结构必须实现“ save_stack_trace_tsk_reliable”。 

原则上，可靠的stacktrace函数必须确保： 

- 跟踪包括任务可以返回到的所有功能，并且返回码为零以表明跟踪是可靠的。 
- 返回码非零，表示跟踪不可靠。

注意：在某些情况下，从跟踪中省略特定功能是合法的，但是必须报告所有其他功能。 这些情况将在下面更详细地描述。 

其次，可靠的堆栈跟踪功能必须对堆栈或其他展开状态已损坏或不可靠的情况具有鲁棒性。 该函数应尝试检测此类情况并返回非零错误代码，并且不应陷入无限循环或以不安全的方式访问内存。 具体情况将在下面进一步详细描述。 

3.编译时分析 

为了确保可以在所有情况下正确解开内核代码，体系结构可能需要验证解开器所期望的方式是否已编译了代码。 例如，展开器可能期望函数以有限的方式操纵堆栈指针，或者所有函数都使用特定的序言和结尾序列。 有这种要求的体系结构应使用objtool验证内核编译。 

在某些情况下，展开器可能需要元数据才能正确地展开。 必要时，应在构建时使用objtool生成此元数据。 

4.注意事项 

展开过程因体系结构，各自的过程调用标准和内核配置而异。 本节描述了体系结构应考虑的常见细节。 

4.1确定成功终止 

放卷可能会由于多种原因而提前终止，其中包括： 

- 堆栈或框架指针损坏。 
- 缺少针对罕见情况的解开支持，或解开器中的错误。 
- 动态生成的代码（例如eBPF）或外来代码（例如EFI运行时服务）不遵循展开器期望的约定。 

为了确保即使不会被其他检查捕获，也不会导致功能从跟踪中遗漏，强烈建议架构验证堆栈跟踪在预期的位置结束，例如 

- 在特定功能内，它是内核的入口点。 
- 在堆栈上特定于内核入口点的特定位置。 
- 在预期用于内核入口点的特定堆栈上（例如，如果架构具有单独的任务和IRQ堆栈）。 

## 4.2识别不可撤销的代码 

展开通常依赖于遵循特定约定的代码（例如，操作帧指针），但是可能存在一些代码可能不遵循这些约定，并且可能需要在展开器中进行特殊处理，例如 ：

- 异常向量和条目汇编。 
- 过程链接表（PLT）条目和饰面板功能。 
- 蹦床装配（例如ftrace，kprobes）。 
- 动态生成的代码（例如eBPF，optprobe蹦床）。 
- 外来代码（例如EFI运行时服务）。 

为确保此类情况不会导致函数从跟踪中遗漏，强烈建议体系结构肯定地标识已知可靠的代码以其展开，并拒绝从所有其他代码展开。 

可以使用“ __kernel_text_address（）”将包含模块和eBPF的内核代码与外来代码区分开。 检查是否也有助于检测堆栈损坏。 

架构可以通过几种方式来识别内核代码，这些代码被认为是不可靠的，例如可以从中解脱出来。 

- 将此类代码放入特殊的链接器部分，并拒绝取消这些部分中任何代码的展开。 
- 使用边界信息识别代码的特定部分。 

## 4.3跨中断和异常展开 

在函数调用边界处，堆栈和其他展开状态应处于适用于可靠展开的一致状态，但是在函数途中可能不是这种情况。 例如，在函数序言或结尾过程中，帧指针可能暂时无效，或者在函数体中，返回地址可以保留在任意通用寄存器中。 对于某些架构，这可能会由于动态检测而在运行时发生变化。 

如果在堆栈或其他展开状态处于不一致状态时发生中断或其他异常，则可能无法可靠地展开，并且可能无法确定这种展开是否可靠。 请参阅下面的示例。 

无法确定何时可以放心释放此类情况的体系结构（或永不可靠的位置）必须拒绝跨异常边界的放宽。 请注意，取消某些例外情况（例如IRQ）可能是可靠的，但跨越其他例外情况（例如NMI）则不可靠。 

能够确定何时可以放宽此类情况（或没有此类情况）的体系结构应尝试跨异常边界进行展开，因为这样做可以防止不必要地停止livepatch一致性检查并允许livepatch转换更快地完成。 

 4.4改写返回地址 

一些蹦床会临时修改函数的返回地址，以便在该函数以返回蹦床返回时进行拦截，例如 

- ftrace蹦床可以修改返回地址，以便函数图跟踪可以拦截返回。 
- kprobes（或optprobes）蹦床可以修改返回地址，以便kretprobes可以拦截返回。 

发生这种情况时，原始的寄信人地址将不在其通常的位置。 对于无需现场修补的蹦床，其中放卷机可以可靠地确定原始寄信人地址，并且蹦床不会改变放卷状态，则放卷机可以报告原始的寄信人地址代替蹦床，并报告为可靠。 否则，放卷机必须将这些情况报告为不可靠。 

标识原始的寄信人地址时，需要格外小心，因为在进入蹦床或返回蹦床期间，此信息的位置不一致。 例如，考虑使用x86_64“ return_to_handler”返回蹦床： 

```
SYM_CODE_START(return_to_handler)        UNWIND_HINT_EMPTY        subq  $24, %rsp        /* Save the return values */        movq %rax, (%rsp)        movq %rdx, 8(%rsp)        movq %rbp, %rdi        call ftrace_return_to_handler        movq %rax, %rdi        movq 8(%rsp), %rdx        movq (%rsp), %rax        addq $24, %rsp        JMP_NOSPEC rdiSYM_CODE_END(return_to_handler)
```

当被跟踪的函数在堆栈上运行其返回地址时，将指向return_to_handler的开始，而原始返回地址将存储在任务的cur_ret_stack中。 在这段时间内，展开器可以使用ftrace_graph_ret_addr（）查找返回地址。 

当被跟踪的函数返回return_to_handler时，堆栈上不再有返回地址，尽管原始返回地址仍存储在任务的cur_ret_stack中。 在ftrace_return_to_handler（）中，原始返回地址已从cur_ret_stack中删除，并在被rax返回之前，被编译器临时任意移动。 return_to_handler蹦床在跳到它之前将其移动到rdi中。 

架构可能并不总是能够解开此类序列，例如，当ftrace_return_to_handler（）从cur_ret_stack中删除了地址，并且无法可靠地确定返回地址的位置时。 

建议体系结构平移尚未返回return_to_handler的情况，但是不要求体系结构从return_to_handler的中间展开，并且可以将其报告为不可靠的。 不需要架构从其他修改返回地址的蹦床中解脱出来。 

 4.5遮盖住返回地址 

一些蹦床不重写返回地址以拦截返回，而是暂时破坏返回地址或其他展开状态。 

例如，optprobes的x86_64实现使用针对相关optprobe蹦床的JMP指令修补所探测的功能。 命中探针后，CPU将跳转到optprobe蹦床，并且所探针功能的地址未保存在任何寄存器中或堆栈中。 

类似地，DYNAMIC_FTRACE_WITH_REGS补丁的arm64实现使用以下命令跟踪功能： 

```
MOV X9, X30BL <trampoline>
```

MOV将链接寄存器（X30）保存到X9中，以在BL阻塞链接寄存器并跳转到蹦床之前保留返回地址。 在蹦床开始时，被跟踪函数的地址在X9中，而不是通常情况下的链接寄存器中。 

架构必须确保展开器可靠地消除这种情况，或者将展开报告为不可靠。 

## 4.6链接寄存器不可靠 

在其他一些体系结构上，“调用”指令将返回地址放入链接寄存器，而“返回”指令将使用链接寄存器中的返回地址，而无需修改寄存器。 在这些体系结构上，软件必须在进行函数调用之前将返回地址保存到堆栈中。 在函数调用期间，返回地址可以单独保存在链接寄存器中，也可以单独保存在堆栈中，也可以保存在两个位置。 

展开器通常假设链接寄存器始终处于活动状态，但是此假设可能导致不可靠的堆栈跟踪。 例如，考虑使用以下arm64组件来实现简单功能：

```
function:        STP X29, X30, [SP, -16]!        MOV X29, SP        BL <other_function>        LDP X29, X30, [SP], #16        RET
```

在进入该功能时，链接寄存器（x30）指向调用者，而帧指针（X29）指向包含调用者返回地址的调用者帧。 前两条指令创建一个新的堆栈帧并更新帧指针，此时链接寄存器和帧指针都描述了该函数的返回地址。 此时的跟踪可能会两次描述此函数，并且如果正在跟踪函数返回，则展开器可能会消耗fgraph返回堆栈中的两个条目，而不是一个条目。 

BL调用“ other_function”，链接寄存器指向该函数的LDR，帧指针指向该函数的stackframe。 返回“ other_function”时，链接寄存器指向BL，因此此时的跟踪可能会导致“ function”在回溯中出现两次。 

类似地，一个功能可以故意破坏LR，例如。

```
caller:        STP X29, X30, [SP, -16]!        MOV X29, SP        ADR LR, <callee>        BLR LR        LDP X29, X30, [SP], #16        RET
```

在BLR分支到该地址之前，ADR会将“被叫方”的地址放入LR。 如果在ADR之后立即进行跟踪，则“被叫方”将是“主叫方”的父级，而不是子级。 

由于上述情况，可能只能可靠地消耗函数调用边界处的链接寄存器值。 在这种情况下，除非能够可靠地标识应何时使用LR或堆栈值（例如，使用objtool生成的元数据），否则架构必须拒绝跨异常边界展开。 

------------------------

# livepatch内核模块-从编译到加载原理分析

## livepatch子系统分析

### 内核模块代码：kernel/liepatch/

对的，这是一个内核模块，不是一个内核子系统。

cat Makefile

```ruby

```

cat Kbuild

```ruby

```



## livepatch内核模块编译

k_m.c:

```

```

编译后文件分析：

k_m.mod.c

```

```

分析

readelf -a k_m.ko

```

```

分析：

## livpatch 内核模块加载 


















