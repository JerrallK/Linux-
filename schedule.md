

# 进程调度

[TOC]

## 1. 进程调度

### 1.1调度算法发展历史

-----------------------

| 调度算法名称                                   | 存在版本            | 主要优点                                     | 主要缺点                                     |
| :--------------------------------------- | --------------- | ---------------------------------------- | ---------------------------------------- |
| **O(n)**  调度算法                           | Linux-0.11~2.4  | 算法简单                                     | 1. 会随着程序数量增加导致每次计算时间增加，导致整体性能下降。2.交互式进程优化不完善。3.内核不支持抢占。 |
| **O(1)**  调度算法                           | linux-2.5~2.6初期 | 1. 时间花费少。2. 扩展性好。3. 支持内核抢占，更好地支持了实时进程。   | 1. 交互式进程优化不完善，导致交互式进程反应慢。2. 动态优先级调整策略复杂。3. 基于平均睡眠时间和经验公式来区分是否为交互进程。 |
| **SD** 楼梯调度算法（Staircase Deadline Scheduler)） | Linux-2.6早期     | 1. 开始采取完全公平的思路。2. 极大简化了算法代码。3. 响应比O(1)算法更好。4. 可以避免饥饿现象。 | 低优先级进程的等待时间无法确定                          |
| **RSDL** 反转楼梯调度算法 (Rotating Staircase Deadline Scheduler) | Linux-2.6早期     | 进一步改善了SD调度算法的公平性                         |                                          |
| **CFS**  完全公平调度算法(Completely Fair Scheduler) | Linux-2.6.23至今  | 1.基于完全公平的思想，不再区分交互式进程，所有进程统一对待。2.性能优越。   | 算法复杂度为 O(log n)，比之前的O(1)、SD、RSDL算法高      |

### 1.2进程种类

--------------------------------

| 分类       | 类型                            | 描述                                                         | 示例                                                         |
| :--------- | ------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 非实时进程 | 交互式进程(interactive process) | 此类进程经常与用户进行交互, 因此需要花费很多时间等待键盘和鼠标操作. 当接受了用户的输入后, 进程必须很快被唤醒, 否则用户会感觉系统反应迟钝 | shell, 文本编辑程序和图形应用程序                            |
| 非实时进程 | 批处理进程(batch process)       | 此类进程不必与用户交互, 因此经常在后台运行. 因为这样的进程不必很快响应, 因此常受到调度程序的怠慢 | 程序语言的编译程序, 数据库搜索引擎以及科学计算               |
| 实时进程   | 实时进程(real-time process)     | 这些进程由很强的调度需要, 这样的进程绝不会被低优先级的进程阻塞. 并且他们的响应时间要尽可能的短 | 视频音频应用程序, 机器人控制程序以及从物理传感器上收集数据的程序 |

### 1.3 进程优先级

------------------

#### 1.3.1优先级的内核表示

| 进程描述符内与优先级相关的字段<sched.h> | 描述                                                         |
| :-------------------------------------- | :----------------------------------------------------------- |
| int                **prio**;            | 动态优先级                                                   |
| int             **static_prio**;        | 静态优先级，进程启动时分配的优先级，可以通过nice和sched_setscheduler系统调用来进行修改, 否则在进程运行期间会一直保持恒定 |
| int             **normal_prio**;        | 普通优先级，表示基于进程的静态优先级static_prio和调度策略计算出的优先级. 因此即使普通进程和实时进程具有相同的静态优先级, 其普通优先级也是不同的, 进程分叉(fork)时, 子进程会继承父进程的普通优先级 |
| unsigned int    **rt_priority**;        | 实时优先级，实时进程的优先级，用户可以通过sched_setparam()和sched_setscheduler()来改变进程的实时优先级。rt_priority缺省值为0，表示非实时任务。[1，99]表示实时任务 |

​	优先级先关数值的定义。

```c
/* include/linux/sched/prio.h */
#define MAX_NICE    19                                        /*最大nice值*/
#define MIN_NICE    -20                                       /*最小nice值*/   
#define NICE_WIDTH  (MAX_NICE - MIN_NICE + 1)                 /*nice值范围：40*/

#define MAX_USER_RT_PRIO    100
#define MAX_RT_PRIO     MAX_USER_RT_PRIO                     /*实时进程最大优先级*/
#define MAX_PRIO        (MAX_RT_PRIO + NICE_WIDTH)           /*普通进程最大优先级*/
#define DEFAULT_PRIO        (MAX_RT_PRIO + NICE_WIDTH / 2)   /*用于nice值和优先级的转换的中间量*/

#define NICE_TO_PRIO(nice)	((nice) + DEFAULT_PRIO)          /*nice值转换到优先级值*/
#define PRIO_TO_NICE(prio)	((prio) - DEFAULT_PRIO)          /*优先级值转换到nice值*/

#define USER_PRIO(p)        ((p)-MAX_RT_PRIO)
#define TASK_USER_PRIO(p)   USER_PRIO((p)->static_prio)
#define MAX_USER_PRIO       (USER_PRIO(MAX_PRIO))
```

​    在用户空间用户可以使用nice命令设置进程的优先级，nice值在-20到+19之间，数值越小，优先级越高。	

​    内核使用数值范围0~139表示内部优先级, 数值越低, 优先级越高。从1~99的范围专供实时进程使用, nice的值[-20,19]则映射到范围100~139![Ashampoo_Snap_2018年2月4日_19h20m23s_001_](F:\screenshoot\Ashampoo_Snap_2018年2月4日_19h20m23s_001_.png)

​	普通进程的静态优先级和动态优先级数值范围为[100,139]，实时进程的实时优先级范围为[0，99]。

> 普通进程的动态优先级=max(100 , min(静态优先级 – bonus + 5) , 139)  

​	bonus的范围为[0,10]

#### 1.3.2 计算优先级

​	计算优先级只需一行代码：`p->prio = effective_prio(p);`effective_prio不仅计算了动态优先级，同时在其内部用normal_prio计算了普通优先级。

```c
/* kernel/sched/core.c*/
static int effective_prio(struct task_struct *p)
{
	p->normal_prio = normal_prio(p);
	/*
	 * 如果是实时进程或已经提高到实时优先级，则保持动态优先级不变。否则，返回普通优先级
	 */
	if (!rt_prio(p->prio))/*rt_prio用于判断当前进程是否为实时进程,是的话rt_prio返回1*/
		return p->normal_prio;/*不是实时进程时返回普通优先级*/
	return p->prio;
}
```

​	根据以上计算可知，若当前进程是普通进程时，动态、静态、普通优先级都为同一个值：静态优先级。

​	effective_prio调用的normal_prio函数具体如下：

```c
/* kernel/sched/core.c*/
static inline int normal_prio(struct task_struct *p)
{
	int prio;

	if (task_has_dl_policy(p))/*判断进程调度策略是否是idle*/
		prio = MAX_DL_PRIO-1;/*MAX_DL_PRIO = 0*/
	else if (task_has_rt_policy(p))/*判断进程调度策略是否是rt进程*/
		prio = MAX_RT_PRIO-1 - p->rt_priority;/*MAX_RT_PRIO = 100*/
	else
		prio = __normal_prio(p);/* __normal_prio返回静态优先级*/
	return prio;

```

​	此处存在一个问题，就是为什么在**effective_prio**里调用**normal_prio**时已经用`task_has_rt_policy(p)`判断了是否为实时进程，为什么还要在调用**normal_prio**后再调用`rt_prio(p->prio)`判断是否为实时进程。因为`rt_prio(p->prio)`是基于优先级数值判断，`task_has_rt_policy(p)`是基于进程的调度策略判断的，对于临时提高优先级至实时优先级的非实时进程来说是很有必要的。

综上可以得到不同类型进程的优先级计算结果：

| 进程类型                           | 静态优先级  |           普通优先级           | 动态优先级  |
| :--------------------------------- | :---------: | :----------------------------: | :---------: |
| 普通进程                           | static_prio |          static_prio           | static_prio |
| 实时进程                           | static_prio | MAX_RT_PRIO-1 - p->rt_priority |    不变     |
| 优先级提高至实时优先级的非实时进程 | static_prio |          static_prio           |    不变     |
| SCHED_DEADLINE调度策略的进程       | static_prio |         MAX_DL_PRIO-1          |    不变     |

​	**在进程分支出子进程时，子进程的静态优先级继承自父进程。子进程的动态优先级，即task_struct->prio，则设置为父进程的普通优先级。**



### 1.4 调度器简介

------------------------------------

​	调度器的存在就是为了让进程尽可能公平地共享CPU时间，创造并行执行的效果。所以调度器的主要任务是：**1. 根据调度策略对进程进行调度。2. 调度后进行上下文切换。**

