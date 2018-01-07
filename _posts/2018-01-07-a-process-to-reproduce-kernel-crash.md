---
layout: post
title:  "一次重现内核 Bug 的经历"
date:   2018-01-07 22:30:40 +0800
categories: kernel cgroup
---

## 现象

在一次平台升级过程中, 为了增强 Hadoop 平台资源隔离特性, 启用了 CGroup 的 CPU 隔离机制. 由于这次升级, 隔天就导致每天都有十多台服务器重启. 我需要定位出导致服务器重启的真正原因, 并给出解决方案. 

同事提到这个功能之前上过一段时间, 并没有出现重启. 这段时间内核, Docker 以及 Hadoop 中 CGroup 隔离功能都未更改. 只是重启的服务器都是最近半年新上的服务器, 这些服务器在开启 CGroup 时一直都是提供给某部门使用.  这些线索反倒让我无从下手, 只能从下面的过程一步步搜寻 Bug 的根源:

1. 分析系统日志目录 `/var/log/`
2. 对比系统参数 `sysctl -a` 和内核版本 `uname -r`
3. 分析崩溃时刻的 YARN 日志
4. 分析 `/sys/fs/cgroup` 目录下 CGroup 的参数
5. 分析内核崩溃时留下的 vmcore

通过 vmcore 给出的线索, 在网上搜索到相关的主题, 定位到重启确实是由于内核的一个 Bug 导致, 但是我需要找出是谁触发了这个 Bug. 之前从未有 vmcore/kernel 方面的知识储备, 这着实让人有点抓狂. 但只能对着分析 vmcore 的工具 crash 的文档漫无目的的尝试, 最后还真让我找到了一个线索, 定位到了触发重启的罪魁祸首.

### 环境

重启的服务器的内核, 重启原因基本一致, 都是由于 Watchdog 检测到 hard LOCKUP.

内核: `3.10.0-327.28.3.el7.x86_64`

内核崩溃的原因:

```
crash> sys
      KERNEL: /usr/lib/debug/lib/modules/3.10.0-327.28.3.el7.x86_64/vmlinux
    DUMPFILE: vmcore  [PARTIAL DUMP]
        CPUS: 48
        DATE: Thu Dec  7 02:59:00 2017
      UPTIME: 15 days, 18:17:54
LOAD AVERAGE: 12.51, 9.27, 10.21
       TASKS: 2101
    NODENAME: XXXXXX-YYY-ZZZ.SUFFIX
     RELEASE: 3.10.0-327.28.3.el7.x86_64
     VERSION: #1 SMP Thu Aug 18 19:05:49 UTC 2016
     MACHINE: x86_64  (2194 Mhz)
      MEMORY: 255.9 GB
       PANIC: "Kernel panic - not syncing: Watchdog detected hard LOCKUP on cpu 34"
         PID: 0
     COMMAND: "swapper/34"
        TASK: ffff881fd2e65c00  (1 of 48)  [THREAD_INFO: ffff881fd2e90000]
         CPU: 34
       STATE: TASK_RUNNING (PANIC)
```

内核崩溃时的堆栈:

