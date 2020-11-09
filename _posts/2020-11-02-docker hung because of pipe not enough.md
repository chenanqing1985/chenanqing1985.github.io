# 我的docker hung住啦-1
## 背景：这个是之前遇到的老问题，最近docker社区里面其他人报了这问题暂时还没解决，issue的链接是：https://github.com/containerd/containerd/issues/4434
### 下面列一下我们是怎么排查并解这个问题的。

#### 一、故障现象

Oppo云智能监控发现lxcfs 服务不是处于工作态超过配置的阈值：
```
# systemctl status lxcfs
● lxcfs.service - FUSE filesystem for LXC
Loaded: loaded (/usr/lib/systemd/system/lxcfs.service; enabled; vendor preset: disabled)
Active: activating (start-post) since Tue 2020-06-23 14:37:50 CST; 5min ago---这个是6月份的案例，非active (running) 状态
Docs: man:lxcfs(1)
Process: 415455 ExecStopPost=/bin/sh -c if mount |grep "baymax\/lxcfs"; then fusermount -u /var/lib/baymax/lxcfs; fi (code=exited, status=0/SUCCESS)
Main PID: 415526 (lxcfs); : 415529 (lxcfs-remount-c)
Tasks: 43
Memory: 28.9M
CGroup: /system.slice/lxcfs.service
├─415526 /usr/bin/lxcfs -o nonempty /var/lib/baymax/lxcfs/
└─control
├─415529 /bin/sh /usr/bin/lxcfs-remount-containers
├─416923 /bin/sh /usr/bin/lxcfs-remount-containers
└─419090 docker exec 1eb2f723b69e sh -c \ export f=/proc/cpuinfo && test -f /var/lib/baymax/lxcfs/$f && (umount $f; mount --bind...
```


#### 二、故障现象分析

查看对应的进程树，发现 1eb2f723b69e 容器的runc阻塞没有返回。

```
# ps -ef |grep -i runc |grep -v shim 
root 172169 138974 0 14:43 pts/2 00:00:00 grep --color -i runc
root 420924 70170 0 14:37 ? 00:00:00 runc --root /var/run/docker/runtime-runc/moby --log /run/docker/containerd/daemon/io.containerd.runtime.v1.linux/moby/1eb2f723b69e2dba83bc490d3fab66922a13a4787be8bcb4cd486e97843ffef5/log.json --log-format json exec --process /tmp/runc-process904568476 --detach --pid-file /run/docker/containerd/daemon/io.containerd.runtime.v1.linux/moby/1eb2f723b69e2dba83bc490d3fab66922a13a4787be8bcb4cd486e97843ffef5/4dfeee72cd794ebec396fb8450f8944499cdde99d22054c950e5a80fb56f0968.pid 1eb2f723b69e2dba83bc490d3fab66922a13a4787be8bcb4cd486e97843ffef5
root 423656 420924 0 14:37 ? 00:00:00 runc init
```

通过cat /proc/423656/stack 发现它阻塞在pipe的write

然后通过crash工具在线查看对应 423656  的堆栈详细信息：
```
PID: 423656  TASK: ffffa0e872d56180  CPU: 28  COMMAND: "runc:[2:INIT]"
。。。。。。
 #1 [ffffa13093eb3d08] schedule at ffffffffb6969f19
    ffffa13093eb3d10: ffffa13093eb3d60 ffffffffb644bd50 
 #2 [ffffa13093eb3d18] pipe_wait at ffffffffb644bd50
    ffffa13093eb3d20: 0000000000000000 ffffa0e872d56180 
    ffffa13093eb3d30: ffffffffb62c3f50 ffffa0f072d87108 
    ffffa13093eb3d40: ffffa1032ad42030 00000000e3a7c164 
    ffffa13093eb3d50: ffffa1032ad42000 0000000000000010 -----分析堆栈，pipe的inode压栈在此
    ffffa13093eb3d60: ffffa13093eb3de8 ffffffffb644bff9 
 #3 [ffffa13093eb3d68] pipe_write at ffffffffb644bff9
    ffffa13093eb3d70: ffffa1032ad42028 ffffa0e872d56180 
ffffa13093eb3d80: ffffa13093eb3df8 0000000000000000 
。。。。限于篇幅，省略部分堆栈
    ffffa13093eb3f50: ffffffffb6976ddb 
 #7 [ffffa13093eb3f50] system_call_fastpath at ffffffffb6976ddb
    RIP: 000000000045b8a5  RSP: 000000c000008be8  RFLAGS: 00010206
    RAX: 0000000000000001  RBX: 0000000000000000  RCX: 000000c000000000
    RDX: 0000000000000010  RSI: 000000c000008bf0  RDI: 0000000000000002
    RBP: 000000c000008b90   R8: 0000000000000001   R9: 00000000006c0fab
    R10: 0000000000000000  R11: 0000000000000202  R12: 0000000000000000
    R13: 0000000000000000  R14: 000000000086d0d8  R15: 0000000000000000
    ORIG_RAX: 0000000000000001  CS: 0033  SS: 002b
```

看情况是卡在pipe的write，然后看下它打开的文件，找到对应的inode信息：

