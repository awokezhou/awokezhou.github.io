---
title: pm-freezing-task
date: 2019-12-11 17:57:57
tags: [Kernel, PM]
categories:
- Linux Kernel
- 电源管理
comments: true
---

Linux PM suspend会冻结所有用户空间进程和大部分内核线程，本文分析梳理冻结过程是如何实现的

参考文档
* kernel document：Document/power/freezing-of-tasks.txt
*  [蜗窝科技-Linux进程冻结技术](http://www.wowotech.net/pm_subsystem/237.html)

## 为什么要冻结进程
主要是以下几点原因
* protect fs
主要原因是防止文件系统在hibernation过程中损坏。目前，kernel还没有检查文件系统的简单方法，因此，如果对磁盘上的文件系统数据或元数据进行了任何修改，kernel就无法将它们恢复到修改之前的状态。同时，每个hibernation镜像都包含一些与文件系统相关的信息，这些信息必须与从镜像恢复系统内存状态后磁盘上数据和元数据的状态一致(否则文件系统将受到严重损坏，通常使它们几乎无法修复)。因此，kernel会冻结那些可能导致磁盘上文件系统的数据和元数据在创建hibernation镜像之后和系统最终关闭之前被修改的任务。其中大多数是用户空间进程，但是如果任何内核线程可能导致这样的事情发生，它们必须是可释放的
* memory
为了创建hibernation镜像，kernel需要在设备停用之前free大量内存(接近50%的有效RAM空间)，因为kernel需要用这些内存做swapping out。在镜像的内存被释放之后，kernel不希望任务分配额外的内存，因此通过提前冻结它们来防止。
* protect devices
防止用户空间进程和一些内核线程干扰设备的suspend和resume。例如，当kernel suspend设备时，在第二个CPU上运行的用户空间进程可能会很麻烦，如果没有任务冻结，kernel需要一些保护措施，以防止在这种情况下可能发生的竞争情况

## 实现

进程冻结的代码实现主要位于以下文件中
* kernel/kernel/power/suspend.c
* kernel/kernel/power/process.c
* kernel/kernel/power/freezer.c

进程冻结的入口函数为`suspend_freeze_processes()`，调用链为
```cmd
enter_state -->
  suspend_prepare -->
    suspend_freeze_processes
```

函数`suspend_freeze_processes()`代码如下
```c
static inline int suspend_freeze_processes(void)
{
	int error;

	error = freeze_processes();
	/*
	 * freeze_processes() automatically thaws every task if freezing
	 * fails. So we need not do anything extra upon error.
	 */
	if (error)
		return error;

	error = freeze_kernel_threads();
	/*
	 * freeze_kernel_threads() thaws only kernel threads upon freezing
	 * failure. So we have to thaw the userspace tasks ourselves.
	 */
	if (error)
		thaw_processes();

	return error;
}
```

首先会调用`freeze_processes()`来冻结进程，注释说的很清楚，如果失败会在函数内自动解冻进程；然后会调用`freeze_kernel_threads()`函数来冻结内核线程，如果失败会在函数内部自动解冻内核线程，因此如果失败还需要再调用`thaw_processes()`解冻进程

###  freeze_processes

`freeze_processes()`函数代码如下
```c
int freeze_processes(void)
{
	int error;

	error = __usermodehelper_disable(UMH_FREEZING);
	if (error)
		return error;

	/* Make sure this task doesn't get frozen */
	current->flags |= PF_SUSPEND_TASK;

	if (!pm_freezing)
		atomic_inc(&system_freezing_cnt);

	pm_wakeup_clear();
	pr_info("Freezing user space processes ... ");
	pm_freezing = true;
	error = try_to_freeze_tasks(true);
	if (!error) {
		__usermodehelper_set_disable_depth(UMH_DISABLED);
		pr_cont("done.");
	}
	pr_cont("\n");
	BUG_ON(in_atomic());

	/*
	 * Now that the whole userspace is frozen we need to disbale
	 * the OOM killer to disallow any further interference with
	 * killable tasks.
	 */
	if (!error && !oom_killer_disable())
		error = -EBUSY;

	if (error)
		thaw_processes();
	return error;
}
```

首先禁用了usermodehelper，将当前task标志位`PF_SUSPEND_TASK`置位，将当前task设置为执行suspend的task，以确保该task不会被冻结。因此冻结进程的实际代码位于函数`try_to_freeze_tasks()`中

`try_to_freeze_tasks()`函数代码如下
```c
static int try_to_freeze_tasks(bool user_only)
{
	struct task_struct *g, *p;
	unsigned long end_time;
	unsigned int todo;
	bool wq_busy = false;
	struct timeval start, end;
	u64 elapsed_msecs64;
	unsigned int elapsed_msecs;
	bool wakeup = false;
	int sleep_usecs = USEC_PER_MSEC;

	do_gettimeofday(&start);

	end_time = jiffies + msecs_to_jiffies(freeze_timeout_msecs);

	if (!user_only)
		freeze_workqueues_begin();

	while (true) {
		todo = 0;
		read_lock(&tasklist_lock);
		for_each_process_thread(g, p) {
			if (p == current || !freeze_task(p))
				continue;

			if (!freezer_should_skip(p))
				todo++;
		}
		read_unlock(&tasklist_lock);

		if (!user_only) {
			wq_busy = freeze_workqueues_busy();
			todo += wq_busy;
		}

		if (!todo || time_after(jiffies, end_time))
			break;

		if (pm_wakeup_pending()) {
			wakeup = true;
			break;
		}

		/*
		 * We need to retry, but first give the freezing tasks some
		 * time to enter the refrigerator.  Start with an initial
		 * 1 ms sleep followed by exponential backoff until 8 ms.
		 */
		usleep_range(sleep_usecs / 2, sleep_usecs);
		if (sleep_usecs < 8 * USEC_PER_MSEC)
			sleep_usecs *= 2;
	}

	do_gettimeofday(&end);
	elapsed_msecs64 = timeval_to_ns(&end) - timeval_to_ns(&start);
	do_div(elapsed_msecs64, NSEC_PER_MSEC);
	elapsed_msecs = elapsed_msecs64;

	if (todo) {
		pr_cont("\n");
		pr_err("Freezing of tasks %s after %d.%03d seconds "
		       "(%d tasks refusing to freeze, wq_busy=%d):\n",
		       wakeup ? "aborted" : "failed",
		       elapsed_msecs / 1000, elapsed_msecs % 1000,
		       todo - wq_busy, wq_busy);

		if (!wakeup) {
			read_lock(&tasklist_lock);
			for_each_process_thread(g, p) {
				if (p != current && !freezer_should_skip(p)
				    && freezing(p) && !frozen(p))
					sched_show_task(p);
			}
			read_unlock(&tasklist_lock);
		}
	} else {
		pr_cont("(elapsed %d.%03d seconds) ", elapsed_msecs / 1000,
			elapsed_msecs % 1000);
	}

	return todo ? -EBUSY : 0;
}
```

首先设置了一个`end_time`，该操作的作用是确保冻结进程的执行在规定时间内完成，这个时间由全局变量`freeze_timeout_msecs`控制，它有个初始值，并且可由/sys/power/pm_freeze_timeout节点进行设置

冻结的对象是内核中可以被调度执行的实体，包括用户进程、内核线程和work_queue，`freeze_workqueues_begin()`函数的作用就是冻结了work_queue。然后在一个循环体内遍历`tasklist_lock`线程链表，对每个task执行`freeze_task()`来冻结，循环会通过`todo`记录某些仍然需要再操作的task，在循环退出后再做一些操作

#### 冻结标志
这里需要对task的冻结标志位进行介绍，一共有3个和冻结相关标志位
* PF_NOFREEZE：表明该task不能被冻结
* PF_FROZEN：表明该task已经被冻结
* PF_FREEZER_SKIP：这个标志是辅助用的

未被标记为`PF_NOFREEZE`的task(所有用户空间的进程和部分内核线程)将被视为‘freezable’可冻结的

这里有几个函数和冻结标志紧密相关
* freezing：判断task是否正在冻结中
* frozen：判断task是否已经被冻结
* freezer_should_skip：判断task是否需要跳过冻结
* freeze_task：冻结task

循环体中主要执行的函数是`freeze_task()`，代码如下
```c
/**
 * freeze_task - send a freeze request to given task
 * @p: task to send the request to
 *
 * If @p is freezing, the freeze request is sent either by sending a fake
 * signal (if it's not a kernel thread) or waking it up (if it's a kernel
 * thread).
 *
 * RETURNS:
 * %false, if @p is not freezing or already frozen; %true, otherwise
 */
bool freeze_task(struct task_struct *p)
{
	unsigned long flags;

	/*
	 * This check can race with freezer_do_not_count, but worst case that
	 * will result in an extra wakeup being sent to the task.  It does not
	 * race with freezer_count(), the barriers in freezer_count() and
	 * freezer_should_skip() ensure that either freezer_count() sees
	 * freezing == true in try_to_freeze() and freezes, or
	 * freezer_should_skip() sees !PF_FREEZE_SKIP and freezes the task
	 * normally.
	 */
	if (freezer_should_skip(p))
		return false;

	spin_lock_irqsave(&freezer_lock, flags);
	if (!freezing(p) || frozen(p)) {
		spin_unlock_irqrestore(&freezer_lock, flags);
		return false;
	}

	if (!(p->flags & PF_KTHREAD))
		fake_signal_wake_up(p);
	else
		wake_up_state(p, TASK_INTERRUPTIBLE);

	spin_unlock_irqrestore(&freezer_lock, flags);
	return true;
}
```

首先是进行了几个判断，如果task跳过冻结、正在冻结中、或者是已经冻结，则返回false，循环体的`todo`不会增加。那么接下来需要处理的就是需要被冻结的task。如果是用户空间进程，则通过`fake_signal_wake_up()`函数发送一个假信号用来唤醒进程，如果是kernel线程，则调用`wake_up_state()`来中断唤醒线程。所有可冻结的task必须通过调用`try_to_freeze()`来响应该唤醒

对于用户空间进程而言，在其信号处理句柄中会自动调用`try_to_freeze()`，但是可冻结的内核线程必须在适当的位置显式的调用`wait_event_freezable()`或者`wait_event_freezable_timeout()`来间接的调用`try_to_freeze()`，并作一些安全检查

#### try_to_freeze
该函数定义在include/linux/freeze.h中，代码如下
```c
/*
 * DO NOT ADD ANY NEW CALLERS OF THIS FUNCTION
 * If try_to_freeze causes a lockdep warning it means the caller may deadlock
 */
static inline bool try_to_freeze_unsafe(void)
{
	might_sleep();
	if (likely(!freezing(current)))
		return false;
	return __refrigerator(false);
}

static inline bool try_to_freeze(void)
{
	if (!(current->flags & PF_NOFREEZE))
		debug_check_no_locks_held();
	return try_to_freeze_unsafe();
}
```

最终会调用`__refrigerator()`函数，代码如下
```c
/* Refrigerator is place where frozen processes are stored :-). */
bool __refrigerator(bool check_kthr_stop)
{
	/* Hmm, should we be allowed to suspend when there are realtime
	   processes around? */
	bool was_frozen = false;
	long save = current->state;

	pr_debug("%s entered refrigerator\n", current->comm);

	for (;;) {
		set_current_state(TASK_UNINTERRUPTIBLE);

		spin_lock_irq(&freezer_lock);
		current->flags |= PF_FROZEN;
		if (!freezing(current) ||
		    (check_kthr_stop && kthread_should_stop()))
			current->flags &= ~PF_FROZEN;
		spin_unlock_irq(&freezer_lock);

		if (!(current->flags & PF_FROZEN))
			break;
		was_frozen = true;
		schedule();
	}

	pr_debug("%s left refrigerator\n", current->comm);

	/*
	 * Restore saved task state before returning.  The mb'd version
	 * needs to be used; otherwise, it might silently break
	 * synchronization which depends on ordered task state change.
	 */
	set_current_state(save);

	return was_frozen;
}
EXPORT_SYMBOL(__refrigerator);
```

设置task不可被中断，然后将task标志`PF_FROZEN`置位，表明task已经被冻结，本质上是在一个死循环中，只有当标志位`PF_FROZEN`被清除后才会退出

### freeze_kernel_threads
函数`freeze_kernel_threads()`内部实际上还是通过调用`try_to_freeze_tasks()`来处理
```c
/**
 * freeze_kernel_threads - Make freezable kernel threads go to the refrigerator.
 *
 * On success, returns 0.  On failure, -errno and only the kernel threads are
 * thawed, so as to give a chance to the caller to do additional cleanups
 * (if any) before thawing the userspace tasks. So, it is the responsibility
 * of the caller to thaw the userspace tasks, when the time is right.
 */
int freeze_kernel_threads(void)
{
	int error;

	pr_info("Freezing remaining freezable tasks ... ");

	pm_nosig_freezing = true;
	error = try_to_freeze_tasks(false);
	if (!error)
		pr_cont("done.");

	pr_cont("\n");
	BUG_ON(in_atomic());

	if (error)
		thaw_kernel_threads();
	return error;
}
```

### thaw_processes
`thaw_processes()`代码如下
```c
void thaw_processes(void)
{
	struct task_struct *g, *p;
	struct task_struct *curr = current;

	trace_suspend_resume(TPS("thaw_processes"), 0, true);
	if (pm_freezing)
		atomic_dec(&system_freezing_cnt);
	pm_freezing = false;
	pm_nosig_freezing = false;

	oom_killer_enable();

	pr_info("Restarting tasks ... ");

	__usermodehelper_set_disable_depth(UMH_FREEZING);
	thaw_workqueues();

	read_lock(&tasklist_lock);
	for_each_process_thread(g, p) {
		/* No other threads should have PF_SUSPEND_TASK set */
		WARN_ON((p != curr) && (p->flags & PF_SUSPEND_TASK));
		__thaw_task(p);
	}
	read_unlock(&tasklist_lock);

	WARN_ON(!(curr->flags & PF_SUSPEND_TASK));
	curr->flags &= ~PF_SUSPEND_TASK;

	usermodehelper_enable();

	schedule();
	pr_cont("done.\n");
	trace_suspend_resume(TPS("thaw_processes"), 0, false);
}
```

首先启动了`OOM killer`线程，然后禁用usermodehelper，`thaw_workqueues()`函数解冻了work_queue。遍历`tasklist_lock`，对每个task调用`__thaw_task()`函数

`__thaw_task()`函数代码如下
```c
void __thaw_task(struct task_struct *p)
{
	unsigned long flags;

	spin_lock_irqsave(&freezer_lock, flags);
	if (frozen(p))
		wake_up_process(p);
	spin_unlock_irqrestore(&freezer_lock, flags);
}
```

`wake_up_process()`的处理涉及到Linux进程管理相关内容，暂不做分析

## 总结
由于休眠需要保护文件系统和设备等资源以防止用户空间和部分内核空间线程的操作，因此在休眠的第一步kernel冻结了所有用户空间进程和部分内核线程

冻结task的实质是通过`PF_FROZEN`标志位和信号来控制task进入/退出一个死循环，达到户空间进程和部分内核线程不进行任何实际操作的目的