```
crash> bt
PID: 0      TASK: ffff881fd2e65c00  CPU: 34  COMMAND: "swapper/34"
 #0 [ffff881fffd859c8] machine_kexec at ffffffff81051e9b
 #1 [ffff881fffd85a28] crash_kexec at ffffffff810f27c2
 #2 [ffff881fffd85af8] panic at ffffffff8162fcee
 #3 [ffff881fffd85b78] watchdog_overflow_callback at ffffffff8111b9b2
 #4 [ffff881fffd85b88] __perf_event_overflow at ffffffff8115f201
 #5 [ffff881fffd85c00] perf_event_overflow at ffffffff8115fcd4
 #6 [ffff881fffd85c10] intel_pmu_handle_irq at ffffffff810325d8
 #7 [ffff881fffd85e60] perf_event_nmi_handler at ffffffff8163fe8b
 #8 [ffff881fffd85e80] nmi_handle at ffffffff8163f5d9
 #9 [ffff881fffd85ec8] do_nmi at ffffffff8163f6f0
#10 [ffff881fffd85ef0] end_repeat_nmi at ffffffff8163ea13
    [exception RIP: enqueue_entity+257]
    RIP: ffffffff810c34c1  RSP: ffff881fffd83de0  RFLAGS: 00000016
    RAX: 0004d7dfb754ffa1  RBX: ffff883fcc8fc3c0  RCX: ffff883fcb8f7e00
    RDX: ffff883fff596840  RSI: ffff883fcc8fc3c0  RDI: 0000000000000000
    RBP: ffff881fffd83e18   R8: 0000000000000000   R9: 0000000000000001
    R10: 0000000000000000  R11: 0000000000000000  R12: ffff883fff596840
    R13: ffff883fcda92400  R14: 0000000000008444  R15: 0000000000000000
    ORIG_RAX: ffffffffffffffff  CS: 0010  SS: 0018
--- <NMI exception stack> ---
#11 [ffff881fffd83de0] enqueue_entity at ffffffff810c34c1
#12 [ffff881fffd83e20] unthrottle_cfs_rq at ffffffff810c4414
#13 [ffff881fffd83e58] distribute_cfs_runtime at ffffffff810c4662
#14 [ffff881fffd83ea0] sched_cfs_period_timer at ffffffff810c4847
#15 [ffff881fffd83ed8] __hrtimer_run_queues at ffffffff810a9d82
#16 [ffff881fffd83f30] hrtimer_interrupt at ffffffff810aa320
#17 [ffff881fffd83f80] local_apic_timer_interrupt at ffffffff810495c7
#18 [ffff881fffd83f98] smp_apic_timer_interrupt at ffffffff816490cf
#19 [ffff881fffd83fb0] apic_timer_interrupt at ffffffff8164779d
--- <IRQ stack> ---
#20 [ffff881fd2e93de8] apic_timer_interrupt at ffffffff8164779d
    [exception RIP: native_safe_halt+6]
    RIP: ffffffff81058e96  RSP: ffff881fd2e93e98  RFLAGS: 00000286
    RAX: 00000000ffffffed  RBX: ffff881fffd8cf00  RCX: 0100000000000000
    RDX: 0000000000000000  RSI: 0000000000000000  RDI: 0000000000000046
    RBP: ffff881fd2e93e98   R8: 0000000000000000   R9: 0000000000000000
    R10: 0000000000000000  R11: 0000000000000000  R12: 0004d69c69cb0e40
    R13: ffff881fffd8fbc0  R14: e343a8a80388bb48  R15: 0000000000000086
    ORIG_RAX: ffffffffffffff10  CS: 0010  SS: 0018
#21 [ffff881fd2e93ea0] default_idle at ffffffff8101dbff
#22 [ffff881fd2e93ec0] arch_cpu_idle at ffffffff8101e506
#23 [ffff881fd2e93ed0] cpu_startup_entry at ffffffff810d6485
#24 [ffff881fd2e93f28] start_secondary at ffffffff8104768a
```

## 目的

我需要定位出导致服务器重启的真正原因, 并能重现重启现象, 并给出解决方案,  确保重启不会再重现.

## 分析过程

1. ### 分析系统日志目录 `/var/log/`
   通过分析所有重启服务器 `/var/log/` 目录下的日志文件, 没有任何记录表明重启时的异常. 这就让我一下子陷入了懵逼状态, 好在后来运维组的同事告诉我部分服务器 `/var/crash/` 目录下生成了 vmcore.


2. ### 对比系统参数 `sysctl -a` 和内核版本 `uname -r`

   通过抽样集群里面正常和异常的服务器的配置参数和内核版本, 也没有发生异样.

