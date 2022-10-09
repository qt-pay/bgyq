## CPU soft lookup--draft

### what is soft lookup

A soft lockup is a situation usually caused by a bug, when a task is executing in kernel space on a CPU without rescheduling. The task also does not allow any other task to execute on that particular CPU. As a result, a warning is displayed to a user through the system console. This problem is also referred to as the soft lockup firing.

所谓lockup，是指某段内核代码占着CPU不放。Lockup严重的情况下会导致整个系统失去响应。Lockup有几个特点：

- 首先只有内核代码才能引起lockup，因为用户代码是可以被抢占的，不可能形成lockup（只有一种情况例外，就是SCHED_FIFO优先级为99的实时进程即使在用户态也可能使[watchdog/x]内核线程抢不到CPU而形成soft lock，参见《[Real-Time进程会导致系统Lockup吗？](http://linuxperf.com/?p=197)》）
- 其次内核代码必须处于禁止内核抢占的状态(preemption disabled)，因为Linux是可抢占式的内核，只在某些特定的代码区才禁止抢占，在这些代码区才有可能形成lockup。

Lockup分为两种：soft lockup 和 hard lockup，它们的区别是 hard lockup 发生在CPU屏蔽中断的情况下。

- Soft lockup是指CPU被内核代码占据，以至于无法执行其它进程。检测soft lockup的原理是给每个CPU分配一个定时执行的内核线程[watchdog/x]，如果该线程在设定的期限内没有得到执行的话就意味着发生了soft lockup，[watchdog/x]是SCHED_FIFO实时进程，优先级为最高的99，拥有优先运行的特权。
- Hard lockup比soft lockup更加严重，CPU不仅无法执行其它进程，而且不再响应中断。检测hard lockup的原理利用了PMU的NMI perf event，因为NMI中断是不可屏蔽的，在CPU不再响应中断的情况下仍然可以得到执行，它再去检查时钟中断的计数器hrtimer_interrupts是否在保持递增，如果停滞就意味着时钟中断未得到响应，也就是发生了hard lockup。

Linux kernel设计了一个检测lockup的机制，称为**NMI Watchdog**，是利用NMI中断实现的，用NMI是因为lockup有可能发生在中断被屏蔽的状态下，这时唯一能把CPU抢下来的方法就是通过NMI，因为NMI中断是不可屏蔽的。

常见导致cpu softlockup的原因：

1、CPU高负载时间过长
2、服务器电源供电不足，导致CPU电压不稳定
3、vcpus超过物理cpu cores
4、虚机所在的宿主机的CPU太忙或磁盘IO太高
5、虚机机的CPU太忙或磁盘IO太高
6、VM网卡驱动存在bug，处理高水位流量时存在bug导致CPU死锁
7、BIOS开启了超频，导致超频时电压不稳，容易出现CPU死锁
8、Linux kernel或KVM存在bug
9、BIOS Intel C-State开启导致，关闭可解决
10、BIOS spread spectrum开启导致




### nmi_watchdog

Controls whether lockup detection mechanisms (`watchdogs`) are active or not. This parameter is of integer type.

| Value | Effect                   |
| :---- | :----------------------- |
| 0     | disables lockup detector |
| 1     | enables lockup detector  |

The hard lockup detector monitors each CPU for its ability to respond to interrupts.

### resolution soft lookup：调整检测时间阈值，尽量避免

This 'soft lockup' can happen if the kernel is busy, working on a huge amount of objects which need to be scanned, freed, or allocated, respectively.
The stack traces of those tasks can give a first idea about what the tasks were doing. However, to be able to examine the cause behind the messages, a kernel dump would be needed.

While these messages cannot be disabled entirely, in some situations, increasing the time before these

soft lockups are fired can relax the situation.

To do so, increase the following sysctl parameter: `kernel.watchdog_thresh`

Default value for this parameter is 10 and to double the value might be a good start.

 ```bash
 $ echo "kernel.watchdog_thresh=20" > /etc/sysctl.d/99-watchdog_thresh.conf
 $ sysctl -p  /etc/sysctl.d/99-watchdog_thresh.conf
 
 ```

A 'soft lockup' is defined as a bug that causes the kernel to loop in kernel mode for more than 20 seconds without giving other tasks a chance to run.
The watchdog daemon will send an non-maskable interrupt (NMI) to all CPUs in the system who, in turn, print the stack traces of their currently running tasks.

### panic softlockup on vm

现象： k8s node 一个gpu应用卡死cpu，kill进程cpu softlockup先不能消除、

解决：重启vm，so，还是需要panic softlockup

you should not enable the `softlockup_panic` and `nmi_watchdog` kernel parameters, **as the virtualized environment may trigger a spurious soft lockup that should not require a system panic.**

> 物理机上几乎不会出现 cpu soft lockup

**The soft lockup firing on physical hosts,** as described in What is a soft lockup usually represents a kernel or hardware bug. The same phenomenon happening on guest operating systems in virtualized environments may represent a false warning.

当内核发生一些死锁，系统并不会重启，只会卡死。这种情况下并不会造成内核panic，需要构造条件使内核panic，这样才能生成vmcore文件。

内核提供了多种场景构造panic：

- 进程出现挂起时引发panic

  `echo 1 > /proc/sys/kernel/hung_task_panic `

- 进程hangtask机制超时时间

  `echo 60 > /proc/sys/kernel/hung_task_timeout_secs `

- 软锁（soft lockup）触发panic

  `echo 1 > /proc/sys/kernel/softlockup_panic `

- 设置Kernel遇到OOM触发panic

  `echo 1 > /proc/sys/vm/panic_on_oom `

- 设置内核中出现警告产生panic

  `echo 1 > /proc/sys/kernel/panic_on_warn `

通过以上命令修改的相关配置为临时生效，重启后失效。如需要重启自动配置，可按照以下步骤进行操作。

1. 编辑“etc/sysctl.conf”文件。

   `vim etc/sysctl.conf `

   写入如下参数：

   ```bash
   kernel.hung_task_panic=1
   kernel.hung_task_timeout_secs=60
   kernel.softlockup_panic=1
   vm.panic_on_oom=1
   kernel.panic_on_warn=1
   ```

2. 使配置生效。

   `sysctl -p `

### 脑洞

可以使用eBPF起个hook打印soft lookup的信息么

### 引用：

1. http://linuxperf.com/?p=83
2. https://www.suse.com/support/kb/doc/?id=000018705
3. https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/managing_monitoring_and_updating_the_kernel/keeping-kernel-panic-parameters-disabled-in-virtualized-environments_managing-monitoring-and-updating-the-kernel
4. https://support.huaweicloud.com/ref-kunpenggrf/kunpenggrffaq_10_0008.html
5. https://blog.51cto.com/StarBigData/3643834