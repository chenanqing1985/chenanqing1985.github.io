# 我的docker hung住啦-2
## 背景：内核版本centos7.6
### runc陷入内核态死循环导致docker阻塞

#### 一、故障现象
运维同学报障，某个容器，远程ssh不上，但是查看智能监控，容器存活正常，业务也正常,且查看宿主机的监控，发现load在慢慢上升。

#### 二、故障现象分析
根据对应卡住的容器，找到对应的runc的堆栈如下：
```
[<ffffffffba3da5f5>] wait_iff_congested+0x135/0x150
[<ffffffffba3cce5a>] shrink_inactive_list+0x30a/0x5d0
[<ffffffffba3cd815>] shrink_lruvec+0x385/0x730
[<ffffffffba3cdc36>] shrink_zone+0x76/0x1a0
[<ffffffffba3ce140>] do_try_to_free_pages+0xf0/0x4e0
[<ffffffffba3ce62c>] try_to_free_pages+0xfc/0x180
[<ffffffffba47941e>] free_more_memory+0xae/0x100
[<ffffffffba47a73b>] __getblk+0x15b/0x300---这个地方会有一个循环
[<ffffffffc075bd43>] __ext4_get_inode_loc+0xe3/0x3c0 [ext4]
[<ffffffffc075e49d>] ext4_get_inode_loc+0x1d/0x20 [ext4]
[<ffffffffc0760256>] ext4_reserve_inode_write+0x26/0xa0 [ext4]
[<ffffffffc0760323>] ext4_mark_inode_dirty+0x53/0x210 [ext4]
[<ffffffffc0763b20>] ext4_dirty_inode+0x40/0x60 [ext4]
[<ffffffffba4715ad>] __mark_inode_dirty+0x16d/0x270
[<ffffffffba45f6a9>] update_time+0x89/0xd0
[<ffffffffba45f790>] file_update_time+0xa0/0xf0
[<ffffffffc0763d4c>] ext4_page_mkwrite+0x6c/0x470 [ext4]
[<ffffffffba3e5a3a>] do_page_mkwrite+0x8a/0xe0
[<ffffffffba3e60d6>] do_shared_fault.isra.62+0x86/0x280
[<ffffffffba3ea664>] handle_pte_fault+0xe4/0xd10
[<ffffffffba3ed3ad>] handle_mm_fault+0x39d/0x9b0
[<ffffffffba971603>] __do_page_fault+0x203/0x4f0
[<ffffffffba971925>] do_page_fault+0x35/0x90
[<ffffffffba96d768>] page_fault+0x28/0x30
[<ffffffffffffffff>] 0xffffffffffffffff
```
从堆栈看，runc由于pagefault进来，然后因为申请内存不到而在内核态死循环的。
查看此时pagefault的oom的执行需要mem_cgroup_oom_enable，也就是用户态过来的异常，才会使能oom，也要求 mem_cgroup_oom_synchronize 能够顺利执行才会最终杀进程，而很多时候循环在__getblk 的循环中，如果当前进程被选择为需要oom的进程，则必须退出循环才能处理。所以出现了软死锁，这个非常关键，我们要跳出循环才能执行oom杀进程，所以不是没有oom，而是oom在这种情况下没法执行。这可以理解为一个内核bug，

do_try_to_free_pages-->shrink_zone--->shrink_lruvec--->shrink_list--->shrink_inactive_list-→wait_iff_congested的流程下，
其实最终会进入 

__mem_cgroup_try_charge 这个函数的如下，
```
{

    if (unlikely(task_in_memcg_oom(current)))
        goto nomem;



nomem:
    if (!(gfp_mask & __GFP_NOFAIL)) {
        *ptr = NULL;
        return -ENOMEM;
    }

}
```

而一旦进入ENOMEM，又会重复循环，所以处理有两种：

用户态处理：给对应的进程kill -9 一个信号，让它进入
```
    if (unlikely(test_thread_flag(TIF_MEMDIE)
             || fatal_signal_pending(current)))
        goto bypass;—这个流程里面就会跳出nomem的流程
```
内核态处理：写一个模块，修改这次申请内存的标志
```
if (unlikely(current->flags & PF_MEMALLOC))
        goto bypass;
```
以上这两种都尝试过，并且都行之有效，用户态处理有一个不太好的地方就是，我们kill的进程，可能不是oom该选的进程。



