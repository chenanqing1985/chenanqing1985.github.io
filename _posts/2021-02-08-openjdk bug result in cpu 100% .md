# 容器内就获取个cpu利用率，怎么就占用单核100%了呢
## 背景：这个是在centos7 + lxcfs 和jdk11 的环境上复现的
### 下面列一下我们是怎么排查并解这个问题的。

#### 一、故障现象
oppo内核团队接到jvm的兄弟甘兄发来的一个案例,
现象是java的热点一直是如下函数，占比很高：
```
at sun.management.OperatingSystemImpl.getProcessCpuLoad(Native Method)
```
我们授权后登陆oppo云宿主，发现top显示如下：
```
   PID USER      PR  NI    VIRT    RES    SHR S %CPU %MEM     TIME+ COMMAND                                                                       
617248 service   20   0   11.9g   8.6g   9300 R 93.8  2.3   3272:09 java                                                                          
179526 service   20   0   11.9g   8.6g   9300 R 93.8  2.3  77462:17 java     
```

#### 二、故障现象分析

java的线程，能把cpu使用率打这么高，常见一般是gc或者跑jni的死循环，
从jvm团队反馈的热点看，这个线程应该在一直获取cpu利用率，而不是以前常见的问题。
从perf的角度看，该线程一直在触发read调用。
然后strace查看read的返回值，打印如下：
```
read(360, 0x7f6f9000b000, 4096)         = -1 ENOTCONN (Transport endpoint is not connected)
read(360, 0x7f6f9000b000, 4096)         = -1 ENOTCONN (Transport endpoint is not connected)
read(360, 0x7f6f9000b000, 4096)         = -1 ENOTCONN (Transport endpoint is not connected)
read(360, 0x7f6f9000b000, 4096)         = -1 ENOTCONN (Transport endpoint is not connected)
read(360, 0x7f6f9000b000, 4096)         = -1 ENOTCONN (Transport endpoint is not connected)
read(360, 0x7f6f9000b000, 4096)         = -1 ENOTCONN (Transport endpoint is not connected)
```
接着查看360这个fd到底是什么类型：
```
# ll /proc/22345/fd/360
lr-x------ 1 service service 64 Feb  4 11:24 /proc/22345/fd/360 -> /proc/stat
```
发现是在读取/proc/stat ,和上面的java热点读取cpu利用率是能吻合的。

根据/proc/stat在容器内的挂载点，
```
# cat /proc/mounts  |grep stat
tmpfs /proc/timer_stats tmpfs rw,nosuid,size=65536k,mode=755 0 0
lxcfs /proc/stat fuse.lxcfs rw,nosuid,nodev,relatime,user_id=0,group_id=0,allow_other 0 0
lxcfs /proc/stat fuse.lxcfs rw,nosuid,nodev,relatime,user_id=0,group_id=0,allow_other 0 0
lxcfs /proc/diskstats fuse.lxcfs rw,nosuid,nodev,relatime,user_id=0,group_id=0,allow_other 0 0
```
很奇怪的是，发现有两个挂载点都是/proc/stat，
尝试手工读取一下这个文件：
```
# cat /proc/stat
cpu  12236554 0 0 9808893 0 0 0 0 0 0
cpu0 10915814 0 0 20934 0 0 0 0 0 0
cpu1 1320740 0 0 9787959 0 0 0 0 0 0
```
说明bash访问这个挂载点没有问题，难道bash访问的挂载点和出问题的java线程访问的挂载点不是同一个？
查看对应java打开的fd的super_block和mount，
```
crash> files 22345 |grep -w 360
360 ffff914f93efb400 ffff91541a16d440 ffff9154c7e29500 REG  /rootfs/proc/stat/proc/stat
crash> inode.i_sb ffff9154c7e29500
  i_sb = 0xffff9158797bf000
```
然后查看对应进程归属mount的namespace中的挂载点信息：
```
crash> mount -n 22345 |grep lxcfs
ffff91518f28b000 ffff9158797bf000 fuse   lxcfs     /rootfs/proc/stat
ffff911fb0209800 ffff914b3668e800 fuse   lxcfs     /rootfs/var/lib/baymax/var/lib/baymax/lxcfs
ffff915535becc00 ffff914b3668e800 fuse   lxcfs     /rootfs/proc/cpuinfo
ffff915876b45380 ffff914b3668e800 fuse   lxcfs     /rootfs/proc/meminfo
ffff912415b56e80 ffff914b3668e800 fuse   lxcfs     /rootfs/proc/uptime
ffff91558260f300 ffff914b3668e800 fuse   lxcfs     /rootfs/proc/stat/proc/stat
ffff911f52a6bc00 ffff914b3668e800 fuse   lxcfs     /rootfs/proc/diskstats
ffff914d24617180 ffff914b3668e800 fuse   lxcfs     /rootfs/proc/swaps
ffff911de87d0180 ffff914b3668e800 fuse   lxcfs     /rootfs/sys/devices/system/cpu/online
```
由于cat操作较短，写一个简单的demo代码，open 之后不关闭并且read 对应的/proc/stat路径，发现它
访问的挂载点的super_blcok是 ffff914b3668e800，由于代码极为简单，在此不列出。

