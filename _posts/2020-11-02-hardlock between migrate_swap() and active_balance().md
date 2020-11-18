# 关于migrate_swap() 和 active_balance()之间的hardlock
## 背景：这个是在3.10.0-957.el7.x86_64 遇到的一例crash
### 下面列一下我们是怎么排查并解这个问题的。

#### 一、故障现象

Oppo云智能监控发现机器down机：
```
 KERNEL: /usr/lib/debug/lib/modules/3.10.0-957.el7.x86_64/vmlinux 
 ....
       PANIC: "Kernel panic - not syncing: Hard LOCKUP"
         PID: 14
     COMMAND: "migration/1"
        TASK: ffff8f1bf6bb9040  [THREAD_INFO: ffff8f1bf6bc4000]
         CPU: 1
       STATE: TASK_INTERRUPTIBLE (PANIC)

crash> bt
PID: 14     TASK: ffff8f1bf6bb9040  CPU: 1   COMMAND: "migration/1"
 #0 [ffff8f4afbe089f0] machine_kexec at ffffffff83863674
 #1 [ffff8f4afbe08a50] __crash_kexec at ffffffff8391ce12
 #2 [ffff8f4afbe08b20] panic at ffffffff83f5b4db
 #3 [ffff8f4afbe08ba0] nmi_panic at ffffffff8389739f
 #4 [ffff8f4afbe08bb0] watchdog_overflow_callback at ffffffff83949241
 #5 [ffff8f4afbe08bc8] __perf_event_overflow at ffffffff839a1027
 #6 [ffff8f4afbe08c00] perf_event_overflow at ffffffff839aa694
 #7 [ffff8f4afbe08c10] intel_pmu_handle_irq at ffffffff8380a6b0
 #8 [ffff8f4afbe08e38] perf_event_nmi_handler at ffffffff83f6b031
 #9 [ffff8f4afbe08e58] nmi_handle at ffffffff83f6c8fc
#10 [ffff8f4afbe08eb0] do_nmi at ffffffff83f6cbd8
#11 [ffff8f4afbe08ef0] end_repeat_nmi at ffffffff83f6bd69
    [exception RIP: native_queued_spin_lock_slowpath+462]
    RIP: ffffffff839121ae  RSP: ffff8f1bf6bc7c50  RFLAGS: 00000002
    RAX: 0000000000000001  RBX: 0000000000000082  RCX: 0000000000000001
    RDX: 0000000000000101  RSI: 0000000000000001  RDI: ffff8f1afdf55fe8---锁
    RBP: ffff8f1bf6bc7c50   R8: 0000000000000101   R9: 0000000000000400
    R10: 000000000000499e  R11: 000000000000499f  R12: ffff8f1afdf55fe8
    R13: ffff8f1bf5150000  R14: ffff8f1afdf5b488  R15: ffff8f1bf5187818
    ORIG_RAX: ffffffffffffffff  CS: 0010  SS: 0018
--- <NMI exception stack> ---
#12 [ffff8f1bf6bc7c50] native_queued_spin_lock_slowpath at ffffffff839121ae
#13 [ffff8f1bf6bc7c58] queued_spin_lock_slowpath at ffffffff83f5bf4b
#14 [ffff8f1bf6bc7c68] _raw_spin_lock_irqsave at ffffffff83f6a487
#15 [ffff8f1bf6bc7c80] cpu_stop_queue_work at ffffffff8392fc70
#16 [ffff8f1bf6bc7cb0] stop_one_cpu_nowait at ffffffff83930450
#17 [ffff8f1bf6bc7cc0] load_balance at ffffffff838e4c6e
#18 [ffff8f1bf6bc7da8] idle_balance at ffffffff838e5451
#19 [ffff8f1bf6bc7e00] __schedule at ffffffff83f67b14
#20 [ffff8f1bf6bc7e88] schedule at ffffffff83f67bc9
#21 [ffff8f1bf6bc7e98] smpboot_thread_fn at ffffffff838ca562
#22 [ffff8f1bf6bc7ec8] kthread at ffffffff838c1c31
#23 [ffff8f1bf6bc7f50] ret_from_fork_nospec_begin at ffffffff83f74c1d
crash> 
```