3. ### 分析崩溃时刻的 YARN 日志

   这些服务器在启用 CGroup 前后一直都是提供给某部门使用, 按说应该考虑用他们的程序来尝试重现. 在分析了 YARN 日志后, 没有发现与应用程序相关的有用价值:

   * 应用程序使用的内存没有超限.
   * ganglia 显示重启时刻整体的 CPU 利用率不高, 系统调用最高也只有 5%, 相比那个时刻的用户态 CPU 15-35% 并没有异常
   * 其它各项系统指标(net/disk)也没有异常.

   鉴于上面的原因, 加上重启时刻不止一个运行中的应用程序, 故没有尝试使用他们的应用程序进行测试. 期间在 Apache JIRA 上搜索到 [CGroup 控制组的数量可能是罪魁祸首][1]. 后来尝试在 CPU 子系统下手动创建大量控制组数量, 并没有导致系统崩溃.

   注: 在此期间注意到另一个问题, 我们的集群具备启动 Docker 容器, Docker 在 `/sys/fs/cgroup/\*/hadoop-yarn` 控制组下面会创建大量子控制组, 我们在删除容器时, 没有处理除 CPU 子系统外的其它系统(memory/device..., 他们的目录下会存在上千个子控制组, 后来在重启服务器上测试, 他们也不会系统的正常运行.

4. ### 分析 `/sys/fs/cgroup` 目录下 CGroup 的参数

   应用程序的 CGroup 参数由 Hadoop YARN 项目下的 container-executor 程序控制. 我做了三个尝试:

   * 编写了一个多线程死循环 Spark 应用, 消耗掉所有重启服务器的全部 CPU 资源
   * 编写了一个多线程死循环 Spark 应用,  其中一个线程循环调用系统调用(打开/写入/关闭文件), 消耗掉所有重启服务器的全部 CPU 资源
   * 用 C++ 编写了一个多线程死循环程序, 参考 [如何使用 CGroup][2] 和 [Cgroup – Linux的网络资源隔离][3] 模拟崩溃时的场景

   无论如何, 模拟程序都不会导致服务器重现相同的重启表现.

   在这个过程中发现了另一个问题, 对于我们的内核版本, 如果在 `/sys/fs/cgroup/cpu/hadoop-yarn` 目录限制 `cpu.cfs_quota_us` (非-1)会导致服务器重启, 是[内核空指针异常][4], 内核升级到 `3.10.0-693.11.1.el7.x86_64` 解决.

5. ### 分析内核崩溃时留下的 vmcore
   多亏同事的帮助, 发现部分服务器在重启前生成了 vmcore. 参考[一份kernel dump分析报告][5], 分析我们的 vmcore, 通过堆栈信息匹配到 [a panic of watchdog hard lockup][6], 这篇文章已经明确指出了这是个内核 Bug, 我还是需要重现重启, 不然即使升级内核, 也不能确定将来还会不会随机出现服务器重启.

   通过[内核如何检测SOFT LOCKUP与HARD LOCKUP][7]了解软硬 LOCKUP, 又通过 [System panics due to an NMI hard lockup in RHEL 7][8] 了解 watchdog_timer_fn 和 watchdog_overflow_callback 的工作机制. 了解了这些原理后, 分析我这个问题的 vmcore 就不感觉一头雾水了, 但是这些还是不能给出任何线索.

   在毫无头绪的情况下, 我还是坚信从 vmcore 里面一定可以找到问题根源(其实也是没有其它办法了), 借助 crash 里面的帮助文档(这个文档写的真的太好了, 很适合初次接触 vmcore 的同学), 一个一个命令试, 期待能发现什么问题. 真的是皇天不负有心人, 最后我观察到 `runq -g` 的输出有一个共性:

   ```
   crash> runq -g
   CPU 0
     CURRENT: PID: 0      TASK: ffffffff81951440  COMMAND: "swapper/0"
     ROOT_TASK_GROUP: ffffffff81db4080  RT_RQ: ffff881fff816950
        [no tasks queued]
     ROOT_TASK_GROUP: ffffffff81db4080  CFS_RQ: ffff881fff816840
        TASK_GROUP: ffff883fd2b62800  CFS_RQ: ffff881fc2d48800  <hadoop-yarn>
           TASK_GROUP: ffff881d9e74c000  CFS_RQ: ffff882daa3dc000  <container_e06_1512098667219_988894_01_000020> (THROTTLED)
              TASK_GROUP: ffff881fd268a800  CFS_RQ: ffff882083fb8000  <5fa97be56cd60433d6e21586b995303ce3d518afb3b85b64e902818d3a0e2171>
                 [120] PID: 46235  TASK: ffff8833d13a1700  COMMAND: "python"
                 [120] PID: 46264  TASK: ffff8837363a0b80  COMMAND: "python"
   ... # 省略部分 CPU 信息
   CPU 7
     CURRENT: PID: 0      TASK: ffff881fd2de2280  COMMAND: "swapper/7"
     ROOT_TASK_GROUP: ffffffff81db4080  RT_RQ: ffff881fff9d6950
        [no tasks queued]
     ROOT_TASK_GROUP: ffffffff81db4080  CFS_RQ: ffff881fff9d6840
        TASK_GROUP: ffff883fd2b62800  CFS_RQ: ffff881fc2d49600  <hadoop-yarn>
           TASK_GROUP: ffff881d9e74c000  CFS_RQ: ffff88360931c000  <container_e06_1512098667219_988894_01_000020> (THROTTLED)
              TASK_GROUP: ffff881fd268a800  CFS_RQ: ffff882e0c494000  <5fa97be56cd60433d6e21586b995303ce3d518afb3b85b64e902818d3a0e2171>
                 [120] PID: 46266  TASK: ffff8837363a5080  COMMAND: "python"
   ... # 省略部分 CPU 信息
   CPU 34
     CURRENT: PID: 0      TASK: ffff881fd2e65c00  COMMAND: "swapper/34"
     ROOT_TASK_GROUP: ffffffff81db4080  RT_RQ: ffff881fffd96950
        [no tasks queued]
     ROOT_TASK_GROUP: ffffffff81db4080  CFS_RQ: ffff881fffd96840
        [120] PID: 15215  TASK: ffff88001d4cb980  COMMAND: "kworker/34:1"
   ... # 省略部分 CPU 信息
   ```

   回到上文的[环境](#环境)部分, 内核 panic 发生在 34 号 CPU, 但是该 CPU 没有任何用户态进程, 而其它 CPU 上的运行队列里面都指向了同一个 YARN 容器 container_e06_1512098667219_988894_01_000020, 该容器运行在 CGroup 机制下(hadoop-yarn), 并且基本处于 THROTTLED. 这个信息让我如获珍宝, 我又对其它几个 vmcore 进行了类似分析, 其中有几台服务器的 CPU 的运行队列里面也是被 THROTTLED 在同一个 YARN 应用(application_**1512098667219_988894**). 

   反向查找到这个应用的源码和作者, 了解到这是一个机器学习应用, 里面用到了 prophet 库(目前只是怀疑). 我们把这个应用跑在之前重启的服务器上, 100% 出现重启, 把内核升级到 `3.10.0-693.11.1.el7.x86_64` 解决.

   至此, 这个问题的根源已经明确, 是一个内核 Bug, 触发该 Bug 的罪魁祸首也落网了, 至于为什么这个应用程序一定导致内核崩溃还未知, 但之前上 CGroup 隔离没有服务器重启可能是那时候还没有相关的机器学习应用.


## 附录

在跟踪问题过程中, 用到了一些命令, 在此也记录下

* 监视 /sys/fs/cgroup: `inotifywait -r -m --timefmt '%H:%M:%S' --format '%T %w%f %e' /sys/fs/cgroup`
* 取特定进程 `ps uax | grep  thread | grep -vP \(kthreadd\|color\) | cut -c 10-14 | xargs -I {} echo -n {},` 复制输出, 但不要最后的逗号, 作为 PID_LIST, 只观察特定进程 `top -d 1 -o COMMAND -p $PID_LIST`
* 对比各个 CPU 运行队列的时间: `runq -t | grep CPU | sort -k3r | awk 'NR==1{now=strtonum("0x"$3)}1{printf"%s\t%7.2fs behind\n",$0,(now-strtonum("0x"$3))/1000000000}'`

## 参考

[1]: https://issues.apache.org/jira/browse/YARN-4048?focusedCommentId=16172195&amp;amp;page=com.atlassian.jira.plugin.system.issuetabpanels:comment-tabpanel#comment-16172195	"CGroup 控制组的数量可能是罪魁祸首"
[2]: http://tiewei.github.io/devops/howto-use-cgroup/	"how to use cgroup"
[3]: http://liwei.life/2016/02/05/cgroup_linux_network_traffic_control/	"Cgroup – Linux的网络资源隔离"
[4]: https://bugs.centos.org/view.php?id=13731	"内核空指针异常"
[5]: http://mp.weixin.qq.com/s/1AbiZwkJteYe9ayo0eZTUw "一份kernel dump分析报告"
[6]: http://linux323.tk/a-panice-of-watchdog/ "a panic of watchdog hard lockup"
[7]: http://linuxperf.com/?p=83 "内核如何检测SOFT LOCKUP与HARD LOCKUP?"
[8]: https://access.redhat.com/solutions/1354963 "System panics due to an NMI hard lockup in RHEL 7"
[10]: https://en.wikipedia.org/wiki/FLAGS_register "RFLAGS"
[11]: https://en.wikipedia.org/wiki/Interrupt_handler "ISR"
[12]: https://people.redhat.com/anderson/crash_whitepaper/ "White Paper: Red Hat Crash Utility"