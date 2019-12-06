---
title: pm-debugging
date: 2019-12-05 13:53:05
tags: [Kernel, PM]
categories:
- Linux Kernel
- 电源管理
comments: true
---

电源管理的调试对于开发需要Suspend to Disk(STD)或者Suspend to Ram(STR)的系统来说，非常必要和重要

由于系统在suspend/resume过程会进行非常复杂的一系列操作，如禁用console、冻结进程等，会导致常规的调试方法难以排查定位问题和跟踪流程

Linux的电源管理框架提供了专门的调试方法，用于方便开发者调试不同类型、不同深度的suspend/resume，本文介绍一些常用的工具和使用方法，并在实际环境中验证

**参考文档**
* kernel document : /Document/power/basic-pm-debugging.txt
* kernel document : /Document/power/drivers-testing.txt
* kernel document : /Document/power/s2ram.txt
* [ubuntu wiki - DebuggingKernelSuspend](https://wiki.ubuntu.com/DebuggingKernelSuspend)
* [stackexchange - How to debug a suspend problem?](https://unix.stackexchange.com/questions/28097/how-to-debug-a-suspend-problem)
* [inter open source blog - BEST PRACTICE TO DEBUG LINUX* SUSPEND/HIBERNATE ISSUES](https://01.org/zh/blogs/rzhang/2015/best-practice-debug-linux-suspend/hibernate-issues?langredirect=1)

**测试环境**
本测试在ubuntu上使用qemu创建arm虚拟机环境，linux源码版本4.0，编写内核模块qksleep_test用于调试，关于qemu调试arm kernel参见文章[Qemu+gdb调试内核](sadadasd)
* 宿主机 : 18.04.1-Ubuntu x86_64
* 虚拟机 : qemu-system-arm vexpress-a9
  * kernel : linux_4.0

## qksleep_test
源码下载
* [pm-debugging.rar](/download/pm-debugging/pm-debugging.rar)
qksleep_test以platform_driver方式向系统注册驱动，在其`pm`操作域上挂接私有的suspend/resume函数
```c
static int qksleep_suspend(struct device *dev)
{
	qksleep_dev_t *qkdev = get_qkdev();
	qksleep_dev_priv *priv = &qkdev->priv_data;
	
	qksleep_debug("qksleep suspend in");

	if (priv->suspend_wakeup_timeout > 0)
		schedule_delayed_work(&priv->suspend_wakeup, msecs_to_jiffies(priv->suspend_wakeup_timeout));
		
	if (priv->suspend_errlock)
		qksleep_lockerr(priv);

	if (priv->suspend_timeout)
		qksleep_vsleep(priv, priv->suspend_timeout);

	return priv->suspend_ret;
}

static int qksleep_resume(struct device *dev)
{
	qksleep_dev_t *qkdev = get_qkdev();
	qksleep_dev_priv *priv = &qkdev->priv_data;
	
	qksleep_debug("qksleep resume in");

	if (priv->resume_errlock)
		qksleep_lockerr(priv);

	if (priv->resume_timeout)
		qksleep_vsleep(priv, priv->resume_timeout);

	return priv->resume_ret;
}

static struct dev_pm_ops qk_sleep_pm = {
	.suspend = qksleep_suspend,
	.resume = qksleep_resume,
};

static struct platform_driver qksleep_driver = {
	.driver = {
		.name = QK_SLEEP_DRV_NAME,
		.pm = &qk_sleep_pm,
		.owner = THIS_MODULE,
	},
	.probe = qksleep_probe,
	.remove = qksleep_remove,
};
```

在platform_driver的probe函数中，向系统注册了7个sysfs节点
```c
static struct attribute *qksleep_attr[] = {
	&dev_attr_suspend_wakeup_timeout.attr,
	&dev_attr_suspend_ret.attr,	
	&dev_attr_suspend_timeout.attr,
	&dev_attr_suspend_errlock.attr,
	&dev_attr_resume_ret.attr,	
	&dev_attr_resume_timeout.attr,	
	&dev_attr_resume_errlock.attr,
	NULL,
};
```

* suspend_wakeup_timeout：suspend后多久唤醒系统
* suspend_ret：suspend函数返回值
* suspend_timeout：suspend函数中模拟一个超时时间
* suspend_errlock：suspend函数中模拟一个错误的锁操作
* resume_xxx：同上

编译kernel，并用quem启动后，在/sys/devices/platform/qksleep路径下生成了该设备的所有sysfs节点
```shell
/ # cd /sys/devices/platform/qksleep/
/sys/devices/platform/qksleep # ls
driver                  resume_ret              suspend_timeout
driver_override         resume_timeout          suspend_wakeup_timeout
modalias                subsystem               uevent
power                   suspend_errlock
resume_errlock          suspend_ret
/sys/devices/platform/qksleep # 
```

查看suspend/resume操作默认值
```shell
/sys/devices/platform/qksleep # cat  suspend_*
suspend errlock:0
suspend ret:0
suspend timeout:0(ms)
suspend wakeup timeout:10000(ms)
/sys/devices/platform/qksleep # 
/sys/devices/platform/qksleep # cat resume_*
resume errlock:0
resume ret:0
resume timeout:0(ms)
/sys/devices/platform/qksleep #
```

执行以下命令可控制qksleep设备suspend 2秒后唤醒
```shell
echo 20000 > suspend_wakeup_timeout
```

执行以下命令可控制qksleep设备suspend函数返回错误值-1
```shell
echo -1 > suspend_ret
```

执行以下命令可控制qksleep设备在suspend函数中模拟一个死锁
```shell
echo 1 > suspend_errlock
```

执行以下命令可控制qksleep设备在suspend函数中模拟一个2秒的超时
```shell
echo 2000 > suspend_timeout
```

resume同suspend

操作/sys/power/state节点，手动进入休眠状态，qksleep默认会在10秒后唤醒系统
```shell
/sys/devices/platform/qksleep # echo mem > /sys/power/state
[  615.345438] PM: Syncing filesystems ... done.
[  615.346378] PM: Preparing system for mem sleep
[  615.374765] Freezing user space processes ... (elapsed 0.011 seconds) done.
[  615.387040] Freezing remaining freezable tasks ... (elapsed 0.008 seconds) done.
[  615.396303] PM: Entering mem sleep
[  615.396496] Suspending console(s) (use no_console_suspend to debug)
[  615.401243] [debug] [qksleep_suspend:89] qksleep suspend in
[  615.401248] PM: suspend of devices complete after 3.752 msecs
[  615.401269] PM: suspend devices took 0.000 seconds
[  615.402693] PM: late suspend of devices complete after 1.392 msecs
[  615.404024] PM: noirq suspend of devices complete after 1.300 msecs
[  615.404050] PM: suspend-to-idle
[  625.426911] [debug] [qksleep_work_suspend_wakeup:51] qksleep is suspended, wakeup...
[  625.426961] PM: resume from suspend-to-idle
[  625.430512] PM: noirq resume of devices complete after 3.254 msecs
[  625.432153] PM: early resume of devices complete after 1.157 msecs
[  625.435256] [debug] [qksleep_resume:108] qksleep resume in
[  625.435260] PM: resume of devices complete after 3.047 msecs
[  625.436082] PM: resume devices took 0.010 seconds
[  625.439150] PM: Finishing wakeup.
[  625.439304] Restarting tasks ... done.
/sys/devices/platform/qksleep # 
```

后续的调试验证将在qksleep设备节点的基础上来做

## 调试工具

kernel的电源管理框架在“/sys/power”目录下创建了一系列供用户空间操作的sysfs节点，kernel文档“/Document/power/basic-pm-debugging.txt”中详细说明了如何将`pm_test`节点用于调试suspend/resume过程。“/Document/power/s2ram.txt”文档介绍了使用“s2ram”工具来调试和排查suspend/resume问题。stackexchange问题“How to debug a suspend problem?”的回答中介绍了“pm_utils”工具。英特尔开源社区的文章“BEST PRACTICE TO DEBUG LINUX* SUSPEND/HIBERNATE ISSUES”中系统的介绍了系统级的调试方法和遇到问题的排查步骤

总的来说，Linux电源管理的调试工具主要分为3大类：系统级调试工具(system debug tools)、PM专用调试方法(pm debug tools)和应用层开发的工具(application tools)

* system debug tools
主要是一些kernel启动参数的控制，用于增加更多打印信息
  * initcall_debug
  * no_console_suspend
  * ignore_loglevel
* pm debug tools
PM创建的sysfs节点
  * pm_test
  * pm_trace
  * pm_async
* applaction tools
结合PM sysfs编写的PM调试app
  * pm_utils
  * s2ram
  * analyze_suspend.py

### initcall_debug
通过将`initcall_debug`作为启动参数传入kernel，可以跟踪kernel的initcalls和驱动在boot、suspend和resume时的调用情况。通过这种方式可以追踪由于特定组件或驱动引起的suspend/resume问题

#### 验证
在qksleep的resume过程设置2秒的超时
```shell
/sys/devices/platform/qksleep # echo 2000 > resume_timeout 
```

手动进入休眠，系统唤醒后可看到suspend过程耗时0.010秒，resume操作耗时2.020秒
```shell
/sys/devices/platform/qksleep # echo mem > /sys/power/state
[   46.630363] PM: Syncing filesystems ... done.
[   46.632115] PM: Preparing system for mem sleep
[   46.695899] Freezing user space processes ... (elapsed 0.035 seconds) done.
[   46.732478] Freezing remaining freezable tasks ... (elapsed 0.017 seconds) done.
[   46.751186] PM: Entering mem sleep
[   46.751560] Suspending console(s) (use no_console_suspend to debug)
[   46.764225] [debug] [qksleep_suspend:89] qksleep suspend in
[   46.764237] PM: suspend of devices complete after 10.832 msecs
[   46.764522] PM: suspend devices took 0.010 seconds
[   46.766748] PM: late suspend of devices complete after 1.970 msecs
[   46.769322] PM: noirq suspend of devices complete after 2.371 msecs
[   46.769689] PM: suspend-to-idle
[   56.771522] [debug] [qksleep_work_suspend_wakeup:51] qksleep is suspended, wakeup...
[   56.771526] PM: resume from suspend-to-idle
[   56.773263] PM: noirq resume of devices complete after 1.499 msecs
[   56.775273] PM: early resume of devices complete after 1.512 msecs
[   58.788915] [debug] [qksleep_resume:108] qksleep resume in
[   58.788927] PM: resume of devices complete after 2013.258 msecs
[   58.792183] PM: resume devices took 2.020 seconds
[   58.798512] PM: Finishing wakeup.
[   58.799016] Restarting tasks ... done.
/sys/devices/platform/qksleep # 
```

虽然能看出来系统resume过程耗时明显过长，但是无法知道是在哪里耗时过长，尝试用`initcall_debug`来查看。系统启动时传入参数`initcall_debug`，开启该参数后，启动虚拟机，会打印所有initcall信息
```shell
...
Console: colour dummy device 80x30
[    0.017370] kmemleak: Kernel memory leak detector disabled
[    0.024047] Calibrating delay loop... 915.86 BogoMIPS (lpj=4579328)
[    0.097768] pid_max: default: 32768 minimum: 301
[    0.101635] Mount-cache hash table entries: 2048 (order: 1, 8192 bytes)
[    0.101702] Mountpoint-cache hash table entries: 2048 (order: 1, 8192 bytes)
[    0.133941] CPU: Testing write buffer coherency: ok
[    0.152375] CPU0: thread -1, cpu 0, socket 0, mpidr 80000000
[    0.153210] calling  trace_init_flags_sys_enter+0x0/0x28 @ 1
[    0.153285] initcall trace_init_flags_sys_enter+0x0/0x28 returned 0 after 0 usecs
[    0.153483] calling  trace_init_flags_sys_exit+0x0/0x28 @ 1
[    0.153525] initcall trace_init_flags_sys_exit+0x0/0x28 returned 0 after 0 usecs
[    0.153643] calling  cpu_suspend_alloc_sp+0x0/0x1ec @ 1
[    0.153865] initcall cpu_suspend_alloc_sp+0x0/0x1ec returned 0 after 0 usecs
[    0.153980] calling  init_static_idmap+0x0/0x74 @ 1
[    0.154138] Setting up static identity map for 0x60b26600 - 0x60b26658
[    0.154337] initcall init_static_idmap+0x0/0x74 returned 0 after 0 usecs
[    0.154453] calling  dcscb_init+0x0/0x1a8 @ 1
[    0.154738] initcall dcscb_init+0x0/0x1a8 returned -19 after 0 usecs
[    0.154848] calling  tc2_pm_init+0x0/0x250 @ 1
[    0.155097] initcall tc2_pm_init+0x0/0x250 returned -19 after 0 usecs
[    0.155207] calling  spawn_ksoftirqd+0x0/0x60 @ 1
[    0.159450] initcall spawn_ksoftirqd+0x0/0x60 returned 0 after 9765 usecs
[    0.159624] calling  init_workqueues+0x0/0x800 @ 1
[    0.170386] initcall init_workqueues+0x0/0x800 returned 0 after 9765 usecs
[    0.170520] calling  migration_init+0x0/0xa4 @ 1
[    0.170714] initcall migration_init+0x0/0xa4 returned 0 after 0 usecs
[    0.170830] calling  check_cpu_stall_init+0x0/0x24 @ 1
[    0.170876] initcall check_cpu_stall_init+0x0/0x24 returned 0 after 0 usecs
[    0.170985] calling  rcu_spawn_gp_kthread+0x0/0x1f0 @ 1
[    0.172303] initcall rcu_spawn_gp_kthread+0x0/0x1f0 returned 0 after 0 usecs
...
```

设置resume超时2秒，手动进入休眠
```shell
/sys/devices/platform/qksleep # echo 2000 > resume_timeout 
/sys/devices/platform/qksleep # echo mem > /sys/power/state
[   30.075565] PM: Syncing filesystems ... done.
[   30.077755] PM: Preparing system for mem sleep
[   30.128189] Freezing user space processes ... (elapsed 0.019 seconds) done.
[   30.149985] Freezing remaining freezable tasks ... (elapsed 0.007 seconds) done.
[   30.158478] PM: Entering mem sleep
[   30.158851] Suspending console(s) (use no_console_suspend to debug)
[   30.161310] calling  input2+ @ 768, parent: serio1
[   30.163481] call input2+ returned 0 after 2011 usecs
[   30.163599] calling  oprofile-perf.0+ @ 768, parent: platform
[   30.163768] call oprofile-perf.0+ returned 0 after 144 usecs
...
[   30.168789] calling  qksleep+ @ 768, parent: platform
[   30.169022] [debug] [qksleep_suspend:89] qksleep suspend in
[   30.169027] call qksleep+ returned 0 after 202 usecs
[   30.169428] ...
[   30.175275] PM: suspend of devices complete after 14.735 msecs
[   30.175416] PM: suspend devices took 0.010 seconds
[   30.178217] PM: late suspend of devices complete after 2.592 msecs
[   30.180898] PM: noirq suspend of devices complete after 2.424 msecs
[   30.181213] PM: suspend-to-idle
[   40.260326] [debug] [qksleep_work_suspend_wakeup:51] qksleep is suspended, wakeup...
[   40.260340] PM: resume from suspend-to-idle
[   40.263683] PM: noirq resume of devices complete after 2.594 msecs
[   40.265601] PM: early resume of devices complete after 1.461 msecs
[   40.266288] calling  reg-dummy+ @ 768, parent: platform
[   40.266395] call reg-dummy+ returned 0 after 84 usecs
[   40.266438] calling  10000000.sysreg+ @ 768, parent: platform
[   40.266455] call 10000000.sysreg+ returned 0 after 1 usecs
[   40.266487] calling  syscon.0.auto+ @ 768, parent: 10000000.sysreg
...
[   40.271348] calling  qksleep+ @ 768, parent: platform
[   42.269188] [debug] [qksleep_resume:108] qksleep resume in
[   42.269209] call qksleep+ returned 0 after 1950845 usecs
...
[   42.282800] PM: resume of devices complete after 2016.935 msecs
[   42.286160] PM: resume devices took 2.020 seconds
[   42.389269] PM: Finishing wakeup.
[   42.390769] Restarting tasks ... done.
/sys/devices/platform/qksleep # 

```

明显可以看出来在qksleep设备的resume过程耗时1950845微秒
```shell
[   42.269209] call qksleep+ returned 0 after 1950845 usecs
```

### no_console_suspend
默认的kernel休眠过程中会禁用console，因此suspend的任何打印只有在系统唤醒以后才能看到，对于suspend这部分的执行过程，kernel是不会输出的。而开启`no_console_suspend`选项，可以让kernel进入suspend的过程，仍然输出打印

no_console_suspend的控制位于/kernel/power/suspend.c和/kernel/printk.c文件中，函数`console_suspend_disable`用于处理kernel参数no_console_suspend，当设置了该参数后，全局变量`console_suspend_enabled`的值为false
```c
static int __init console_suspend_disable(char *str)
{
	console_suspend_enabled = false;
	return 1;
}
__setup("no_console_suspend", console_suspend_disable);
```

kernel休眠时会调用`suspend_devices_and_enter`来执行设备的休眠流程，在其中会调用`suspend_console`来执行console的suspend
```c
/**
 * suspend_console - suspend the console subsystem
 *
 * This disables printk() while we go into suspend states
 */
void suspend_console(void)
{
	if (!console_suspend_enabled)
		return;
	printk("Suspending console(s) (use no_console_suspend to debug)\n");
	console_lock();
	console_suspended = 1;
	up_console_sem();
}
```

如果`console_suspend_enabled`为真，则会执行下面的锁定console操作，此后所有`printk`的打印信息不会立刻打印到终端；如果为否，直接退出，此后`printk`函数的打印信息不受影响

`no_console_suspend`参数比较适用于要跟踪调试kernel代码的情况，且适用于suspend后唤醒不正常的情况

#### 验证
qksleep的resume设置模拟死锁，并分别在带`no_console_suspend`参数和不带参数情况下调试

qksleep的resume设置死锁
```shell
/sys/devices/platform/qksleep # echo 1 > resume_errlock
```

不带`no_console_suspend`情况下，手动进入休眠，打印在“Suspending console(s) (use no_console_suspend to debug)”处停止
```shell
/sys/devices/platform/qksleep # echo mem > /sys/power/state
[  160.900817] PM: Syncing filesystems ... done.
[  160.902427] PM: Preparing system for mem sleep
[  160.962134] Freezing user space processes ... (elapsed 0.036 seconds) done.
[  161.000256] Freezing remaining freezable tasks ... (elapsed 0.018 seconds) done.
[  161.020661] PM: Entering mem sleep
[  161.021063] Suspending console(s) (use no_console_suspend to debug)
```

带`no_console_suspend`的情况下，手动进入休眠，死锁前的打印都能看到
```shell
/sys/devices/platform/qksleep # echo mem > /sys/power/state
[   41.220243] PM: Syncing filesystems ... done.
[   41.222718] PM: Preparing system for mem sleep
[   41.274511] Freezing user space processes ... (elapsed 0.026 seconds) done.
[   41.301588] Freezing remaining freezable tasks ... (elapsed 0.011 seconds) done.
[   41.313599] PM: Entering mem sleep
[   41.322571] [debug] [qksleep_suspend:89] qksleep suspend in
[   41.326340] PM: suspend of devices complete after 11.215 msecs
[   41.326920] PM: suspend devices took 0.010 seconds
[   41.330000] PM: late suspend of devices complete after 2.664 msecs
[   41.332451] PM: noirq suspend of devices complete after 2.001 msecs
[   41.333041] PM: suspend-to-idle
[   51.341481] [debug] [qksleep_work_suspend_wakeup:51] qksleep is suspended, wakeup...
[   51.344717] PM: resume from suspend-to-idle
[   51.347243] PM: noirq resume of devices complete after 1.331 msecs
[   51.349427] PM: early resume of devices complete after 1.406 msecs
[   51.352296] [debug] [qksleep_resume:108] qksleep resume in
```

### ignore_loglevel
开启`ignore_loglevel`参数后，kernel的打印会无视log级别限制，所有打印都能看到，适用于代码中有很多log级别区分的调试

### pm_test
该功能依赖kernel配置宏`CONFIG_PM_DEBUG`，在make menuconfig中配置路径为
```shell
make menuconfig
  --> Power management options
    [*] Power Management Debug Support
```

开启此配置后，可通过写入/sys/power/pm_test节点让PM core以测试模式运行，测试模式有5个级别，对应于不同深度的休眠
* freezer：测试进程冻结
* devices：测试进程冻结和设备suspend
* platform：测试进程冻结、设备suspend、平台架构相关suspend
* processors：测试进程冻结、设备suspend、平台架构相关suspend、禁用非引导CPU
* core：测试进程冻结、设备suspend、平台架构相关suspend、禁用非引导CPU、系统suspend

这5个级别由浅入深，正好对应kernel的休眠流程，kernel在不同地方都设置了测试点`suspend_test`，当设置了测试模式后，对应级别的测试点会delay 5秒钟然后唤醒系统
```c
static int suspend_test(int level)
{
#ifdef CONFIG_PM_DEBUG
	if (pm_test_level == level) {
		printk(KERN_INFO "suspend debug: Waiting for 5 seconds.\n");
		mdelay(5000);
		return 1;
	}
#endif /* !CONFIG_PM_DEBUG */
	return 0;
}
```

使用这种方法能够在不添加额外唤醒源的情况下测试suspend/resume的完整流程，且能够分阶段进行调试

#### 验证
禁用qksleep原本的suspend唤醒
```shell
/sys/devices/platform/qksleep # echo 0 > suspend_wakeup_timeout
```

设置PM core调试模式为devices
```shell
/sys/devices/platform/qksleep # echo devices > /sys/power/pm_test
```

设置qksleep设备resume返回错误值-1
```shell
/sys/devices/platform/qksleep # echo -1 > resume_ret
```

手动进入休眠，可看到kernel在5秒后自动唤醒了
```shell
/sys/devices/platform/qksleep # echo mem > /sys/power/state
[  229.676306] PM: Syncing filesystems ... done.
[  229.676943] PM: Preparing system for mem sleep
[  229.701077] Freezing user space processes ... (elapsed 0.013 seconds) done.
[  229.716141] Freezing remaining freezable tasks ... (elapsed 0.008 seconds) done.
[  229.725218] PM: Entering mem sleep
[  229.725415] Suspending console(s) (use no_console_suspend to debug)
[  229.730420] [debug] [qksleep_suspend:89] qksleep suspend in
[  229.730427] PM: suspend of devices complete after 4.232 msecs
[  229.730448] PM: suspend devices took 0.000 seconds
[  229.730459] suspend debug: Waiting for 5 seconds.
[  234.941168] [debug] [qksleep_resume:108] qksleep resume in
[  234.941173] dpm_run_callback(): platform_pm_resume+0x0/0x90 returns -1
[  234.941196] PM: Device qksleep failed to resume: error -1
[  234.942311] PM: resume of devices complete after 2.948 msecs
[  234.943057] PM: resume devices took 0.000 seconds
[  234.945443] PM: Finishing wakeup.
[  234.945607] Restarting tasks ... done.
/sys/devices/platform/qksleep # 
```