#### 二、故障现象分析

hardlock一般是由于关中断时间过长，从堆栈看，上面的"migration/1" 进程在抢spinlock，
由于_raw_spin_lock_irqsave 会先调用 arch_local_irq_disable,然后再去拿锁，而
arch_local_irq_disable 是常见的关中断函数,下面分析这个进程想要拿的锁被谁拿着。
```
x86架构下，native_queued_spin_lock_slowpath的rdi就是存放锁地址的

crash> arch_spinlock_t ffff8f1afdf55fe8
struct arch_spinlock_t {
  val = {
    counter = 257
  }
}
```
下面，我们需要了解，这个是一把什么锁。
从调用链分析 idle_balance-->load_balance-->stop_one_cpu_nowait-->cpu_stop_queue_work
反汇编 cpu_stop_queue_work 拿锁阻塞的代码：
```
crash> dis -l ffffffff8392fc70
/usr/src/debug/kernel-3.10.0-957.el7/linux-3.10.0-957.el7.x86_64/kernel/stop_machine.c: 91
0xffffffff8392fc70 <cpu_stop_queue_work+48>:    cmpb   $0x0,0xc(%rbx)

     85 static void cpu_stop_queue_work(unsigned int cpu, struct cpu_stop_work *work)
     86 {
     87         struct cpu_stopper *stopper = &per_cpu(cpu_stopper, cpu);
     88         unsigned long flags;
     89 
     90         spin_lock_irqsave(&stopper->lock, flags);---所以是卡在拿这把锁
     91         if (stopper->enabled)
     92                 __cpu_stop_queue_work(stopper, work);
     93         else
     94                 cpu_stop_signal_done(work->done, false);
     95         spin_unlock_irqrestore(&stopper->lock, flags);
     96 }
```
看起来 需要根据cpu号，来获取对应的percpu变量 cpu_stopper,这个入参在 load_balance 函数中找到的
最忙的rq，然后获取其对应的cpu号:
```
   6545 static int load_balance(int this_cpu, struct rq *this_rq,
   6546                         struct sched_domain *sd, enum cpu_idle_type idle,
   6547                         int *should_balance)
   6548 {
....
   6735                         if (active_balance) {
   6736                                 stop_one_cpu_nowait(cpu_of(busiest),
   6737                                         active_load_balance_cpu_stop, busiest,
   6738                                         &busiest->active_balance_work);
   6739                         }
....
  6781 }

  crash> dis -l load_balance |grep stop_one_cpu_nowait -B 6
0xffffffff838e4c4d <load_balance+2045>:	callq  0xffffffff83f6a0e0 <_raw_spin_unlock_irqrestore>
/usr/src/debug/kernel-3.10.0-957.el7/linux-3.10.0-957.el7.x86_64/kernel/sched/fair.c: 6736
0xffffffff838e4c52 <load_balance+2050>:	mov    0x930(%rbx),%edi------------根据rbx可以取cpu号，rbx就是最忙的rq
0xffffffff838e4c58 <load_balance+2056>:	lea    0x908(%rbx),%rcx
0xffffffff838e4c5f <load_balance+2063>:	mov    %rbx,%rdx
0xffffffff838e4c62 <load_balance+2066>:	mov    $0xffffffff838de690,%rsi
0xffffffff838e4c69 <load_balance+2073>:	callq  0xffffffff83930420 <stop_one_cpu_nowait>
```

然后我们再栈中取的数据如下：
```
最忙的组是：
crash> rq.cpu ffff8f1afdf5ab80
  cpu = 26
```
也就是说，1号cpu在等 percpu变量cpu_stopper 的26号cpu的锁。

然后我们搜索这把锁在其他哪个进程的栈中，找到了如下：
```
ffff8f4957fbfab0: ffff8f1afdf55fe8 --------这个在  355608 的栈中
crash> kmem ffff8f4957fbfab0
    PID: 355608
COMMAND: "custom_exporter"
   TASK: ffff8f4aea3a8000  [THREAD_INFO: ffff8f4957fbc000]
    CPU: 26--------刚好也是运行在26号cpu的进程
  STATE: TASK_RUNNING (ACTIVE)
```