```
PID: 423656  TASK: ffffa0e872d56180  CPU: 28  COMMAND: "runc:[2:INIT]"
ROOT: /rootfs    CWD: /rootfs
 FD       FILE            DENTRY           INODE       TYPE PATH
  0 ffffa0feca33cf00 ffffa1333b800240 ffffa1333f568850 CHR  /dev/null
  1 ffffa1031adba700 ffffa0f10efb83c0 ffffa0d78de55c80 FIFO 
  2 ffffa12f3d7df300 ffffa0f10efb8a80 ffffa0d78de56f00 FIFO -----对应那个pipe
  3 ffffa133026b9700 ffffa11382e2f080 ffffa12a2b07ad30 SOCK UNIX
```
验证下这个pipe：

```
crash> struct file.private_data ffffa12f3d7df300
  private_data = 0xffffa1032ad42000

crash> pipe_inode_info 0xffffa1032ad42000--------和上面的堆栈对的上
struct pipe_inode_info {
  mutex = {
    count = {
      counter = 1
    }, 
。。。。。
    }
  }, 
  nrbufs = 1, ----只有一个buf，说明pipe创建的时候，page不够，这个主要受限于 pipe-user-pages-soft的默认配置以及pipe-max-size
  curbuf = 0, 
  buffers = 1, 
  readers = 1, 
  writers = 1, 
  files = 2, 
  waiting_writers = 1, 
  r_counter = 1, 
  w_counter = 1, 
  tmp_page = 0x0, 
  fasync_readers = 0x0, 
  fasync_writers = 0x0, 
  bufs = 0xffffa132c3a3d100, 
  user = 0xffffffffb6e4d700
}
```

看下pipe中的内容：
```
crash> pipe_buffer 0xffffa132c3a3d100
struct pipe_buffer {
  page = 0xffffe392f992cb00, 
  offset = 0, 
  len = 4081, ---内容的长度
  ops = 0xffffffffb6a2e000, 
  flags = 0, 
  private = 0
}
```
查看对应的page的物理地址
```
crash> kmem -p |grep ffffe392f992cb00
ffffe392f992cb00 2e64b2c000                0        0  1 2fffff00000000
```
直接rd一下对应的物理页面
```
crash> rd  -a -p 2e64b2c000 4081-----------这个4081就是上面的长度
      2e64b2c000:  runtime/cgo: pthread_create failed: Resource temporarily una
      2e64b2c03c:  vailable
      2e64b2c045:  SIGABRT: abort-----------------
      2e64b2c054:  PC=0x6c0fab m=0 sigcode=18446744073709551610
      2e64b2c082:  goroutine 0 [idle]:
      2e64b2c096:  runtime: unknown pc 0x6c0fab
      2e64b2c0b3:  stack: frame={sp:0x7ffc54fb5b18, fp:0x0} stack=[0x7ffc547b6f
      2e64b2c0ef:  a8,0x7ffc54fb5fd0)
      2e64b2c102:  00007ffc54fb5a18:  0000000000004000  0000000000000000 
      2e64b2c139:  00007ffc54fb5a28:  0000000000d0eb80  00007fe3c913f000 
限于篇幅，省略部分打印。。。。
    
      2e64b2cf70:  00007ffc54fb5bd8:  0000000000cd4603  0000000000a9d760 
      2e64b2cfa7:  00007ffc54fb5be8:  00000000006ea87b  0000000000cd4580 
      2e64b2cfde:  00007ffc54fb5bf8:  
```
很明显，pipe的write阻塞是因为page的空间不够了，默认的阻塞模式，如果对端能够及时读取，应该会将空间空出来才对，

所以下面需要看下对端为啥没有来读：
```
crash> pipe_inode_info.wait 0xffffa1032ad42000
  wait = {
    lock = {
      {
        rlock = {
          raw_lock = {
            val = {
              counter = 0
            }
          }
        }
      }
    }, 
    task_list = {
      next = 0xffffa13093eb3d38, --------__wait_queue的task_list链串在此
      prev = 0xffffa0f072d87108
    }
  }
```
查看对应的等待队列
```
crash> __wait_queue 0xffffa13093eb3d20
struct __wait_queue {
  flags = 0, 
  private = 0xffffa0e872d56180, ----对应的就是 423656本身的等待
  func = 0xffffffffb62c3f50, 
  task_list = {
    next = 0xffffa0f072d87108, 
    prev = 0xffffa1032ad42030
  }
}
```
根据fd的对端信息，可以找到其父进程，也就是containerd-shim进程,而根据如下代码：
```
func (r *Runc) Exec(context context.Context, id string, spec specs.Process, opts *ExecOpts) error {
。。。。
cmd := r.command(context, append(args, id)...)
。。。。
ec, err := Monitor.Start(cmd)
。。。。
status, err := Monitor.Wait(cmd, ec)
}
```
由于shim的代码里面是等待runc退出再去读取pipe，而runc又没有pipe容量不够而不退出，所以形成了死锁。
有兴趣的同学也可以了解下这个：https://github.com/opencontainers/runc/issues/2530

