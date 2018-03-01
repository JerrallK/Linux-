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
| unsigned int    **rt_priority**;        | 实时优先级，实时进程的优先级，用户可以通过sched_setparam()和sched_setscheduler()来改变进程的实时优先级 |

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

​	在用户空间用户可以使用nice命令设置进程的优先级，nice值在-20到+19之间，数值越小，优先级越高。	

​	内核使用数值范围0~139表示内部优先级, 数值越低, 优先级越高。从1~99的范围专供实时进程使用, nice的值[-20,19]则映射到范围100~139![Ashampoo_Snap_2018年2月4日_19h20m23s_001_](F:\screenshoot\Ashampoo_Snap_2018年2月4日_19h20m23s_001_.png)

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

​	在进程分支出子进程时，子进程的静态优先级继承自父进程。子进程的动态优先级，即task_struct->prio，则设置为父进程的普通优先级。



### 1.4 调度器简介

------------------------------------

​	调度器的存在就是为了让进程尽可能公平地共享CPU时间，创造并行执行的效果。所以调度器的主要任务是：**1. 根据调度策略对进程进行调度。2. 调度后进行上下文切换。**

​	目前Linux有两种调度器：一种是进程由于打算睡眠或者其他原因放弃CPU而直接激活的调度器，叫做**主调度器**；另一种就是通过周期性机制，以固定频率运行检测是否有程序需要进行进程切换，叫做**周期性调度器**。两者统称为：**通用调度器**（generic scheduler）或**核心调度器**（core scheduler）。

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

​	在内核中的许多地方，如果要将CPU分配给与当前活动进程不同的另一个进程，都会直接调用主调度器函数（schedule）。内核需要知道在什么时候调用schedule，所以内核给进程描述符提供了一个**TIF_NEED_RESCHED**标志来表明需要再次执行schedule，当某个进程应当被抢占时，schedule_tick就会设置这个标志，当一个优先级高的进程进入可执行状态时，try_to_wake_up也会设置这个标志。在从系统调用返回之后，内核会检查当前进程是否设置了重调度标志TIF_NEED_RESCHED，如果设置了，内核就会调用schedule。

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

2. 调度全局的pick_next_task选择抢占的进程

   - 如果当前cpu上所有的进程都是cfs调度的普通非实时进程, 则直接用cfs调度, 如果无程序可调度则调度idle进程
   - 否则从优先级最高的调度器类sched_class_highest(目前是stop_sched_class)开始依次遍历所有调度器类的pick_next_task函数, 选择最优的那个进程执行

3. context_switch完成进程上下文切换

   - 调用switch_mm(), 把虚拟内存从一个进程映射切换到新进程中
   - 调用switch_to(),从上一个进程的处理器状态切换到新进程的处理器状态。这包括保存、恢复栈信息和寄存器信息




####1.4.2 周期性调度器 

​	周期性调度器在scheduler_tick中实现。如果系统正在活动中，内核会按照频率HZ自动调用该
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

​	进程被抢占时, 操作系统保存其上下文信息, 同时将新的活动进程的上下文信息加载进来, 这个过程就是**上下文切换**。当这个进程再次成为活动的, 它可以恢复自己的上下文继续从被抢占的位置开始执行。由于上下文切换过程较为繁杂，所以简单介绍。