下面，就需要分析，为什么位于26号cpu的进程 custom_exporter 会长时间拿着 ffff8f1afdf55fe8 

我们来分析26号cpu的堆栈：

```
crash> bt -f 355608
PID: 355608  TASK: ffff8f4aea3a8000  CPU: 26  COMMAND: "custom_exporter"
.....
 #3 [ffff8f1afdf48ef0] end_repeat_nmi at ffffffff83f6bd69
    [exception RIP: try_to_wake_up+114]
    RIP: ffffffff838d63d2  RSP: ffff8f4957fbfa30  RFLAGS: 00000002
    RAX: 0000000000000001  RBX: ffff8f1bf6bb9844  RCX: 0000000000000000
    RDX: 0000000000000001  RSI: 0000000000000003  RDI: ffff8f1bf6bb9844
    RBP: ffff8f4957fbfa70   R8: ffff8f4afbe15ff0   R9: 0000000000000000
    R10: 0000000000000000  R11: 0000000000000000  R12: 0000000000000000
    R13: ffff8f1bf6bb9040  R14: 0000000000000000  R15: 0000000000000003
    ORIG_RAX: ffffffffffffffff  CS: 0010  SS: 0000
--- <NMI exception stack> ---
 #4 [ffff8f4957fbfa30] try_to_wake_up at ffffffff838d63d2
    ffff8f4957fbfa38: 000000000001ab80 0000000000000086 
    ffff8f4957fbfa48: ffff8f4afbe15fe0 ffff8f4957fbfb48 
    ffff8f4957fbfa58: 0000000000000001 ffff8f4afbe15fe0 
    ffff8f4957fbfa68: ffff8f1afdf55fe0 ffff8f4957fbfa80 
    ffff8f4957fbfa78: ffffffff838d6705 
 #5 [ffff8f4957fbfa78] wake_up_process at ffffffff838d6705
    ffff8f4957fbfa80: ffff8f4957fbfa98 ffffffff8392fc05 
 #6 [ffff8f4957fbfa88] __cpu_stop_queue_work at ffffffff8392fc05
    ffff8f4957fbfa90: 000000000000001a ffff8f4957fbfbb0 
    ffff8f4957fbfaa0: ffffffff8393037a 
 #7 [ffff8f4957fbfaa0] stop_two_cpus at ffffffff8393037a
.....
    ffff8f4957fbfbb8: ffffffff838d3867 
 #8 [ffff8f4957fbfbb8] migrate_swap at ffffffff838d3867
    ffff8f4957fbfbc0: ffff8f4aea3a8000 ffff8f1ae77dc100 -------栈中的 migration_swap_arg
    ffff8f4957fbfbd0: 000000010000001a 0000000080490f7c 
    ffff8f4957fbfbe0: ffff8f4aea3a8000 ffff8f4957fbfc30 
    ffff8f4957fbfbf0: 0000000000000076 0000000000000076 
    ffff8f4957fbfc00: 0000000000000371 ffff8f4957fbfce8 
    ffff8f4957fbfc10: ffffffff838dd0ba 
 #9 [ffff8f4957fbfc10] task_numa_migrate at ffffffff838dd0ba
    ffff8f4957fbfc18: ffff8f1afc121f40 000000000000001a 
    ffff8f4957fbfc28: 0000000000000371 ffff8f4aea3a8000 ---这里ffff8f4957fbfc30 就是 task_numa_env 的存放在栈中的地址
    ffff8f4957fbfc38: 000000000000001a 000000010000003f 
    ffff8f4957fbfc48: 000000000000000b 000000000000022c 
    ffff8f4957fbfc58: 00000000000049a0 0000000000000012 
    ffff8f4957fbfc68: 0000000000000001 0000000000000003 
    ffff8f4957fbfc78: 000000000000006f 000000000000499f 
    ffff8f4957fbfc88: 0000000000000012 0000000000000001 
    ffff8f4957fbfc98: 0000000000000070 ffff8f1ae77dc100 
    ffff8f4957fbfca8: 00000000000002fb 0000000000000001 
    ffff8f4957fbfcb8: 0000000080490f7c ffff8f4aea3a8000 ---rbx压栈在此，所以这个就是current
    ffff8f4957fbfcc8: 0000000000017a48 0000000000001818 
    ffff8f4957fbfcd8: 0000000000000018 ffff8f4957fbfe20 
    ffff8f4957fbfce8: ffff8f4957fbfcf8 ffffffff838dd4d3 
#10 [ffff8f4957fbfcf0] numa_migrate_preferred at ffffffff838dd4d3
    ffff8f4957fbfcf8: ffff8f4957fbfd88 ffffffff838df5b0 
.....
crash> 
crash> 
```
整体上看，26号上的cpu也正在进行numa的balance动作，简单展开介绍一下numa在balance下的动作
在 task_tick_fair 函数中：
```
static void task_tick_fair(struct rq *rq, struct task_struct *curr, int queued)
{
	struct cfs_rq *cfs_rq;
	struct sched_entity *se = &curr->se;

	for_each_sched_entity(se) {
		cfs_rq = cfs_rq_of(se);
		entity_tick(cfs_rq, se, queued);
	}

	if (numabalancing_enabled)----------如果开启numabalancing，则会调用task_tick_numa
		task_tick_numa(rq, curr);

	update_rq_runnable_avg(rq, 1);
}
```
而 task_tick_numa 会根据扫描情况，将当前进程需要numa_balance的时候推送到一个work中。
通过调用change_prot_numa将所有映射到VMA的PTE页表项该为PAGE_NONE，使得下次进程访问页表的时候
产生缺页中断，handle_pte_fault 函数 会由于缺页中断的机会来根据numa 选择更好的node，具体不再展开。