也就是进一步地确认了，出问题的java进程访问的是一个出问题的挂载点。
那为什么访问这个挂载点会返回ENOTCONN呢？
```
crash> struct file.private_data ffff914f93efb400
  private_data = 0xffff911e57b49b80---根据fuse模块代码，这个是一个fuse_file

crash> fuse_file.fc 0xffff911e57b49b80
  fc = 0xffff9158797be000
crash> fuse_conn.connected 0xffff9158797be000
  connected = 0
```
果然对应的fuse_conn的连接状态是0，从打点看，是在fuse_get_req返回的：
调用链如下：
```
sys_read-->vfs_read-->fuse_direct_read-->__fuse_direct_read-->fuse_direct_io
    --->fuse_get_req--->__fuse_get_req

static struct fuse_req *__fuse_get_req(struct fuse_conn *fc, unsigned npages,
				       bool for_background)
{
...
	err = -ENOTCONN;
	if (!fc->connected)
		goto out;
...
}
```

下面要定位的就是，为什么出问题的java线程会没有判断read的返回值，而不停重试呢？
gdb跟踪一下：
```
(gdb) bt
#0  0x00007fec3a56910d in read () at ../sysdeps/unix/syscall-template.S:81
#1  0x00007fec3a4f5c04 in _IO_new_file_underflow (fp=0x7fec0c01ae00) at fileops.c:593
#2  0x00007fec3a4f6dd2 in __GI__IO_default_uflow (fp=0x7fec0c01ae00) at genops.c:414
#3  0x00007fec3a4f163e in _IO_getc (fp=0x7fec0c01ae00) at getc.c:39
#4  0x00007febeacc6738 in get_totalticks.constprop.4 () from /usr/local/paas-agent/HeyTap-Linux-x86_64-11.0.7/lib/libmanagement_ext.so
#5  0x00007febeacc6e47 in Java_com_sun_management_internal_OperatingSystemImpl_getSystemCpuLoad0 ()
   from /usr/local/paas-agent/HeyTap-Linux-x86_64-11.0.7/lib/libmanagement_ext.so
```
确认了用户态在循环进入 _IO_getc的流程，然后jvm的甘兄开始走查代码，发现有一个代码非常可疑：
UnixOperatingSystem.c：get_totalticks 函数，会有一个循环如下：
```
static void next_line(FILE *f) {
    while (fgetc(f) != '\n');
}
```

从fuse模块的代码看，fc->connected = 0的地方并不是很多，基本都是abort或者fuse_put_super调用导致，通过对照java线程升高的时间点以及查看journal的日志，进一步fuse的挂载点失效有用户态文件系统lxcfs退出，而挂载点还有inode在访问导致，导致又无法及时地回收这个super_block,新的挂载使用的是：
```
remount_container() {
    export f=/proc/stat    && test -f /var/lib/baymax/lxcfs/$f && (umount $f;  mount --bind /var/lib/baymax/lxcfs/$f $f)
```
使用了bind模式，具体bind的mount的特性可以参照网上的资料。
并且在journal日志中，查看到了关键的：
```
: umount: /proc/stat: target is busy
```
这行日志表明，当时卸载时出现失败，不是所有的访问全部关闭。