另外，不是所有的pagefault进来都是这个堆栈，比如是因为写时复制的原因进来，这个时候堆栈大多数情况下是如下：
```
PID: 112272 TASK: ffff9ae9eaff8000 CPU: 21 COMMAND: "runc:[2:INIT]"
#0 [ffff9adf48673608] __schedule at ffffffffb6769a72
#1 [ffff9adf48673698] schedule at ffffffffb6769f19
#2 [ffff9adf486736a8] schedule_timeout at ffffffffb6767968
#3 [ffff9adf48673758] io_schedule_timeout at ffffffffb67695ed
#4 [ffff9adf48673788] wait_iff_congested at ffffffffb61da5f5
#5 [ffff9adf486737e8] shrink_inactive_list at ffffffffb61cce5a
#6 [ffff9adf486738b0] shrink_lruvec at ffffffffb61cd815
#7 [ffff9adf486739b0] shrink_zone at ffffffffb61cdc36
#8 [ffff9adf48673a08] do_try_to_free_pages at ffffffffb61ce140
#9 [ffff9adf48673a80] try_to_free_mem_cgroup_pages at ffffffffb61ce78a
#10 [ffff9adf48673b18] mem_cgroup_reclaim at ffffffffb6234d1e
#11 [ffff9adf48673b58] __mem_cgroup_try_charge at ffffffffb62356dc
#12 [ffff9adf48673c10] mem_cgroup_charge_common at ffffffffb6236049
#13 [ffff9adf48673c58] wp_page_copy at ffffffffb61e6b50
#14 [ffff9adf48673cc8] do_wp_page at ffffffffb61e8f6b
#15 [ffff9adf48673d70] handle_pte_fault at ffffffffb61ea8fd
#16 [ffff9adf48673e08] handle_mm_fault at ffffffffb61ed3ad
#17 [ffff9adf48673eb0] __do_page_fault at ffffffffb6771603
#18 [ffff9adf48673f20] do_page_fault at ffffffffb6771925
#19 [ffff9adf48673f50] page_fault at ffffffffb676d768
```
这种情况下，runc会由于写时复制而出现没有内存的情况，返回值一般是：

mem_cgroup_charge_common return=0xfffffffffffffff4，这个就是-12了，也就是没有内存。

这种情况虽然不会出现内核态死循环，但是由于用户态确实需要访问某个地址触发写时复制，这样会导致它长期地持有

mmap_sem这把锁，也就是从用户态频繁进入到内核态，虽然没有直接在内核态死循环，但是也是一直在一个地址循环，如下：
```
:Sat Aug 29 06:32:19 2020,runc:[2:INIT],call handle_mm_fault tid=112272,address=cd3468,vma=0xffff9adf54e2c000,flags=a9

:Sat Aug 29 06:32:23 2020,runc:[2:INIT],call handle_mm_fault tid=112272,address=cd3468,vma=0xffff9adf54e2c000,flags=a9

:Sat Aug 29 06:32:33 2020,runc:[2:INIT],call handle_mm_fault tid=112272,address=cd3468,vma=0xffff9adf54e2c000,flags=a9
```
耗时也很长，一般都是秒级，这对于cpu来说也是一种浪费。

3.还有一种load高，是因为内存翻转导致，比如提交的io飞了，在oppo的dell环境遇到过三次。

4.很多ps进程阻塞，这个也是表象，ps阻塞还出现过读锁堵塞读，只要是因为防止写饥饿，因为虽然读者持有sem的读端，当写者来写时，排队会阻塞后端再来的读。


#### 三、故障复现
这种问题，一般需要将内存限制住，然后不停申请内存，申请的内存最好能够通过pagefault进入内核，比如使用loop-pool来做docker的存储后端

#### 四、故障规避或解决
1、在循环的地方加次数限制。
2、监控容器的内存使用，必要时在用户态oomguard。
3、创建内核模块，便利task链，检测这种循环的情况，将对应的task丢到一个work中去kill。