在 26号cpu的调用链中，stop_two_cpus-->cpu_stop_queue_two_works-->__cpu_stop_queue_work 函数
由于 cpu_stop_queue_two_works 被内联了，但是 cpu_stop_queue_two_works 调用 __cpu_stop_queue_work
有两次，所以需要根据压栈地址判断当前是哪次调用出现问题。

```
    227 static int cpu_stop_queue_two_works(int cpu1, struct cpu_stop_work *work1,
    228                                     int cpu2, struct cpu_stop_work *work2)
    229 {
    230         struct cpu_stopper *stopper1 = per_cpu_ptr(&cpu_stopper, cpu1);
    231         struct cpu_stopper *stopper2 = per_cpu_ptr(&cpu_stopper, cpu2);
    232         int err;
    233 
    234         lg_double_lock(&stop_cpus_lock, cpu1, cpu2);
    235         spin_lock_irq(&stopper1->lock);---注意到这里已经持有了stopper1的锁
    236         spin_lock_nested(&stopper2->lock, SINGLE_DEPTH_NESTING);
.....
    243         __cpu_stop_queue_work(stopper1, work1);
    244         __cpu_stop_queue_work(stopper2, work2);
.....
    251 }

```
根据压栈的地址:
```
 #5 [ffff8f4957fbfa78] wake_up_process at ffffffff838d6705
    ffff8f4957fbfa80: ffff8f4957fbfa98 ffffffff8392fc05 
 #6 [ffff8f4957fbfa88] __cpu_stop_queue_work at ffffffff8392fc05
    ffff8f4957fbfa90: 000000000000001a ffff8f4957fbfbb0 
    ffff8f4957fbfaa0: ffffffff8393037a 
 #7 [ffff8f4957fbfaa0] stop_two_cpus at ffffffff8393037a
    ffff8f4957fbfaa8: 0000000100000001 ffff8f1afdf55fe8 

crash> dis -l ffffffff8393037a 2
/usr/src/debug/kernel-3.10.0-957.el7/linux-3.10.0-957.el7.x86_64/kernel/stop_machine.c: 244
0xffffffff8393037a <stop_two_cpus+394>: lea    0x48(%rsp),%rsi
0xffffffff8393037f <stop_two_cpus+399>: mov    %r15,%rdi
```
说明压栈的是244行的地址，也就是说目前调用的是243行的 __cpu_stop_queue_work。