> 一个进程的[上下文](https://www.cnblogs.com/wanghuaijun/p/6985672.html)可以分为三个部分:**用户级上下文**、**寄存器上下文**以及**系统级上下文**。
>
> - 用户级上下文: 正文、数据、用户堆栈以及共享存储区；
> - 寄存器上下文: 通用寄存器、程序寄存器(IP)、处理器状态寄存器(EFLAGS)、栈指针(ESP)；
> - 系统级上下文: 进程控制块task_struct、内存管理信息(mm_struct、vm_area_struct、pgd、pte)、内核栈。

​	上下文切换只能发生在内核态中，上下文切换通常是计算密集型的，对系统来说意味着消耗大量的 CPU 时间，事实上，可能是操作系统中时间消耗最大的操作。进程调度调度时，内核选择新进程之后进行抢占，通过**context_switch**完成进程上下文切换，主要通过以下两个处理函数完成：	

(1) **switch_mm**更换通过task_struct->mm描述的内存管理上下文。该工作的细节取决于处理器，主要包括加载页表、刷出地址转换后备缓冲器（部分或全部）、向内存管理单元（MMU）提供新的信息。
(2) **switch_to**切换处理器寄存器内容和内核栈，从上一个进程的处理器状态切换到新进程的处理器状态。这包括保存、恢复栈信息和寄存器信息（虚拟地址空间的用户部分在第一步已经变更，其中也包括了用户状态下的栈，因此用户栈就不需要显式变更了）。**此项工作在不同的体系结构下可能差别很大，代码通常都使用汇编语言编写。**

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

​	(1) 调度类用于判断接下来运行哪个进程。内核支持不同的调度策略（完全公平调度、实时调度、在无事可做时调度空闲进程），调度类使得能够以模块化方法实现这些策略，即一个类的代码不需要与其他类的代码交互。在调度器被调用时，它会查询调度器类，得知接下来运行哪个进程。
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



### 1.7 就绪队列

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

​	在内核中，进程组由task_group进行管理，其中涉及的内容很多都是cgroup控制机制 ，此处指重点描述组调度的部分，具体见如下注释。

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

	if (entity_is_task(curr)) {
		struct task_struct *curtask = task_of(curr);

		trace_sched_stat_runtime(curtask, delta_exec, curr->vruntime);
		cpuacct_charge(curtask, delta_exec);
		account_group_exec_runtime(curtask, delta_exec);
	}

	account_cfs_rq_runtime(cfs_rq, delta_exec);
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

[参考链接](https://www.ibm.com/developerworks/cn/linux/l-cfs/index.html)

​	当一个普通新进程进入执行状态，CFS就会立即开启对虚拟运行时间的追踪，每个时钟中断来临的时候更新vruntime，调度器就根据vruntime的红黑树选择vruntime最小的进程来执行。

具体的方法是：根据当前系统的情况计算targeted latency（调度周期），在这个调度周期中计算当前进程应该获得的时间片（物理时间），然后计算当前进程已经累积执行的物理时间，如果大于当前应该获得的时间片，那么更新本进程的vruntime并标记need resched flag，并在最近的一个调度点发起调度。





### 1.10 RT调度类

​	RT调度类`rt_sched_class`针对的是实时进程，即针对的是调度策略为SCHED_RR和SCHED_FIFO的进程。且只要系统中出现实时进程，就会立即抢占非实时进程。

- SCHED_FIFO

  先进先出调度策略，不使用时间片，使用SCHED_FIFO策略的实时进  程可以一直执行下去，直到阻塞或者释放处理器，或者被更高优先级的实时进程抢占才会结束运行，相同优先级的任务先到先服务，高优先级的任务可以抢占低优先级的任务。

- SCHED_RR

  时间片轮转调度，SCHED_RR的实质是带有时间片的SCHED_FIFO，SCHED_RR的进程消耗完时间片后调度同一优先级的进程。









## 3. 调度相关系统调用

#### 3.1 与优先级相关的系统调用

[（参考网址）](https://access.redhat.com/documentation/en-us/red_hat_enterprise_mrg/?version=2)

- 普通进程相关：

|                           系统调用                           | 描述                                                         |
| :----------------------------------------------------------: | :----------------------------------------------------------- |
| [nice()](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux_for_real_time/7/html/reference_guide/sect-using_library_calls_to_set_priority) | 给进程的静态优先级增加一个给定的值降低优先级，但只有超级用户才能使用负值来提高优先级 |
| [getpriority()](http://www.jollen.org/blog/2006/10/getpriority_setpriority.html) |                                                              |
|                        setpriority()                         |                                                              |

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

***
```



#### 问题：为什么sched_get_priority_min()返回的是1而不是0？

#### 3.2 与调度策略相关的系统调用

| 系统调用                                                     | 描述                                         |
| ------------------------------------------------------------ | -------------------------------------------- |
| sched_setscheduler()                                         | 设置进程的调度策略（需要超级用户权限）       |
| [sched_getscheduler()](https://linux.die.net/man/2/sched_getscheduler) | 获取进程的调度策略                           |
| sched_rr_get_interval()                                      | 获取SCHED_RR进程的时间片（需要超级用户权限） |

#### 3.3 与处理器相关的系统调用

| 系统调用            | 描述                                     |
| ------------------- | ---------------------------------------- |
| sched_yield()       | 暂时让出处理器（主要用于SCHED_FIFO进程） |
| sched_getaffinity() | 获取当前进程的cpus_allowed位掩码         |
| sched_setaffinity() | 设置当前进程的cpus_allowed位掩码         |

