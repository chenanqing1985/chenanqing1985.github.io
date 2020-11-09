# 为啥我的docker update 不生效?
## 背景：内核版本centos7.6,我们使用lxcfs来挂载/proc/meminfo
### 云原生场景lxcfs使用的那些坑

#### 一、故障现象
同事报问题，说经过update操作扩大内存的容器，在容器内看到的内存没有变化.

#### 二、故障现象分析
根据对应的容器，使用free -m 查看，发现内存在update 前后，内存确实没有变化：
比如如下容器：
```
# docker ps |grep 7448ec06e37d
7448ec06e37d        9bc3891445e9   "/usr/local/.baymax/…"   7 days ago       Up 2 days     k8s_fat-container_online-service-orcus-gpu-pod-1_orcus_99fef030-19da-11eb-9c0d-e4434b4c9d7c_0
```
查看当前内存限制：
# pwd
/sys/fs/cgroup/memory/kubepods/burstable/pod99fef030-19da-11eb-9c0d-e4434b4c9d7c/7448ec06e37de54396f66182175edfa48137f41b469bf7b8f3157dfbb9524db3
# cat memory.limit_in_bytes 
36507222016
# docker exec 7448ec06e37d free -k
              total        used        free      shared  buff/cache   available
Mem:       35651584      271104    35145468           0      235012    35380480
Swap:             0           0           0

```
从上可以看出，目前limit的值和free -k 看到的值是 一致的。
然后扩充对应的内存：
```
# docker update -m 37507222016 --memory-swap 37507222016 7448ec06e37d

# cat memory.limit_in_bytes 
37507219456
# 
```
以上说明扩充是成功的，但是如果在容器内查看：
# docker exec 7448ec06e37d free -k
              total        used        free      shared  buff/cache   available
Mem:       35651584      270900    35145672           0      235012    35380684
Swap:             0           0           0
```
我们会发现，内存基本没有变化，特别是Mem的total这个值，纹丝不动，我们使用了lxcfs来挂载/proc/meminfo,难道是
因为某种原因，lxcfs 没有去读取对应的 memory.limit_in_bytes？

为了验证这个想法，我们手工创建了一个容器：
```
# docker ps |grep my
905acbb9055f        ubuntu:18.04      "/bin/bash"              2 days ago          Up 2 days   my_ubuntu
```
然后更新前，我们查看下内存：
```
# pwd
/sys/fs/cgroup/memory/docker/905acbb9055f17c942694a66aae4787915cc5d676178b4817d613b3cd1ff9228
# cat memory.limit_in_bytes 
32507219968
# docker exec 905acbb90 free -k
              total        used        free      shared  buff/cache   available
Mem:       31745332       15020    31726272           0        4040    31730312
Swap:             0           0           0
```
在容器内我们查看的内存值和memory.limit_in_bytes 是一致的。
然后我们更新一下：
```
# docker update -m 33507219968 --memory-swap 33507222016 905acbb905
905acbb905
# cat memory.limit_in_bytes 
33507217408
# docker exec 905acbb90 free -k
              total        used        free      shared  buff/cache   available
Mem:       32721892       16184    32701668           0        4040    32705708
Swap:             0           0           0

```
对比update之前的 free -k的结果，可以明显确定，31745332 变更为 32721892 了。

这个案例，回答了我们之前的疑问，就是lxcfs是否会不去读取limit_in_bytes，看样子应该是不会的。
然后走读lxcfs的代码，发现了问题的根因是：
```
  3405 static unsigned long get_min_memlimit(const char *cgroup, const char *file)
   3406 {
   3407     char *copy = strdupa(cgroup);
   3408     unsigned long memlimit = 0, retlimit;
   3409 
   3410     retlimit = get_memlimit(copy, file);
   3411 
   3412     while (strcmp(copy, "/") != 0) {
   3413         copy = dirname(copy);
   3414         memlimit = get_memlimit(copy, file);
   3415         if (memlimit != -1 && memlimit < retlimit)----这里有个限制条件，就是只找父子链路中最小的那个限制值作为总内存
   3416             retlimit = memlimit;
   3417     };
   3418 
   3419     return retlimit;
   3420 }
```
我们k8s更新容器的内存时，并没有更新pod的内存，导致pod的内存并没有增长，而更新后的容器的内存值大于pod的内存值，被lxcfs限制。

```
# pwd
/sys/fs/cgroup/memory/kubepods/burstable/pod99fef030-19da-11eb-9c0d-e4434b4c9d7c/7448ec06e37de54396f66182175edfa48137f41b469bf7b8f3157dfbb9524db3
# cat memory.limit_in_bytes 
37507219456
# cd ../
# pwd
/sys/fs/cgroup/memory/kubepods/burstable/pod99fef030-19da-11eb-9c0d-e4434b4c9d7c
# cat memory.limit_in_bytes 
36507222016
```
而我们手工创建的容器，却没有这个限制，因为它的上级的内存限制为-1。

#### 四、故障规避或解决
1、需要先更新pod的内存，再更新容器的内存。