然后分析对应的入参：
```
crash> task_numa_env ffff8f4957fbfc30
struct task_numa_env {
  p = 0xffff8f4aea3a8000, 
  src_cpu = 26, 
  src_nid = 0, 
  dst_cpu = 63, 
  dst_nid = 1, 
  src_stats = {
    nr_running = 11, 
    load = 556, ---load高
    compute_capacity = 18848, ---容量相当
    task_capacity = 18, 
    has_free_capacity = 1
  }, 
  dst_stats = {
    nr_running = 3, 
    load = 111, ---load低，且容量相当，要迁移过来
    compute_capacity = 18847, ---容量相当
    task_capacity = 18, 
    has_free_capacity = 1
  }, 
  imbalance_pct = 112, 
  idx = 0, 
  best_task = 0xffff8f1ae77dc100, ---要对调的task，是通过 task_numa_find_cpu-->task_numa_compare-->task_numa_assign 来获取的
  best_imp = 763, 
  best_cpu = 1---最佳的swap的对象对于1号cpu
}

crash> migration_swap_arg ffff8f4957fbfbc0 
struct migration_swap_arg {
  src_task = 0xffff8f4aea3a8000, 
  dst_task = 0xffff8f1ae77dc100, 
  src_cpu = 26, 
  dst_cpu = 1-----选择的dst cpu为1
}
```
根据 cpu_stop_queue_two_works 的代码，它在持有 cpu_stopper:26号cpu锁的情况下，去
调用try_to_wake_up ，wake的对象是 用来migrate的 kworker。

```
static void __cpu_stop_queue_work(struct cpu_stopper *stopper,
					struct cpu_stop_work *work)
{
	list_add_tail(&work->list, &stopper->works);
	wake_up_process(stopper->thread);//其实一般就是唤醒 migration
}
```
由于最佳的cpu对象为1，所以需要cpu上的migrate来拉取进程。
```
crash> p cpu_stopper:1
per_cpu(cpu_stopper, 1) = $33 = {
  thread = 0xffff8f1bf6bb9040, ----需要唤醒的目的task
  lock = {
    {
      rlock = {
        raw_lock = {
          val = {
            counter = 1
          }
        }
      }
    }
  }, 
  enabled = true, 
  works = {
    next = 0xffff8f4957fbfac0, 
    prev = 0xffff8f4957fbfac0
  }, 
  stop_work = {
    list = {
      next = 0xffff8f4afbe16000, 
      prev = 0xffff8f4afbe16000
    }, 
    fn = 0xffffffff83952100, 
    arg = 0x0, 
    done = 0xffff8f1ae3647c08
  }
}
crash> kmem 0xffff8f1bf6bb9040
CACHE            NAME                 OBJSIZE  ALLOCATED     TOTAL  SLABS  SSIZE
ffff8eecffc05f00 task_struct             4152       1604      2219    317    32k
  SLAB              MEMORY            NODE  TOTAL  ALLOCATED  FREE
  fffff26501daee00  ffff8f1bf6bb8000     1      7          7     0
  FREE / [ALLOCATED]
  [ffff8f1bf6bb9040]

    PID: 14
COMMAND: "migration/1"--------------目的task就是对应的cpu上的migration 
   TASK: ffff8f1bf6bb9040  [THREAD_INFO: ffff8f1bf6bc4000]
    CPU: 1
  STATE: TASK_INTERRUPTIBLE (PANIC)

      PAGE         PHYSICAL      MAPPING       INDEX CNT FLAGS
fffff26501daee40 3076bb9000                0        0  0 6fffff00008000 tail

```
现在的问题是，虽然我们知道了当前cpu26号进程在拿了锁的情况下去唤醒1号cpu上的migrate进程，
那么为什么会迟迟不释放锁，导致1号cpu因为等待该锁时间过长而触发了hardlock的panic呢？

