中断下半部
^^^^^^^^^^^
其实就是软中断。
如:
_local_bh_disable(void);
_local_bh_enable(void);

所以那个驱动的实现有问题，应该是现在软中断里。
 首先明确一个概念软中断（不是软件中断int n）。总来来说软中断就是内核在启动时为每一个内核创建了一个特殊的进程，这个进程会不停的poll检查是否有软中断需要执行，如果需要执行则调用注册的接口函数。所以软中断是运行在进程上下文的，而且可能并发执行在不同CPU上。所谓的软中断就是内核利用内核线程配合抽象的数据结构进行管理线程合适时间调用注册的接口的一套软件管理机制。

     先看管理软中断的数据结构因为数据结构最能说明逻辑内核对软件中断抽象的数据结构主要有如下几个部分。    HI_SOFTIRQ=0,
    TIMER_SOFTIRQ,
    NET_TX_SOFTIRQ,
    NET_RX_SOFTIRQ,
    BLOCK_SOFTIRQ,
    BLOCK_IOPOLL_SOFTIRQ,
    TASKLET_SOFTIRQ,
    SCHED_SOFTIRQ,
    HRTIMER_SOFTIRQ,
    RCU_SOFTIRQ,    /* Preferable RCU should always be the last softirq */

    NR_SOFTIRQS
    
    之所以综上可以知道内核维护了一个struct softirq_action类型的软中断接口数组，而软中断的状态则是由前面的 irq_cpustat_t 类型的数组管理，由定义可以知道状态是和CPU关联的，表示某一个CPU上的软中断状态。下面看看irq_cpustat_t 的定义，也非常的的简单主要就是 其中的 __softirq_pending成员，这个成员的每一个bit表示一种类型的中断类型的状态信息，并且低bit的中断类型的中断优先级高。
    
    
    在通过Tasklet接接口中断的创建就可以知道软件中断的注册(open_softirq)过程就是修改前面定义的softirq_vec数组，就可以完成软件中断的注册,而驱动开发人员也很少直接使用软件中断。
    注意软中断的线程化。
    
    
    enum
{
    HI_SOFTIRQ=0,//最高优先级的软中断类型
    TIMER_SOFTIRQ,//Timer定时器软中断
    NET_TX_SOFTIRQ,//发送网络数据包软中断
    NET_RX_SOFTIRQ,//接收网络数据包软中断
    BLOCK_SOFTIRQ,
    BLOCK_IOPOLL_SOFTIRQ,//块设备软中断
    TASKLET_SOFTIRQ,//专门为tasklet机制准备的软中断
    SCHED_SOFTIRQ,//进程调度以及负载均衡软中断
    HRTIMER_SOFTIRQ,//高精度定时器软中断
    RCU_SOFTIRQ,    /* Preferable RCU should always be the last softirq */----RCU服务软中断

    NR_SOFTIRQS
};



tasklet描述：


struct tasklet_struct
{
    //多个tasklet串成一个链表。
    struct tasklet_struct *next;
     /*
     TASKLET_STATE_SCHED表示tasklet已经被调度，正准备运行； 
     TASKLET_STATE_RUN表示tasklet正在运行中。
    */
    unsigned long state;
    //0表示tasklet处于激活状态；非0表示该tasklet被禁止，不允许执行。
    atomic_t count;
   //该tasklet处理接口
    void (*func)(unsigned long);
   //传递给tasklet处理函数的参数
    unsigned long data;
};


内核定时器执行：



-------------























