​	目前Linux有两种调度器：一种是进程由于打算睡眠或者其他原因放弃CPU而直接激活的调度器，叫做**主调度器**；另一种就是通过周期性机制，以固定频率运行检测是否有程序需要进行进程切换，定时更新调度相关的统计信息，但不负责进程切换，叫做**周期性调度器**。两者统称为：**通用调度器**（generic scheduler）或**核心调度器**（core scheduler）。

![Ashampoo_Snap_2018年2月8日_15h00m34s_005_](F:\screenshoot\Ashampoo_Snap_2018年2月8日_15h00m34s_005_.png)

​	进程描述符中与调度相关的成员：

```c
<sched.h>
struct task_struct {
...
int prio, static_prio, normal_prio;/*动态优先级，静态优先级，普通优先级*/
unsigned int rt_priority;/*实时优先级*/
struct list_head run_list; /*用于实时进程调度的表头*/
const struct sched_class *sched_class;/*调度类*/
struct sched_entity se;/*组调度实体*/
unsigned int policy;/*调度策略*/
cpumask_t cpus_allowed;/*一个位域，在多处理器上使用，用来限制进程可以在哪些CPU上运行*/
unsigned int time_slice;/*指定实时进程可使用的CPU剩余时间段*/
...
}
```



#### 1.4.1 主调度器

​	主调度器主要负责进程的切换。内核需要知道在什么时候调用schedule，所以内核给进程描述符提供了一个**TIF_NEED_RESCHED**标志来表明需要再次执行schedule，当某个进程应当被抢占时，schedule_tick就会设置这个标志，当一个优先级高的进程进入可执行状态时，try_to_wake_up也会设置这个标志。在从系统调用返回之后，内核会检查当前进程是否设置了重调度标志TIF_NEED_RESCHED，如果设置了，内核就会调用schedule。

> 该函数主要完成如下工作：
>
> - 确定当前就绪队列, 并在保存一个指向当前(仍然)活动进程的task_struct指针
> - 检查死锁, 关闭内核抢占后调用__schedule完成内核调度
> - 恢复内核抢占, 然后检查当前进程是否设置了重调度标志TLF_NEDD_RESCHED, 如果该进程被其他进程设置了TIF_NEED_RESCHED标志, 则函数重新执行进行调度

**内核抢占：**

​	内核抢占就是指一个在内核态运行的进程, 可能在执行内核函数期间被另一个进程取代。不支持内核抢占的内核中，内核代码可以一直执行下去，直到完成。

​	内核抢占的条件：进程没有持锁（锁是非抢占区域的标志）。为了支持内核抢占，Linux内核在进程的thread_info引入了`preempt_count`称为**抢占计数器**。该计数器初始值为0，每当使用锁的时候+1，释放锁的时候-1。要进程设置了TIF_NEED_RESCHED标志并且抢占计数器为0时才可以发生进程抢占。

![Ashampoo_Snap_2018年2月8日_11h53m34s_004_](F:\screenshoot\Ashampoo_Snap_2018年2月8日_11h53m34s_004_.png)

​	内核显式抢占：1. 内核进程发生阻塞；2. 显式调用schedule（）；

综上可知内核抢占发生的时机为：

- 从中断返回内核空间

- 内核重新启用抢占

- 内核进程发生阻塞

- 显式调用schedule（）

  ​

**用户抢占：**

​	当内核即将返回用户空间或者从中断返回用户空间, 内核会检查TIF_NEED_RESCHED是否设置, 如果设置, 则调用schedule()，此时，发生用户抢占.

用户抢占发生的时机为：

- 从系统调用返回用户空间
- 从中断返回用户空间

**schedule函数具体实现：**

```c
/*kernel/sched/core.c*/
asmlinkage __visible void __sched schedule(void)
{
	struct task_struct *tsk = current;             /* 获取当前的进程  */

	sched_submit_work(tsk);                        /* 避免死锁 */
	do {
		preempt_disable();                         //关闭内核抢占，防止发生内核抢占使调度被中断
		__schedule(false);                         /* 完成调度  */
		sched_preempt_enable_no_resched();         /* 开启内核抢占  */
	} while (need_resched());/*  如果该进程被其他进程设置了TIF_NEED_RESCHED标志，则函数重新执行进行调度    */
}
```

具体的调度是在`__schedule`函数中完成的

1. 完成一些必要的检查, 并设置进程状态, 处理进程所在的就绪队列
2. 调度全局的**pick_next_task**选择抢占的进程

   - 如果当前cpu上所有的进程都是cfs调度的普通非实时进程, 则直接用cfs调度, 如果无程序可调度则调度idle进程
   - 否则从优先级最高的调度器类sched_class_highest(目前是stop_sched_class)开始依次遍历所有调度器类的pick_next_task函数, 选择最优的那个进程执行
3. **context_switch**完成进程上下文切换

   - 调用switch_mm(), 把虚拟内存从一个进程映射切换到新进程中
   - 调用switch_to(),从上一个进程的处理器状态切换到新进程的处理器状态。这包括保存、恢复栈信息和寄存器信息




####1.4.2 周期性调度器 

​	周期性调度器在scheduler_tick中实现。scheduler_tick（）作用：**更新CPU和当前进行的一些数据，然后根据当前进程的调度类，调用task_tick()函数**。如果系统正在活动中，内核会按照频率HZ自动调用该
函数。如果没有进程在等待调度，那么在计算机电力供应不足的情况下，也可以关闭该调度器以减少
电能消耗。

​	scheduler_tick函数有下面两个主要任务：
​	(1) 管理内核中与整个系统和各个进程的调度相关的统计量。其间执行的主要操作是对各种计数
器加1。

| 函数                     | 描述                                       |
| ---------------------- | ---------------------------------------- |
| update_rq_clock        | 处理就绪队列时钟的更新, 本质上就是增加struct rq当前实例的时钟时间戳  |
| update_cpu_load_active | 负责更新就绪队列的cpu_load数组, 其本质上相当于将数组中先前存储的负荷值向后移动一个位置, 将当前就绪队列的符合记入数组的第一个位置. 另外该函数还引入一些取平均值的技巧, 以确保符合数组的内容不会呈现太多的不联系跳读. |
| calc_global_load_tick  | 跟新cpu的活动计数, 主要是更新全局cpu就绪队列的calc_load_update |

​	(2) 激活负责当前进程的调度类的周期性调度方法。由于调度器的模块化结构, 主体工程其实很简单, 在更新统计信息的同时, 内核将真正的调度工作委托给了特定的调度类方法，内核先找到了就绪队列上当前运行的进程curr, 然后调用curr所属调度类sched_class的周期性调度方法task_tick即

```
curr->sched_class->task_tick(rq, curr, 0);
```

task_tick的实现方法取决于底层的调度器类

```c
void scheduler_tick(void)
{
	int cpu = smp_processor_id();/*在于SMP的情况下，获得当前CPU的ID。如果不是SMP，那么就返回0*/
	struct rq *rq = cpu_rq(cpu);/*获取cpu的全局就绪队列rq, 每个CPU都有一个就绪队列rq */
	struct task_struct *curr = rq->curr;/*获取就绪队列上正在运行的进程curr*/

	sched_clock_tick();

	raw_spin_lock(&rq->lock);/*加锁*/
	update_rq_clock(rq);/*更新rq的当前时间戳.即使rq->clock变为当前时间戳*/
  
   /*执行当前运行进程所在调度类的task_tick函数进行周期性调度*/
	curr->sched_class->task_tick(rq, curr, 0);
  
    /*  更新rq的负载信息,  即就绪队列的cpu_load[]数据，本质是讲数组中先前存储的负荷值向后移动一个位      *置，将当前负荷记入数组的第一个位置 */
	update_cpu_load_active(rq);
	calc_global_load_tick(rq);/*更新cpu的active count活动计数*/
	raw_spin_unlock(&rq->lock);/*解锁*/

	perf_event_task_tick();/*与perf计数事件相关*/

#ifdef CONFIG_SMP
	rq->idle_balance = idle_cpu(cpu);/*判断当前CPU是否空闲*/
	trigger_load_balance(rq);/*如果是，触发负载均衡软中断SCHED_SOFTIRQ*/
#endif
	rq_last_tick_reset(rq);
}
```

####1.4.3 上下文切换

​	进程被抢占时, 操作系统保存其上下文信息, 同时将新的活动进程的上下文信息加载进来, 这个过程就是**上下文切换**。当这个进程再次成为活动的, 它可以恢复自己的上下文继续从被抢占的位置开始执行。主要由`_schedule`内的`context__switch`完成。