下面就分析，为什么它持锁的时间这么长：
```
 #3 [ffff8f1afdf48ef0] end_repeat_nmi at ffffffff83f6bd69
    [exception RIP: try_to_wake_up+114]
    RIP: ffffffff838d63d2  RSP: ffff8f4957fbfa30  RFLAGS: 00000002
    RAX: 0000000000000001  RBX: ffff8f1bf6bb9844  RCX: 0000000000000000
    RDX: 0000000000000001  RSI: 0000000000000003  RDI: ffff8f1bf6bb9844
    RBP: ffff8f4957fbfa70   R8: ffff8f4afbe15ff0   R9: 0000000000000000
    R10: 0000000000000000  R11: 0000000000000000  R12: 0000000000000000
    R13: ffff8f1bf6bb9040  R14: 0000000000000000  R15: 0000000000000003
    ORIG_RAX: ffffffffffffffff  CS: 0010  SS: 0000
--- <NMI exception stack> ---
 #4 [ffff8f4957fbfa30] try_to_wake_up at ffffffff838d63d2
    ffff8f4957fbfa38: 000000000001ab80 0000000000000086 
    ffff8f4957fbfa48: ffff8f4afbe15fe0 ffff8f4957fbfb48 
    ffff8f4957fbfa58: 0000000000000001 ffff8f4afbe15fe0 
    ffff8f4957fbfa68: ffff8f1afdf55fe0 ffff8f4957fbfa80 

    crash> dis -l ffffffff838d63d2
/usr/src/debug/kernel-3.10.0-957.el7/linux-3.10.0-957.el7.x86_64/kernel/sched/core.c: 1790
0xffffffff838d63d2 <try_to_wake_up+114>:        mov    0x28(%r13),%eax

   1721 static int
   1722 try_to_wake_up(struct task_struct *p, unsigned int state, int wake_flags)
   1723 {
.....
   1787          * If the owning (remote) cpu is still in the middle of schedule() with
   1788          * this task as prev, wait until its done referencing the task.
   1789          */
   1790         while (p->on_cpu)---------原来循环在此
   1791                 cpu_relax();
.....
   1814         return success;
   1815 }
```

我们用一个简单的图来表示一下这个hardlock：

```
    CPU1                                    CPU26
    schedule(.prev=migrate/1)               <fault>
      pick_next_task()                        ...
        idle_balance()                          migrate_swap()
          active_balance()                        stop_two_cpus()
                                                    spin_lock(stopper0->lock)
                                                    spin_lock(stopper1->lock)
                                                    try_to_wake_up
                                                      pause() -- waits for schedule()
            stop_one_cpu(1)
              spin_lock(stopper26->lock) -- waits for stopper lock
```
查看上游的补丁，
```
 static void __cpu_stop_queue_work(struct cpu_stopper *stopper,
-					struct cpu_stop_work *work)
+				       struct cpu_stop_work *work,
+				       struct wake_q_head *wakeq)
 {
 	list_add_tail(&work->list, &stopper->works);
-	wake_up_process(stopper->thread);
+	wake_q_add(wakeq, stopper->thread);
 }
```
#### 三、故障复现
1.由于这个是一个race condition导致的hardlock，逻辑上分析已经没有问题了，就没有花时间去复现，
该环境运行一个dpdk的node，不过为了性能设置了只在一个numa节点上，可以频繁造成numa的不均衡，所以要复现的同学，
可以参考单numa节点上运行dpdk来复现，会概率大一些。

#### 四、故障规避或解决

我们的解决方案是：

1.关闭numa的自动balance。
2.手工合入 linux社区的 0b26351b910f 补丁。
3.这个补丁在centos的 3.10.0-974.el7 合入了 
- [kernel] stop_machine, sched: Fix migrate_swap() vs. active_balance() deadlock (Phil Auld) [1557061]
同时红帽又反向合入到了3.10.0-957.27.2.el7.x86_64，所以把centos内核升级到 3.10.0-957.27.2.el7.x86_64也是
一种选择。

 