#### 三、故障复现
```
1、# docker ps |grep busybox
8bebfd8a7b59        busybox                                               "sh"                     6 days ago          Up 2 days

2、# pwd
/sys/fs/cgroup/pids/docker/8bebfd8a7b59748da6bcb154ec2ce428d1f21376b16c3915d962ec4149484e5c

3、# cat pids.current -------查看一下当前线程数
1

4、# echo 3 > pids.max      ----限制一下线程数，由于runc执行过程会创建多个线程，这里设置为3

5、# cat pids.max 
3

6、查看默认pipe配置：
# cat /proc/sys/fs/pipe-user-pages-soft
16384

7、测试是否阻塞：
# docker exec 8bebfd8 ls
OCI runtime exec failed: exec failed: container_linux.go:344: starting container process caused "read init-p: connection reset by peer": unknown
发现并没有阻塞，然后我们将pipe对应的page数量改小：

8、# echo 1 >/proc/sys/fs/pipe-user-pages-soft 
然后再次执行同样命令：
# docker exec 8bebfd8 ls
发现阻塞了，没有结果返回，我们来看阻塞在哪：
9、# ps -ef |grep runc |grep -v shim|grep -v grep
root     122935 642935  0 09:10 ?        00:00:00 runc --root /var/run/docker/runtime-runc/moby --log /run/docker/containerd/daemon/io.containerd.runtime.v1.linux/moby/8bebfd8a7b59748da6bcb154ec2ce428d1f21376b16c3915d962ec4149484e5c/log.json --log-format json exec --process /tmp/runc-process773686128 --detach --pid-file /run/docker/containerd/daemon/io.containerd.runtime.v1.linux/moby/8bebfd8a7b59748da6bcb154ec2ce428d1f21376b16c3915d962ec4149484e5c/7d5955f67c29d2a66c77295f16d416c5baa5277635bf3f3edec381c27f30bafc.pid 8bebfd8a7b59748da6bcb154ec2ce428d1f21376b16c3915d962ec4149484e5c
root     122943 122935  0 09:10 ?        00:00:00 runc init-------------被阻塞，没返回
 
10、查看具体阻塞的堆栈：
# ls /proc/122943/task/ |xargs -I file sh -c "echo file && cat /proc/file/stack"
122943
[<ffffffffadc4bd50>] pipe_wait+0x70/0xc0-------------阻塞在pipe的写，
[<ffffffffadc4bff9>] pipe_write+0x1f9/0x540
[<ffffffffadc41c13>] do_sync_write+0x93/0xe0
[<ffffffffadc42700>] vfs_write+0xc0/0x1f0
[<ffffffffadc4351f>] SyS_write+0x7f/0xf0
[<ffffffffae176ddb>] system_call_fastpath+0x22/0x27
[<ffffffffffffffff>] 0xffffffffffffffff
122944
[<ffffffffadb0e286>] futex_wait_queue_me+0xc6/0x130
[<ffffffffadb0ef6b>] futex_wait+0x17b/0x280
[<ffffffffadb10cb6>] do_futex+0x106/0x5a0
[<ffffffffadb111d0>] SyS_futex+0x80/0x190
[<ffffffffae176ddb>] system_call_fastpath+0x22/0x27
[<ffffffffffffffff>] 0xffffffffffffffff

11、通过如下的stap打点命令，可以确定是pipe容量不足：
printf(" %s,tid=%d,pipe=%d\n",execname(),tid(),
            @cast(@cast($iocb->ki_filp,"struct file")->private_data, "struct pipe_inode_info")->buffers)
打点结果：
runc:[2:INIT],tid=122943,pipe=1--------------当容量足够时，这个值默认为16,是linux内核的默认值

12、查看pipe中的数据：
# cat /proc/122943/fd/2
runtime/cgo: pthread_create failed: Resource temporarily unavailable
SIGABRT: abort
PC=0x6c0fab m=0 sigcode=18446744073709551610

goroutine 0 [idle]:
runtime: unknown pc 0x6c0fab
stack: frame={sp:0x7fffcd06ae18, fp:0x0} stack=[0x7fffcc86c398,0x7fffcd06b3c0)
00007fffcd06ad18:  00000000ffffffff  00007f286a8ba000 
00007fffcd06ad28:  00007fffcd06ad70  00007fffcd06ada8 
00007fffcd06ad38:  0000000000404f61 <runtime.mmap+177>  00007fffcd06ad78 
00007fffcd06ad48:  0000000000000000  0000000000000000 
。。。限于篇幅，其余数据省略
```
以上就是完整复现流程。
#### 四、故障规避或解决

我们的解决方案是：

1.增加 pipe-user-pages-soft 的配置，这个需要根据云的密度来，因为docker各个组件大量使用pipe来通信。

2.监控user_struct.pipe_bufs 的用量，这个要根据用户来。

3.建议不要去动shim 中等待runc退出在读取pipe的逻辑，除非大的故障，不会没事去升级一遍containerd-shim。

4.runc存活时间监控。

5.监控容器pid的用量，超过阈值报警。


ps：docker hung住的问题案例很多，后续我们会有相应的案例展出，以避免大家重复踩坑，共同推进云原生的进步。

 