> 一个进程的[上下文](https://www.cnblogs.com/wanghuaijun/p/6985672.html)可以分为三个部分:**用户级上下文**、**寄存器上下文**以及**系统级上下文**。
>
> - 用户级上下文: 正文、数据、用户堆栈以及共享存储区；
> - 寄存器上下文: 通用寄存器、程序寄存器(IP)、处理器状态寄存器(EFLAGS)、栈指针(ESP)；
> - 系统级上下文: 进程控制块task_struct、内存管理信息(mm_struct、vm_area_struct、pgd、pte)、内核栈。



​	上下文切换只能发生在内核态中，上下文切换通常是计算密集型的，对系统来说意味着消耗大量的 CPU 时间，事实上，可能是操作系统中时间消耗最大的操作。进程调度调度时，内核选择新进程之后进行抢占，**context_switch**内通过以下两个处理函数完成：	

​	(1) **switch_mm** ：更换通过task_struct->mm描述的内存管理上下文。该工作的细节取决于处理器，主要包括加载页表、刷出地址转换后备缓冲器（部分或全部）、向内存管理单元（MMU）提供新的信息。	context_switch( )函数建立next的地址空间。进程描述符的**active_mm**字段指向进程所使用的内存描述符，而**mm**字段指向进程所拥有的内存描述符。对于一般的进程，这两个字段有相同的地址，但是，内核线程没有它自己的地址空间而且它的 mm字段总是被设置为NULL。在context_switch( )函数内：如果next是一个内核线程，它使用prev所使用的地址空间，如果next是一个普通进程，schedule( )函数用next的地址空间替换prev的地址空间。
​	(2) **switch_to** ：切换处理器寄存器内容和内核栈，从上一个进程的处理器状态切换到新进程的处理器状态。这包括保存、恢复栈信息和寄存器信息（虚拟地址空间的用户部分在第一步已经变更，其中也包括了用户状态下的栈，因此用户栈就不需要显式变更了）。**此项工作在不同的体系结构下可能差别很大，代码通常都使用汇编语言编写。**

### 1.5 调度策略

-----------------------

```c
/*include/uapi/linux/sched.h*/
/*
 * Scheduling policies
 */
#define SCHED_NORMAL		0
#define SCHED_FIFO		1
#define SCHED_RR		2
#define SCHED_BATCH		3
/* SCHED_ISO: reserved but not implemented yet */
#define SCHED_IDLE		5
#define SCHED_DEADLINE		6
```

| 字段           | 描述                                                         | 所在调度器类 |
| -------------- | ------------------------------------------------------------ | ------------ |
| SCHED_NORMAL   | （也叫SCHED_OTHER）用于普通进程，通过CFS调度器实现。         | CFS          |
| SCHED_BATCH    | SCHED_NORMAL普通进程策略的分化版本。采用分时策略，根据动态优先级(可用nice()API设置），分配CPU运算资源。针对吞吐量优化, 除了不能抢占外与常规任务一样，允许任务运行更长时间，更好地使用高速缓存，适合于成批处理的工作，用于处理非交互的处理器消耗型进程。 | CFS          |
| SCHED_IDLE     | 优先级最低，在系统空闲时才跑这类进程(如利用闲散计算机资源跑地外文明搜索，蛋白质结构分析等任务，是此调度策略的适用者） | CFS-IDLE     |
| SCHED_FIFO     | 先入先出调度算法（实时调度策略），相同优先级的任务先到先服务，高优先级的任务可以抢占低优先级的任务 | RT           |
| SCHED_RR       | 轮流调度算法（实时调度策略），Roound-Robin 采用时间片，相同优先级的任务当用完时间片会被放到队列尾部，以保证公平性，同样，高优先级的任务可以抢占低优先级的任务。不同要求的实时任务可以根据需要用sched_setscheduler() API设置策略 | RT           |
| SCHED_DEADLINE | 新支持的实时进程调度策略，针对突发型计算，且对延迟和完成时间高度敏感的任务适用。基于Earliest Deadline First (EDF) 调度算法 | D            |

### 1.6 调度类

​	(1) 调度类用于判断接下来运行哪个进程。内核支持不同的调度策略（完全公平调度、实时调度、在无事可做时调度空闲进程），调度类使得能够以模块化方法实现这些策略，即一个类的代码不需要与其他类的代码交互。在调度器被调用时，它会查询调度类，得知接下来运行哪个进程。
​	(2) 在选中将要运行的进程之后，必须执行底层任务切换。这需要与CPU的紧密交互。每个进程都有其调度类，各个调度类负责管理所属的进程。**通用调度器自身完全不涉及进程管理，其工作都委托给调度类。**

```c
struct sched_class {
	const struct sched_class *next;

	void (*enqueue_task) (struct rq *rq, struct task_struct *p, int flags);
	void (*dequeue_task) (struct rq *rq, struct task_struct *p, int flags);
	void (*yield_task) (struct rq *rq);
	bool (*yield_to_task) (struct rq *rq, struct task_struct *p, bool preempt);

	void (*check_preempt_curr) (struct rq *rq, struct task_struct *p, int flags);
	/*
	 *enqueue_task向就绪队列添加一个新进程。在进程从睡眠状态变为可运行状态时，即发生该操作。
	 *
     *dequeue_task提供逆向操作，将一个进程从就绪队列去除。比如使进程从可运行状态切
	 *换到不可运行状态时，就会发生该操作。
	 *
	 *yield_task在进程想要自愿放弃对处理器的控制权时，可使用sched_yield系统调用
	 *
	 *在必要的情况下，会调用check_preempt_curr，用一个新唤醒的进程来抢占当前进程
	 */
  
···
	struct task_struct * (*pick_next_task) (struct rq *rq,
						struct task_struct *prev,
						struct rq_flags *rf);
	void (*put_prev_task) (struct rq *rq, struct task_struct *p);

	void (*set_curr_task) (struct rq *rq);
	void (*task_tick) (struct rq *rq, struct task_struct *p, int queued);
	void (*task_fork) (struct task_struct *p);
	void (*task_dead) (struct task_struct *p);
  /*
   *pick_next_task用于选择下一个将要运行的进程，而put_prev_task则在用另一个进程代替
   *当前运行的进程之前调用。要注意，这些操作并不等价于将进程加入或撤出就绪队列的操作，
   *如enqueue_task和dequeue_task。相反，它们负责向进程提供或撤销CPU。但在不同进程之
   *间切换，仍然需要执行一个底层的上下文切换。
   *
   *task_tick在每次激活周期性调度器时，由周期性调度器调用。
   *
   *new_task用于建立fork系统调用和调度器之间的关联。每次新进程建立后，则用new_task通知调度器
   */
···
};
```

**共有5个调度类,都是sched_class的实例**

| 调度类           | 文件位置                  |
| ---------------- | ------------------------- |
| stop_sched_class | /kernel/sched/stop_task.c |
| dl_sched_class   | /kernel/sched/deadline.c  |
| rt_sched_class   | /kernel/sched/rt.c        |
| fair_sched_class | /kernel/sched/fair.c      |
| idle_sched_class | /kernel/sched/idle_task.c |



| **调度类**       | **调度策略**             | **调度策略对应的调度算法**                      | **调度实体**    | **调度实体对应的调度对象**                                   |
| ---------------- | ------------------------ | ----------------------------------------------- | --------------- | ------------------------------------------------------------ |
| stop_sched_class | 无                       | 无                                              | 无              | 特殊情况, 发生在cpu_stop_cpu_callback 进行cpu之间任务迁移migration或者HOTPLUG_CPU的情况下关闭任务 |
| dl_sched_class   | SCHED_DEADLINE           | Earliest-Deadline-First最早截至时间有限算法     | sched_dl_entity | 采用DEF最早截至时间有限算法调度实时进程                      |
| rt_sched_class   | SCHED_RR SCHED_FIFO      | Roound-Robin时间片轮转算法 。FIFO先进先出算法。 | sched_rt_entity | 采用Roound-Robin或者FIFO算法调度的实时调度实体               |
| fair_sched_class | SCHED_NORMAL SCHED_BATCH | CFS完全公平懂调度算法                           | sched_entity    | 采用CFS算法普通非实时进程                                    |
| idle_sched_class | SCHED_IDLE               | 无                                              | 无              | 特殊进程, 用于cpu空闲时调度空闲进程idle                      |

其所属进程优先级顺序为：

`stop_sched_class > dl_sched_class > rt_sched_class > fair_sched_class > idle_sched_class`



### 1.7 就绪队列与等待队列

#### 1.7.1 就绪队列

​	核心调度器用来管理活动进程的主要数据结构成为就绪队列。各个CPU都有自身的就绪队列，也就是 struct rq 结构，其用于描述在此CPU上所运行的所有进程，包括一个实时进程队列和一个根CFS运行队列，在调度时，调度器首先会先去实时进程队列找是否有实时进程需要运行，如果没有才会去CFS运行队列找是否有进行需要运行，这就是为什么常说的实时进程优先级比普通进程高，不仅仅体现在prio优先级上，还体现在调度器的设计上，各个活动进程只能出现在一个就绪队列中，就是说一个进程在多个CPU上同时运行是不可能的，但是发源于同一个进程的线程组中的各个线程可以在不同CPU上运行。

```c
/* kernel/sched/sched.h 中rq部分代码*/

struct rq {
···
	unsigned int nr_running;/*指定了队列上可运行进程的数目，不考虑其优先级或调度类*/
	#define CPU_LOAD_IDX_MAX 5
	unsigned long cpu_load[CPU_LOAD_IDX_MAX];
···
	/* capture load from *all* tasks on this cpu: */
	struct load_weight load;
	unsigned long nr_load_updates;
	u64 nr_switches;
	struct cfs_rq cfs;/*cfs,rt,dl是嵌入的子就绪队列*/
	struct rt_rq rt;
	struct dl_rq dl;
···
	struct task_struct *curr, *idle, *stop;/*curr指向当前运行进程的实例，idle和 stop分别指向idle和stop进程的实例*/

	unsigned int clock_update_flags;
	u64 clock;/*就绪队列自身的时钟，每次调用周期性调度器时，都会更新clock的值。*/
	u64 clock_task;
···
};

```

​	系统的所有就绪队列都在[runqueues](https://www.cnblogs.com/openix/p/3264099.html)数组中，该数组的每个元素分别对应于系统中的一个CPU。在单处理器系统中，由于只需要一个就绪队列，数组只有一个元素。

#### 1.7.2 等待队列

​           等待队列可以看作保存进程的容器，在阻塞进程时，将进程放入等待队列；当唤醒进程时，从等待队列中取出进程。

​	**睡眠和唤醒操作**：对等待队列的操作包括睡眠和唤醒（相关函数保存在源代码树的/kernel/sched.c和include/linux/sched.h中）。思想是更改当前进程（CURRENT）的任务状态，并要求重新调度，因为这时这个进程的状态已经改变，不再在调度表的就绪队列中，因此无法再获得执行机会，进入"睡眠"状态，直至被"唤醒"，即其任务状态重新被修改回就绪态。

​	常用的睡眠操作有interruptible_sleep_on和sleep_on。两个函数类似，只不过前者将进程的状态从就绪态（TASK_RUNNING）设置为TASK_INTERRUPTIBLE，允许通过发送signal唤醒它（即可中断的睡眠状态）；而后者将进程的状态设置为TASK_UNINTERRUPTIBLE，在这种状态下，不接收任何singal。

### 1.8 调度实体



​	调度实体是内核进行调度的基本实体单位, 其可能包含一个或者多个进程。对于根的红黑树而言，一个进程组就相当于一个调度实体，一个进程也相当于一个调度实体。

在 struct sched_entity 结构中，值得我们注意的成员是：

- **load：**权重，通过优先级转换而成，是vruntime计算的关键。
- **on_rq：**表明是否处于CFS红黑树运行队列中，需要明确一个观点就是，CFS运行队列里面包含有一个红黑树，但这个红黑树并不是CFS运行队列的全部，因为红黑树仅仅是用于选择出下一个调度程序的算法。很简单的一个例子，普通程序运行时，其并不在红黑树中，但是还是处于CFS运行队列中，其on_rq为真。只有准备退出、即将睡眠等待和转为实时进程的进程其CFS运行队列的on_rq为假。
- **vruntime：**虚拟运行时间，调度的关键，其计算公式：一次调度间隔的虚拟运行时间 = 实际运行时间 * (NICE_0_LOAD / 权重)。可以看出跟实际运行时间和权重有关，红黑树就是以此作为排序的标准，优先级越高的进程在运行时其vruntime增长的越慢，其可运行时间相对就长，而且也越有可能处于红黑树的最左结点，调度器每次都选择最左边的结点为下一个调度进程。注意其值为单调递增，在每个调度器的时钟中断时当前进程的虚拟运行时间都会累加。单纯的说就是进程们都在比谁的vruntime最小，最小的将被调度。
- **cfs_rq：**此调度实体所处于的CFS运行队列。
- **my_q：**如果此调度实体代表的是一个进程组，那么此调度实体就包含有一个自己的CFS运行队列，其CFS运行队列中存放的是此进程组中的进程，这些进程就不会在其他CFS运行队列的红黑树中被包含(包括顶层红黑树也不会包含他们，他们只属于这个进程组的红黑树)。

　　对于怎么理解一个进程组有它自己的CFS运行队列，其实很好理解，比如在根CFS运行队列的红黑树上有一个进程A一个进程组B，各占50%的CPU，对于根的红黑树而言，他们就是两个调度实体。调度器调度的不是进程A就是进程组B，而如果调度到进程组B，进程组B自己选择一个程序交给CPU运行就可以了，而进程组B怎么选择一个程序给CPU，就是通过自己的CFS运行队列的红黑树选择，如果进程组B还有个子进程组C，原理都一样，就是一个层次结构。

> 组调度是cgroup里面的概念，指将N个进程视为一个整体，参与系统中的调度过程，为何要引入**[组调度](http://oenhan.com/task-group-sched)**呢？假设用户A和B共用一台机器。我们可能希望A和B能公平的分享CPU资源，但是如果用户A使用创建了8个进程、而用户B只创建1个进程，根据CFS调度是基于进程的（假设用户进程的优先级相同），A用户占用的CPU的时间将是B用户的8倍。如何保证A、B用户公平分享CPU呢？**组调度**就能做到这一点。把属于用户A和B的进程各分为一组，调度程序将先从两个组中选择一个组，再从选中的组中选择一个进程来执行。如果两个组被选中的机率相当，那么用户A和B将各占有约50%的CPU。

​	在内核中，进程组由task_group进行管理，其中涉及的内容很多都是cgroup控制机制 ，此处指重点描述组调度的部分，对于一个task_group来说，它的调度实体和运行队列都是每CPU一份的，一个（task_group对应的）调度实体只会被加入到相同CPU所对应的运行队列。而对于task来说，它的调度实体则只有一份（没有按CPU划分），调度程序的负载均衡功能可能会将（task对应的）调度实体从不同CPU所对应的运行队列移来移去。。

| 调度实体            | 名称           | 描述                              | 对应调度器类           |
| --------------- | ------------ | ------------------------------- | ---------------- |
| sched_dl_entity | DEADLINE调度实体 | 采用EDF算法调度的实时调度实体                | dl_sched_class   |
| sched_rt_entity | RT调度实体       | 采用Roound-Robin或者FIFO算法调度的实时调度实体 | rt_sched_class   |
| sched_entity    | CFS调度实体      | 采用CFS算法调度的普通非实时进程的调度实体          | fair_sched_class |



### 1.9 CFS 调度类

​	CFS调度器类针对的是SCHED_NORMAL、SCHED_IDLE和SCHED_BATCH的进程。CFS摒弃了传统调度器的时间片概念，重点关注进程的等待时间。引入了虚拟运行时间的概念，CFS的就绪队列是一棵以vruntime为键值的红黑树，当CFS需要选择下一个进程运行的时候，就会选择具有最小vruntime的进程，对应的就是红黑树最左侧的叶子节点。所有与虚拟时钟有关的计算都在`update_curr`中执行，该函数在系统中各个不同地方调用，包括周期性调度器之内。

#### 1.9.1负荷权重

负荷权重的数据结构，进程的负荷权重由`set_load_weight`计算：

```c
struct load_weight {
    unsigned long weight;    /*存储了权重的信息 */
    u32 inv_weight;          /*存储了权重值用于重除的结果 weight * inv_weight = 2^32 */
};
```

调度实体的负荷权重：

```c
struct sched_entity {
    struct load_weight      load;           /* for load-balancing */
···
};
```

**nice值与权重的关系：**

```c
/*kernel/sched/sched.h*/
static const int prio_to_weight[40] = {
 /* -20 */     88761,     71755,     56483,     46273,     36291,
 /* -15 */     29154,     23254,     18705,     14949,     11916,
 /* -10 */      9548,      7620,      6100,      4904,      3906,
 /*  -5 */      3121,      2501,      1991,      1586,      1277,
 /*   0 */      1024,       820,       655,       526,       423,
 /*   5 */       335,       272,       215,       172,       137,
 /*  10 */       110,        87,        70,        56,        45,
 /*  15 */        36,        29,        23,        18,        15,
};
```



#### 1.9.2 最小粒度与目标延迟

​	CFS调度算法是基于理想的完全公平的调度算法，就是说无论**观察时间**多么短，CPU的时间片都是公平分配给各个进程的。由此就存在一个问题，假设观察时间是100ns，进程队列中有10个进程，那么基于完全公平的思想，每个进程就只能分到10ns，但是如果这样的话cpu的时间几乎都会花费在进程的切换上，真正执行进程的CPU时间就少得可怜，因此需要规定一个值，用来保证每个进程至少能占用一定的CPU时间从而在尽可能保证公平性的前提下也尽可能保证CPU时间能高效利用，这个时间就是最小粒度（`sysctl_sched_min_granularity`）。

​	传统调度器的调度延迟是和系统负载相关的，当系统负载增加的时候，用户更容易观察到卡顿现象。比如有A、B、C三个进程交替运行，每个执行100ms，A执行完，就必须等B、C执行完，即需要等20ms，当进程增加了，等待时间也相应的增加，从而导致了较大的延迟。因此CFS设计时增加了目标延迟这一参数，这是一个类似于调度周期概念的参数，它保证了在一个目标延时（targeted latency）中，所有的runnable进程都会至少执行一次，从而保证了有界的、可预测的调度延迟。

```c
/*kernel/sched/fair.c*/
/*计算目标延时*/
static u64 __sched_period(unsigned long nr_running)
{
	if (unlikely(nr_running > sched_nr_latency))/*sched_nr_latency=8*/
        /*最小粒度sysctl_sched_min_granularity=0.75ms*/
		return nr_running * sysctl_sched_min_granularity;/*为了保证最小粒度*/
	else
		return sysctl_sched_latency;/*sysctl_sched_latency=6ms ，8*0.75=6*/
}

/*计算一个周期内各个进程实际运行的时间*/
static u64 sched_slice(struct cfs_rq *cfs_rq, struct sched_entity *se)
{
	u64 slice = __sched_period(cfs_rq->nr_running + !se->on_rq);

	for_each_sched_entity(se) {
		struct load_weight *load;
		struct load_weight lw;

		cfs_rq = cfs_rq_of(se);
		load = &cfs_rq->load;

		if (unlikely(!se->on_rq)) {
			lw = cfs_rq->load;

			update_load_add(&lw, se->load.weight);
			load = &lw;
		}
		slice = __calc_delta(slice, se->load.weight, load);
	}
	return slice;
}
```

​	在时钟中断中，调度相关的一些时间计数量会被更新。同时会检查一下目前运行的进程运行时间是不是超过了一个slice，如果超过了这个间隔，就会设置重新调度标记。会在schedule函数中完成调度并完成进程切换。如果没有小于这个slice量，那就不会触发重新调度。

#### 1.9.3 计算虚拟运行时间

​	[虚拟运行时间](http://blog.csdn.net/jus3ve/article/details/78683044)：virtual runtime(vruntime)，CFS调度内是没有时间片这种说法的，主要思想就是让就绪队列里所有进程均分目标延时，这里所说的均分是考虑了进程负荷权重的均分，也就是考虑了进程优先级。因此引入了虚拟运行时间。

- 计算单个调度周期内分配给进程的总时间：

  `分配给进程的实际运行时间=调度周期 *进程权重 /所有进程权重之和;` 

  调度周期：`unsigned int sysctl_sched_latency = 6000000ULL;`

- 虚拟运行时间计算方法：

  ​

```c
static void update_curr(struct cfs_rq *cfs_rq)
{
	struct sched_entity *curr = cfs_rq->curr;/*当前进程的可调度实体*/
	u64 now = rq_clock_task(rq_of(cfs_rq));/*rq_of返回cfs队列所在的全局就绪队列 */
	u64 delta_exec; 

	if (unlikely(!curr))
		return;

	delta_exec = now - curr->exec_start;/*当前时间和上一次更新负荷权重之间的时间差*/
	if (unlikely((s64)delta_exec <= 0))
		return;

	curr->exec_start = now;/*更新‘更新负荷权重时间’到当前时间*/

	schedstat_set(curr->statistics.exec_max,
		      max(delta_exec, curr->statistics.exec_max));
    /*计算进程花费的总物理时间，将时间差加到先前统计的时间即可*/
	curr->sum_exec_runtime += delta_exec;
    
	schedstat_add(cfs_rq, exec_clock, delta_exec);

	curr->vruntime += calc_delta_fair(delta_exec, curr);/*计算虚拟时间*/
	update_min_vruntime(cfs_rq);/*立即更新最小虚拟时间，因为必须小心保证该值是单调递增*/

···
}
```

​	而在内核中，进程的虚拟运行时间是自进程诞生以来进行**累加**的，每个调度周期内一个进程的虚拟运行时间是通过 `calc_delta_fair(delta_exec, curr)`函数计算的，具体算法如下：

> **一次调度间隔的虚拟运行时间=上一次运行真实时间\*（NICE_0_LOAD/权重）**

​	其中，NICE_0_LOAD是nice为0时的权重。也就是说，nice值为0的进程实际运行时间和虚拟运行时间相同。通过这个公式可以看到，**权重越大的进程获得的虚拟运行时间越小，那么它将被调度器所调度的机会就越大**。

更新进程的vruntime的时机：

- 进程创建时计算进程初始的vruntime值copy_process->sched_fork->task_fork=task_fork_fair->place_entity(cfs_rq,se, 1);
- 唤醒一个进程时：activate_task(rq,p, 0)->enqueue_task_fair->place_entity
- 时钟中断发生时: (task_tick= task_tick_fair)-> update_curr

#### 1.9.4 CFS调度类是如何运作的

[参考例子](http://www.cnblogs.com/openix/p/3254394.html)

​	当一个普通新进程进入执行状态，CFS就会立即开启对虚拟运行时间的追踪，每个时钟中断来临的时候，调用scheduler_tick()，然后根据当前进程的调度类，调用task_tick_fair()函数。task_tick_fair()中的entity_tick（）会调用update_curr(cfs_rq)更新vruntime，在entity_tick()中，首先会更新当前进程的实际运行时间和虚拟运行时间，这里很重要，在entity_tick()函数中最后面的check_preempt_tick()函数就是用来判断进程是否需要被调度的，其判断的标准有两个：

- 先判断当前进程的实际运行时间是否超过CPU分配给这个进程的CPU时间，如果超过，则需要调度。
- 再判断当前进程的vruntime是否大于下个进程的vruntime，如果大于，则需要调度。

如果进程不需要被调度，则继续执行。如果进程需要被调度，就被标记上TIF_NEED_SCHED，并在最近的一个调度点发起调度，调度器就根据vruntime的红黑树选择vruntime最小的进程作为下一个要执行的进程。



### 1.10 RT调度类

​	RT调度类`rt_sched_class`针对的是实时进程，即针对的是调度策略为SCHED_RR和SCHED_FIFO的进程。且只要系统中出现实时进程，就会立即抢占非实时进程。

- SCHED_FIFO

  先进先出调度策略，不使用时间片，使用SCHED_FIFO策略的实时进  程可以一直执行下去，直到阻塞或者释放处理器，或者被更高优先级的实时进程抢占才会结束运行，相同优先级的任务先到先服务，高优先级的任务可以抢占低优先级的任务。

- SCHED_RR

  时间片轮转调度，SCHED_RR的实质是带有时间片的SCHED_FIFO，SCHED_RR的进程消耗完时间片后调度同一优先级的进程。







## 2. 调度相关系统调用

#### 2.1 与优先级相关的系统调用

[（参考网址）](https://access.redhat.com/documentation/en-us/red_hat_enterprise_mrg/?version=2)

- 普通进程相关：

|                           系统调用                           | 描述                                                         |
| :----------------------------------------------------------: | :----------------------------------------------------------- |
| [nice()](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux_for_real_time/7/html/reference_guide/sect-using_library_calls_to_set_priority) | 给进程的静态优先级增加一个给定的值降低优先级，但只有超级用户才能使用负值来提高优先级 |
| [getpriority()](http://www.jollen.org/blog/2006/10/getpriority_setpriority.html) | 获取普通进程的nice值（也可以设置实时进程的nice值）           |
|                        setpriority()                         | 设置普通进程的nice值（也可以设置实时进程的nice值）           |

- 实时进程相关

| 系统调用                 | 描述                                                         |
| ------------------------ | ------------------------------------------------------------ |
| sched_setparam()         | 设置实时进程的优先级                                         |
| sched_getparam()         | 获取实时进程的优先级                                         |
| sched_get_priority_max() | 返回给定调度策略的最大优先级(如果是实时进程返回99，非实时进返回0) |
| sched_get_priority_min() | 返回给定调度策略的最小优先级(如果是实时进程返回1，非实时进返回0) |

```c
#include <stdio.h>
#include <unistd.h>
#include <sched.h>
main(){
  printf("Valid priority range for SCHED_OTHER: %d - %d\n",
         sched_get_priority_min(SCHED_OTHER),
         sched_get_priority_max(SCHED_OTHER));

  printf("Valid priority range for SCHED_FIFO: %d - %d\n",
         sched_get_priority_min(SCHED_FIFO),
         sched_get_priority_max(SCHED_FIFO));

  printf("Valid priority range for SCHED_RR: %d - %d\n",
         sched_get_priority_min(SCHED_RR),
         sched_get_priority_max(SCHED_RR));
}

结果：
Valid priority range for SCHED_OTHER: 0 - 0
Valid priority range for SCHED_FIFO: 1 - 99
Valid priority range for SCHED_RR: 1 - 99

```



#### 2.2 与调度策略相关的系统调用

| 系统调用                                                     | 描述                                         |
| ------------------------------------------------------------ | -------------------------------------------- |
| sched_setscheduler()                                         | 设置进程的调度策略（需要超级用户权限）       |
| [sched_getscheduler()](https://linux.die.net/man/2/sched_getscheduler) | 获取进程的调度策略                           |
| sched_rr_get_interval()                                      | 获取SCHED_RR进程的时间片（需要超级用户权限） |

#### 2.3 与处理器相关的系统调用

| 系统调用            | 描述                                     |
| ------------------- | ---------------------------------------- |
| sched_yield()       | 暂时让出处理器（主要用于SCHED_FIFO进程） |
| sched_getaffinity() | 获取当前进程的cpus_allowed位掩码         |
| sched_setaffinity() | 设置当前进程的cpus_allowed位掩码         |

## 3. 相关疑问与解答

**1.为什么要有很多种类的优先级，为什么要分成普通调度和实时调度**

​	linux针对普通进程和实时进程分别使用静态优先级static_prio和实时优先级rt_priority来指定其默认的优先级别, 然后通过normal_prio函数将他们分别转换为普通优先级normal_prio, 最终换算出动态优先级prio, 动态优先级prio才是内核调度时候优先考虑的优先级字段。



**2.调度器是怎么运行的，会不会存在被其他进程抢占（跟中断有关）**

​	Linux 的调度算法通过模块的方式实现，每种类型的调度器会有一个优先级，调度时会按照优先级遍历调度类，拥有一个可执行进程的最高优先级的调度器类胜出，去选择下面要执行的那个程序。



**3.非实时进程什么时候会把优先级提高到实时优先级**

​	当实时进程需要的资源被普通进程占用，此时需要临时提高此普通进程的优先级到最高优先级，或者比这个实时进程的优先级高一些，通常采取第二种方法。



**4.每个进程都是一个调度实体吗？**

是的，调度实体是调度的基本单位。



**5.一个调度周期中的进程是如何切换的，每个调度周期结束后会发生什么。**

​	通过scheduler_tick()，周期性地寻找vruntime最小的进程，找到之后，如果当前进程的运行时间已经超过了它的slice则设置一个请求重新调度的标志(TIF_NEED_RESCHED)，然后在中断返回时会执行抢占。当前进程被放进运行队列。-------在时钟中断中，调度相关的一些时间计数量会被更新。同时会检查一下目前运行的进程运行时间是不是超过了一个slice，如果超过了这个间隔，就会设置重新调度标记。会在schedule函数中完成调度并完成进程切换。如果没有小于这个slice量，那就不会触发重新调度。



**6.新开进程的vruntime是多少？**

​	假如新进程的vruntime初值为0的话，比老进程的值小很多，那么它在相当长的时间内都会保持抢占CPU的优势，老进程就要饿死了，这显然是不公平的。所以CFS是这样做的：每个CPU的运行队列cfs_rq都维护一个*min_vruntime*字段，记录该运行队列中所有进程的vruntime最小值，新进程的初始vruntime值就以它所在运行队列的min_vruntime为基础来设置，与老进程保持在合理的差距范围内。

> 新进程的vruntime初值的设置与两个参数有关：
>
> > > - sched_child_runs_first*：规定fork之后让子进程先于父进程运行;
> > >
> > >
> > > - sched_features的*START_DEBIT*位：规定新进程的第一次运行要有延迟。
> > >
> > >  子进程在创建时，vruntime初值首先被设置为min_vruntime；然后，如果sched_features中设置了START_DEBIT位，vruntime会在min_vruntime的基础上再增大一些。设置完子进程的vruntime之后，检查sched_child_runs_first参数，如果为1的话，就比较父进程和子进程的vruntime，若是父进程的vruntime更小，就对换父、子进程的vruntime，这样就保证了子进程会在父进程之前运行。



**7.眠进程的vruntime一直保持不变吗？**

​	如果休眠进程的 vruntime 保持不变，而其他运行进程的 vruntime 一直在推进，那么等到休眠进程终于唤醒的时候，它的vruntime比别人小很多，会使它获得长时间抢占CPU的优势，其他进程就要饿死了。这显然是另一种形式的不公平。CFS是这样做的：在休眠进程被唤醒时重新设置vruntime值，以min_vruntime值为基础，给予一定的补偿，但不能补偿太多，在`place_entity`中进行补偿计算。



**8.休眠进程在唤醒时会立刻抢占CPU吗？**

​	这是由CFS的*唤醒抢占 *特性决定的，即sched_features的*`WAKEUP_PREEMPT`位。决定的。由于休眠进程在唤醒时会获得vruntime的补偿，所以它在醒来的时候有能力抢占CPU是大概率事件，这也是CFS调度算法的本意，即保证交互式进程的响应速度，因为交互式进程等待用户输入会频繁休眠。除了交互式进程以外，主动休眠的进程同样也会在唤醒时获得补偿，例如通过调用sleep()、nanosleep()的方式，定时醒来完成特定任务，这类进程往往并不要求快速响应，如果由于主动休眠造成了整体性能的下降，可以禁用唤醒抢占特性或者修改`*sched_wakeup_granularity_ns*`参数，这个参数限定了一个唤醒进程要抢占当前进程之前必须满足的条件：只有当该唤醒进程的vruntime比当前进程的vruntime小、并且两者差距大于`sched_wakeup_granularity_ns`的情况下，才可以抢占，否则不可以。这个参数越大，发生唤醒抢占就越不容易。



**9.进程从一个CPU迁移到另一个CPU上的时候vruntime会不会变？**

​	每个CPU都有各自的运行队列，所以每个队列的最小虚拟运行时间不同，如果一个进程从`min_vruntime`更小的CPU (A) 上迁移到`min_vruntime`更大的CPU (B) 上，可能就会占便宜了，因为CPU (B) 的运行队列中进程的vruntime普遍比较大，迁移过来的进程就会获得更多的CPU时间片。这显然不太公平。可以使用`grep min_vruntime /proc/sched_debug`查看各个CPU队列的`min_vruntime`。

因此CFS是这样做的：
​	当进程从一个CPU的运行队列中出来 (dequeue_entity) 的时候，它的vruntime要减去队列的min_vruntime值；而当进程加入另一个CPU的运行队列 ( enqueue_entiry) 时，它的vruntime要加上该队列的min_vruntime值。这样，进程从一个CPU迁移到另一个CPU之后，vruntime保持相对公平。

```c
static void
dequeue_entity(struct cfs_rq *cfs_rq, struct sched_entity *se, int flags)
{
...
        /*
         * Normalize the entity after updating the min_vruntime because the
         * update can refer to the ->curr item and we need to reflect this
         * movement in our normalized position.
         */
        if (!(flags & DEQUEUE_SLEEP))
                se->vruntime -= cfs_rq->min_vruntime;
...
}

static void
enqueue_entity(struct cfs_rq *cfs_rq, struct sched_entity *se, int flags)
{
        /*
         * Update the normalized vruntime before updating min_vruntime
         * through callig update_curr().
         */
        if (!(flags & ENQUEUE_WAKEUP) || (flags & ENQUEUE_WAKING))
                se->vruntime += cfs_rq->min_vruntime;
...
}
```

​

**10.实时进程会不会导致其它进程得不到运行机会？**

​	如果实时进程占着CPU不放，会不会导致其它进程得不到运行机会，包括管理员的shell也无法运行、连基本的管理任务也进行不了，最终造成整个系统失去控制？

​	通常不会。因为Linux kernel有一个**RealTime Throttling**机制，就是为了防止CPU消耗型的实时进程霸占所有的CPU资源而造成整个系统失去控制。它的原理很简单，就是保证无论如何普通进程都能得到一定比例（默认5%）的CPU时间，可以通过两个内核参数来控制：

- /proc/sys/kernel/sched_rt_period_us
  缺省值是1,000,000 μs (1秒)，表示实时进程的运行粒度为1秒。（注：修改这个参数请谨慎，太大或太小都可能带来问题）。

- /proc/sys/kernel/sched_rt_runtime_us
  缺省值是 950,000 μs (0.95秒)，表示在1秒的运行周期里所有的实时进程一起最多可以占用0.95秒的CPU时间。

  ​	如果这0.05秒内没有任何普通进程需要使用CPU（一直没有TASK_RUNNING状态的普通进程）呢？这种情况下既然普通进程对CPU没有需求，实时进程是否可以运行超过0.95秒呢？不能。在剩下的0.05秒中内核宁可让CPU一直闲着，也不让实时进程使用。

  ​	但是如果sched_rt_runtime_us=-1，表示取消限制，意味着实时进程可以占用100%的CPU时间（慎用，有可能使系统失去控制）。

​        所以，Linux kernel默认情况下保证了普通进程无论如何都可以得到5%的CPU时间，尽管系统可能会慢如蜗牛，但管理员仍然可以利用这5%的时间设法恢复系统，比如停掉失控的实时进程，或者给自己的shell进程赋予更高的实时优先级以便执行管理任务，等等。



**11.如果一个进程组内的进程在不同的CPU上运行，那么如何维护进程组的红黑树？**



**12.为什么对于 ps -el 为什么 nice (NI) 是 0，而 PR (priority) 显示的是 80 而不是 120。**

​	ps 中，如果是 normal process 就显示 60 + p->prio - MAX_RT_PRIO，如果是 realtime process 则只显示 rt。



**13.进程组在每个cpu上都有对应的调度实体和cfs调度队列，那到底同一个进程组中的不同进程能否同时在不同的cpu上运行？**

​	可以运行。

**14.调度实体中组调度的组的优先级怎么确定？**

​	实时进程组的优先级就被定义为“组内最高优先级的进程所拥有的优先级”。

**15.新进程是如何入队的？**

```
fork或vfork后：
do_fork
_do_fork
wake_up_new_task
activate_task
enqueue_task
```



-------------------------

## 4. 相关源码及系统调用测试代码

#### __schedule代码

```c
static void __sched notrace __schedule(bool preempt)
{
	struct task_struct *prev, *next;
	unsigned long *switch_count;
	struct rq_flags rf;
	struct rq *rq;
	int cpu;

	cpu = smp_processor_id();
	rq = cpu_rq(cpu);//跟当前进程相关的runqueue的信息被保存在rq中
	prev = rq->curr;//把当前进程设置为前一个进程

	schedule_debug(prev);//当前进程与时间相关的统计量的检查

	if (sched_feat(HRTICK))
		hrtick_clear(rq);

	local_irq_disable();
	rcu_note_context_switch(preempt);

	/*
	 * Make sure that signal_pending_state()->signal_pending() below
	 * can't be reordered with __set_current_state(TASK_INTERRUPTIBLE)
	 * done by the caller to avoid the race with signal_wake_up().
	 */
	rq_lock(rq, &rf);
	smp_mb__after_spinlock();

	/* Promote REQ to ACT */
	rq->clock_update_flags <<= 1;
	update_rq_clock(rq);//更新就绪队列的自身时钟信息

	switch_count = &prev->nivcsw;
	if (!preempt && prev->state) {//state: -1 unrunnable, 0 runnable, >0 stopped */
		if (unlikely(signal_pending_state(prev->state, prev))) {
			prev->state = TASK_RUNNING;//若没有待处理的信号则将进程的状态设置为RUNNING
		} else {
			deactivate_task(rq, prev, DEQUEUE_SLEEP | DEQUEUE_NOCLOCK);//若有待处理的信号则把当前进程移除队列
			prev->on_rq = 0;

			if (prev->in_iowait) {
				atomic_inc(&rq->nr_iowait);
				delayacct_blkio_start();
			}

			/*
			 * If a worker went to sleep, notify and ask workqueue
			 * whether it wants to wake up a task to maintain
			 * concurrency.
			 */
			if (prev->flags & PF_WQ_WORKER) {
				struct task_struct *to_wakeup;

				to_wakeup = wq_worker_sleeping(prev);
				if (to_wakeup)
					try_to_wake_up_local(to_wakeup, &rf);
			}
		}
		switch_count = &prev->nvcsw;//上下文切换计数
	}
	
	/*核心调度函数*/

	next = pick_next_task(rq, prev, &rf);
	/*清除TIF_NEED_RESCHED标志*/
	clear_tsk_need_resched(prev);
	/*清除PREEMPT_NEED_RESCHED标志*/
	clear_preempt_need_resched();

	if (likely(prev != next)) {
		/* 该CPU进程切换次数加1 */
		rq->nr_switches++;
		 /* 该CPU当前执行进程为新进程 */
		rq->curr = next;
		/*
		 * The membarrier system call requires each architecture
		 * to have a full memory barrier after updating
		 * rq->curr, before returning to user-space. For TSO
		 * (e.g. x86), the architecture must provide its own
		 * barrier in switch_mm(). For weakly ordered machines
		 * for which spin_unlock() acts as a full memory
		 * barrier, finish_lock_switch() in common code takes
		 * care of this barrier. For weakly ordered machines for
		 * which spin_unlock() acts as a RELEASE barrier (only
		 * arm64 and PowerPC), arm64 has a full barrier in
		 * switch_to(), and PowerPC has
		 * smp_mb__after_unlock_lock() before
		 * finish_lock_switch().
		 */
		++*switch_count;

		trace_sched_switch(preempt, prev, next);

		/* Also unlocks the rq: */
		rq = context_switch(rq, prev, next, &rf);
	} else {
		rq->clock_update_flags &= ~(RQCF_ACT_SKIP|RQCF_REQ_SKIP);
		rq_unlock_irq(rq, &rf);
	}

	balance_callback(rq);
}

```

#### pick_next_task代码

```c
static inline struct task_struct *
pick_next_task(struct rq *rq, struct task_struct *prev, struct rq_flags *rf)
{
	const struct sched_class *class;
	struct task_struct *p;

	/*
	 * Optimization: we know that if all tasks are in the fair class we can
	 * call that function directly, but only if the @prev task wasn't of a
	 * higher scheduling class, because otherwise those loose the
	 * opportunity to pull in more work from other CPUs.
	 */
	 /*如果当前进程属于cfs或者idle调度类并且当且运行队列中实体数量等于cfs队列实体数量
	  *
      *
      *
	  */
	if (likely((prev->sched_class == &idle_sched_class ||
		    prev->sched_class == &fair_sched_class) &&
		   rq->nr_running == rq->cfs.h_nr_running)) {

		p = fair_sched_class.pick_next_task(rq, prev, rf);//pick_next_task_fair
	/* May return RETRY_TASK when it finds a higher prio class has runnable tasks.*/
		/*RETRY_TASK表示当前队列里还有除了cfs和idle优先级更高的调度类*/
		if (unlikely(p == RETRY_TASK))
			goto again;

		/* Assumes fair_sched_class->next == idle_sched_class */
		//如果调度队列内没有cfs实体则选择idle实体
		if (unlikely(!p))
			p = idle_sched_class.pick_next_task(rq, prev, rf);

		return p;
	}

again:

/*#define sched_class_highest (&stop_sched_class)
  #define for_each_class(class) \
	   for (class = sched_class_highest; class; class = class->next)
	
	extern const struct sched_class stop_sched_class;
	extern const struct sched_class dl_sched_class;
	extern const struct sched_class rt_sched_class;
	extern const struct sched_class fair_sched_class;
	extern const struct sched_class idle_sched_class;
*/
	for_each_class(class) {
		p = class->pick_next_task(rq, prev, rf);
		if (p) {
			if (unlikely(p == RETRY_TASK))
				goto again;
			return p;
		}
	}
	/* The idle class should always have a runnable task: */
	BUG();
}
```

#### context_switch代码

```c
/*
 * context_switch - switch to the new MM and the new thread's register state.
 */

static __always_inline struct rq *
context_switch(struct rq *rq, struct task_struct *prev,
        struct task_struct *next, struct pin_cookie cookie)
{
 struct mm_struct *mm, *oldmm;
 prepare_task_switch(rq, prev, next);
 mm = next->mm;
 oldmm = prev->active_mm;
 /*
  * For paravirt, this is coupled with an exit in switch_to to
  * combine the page table reload and the switch backend into
  * one hypercall.
  */
 arch_start_context_switch(prev);
/*
  如果next是内核线程，
  则线程使用prev所使用的地址空;schedule( )函数把该线程设置为懒惰TLB模式
*/
 if (!mm) {
  next->active_mm = oldmm;
  atomic_inc(&oldmm->mm_count);
  enter_lazy_tlb(oldmm, next);
 } else
//如果next是一个普通进程，schedule( )函数用next的地址空间替换prev的地址空间
  switch_mm_irqs_off(oldmm, mm, next);
/*
  如果prev是内核线程，context_switch()函数就把指向prev
  内存描述符的指针保存到运行队列的prev_mm字段中，然后重新设置prev->active_mm
*/
 if (!prev->mm) {
  prev->active_mm = NULL;
  rq->prev_mm = oldmm;
 }
 /*
  * Since the runqueue lock will be released by the next
  * task (which is an invalid locking op but in the case
  * of the scheduler it's an obvious special-case), so we
  * do an early lockdep release here:
  */
 lockdep_unpin_lock(&rq->lock, cookie);
 spin_release(&rq->lock.dep_map, 1, _THIS_IP_);
 /* Here we just switch the register state and the stack. */
 /*context_switch()终于可以调用switch_to()执行prev和next之间的进程切换了*/
 /*具体的切换工作，切换堆栈和寄存器*/
 switch_to(prev, next, prev);
 /* 屏障同步, 一般用编译器指令实现
     * 确保了switch_to和finish_task_switch的执行顺序
     * 不会因为任何可能的优化而改变 */
 barrier();
 return finish_task_switch(prev);
}
```

#### 系统调用测试代码

```c
#define _GNU_SOURCE

#include <stdio.h>
#include <sched.h>
#include <zconf.h>
#include <sys/resource.h>
#include <memory.h>

/*
 * >nice() 给进程的静态优先级增加一个给定的值降低优先级，但只有超级用户才能使用负值来提高优先级
 * >sched_setparam()	设置实时进程的优先级
 * >sched_getparam()	获取实时进程的优先级
 * >sched_get_priority_max()	返回给定调度策略的最大优先级(如果是实时进程返回99，非实时进返回0)
 * >sched_get_priority_min()	返回给定调度策略的最小优先级(如果是实时进程返回1，非实时进返回0)
 *
 * >sched_setscheduler()	设置进程的调度策略（需要超级用户权限）
 * >sched_getscheduler()	获取进程的调度策略
 * >sched_rr_get_interval()	获取SCHED_RR进程的时间片（需要超级用户权限）
 *
 * sched_yield()	暂时让出处理器（主要用于SCHED_FIFO进程）
 * sched_getaffinity()	获取当前进程的cpus_allowed位掩码
 * sched_setaffinity()	设置当前进程的cpus_allowed位掩码
 */
int getscheduler(pid_t pid, struct sched_param sp);

void get_RR_ts(pid_t pid);

void line();

void get_prio_width(int policy);

void getaffinity(pid_t pid);

void setaffinity(pid_t pid);

int cpucount = 0;

int priority = 0;

int main() {

    int prio;
    int ret;
    pid_t pid;
    struct sched_param sp;
    line();
    printf("please input pid \n");
    scanf("%d", &pid);
    int policy = getscheduler(pid, sp);
    if (policy == 1) {
        printf(" \nThis is a realtime process\n ");
        line();
        getaffinity(pid);
    } else {
        printf("\nThis is not a realtime process\n");
        line();
        printf("You can change the nice value by setpriority() [-20,19]  \n nice =");
        int nicep;
        scanf("%d", &nicep);
        //nice(nicep);
        int success = setpriority(PRIO_PROCESS, pid, nicep);
        line();
        if (success == 0) {
            printf("change nice value succeed\n");
            printf("-> setpriority\n");
        } else { printf("change nice value failed. need CAP_SYS_NICE or superuser\n"); }
        printf("new nice = %d\n", getpriority(PRIO_PROCESS, pid));
        getaffinity(pid);
        setaffinity(pid);
        getaffinity(pid);
        return;
    }
    get_RR_ts(pid);
    line();
    printf("please input new priority");
    get_prio_width(policy);
    scanf("%d", &prio);
    sp.__sched_priority = prio;
    line();
    printf("Whether to set the schedule policy ?\n1.Yes  2.No\n");
    int choose;
    scanf("%d", &choose);
    if (choose == 1) {
        line();
        printf("please choose schedule policy\n0. SCHED_OTHER\n1. SCHED_FIFO\n2. SCHED_RR\n3. SCHED_BATCH\n5. SCHED_IDLE\n");
        int policy = 0;
        scanf("%d", &policy);
        ret = sched_setscheduler(pid, policy, &sp);
        printf("-> sched_setscheduler\n");
        if (ret == -1) {
            printf("error");
            perror("sched_setscheduler");
            return;

        }
    } else if (choose = 2) {
        sched_setparam(pid, &sp);
        printf("-> sched_setparam\n");
    }
    line();
    getscheduler(pid, sp);
    printf("\n");
    get_RR_ts(pid);
    setaffinity(pid);
    getaffinity(pid);
}

int getscheduler(pid_t pid, struct sched_param sp) {
    printf("\n-> sched_getscheduler\n");
    int policy;
    int ret;
    int sig;
    ret = sched_getparam(pid, &sp);

    policy = sched_getscheduler(pid);
    printf("Scheduler Policy(%2d) for PID: %2d  -> ", policy, pid);
    priority = sp.__sched_priority;
    switch (policy) {
        case SCHED_OTHER:
            printf("SCHED_OTHER");
            sig = 0;
            printf("\n-> getpriority\nnice = %d\n", getpriority(PRIO_PROCESS, pid));
            break;
        case SCHED_RR:
            printf("SCHED_RR");
            sig = 1;
            printf("  priority = %d", priority);
            break;
        case SCHED_FIFO:
            printf("SCHED_FIFO");
            sig = 1;
            printf("  priority = %d", priority);
            break;
        case SCHED_BATCH:
            printf("SCHED_BATCH");
            sig = 0;
            printf("  priority = %d", priority);
            break;
        case SCHED_IDLE:
            printf("SCHED_IDLE");
            sig = 0;
            printf("  priority = %d", priority);
            break;
        default:
            printf("Unknown..");
            sig = 0;
    }
    return sig;

}
void get_RR_ts(pid_t pid) {
    printf("-> get_RR_ts");
    struct timespec ts;
    int ret;
    /* real apps must check return values */
    if (sched_rr_get_interval(pid, &ts) == 0) {
        printf("\nTimeslice: %lu.%lu\n", ts.tv_sec, ts.tv_nsec);
    } else { printf("get timeslice failed"); }
}
void line() {
    printf("---------------------------\n");
}

void get_prio_width(int policy) {
    int min = sched_get_priority_min(policy);
    int max = sched_get_priority_max(policy);
    printf("[%d,%d]\n", min, max);
    printf("-> sched_get_priority_min/sched_get_priority_max\n");

}

void getaffinity(pid_t pid) {
    printf("-> sched_getaffinity");
    cpu_set_t my_set;
    int ret;
    CPU_ZERO(&my_set);
    ret = sched_getaffinity(pid, sizeof(my_set), &my_set);
    char str[80];
    strcpy(str, " ");
    int count = 0;
    int j;

    for (j = 0; j < CPU_SETSIZE; ++j) {
        if (CPU_ISSET(j, &my_set)) {
            ++count;
            char cpunum[3];
            sprintf(cpunum, "%d ", j);
            strcat(str, cpunum);
        }
    }
    cpucount = sysconf(_SC_NPROCESSORS_ONLN);
    printf("\npid %d affinity has %d CPUs : %s\n", pid, count, str);
    line();

}

void setaffinity(pid_t pid) {
    printf("-> sched_setaffinity\n");
//http://blog.saliya.org/2015/07/get-and-set-process-affinity-in-c.html
    cpu_set_t my_set;
    line();
    printf("There are %d CPUs\nPlease select which CPU you need to run on\n", cpucount);
    printf("0.CPU 0\n1.CPU 1\n2.CPU 0 and 1\nYour select:");
    int choose;
    scanf("%d", &choose);
    CPU_ZERO(&my_set);
    int ret;
    if (choose == 0) {
        CPU_SET(0, &my_set);
    } else if (choose == 1) {
        CPU_SET(1, &my_set);
    } else if (choose == 2) {
        CPU_SET(0, &my_set);
        CPU_SET(1, &my_set);
    } else {
        return;
    }
    ret = sched_setaffinity(pid, sizeof(my_set), &my_set);
}
```