#### 三、故障复现
1.针对以上的怀疑点，写了一个很简单的demo如下：
```
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#define SCNd64 "I64d"
#define DEC_64 "%"SCNd64

static void next_line(FILE *f) {
    while (fgetc(f) != '\n');//出错的关键点
}

#define SET_VALUE_GDB 10
int main(int argc,char* argv[])
{
  unsigned long i =1;
  unsigned long j=0;
  FILE * f=NULL;
  FILE         *fh;
  unsigned long        userTicks, niceTicks, systemTicks, idleTicks;
  unsigned long        iowTicks = 0, irqTicks = 0, sirqTicks= 0;
  int             n;

  if((fh = fopen("/proc/stat", "r")) == NULL) {
    return -1;
  }

    n = fscanf(fh, "cpu " DEC_64 " " DEC_64 " " DEC_64 " " DEC_64 " " DEC_64 " "
                   DEC_64 " " DEC_64,
           &userTicks, &niceTicks, &systemTicks, &idleTicks,
           &iowTicks, &irqTicks, &sirqTicks);

    // Move to next line
    while(i!=SET_VALUE_GDB)----------如果挂载点不变，则走这个大循环，单核cpu接近40%
    {
       next_line(fh);//挂载点一旦变化，返回 ENOTCONN，则走这个小循环，单核cpu会接近100%
       j++;
    }
    
   fclose(fh);
   return 0;
}
```
执行以上代码，获得结果如下：
```
#gcc -g -o caq.o caq.c
一开始单独运行./caq.o，会看到cpu占用如下：
628957 root      20   0    8468    612    484 R  32.5  0.0  18:40.92 caq.o 
发现cpu占用率时32.5左右，
此时挂载点信息如下：
crash> mount -n 628957 |grep lxcfs
ffff88a5a2619800 ffff88a1ab25f800 fuse   lxcfs     /rootfs/proc/stat
ffff88cf53417000 ffff88a4dd622800 fuse   lxcfs     /rootfs/var/lib/baymax/var/lib/baymax/lxcfs
ffff88a272f8c600 ffff88a4dd622800 fuse   lxcfs     /rootfs/proc/cpuinfo
ffff88a257b28900 ffff88a4dd622800 fuse   lxcfs     /rootfs/proc/meminfo
ffff88a5aff40300 ffff88a4dd622800 fuse   lxcfs     /rootfs/proc/uptime
ffff88a3db2bd680 ffff88a4dd622800 fuse   lxcfs     /rootfs/proc/stat/proc/stat
ffff88a2836ba400 ffff88a4dd622800 fuse   lxcfs     /rootfs/proc/diskstats
ffff88bcb361b600 ffff88a4dd622800 fuse   lxcfs     /rootfs/proc/swaps
ffff88776e623480 ffff88a4dd622800 fuse   lxcfs     /rootfs/sys/devices/system/cpu/online

由于没有关闭/proc/stat的fd，也就是进行大循环，然后这个时候重启lxcfs挂载：
#systemctl restart lxcfs
重启之后，发现挂载点信息如下：
crash> mount -n 628957 |grep lxcfs
ffff88a5a2619800 ffff88a1ab25f800 fuse   lxcfs     /rootfs/proc/stat
ffff88a3db2bd680 ffff88a4dd622800 fuse   lxcfs     /rootfs/proc/stat/proc/stat------------这个挂载点，由于fd未关闭，所以卸载肯定失败，可以看到super_block是重启前的
ffff887795a8f600 ffff88a53b6c6800 fuse   lxcfs     /rootfs/var/lib/baymax/var/lib/baymax/lxcfs
ffff88a25472ae80 ffff88a53b6c6800 fuse   lxcfs     /rootfs/proc/cpuinfo
ffff88cf75ff1e00 ffff88a53b6c6800 fuse   lxcfs     /rootfs/proc/meminfo
ffff88a257b2ad00 ffff88a53b6c6800 fuse   lxcfs     /rootfs/proc/uptime
ffff88cf798f0d80 ffff88a53b6c6800 fuse   lxcfs     /rootfs/proc/stat/proc/stat/proc/stat--------bind模式挂载，会新生成一个/proc/stat
ffff88cf36ff2880 ffff88a53b6c6800 fuse   lxcfs     /rootfs/proc/diskstats
ffff88cf798f1f80 ffff88a53b6c6800 fuse   lxcfs     /rootfs/proc/swaps
ffff88a53f295980 ffff88a53b6c6800 fuse   lxcfs     /rootfs/sys/devices/system/cpu/online
cpu立刻打到接近100%
628957 root      20   0    8468    612    484 R  98.8  0.0  18:40.92 caq.o     
```
#### 四、故障规避或解决

我们的解决方案是：

1.由于lxcfs在3.1.2 的版本会有段错误导致退出的bug，尽量升级到4.0.0以上版本。

2.在lxcfs进行remount的时候，如果遇到umount失败，则延迟加重试，超过次数通过

日志告警系统抛出告警。

3.在fgetc 中增加判断。


 