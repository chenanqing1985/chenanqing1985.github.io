# 一个较复杂dcache问题
## 背景：这个是在centos7.6的环境上复现的，但问题其实在很多内核版本上都有
### 下面列一下我们是怎么排查并解这个问题的。

#### 一、故障现象
oppo云内核团队发现集群的snmpd的cpu消耗冲高,
snmpd几乎长时间占用一个核，perf发现热点如下：
```
+   92.00%     3.96%  [kernel]    [k]    __d_lookup 
-   48.95%    48.95%  [kernel]    [k] _raw_spin_lock 
     20.95% 0x70692f74656e2f73                       
        __fopen_internal                              
        __GI___libc_open                              
        system_call                                   
        sys_open                                       
        do_sys_open                                    
        do_filp_open                                   
        path_openat                                    
        link_path_walk                                 
      + lookup_fast                                    
-   45.71%    44.58%  [kernel]    [k] proc_sys_compare 
   - 5.48% 0x70692f74656e2f73                          
        __fopen_internal                               
        __GI___libc_open                               
        system_call                                    
        sys_open                                       
        do_sys_open                                    
        do_filp_open                                   
        path_openat                                    
   + 1.13% proc_sys_compare                                                                                                                     
```
几乎都消耗在内核态 __d_lookup的调用中，然后strace看到的消耗为：
```
open("/proc/sys/net/ipv4/neigh/kube-ipvs0/retrans_time_ms", O_RDONLY) = 8 <0.000024>------v4的比较快
open("/proc/sys/net/ipv6/neigh/ens7f0_58/retrans_time_ms", O_RDONLY) = 8 <0.456366>-------v6很慢
```
进一步手工操作，发现进入ipv6的路径很慢：
# time cd /proc/sys/net
real	0m0.000s
user	0m0.000s
sys	0m0.000s
# time cd /proc/sys/net/ipv6

real	0m2.454s
user	0m0.000s
sys	0m0.509s
# time cd /proc/sys/net/ipv4

real	0m0.000s
user	0m0.000s
sys	0m0.000s
可以看到，进入ipv6的路径的时间消耗远远大于ipv4的路径。
#### 二、故障现象分析

我们需要看一下，为什么perf的热点显示为__d_lookup中proc_sys_compare消耗较多，它的流程是怎么样的
proc_sys_compare只有一个调用路径，那就是d_compare回调，从调用链看：
```
__d_lookup--->if (parent->d_op->d_compare(parent, dentry, tlen, tname, name))
struct dentry *__d_lookup(const struct dentry *parent, const struct qstr *name)
{
.....
	hlist_bl_for_each_entry_rcu(dentry, node, b, d_hash) {

		if (dentry->d_name.hash != hash)
			continue;

		spin_lock(&dentry->d_lock);
		if (dentry->d_parent != parent)
			goto next;
		if (d_unhashed(dentry))
			goto next;

		/*
		 * It is safe to compare names since d_move() cannot
		 * change the qstr (protected by d_lock).
		 */
		if (parent->d_flags & DCACHE_OP_COMPARE) {
			int tlen = dentry->d_name.len;
			const char *tname = dentry->d_name.name;
			if (parent->d_op->d_compare(parent, dentry, tlen, tname, name))
				goto next;//caq：返回1则是不相同
		} else {
			if (dentry->d_name.len != len)
				goto next;
			if (dentry_cmp(dentry, str, len))
				goto next;
		}
		....
next:
		spin_unlock(&dentry->d_lock);//caq:再次进入链表循环
 	}		

.....
}
```
集群同物理条件的机器，snmp流程应该一样，所以很自然就怀疑，是不是hlist_bl_for_each_entry_rcu
循环次数过多，导致了parent->d_op->d_compare不停地比较冲突链，
进入ipv6的时候，是否比较次数很多，因为遍历list的过程中肯定会遇到了比较多的cache miss，当遍历了
太多的链表元素，则有可能触发这种情况，下面需要验证下：
```
static inline long hlist_count(const struct dentry *parent, const struct qstr *name)
{
  long count = 0;
  unsigned int hash = name->hash;
  struct hlist_bl_head *b = d_hash(parent, hash);
  struct hlist_bl_node *node;
  struct dentry *dentry;

  rcu_read_lock();
  hlist_bl_for_each_entry_rcu(dentry, node, b, d_hash) {
    count++;
  }
  rcu_read_unlock();
  if(count >COUNT_THRES)
  {
     printk("hlist_bl_head=%p,count=%ld,name=%s,hash=%u\n",b,count,name,name->hash);
  }
  return count;
}

```
kprobe的结果如下：
```
[20327461.948219] hlist_bl_head=ffffb0d7029ae3b0 count = 799259,name=ipv6/neigh/ens7f1_46/base_reachable_time_ms,hash=913731689
[20327462.190378] hlist_bl_head=ffffb0d7029ae3b0 count = 799259,name=ipv6/neigh/ens7f0_51/retrans_time_ms,hash=913731689
[20327462.432954] hlist_bl_head=ffffb0d7029ae3b0 count = 799259,name=ipv6/conf/ens7f0_51/forwarding,hash=913731689
[20327462.675609] hlist_bl_head=ffffb0d7029ae3b0 count = 799259,name=ipv6/neigh/ens7f0_51/base_reachable_time_ms,hash=913731689
```
从冲突链的长度看，确实进入了dcache的hash表中里面一条比较长的冲突链,该链的dentry个数为799259个，
而且都指向ipv6这个dentry。
了解dcache原理的同学肯定知道，位于冲突链中的元素肯定hash值是一样的，而dcache的hash值是用的parent
的dentry加上那么的hash值形成最终的hash值：
```
static inline struct hlist_bl_head *d_hash(const struct dentry *parent,
					unsigned int hash)
{
	hash += (unsigned long) parent / L1_CACHE_BYTES;
	hash = hash + (hash >> D_HASHBITS);
	return dentry_hashtable + (hash & D_HASHMASK);
}
高版本的内核是：
static inline struct hlist_bl_head *d_hash(unsigned int hash)
{
	return dentry_hashtable + (hash >> d_hash_shift);
}
```
表面上看，高版本的内核的dentry->dname.hash值的计算变化了，其实是
hash存放在dentry->d_name.hash的时候，已经加了helper，具体可以参考
如下补丁：
```
commit 8387ff2577eb9ed245df9a39947f66976c6bcd02
Author: Linus Torvalds <torvalds@linux-foundation.org>
Date:   Fri Jun 10 07:51:30 2016 -0700

    vfs: make the string hashes salt the hash
    
    We always mixed in the parent pointer into the dentry name hash, but we
    did it late at lookup time.  It turns out that we can simplify that
    lookup-time action by salting the hash with the parent pointer early
    instead of late.
```
问题分析到这里，有两个疑问如下：

1.冲突链虽然长，那也可能我们的dentry在冲突链前面啊
不一定每次都比较到那么远；

2.proc下的dentry，按道理都是常见和固定的文件名，
为什么会这么长的冲突链呢？

要解决这两个疑问，有必要，对冲突链里面的dentry进一步分析。
我们根据上面kprobe打印的hash头，可以进一步分析其中的dentry如下：
```
crash> list dentry.d_hash -H 0xffff8a29269dc608 -s dentry.d_sb
ffff89edf533d080
  d_sb = 0xffff89db7fd3c800
ffff8a276fd1e3c0
  d_sb = 0xffff89db7fd3c800
ffff8a2925bdaa80
  d_sb = 0xffff89db7fd3c800
ffff89edf5382a80
  d_sb = 0xffff89db7fd3c800
.....
```
由于链表非常长，我们把对应的分析打印到文件，发现所有的这条冲突链中所有的dentry
都是属于同一个super_block，也就是 0xffff89db7fd3c800,
```
crash> list super_block.s_list -H super_blocks -s super_block.s_id,s_nr_dentry_unused >/home/caq/super_block.txt

# grep ffff89db7fd3c800 super_block.txt  -A 2 
ffff89db7fd3c800
  s_id = "proc\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000"
```
0xffff89db7fd3c800 是 proc 文件系统，他为什么会创建这么多ipv6的dentry呢？
继续使用命令看一下dentry对应的d_inode的情况：
```
...
ffff89edf5375b00
  d_inode = 0xffff8a291f11cfb0
ffff89edf06cb740
  d_inode = 0xffff89edec668d10
ffff8a29218fa780
  d_inode = 0xffff89edf0f75240
ffff89edf0f955c0
  d_inode = 0xffff89edef9c7b40
ffff8a2769e70780
  d_inode = 0xffff8a291c1c9750
ffff8a2921969080
  d_inode = 0xffff89edf332e1a0
ffff89edf5324b40
  d_inode = 0xffff89edf2934800
...
```
我们发现，这些同名的，d_name.name均为 ipv6 的dentry，他的inode是不一样的，说明这些proc
下的文件不存在硬链接,所以这个是正常的。
我们继续分析ipv6路径的形成。
/proc/sys/net/ipv6路径的形成，简单地说分为了如下几个步骤：
```
start_kernel-->proc_root_init()//caq:注册proc fs
由于proc是linux系统默认挂载的，所以查找 kern_mount_data 函数
pid_ns_prepare_proc-->kern_mount_data(&proc_fs_type, ns);//caq:挂载proc fs
proc_sys_init-->proc_mkdir("sys", NULL);//caq：proc目录下创建sys目录
net_sysctl_init-->register_sysctl("net", empty);//caq:在/proc/sys下创建net
对于init_net:
ipv6_sysctl_register-->register_net_sysctl(&init_net, "net/ipv6", ipv6_rotable);
对于其他net_namespace,一般是系统调用触发创建
ipv6_sysctl_net_init-->register_net_sysctl(net, "net/ipv6", ipv6_table);//创建ipv6
```
有了这些基础，接下来，我们盯着最后一个，ipv6的创建流程。
ipv6_sysctl_net_init 函数
ipv6_sysctl_register-->register_pernet_subsys(&ipv6_sysctl_net_ops)-->
register_pernet_operations-->__register_pernet_operations-->
ops_init-->ipv6_sysctl_net_init
常见的调用栈如下：
```
 :Fri Mar  5 11:18:24 2021,runc:[1:CHILD],tid=125338.path=net/ipv6
 0xffffffffb9ac66f0 : __register_sysctl_table+0x0/0x620 [kernel]
 0xffffffffb9f4f7d2 : register_net_sysctl+0x12/0x20 [kernel]
 0xffffffffb9f324c3 : ipv6_sysctl_net_init+0xc3/0x150 [kernel]
 0xffffffffb9e2fe14 : ops_init+0x44/0x150 [kernel]
 0xffffffffb9e2ffc3 : setup_net+0xa3/0x160 [kernel]
 0xffffffffb9e30765 : copy_net_ns+0xb5/0x180 [kernel]
 0xffffffffb98c8089 : create_new_namespaces+0xf9/0x180 [kernel]
 0xffffffffb98c82ca : unshare_nsproxy_namespaces+0x5a/0xc0 [kernel]
 0xffffffffb9897d83 : sys_unshare+0x173/0x2e0 [kernel]
 0xffffffffb9f76ddb : system_call_fastpath+0x22/0x27 [kernel]

```
在dcache中，我们/proc/sys/下的各个net_namespace中的dentry都是一起hash的，
那怎么保证一个net_namespace
内的dentry隔离呢？我们来看对应的__register_sysctl_table函数：
```
struct ctl_table_header *register_net_sysctl(struct net *net,
	const char *path, struct ctl_table *table)
{
	return __register_sysctl_table(&net->sysctls, path, table);
}

struct ctl_table_header *__register_sysctl_table(
	struct ctl_table_set *set,
	const char *path, struct ctl_table *table)
{
	.....
	for (entry = table; entry->procname; entry++)
		nr_entries++;//caq:先计算该table下有多少个项

	header = kzalloc(sizeof(struct ctl_table_header) +
			 sizeof(struct ctl_node)*nr_entries, GFP_KERNEL);
....
	node = (struct ctl_node *)(header + 1);
	init_header(header, root, set, node, table);
....
	/* Find the directory for the ctl_table */
	for (name = path; name; name = nextname) {
....//caq:遍历查找到对应的路径
	}

	spin_lock(&sysctl_lock);
	if (insert_header(dir, header))//caq:插入到管理结构中去
		goto fail_put_dir_locked;
....
}
```
具体代码不展开，每个sys下的dentry通过 ctl_table_set 来区分是否可见
然后在查找的时候，比较如下：
```
static int proc_sys_compare(const struct dentry *parent, const struct dentry *dentry,
		unsigned int len, const char *str, const struct qstr *name)
{
....
	return !head || !sysctl_is_seen(head);
}

static int sysctl_is_seen(struct ctl_table_header *p)
{
	struct ctl_table_set *set = p->set;//获取对应的set
	int res;
	spin_lock(&sysctl_lock);
	if (p->unregistering)
		res = 0;
	else if (!set->is_seen)
		res = 1;
	else
		res = set->is_seen(set);
	spin_unlock(&sysctl_lock);
	return res;
}

//不是同一个 ctl_table_set 则不可见
static int is_seen(struct ctl_table_set *set)
{
	return &current->nsproxy->net_ns->sysctls == set;
}
```
由以上代码可以看出，当前去查找的进程，如果它归属的net_ns的set
和dentry 中归属的set不一致，则会返回失败，而snmpd归属的
set其实是init_net的sysctls，而经过查看冲突链中的各个前面绝大多数dentry
的sysctls，都不是归属于init_net的，所以前面都比较失败。


那么，为什么归属于init_net的/proc/sys/net的这个dentry会在冲突链的末尾呢？
那个是因为下面的代码导致的：
```
static inline void hlist_bl_add_head_rcu(struct hlist_bl_node *n,
					struct hlist_bl_head *h)
{
	struct hlist_bl_node *first;

	/* don't need hlist_bl_first_rcu because we're under lock */
	first = hlist_bl_first(h);

	n->next = first;//caq:每次后面添加的时候，是加在链表头
	if (first)
		first->pprev = &n->next;
	n->pprev = &h->first;

	/* need _rcu because we can have concurrent lock free readers */
	hlist_bl_set_first_rcu(h, n);
}
```

已经知道了snmp对冲突链表比较需要遍历到很后的位置的原因，接下来，需要弄
明白，为什么会有这么多dentry。根据打点，我们发现了，如果docker不停地
创建pause容器并销毁，这些net下的ipv6的dentry就会累积，
累积的原因，一个是dentry在没有触发内存紧张的情况下，不会自动销毁，
能缓存则缓存，另一个则是我们没有对冲突链的长度进行限制。

那么问题又来了，为什么ipv4的dentry就没有累积呢？既然ipv6和ipv4的父parent
都是一样的，那么查看一下这个父parent有多少个子dentry呢?
```
然后看 hash表里面的dentry，d_parent很多都指向 0xffff8a0a7739fd40 这个dentry。
crash> dentry.d_subdirs 0xffff8a0a7739fd40 ----查看这个父dentry有多少child
  d_subdirs = {
    next = 0xffff8a07a3c6f710, 
    prev = 0xffff8a0a7739fe90
  }
crash> list 0xffff8a07a3c6f710 |wc -l
1598540----------居然有159万个child
```
159万个子目录，去掉前面冲突链较长的799259个，还有差不多79万个，那既然进入ipv4路径很快，
说明在net目录下，应该还有其他的dentry有很多子dentry，会不会是一个共性问题？

然后查看集群其他机器，也发现类型现象，截取的打印如下：
```
 count=158505,d_name=net,d_len=3,name=ipv6/conf/all/disable_ipv6,hash=913731689,len=4
hlist_bl_head=ffffbd9d5a7a6cc0,count=158507
 count=158507,d_name=net,d_len=3,name=core/somaxconn,hash=1701998435,len=4
hlist_bl_head=ffffbd9d429a7498,count=158506
```
可以看到，ffffbd9d429a7498有着和ffffbd9d5a7a6cc0几乎一样长度的冲突链。
先分析ipv6 链，core链的分析其实是一样的，挑取冲突链的数据分析如下：
```
crash> dentry.d_parent,d_name.name,d_lockref.count,d_inode,d_subdirs ffff9b867904f500
  d_parent = 0xffff9b9377368240
  d_name.name = 0xffff9b867904f538 "ipv6"-----这个是一个ipv6的dentry
  d_lockref.count = 1
  d_inode = 0xffff9bba4a5e14c0
  d_subdirs = {
    next = 0xffff9b867904f950, 
    prev = 0xffff9b867904f950
  }

d_child偏移0x90，则0xffff9b867904f950减去0x90为 0xffff9b867904f8c0
crash> dentry 0xffff9b867904f8c0
struct dentry {
......
  d_parent = 0xffff9b867904f500, 
  d_name = {
    {
      {
        hash = 1718513507, 
        len = 4
      }, 
      hash_len = 18898382691
    }, 
    name = 0xffff9b867904f8f8 "conf"------名称为conf
  }, 
  d_inode = 0xffff9bba4a5e61a0, 
  d_iname = "conf\000bles_names\000\060\000.2\000\000pvs.(*Han", 
  d_lockref = {
......
        count = 1----------------引用计数为1，说明还有人引用
......
  }, 
 ......
  d_subdirs = {
    next = 0xffff9b867904fb90, 
    prev = 0xffff9b867904fb90
  }, 
......
}
既然引用计数为1，则继续往下挖：
crash> dentry.d_parent,d_lockref.count,d_name.name,d_subdirs 0xffff9b867904fb00
  d_parent = 0xffff9b867904f8c0
  d_lockref.count = 1
  d_name.name = 0xffff9b867904fb38 "all"
  d_subdirs = {
    next = 0xffff9b867904ef90, 
    prev = 0xffff9b867904ef90
  }
  再往下：
crash> dentry.d_parent,d_lockref.count,d_name.name,d_subdirs,d_flags,d_inode -x 0xffff9b867904ef00
  d_parent = 0xffff9b867904fb00
  d_lockref.count = 0x0-----------------------------挖到引用计数为0为止
  d_name.name = 0xffff9b867904ef38 "disable_ipv6"
  d_subdirs = {
    next = 0xffff9b867904efa0, --------为空
    prev = 0xffff9b867904efa0
  }
  d_flags = 0x40800ce-------------下面重点分析这个
  d_inode = 0xffff9bba4a5e4fb0
```
可以看到，ipv6的dentry路径为ipv6/conf/all/disable_ipv6,和probe看到的一样，
针对 d_flags ，分析如下：
```
#define DCACHE_FILE_TYPE        0x04000000 /* Other file type */

#define DCACHE_LRU_LIST     0x80000--------这个表示在lru上面

#define DCACHE_REFERENCED   0x0040  /* Recently used, don't discard. */
#define DCACHE_RCUACCESS    0x0080  /* Entry has ever been RCU-visible */

#define DCACHE_OP_COMPARE   0x0002
#define DCACHE_OP_REVALIDATE    0x0004
#define DCACHE_OP_DELETE    0x0008
```
我们看到，disable_ipv6的引用计数为0，但是它是有 DCACHE_LRU_LIST 标志的，
根据如下函数：
```
static void dentry_lru_add(struct dentry *dentry)
{
	if (unlikely(!(dentry->d_flags & DCACHE_LRU_LIST))) {
		spin_lock(&dcache_lru_lock);
		dentry->d_flags |= DCACHE_LRU_LIST;//有这个标志说明在lru上
		list_add(&dentry->d_lru, &dentry->d_sb->s_dentry_lru);
		dentry->d_sb->s_nr_dentry_unused++;//caq:放在s_dentry_lru是空闲的
		dentry_stat.nr_unused++;
		spin_unlock(&dcache_lru_lock);
	}
}
```
到此，说明它是可以释放的，由于是线上业务，我们不敢使用 
echo 2 >/proc/sys/vm/drop_caches
然后编写一个模块去释放，模块的主代码如下,参考 shrink_slab：
```
  spin_lock(orig_sb_lock);
        list_for_each_entry(sb, orig_super_blocks, s_list) {
                if (memcmp(&(sb->s_id[0]),"proc",strlen("proc"))||\
                   memcmp(sb->s_type->name,"proc",strlen("proc"))||\
                    hlist_unhashed(&sb->s_instances)||\
                    (sb->s_nr_dentry_unused < NR_DENTRY_UNUSED_LEN) )
                        continue;
                sb->s_count++;
                spin_unlock(orig_sb_lock);
                printk("find proc sb=%p\n",sb);
                shrinker = &sb->s_shrink;
                
               count = shrinker_one(shrinker,&shrink,1000,1000);
               printk("shrinker_one count =%lu,sb=%p\n",count,sb);
               spin_lock(orig_sb_lock);//caq:再次持锁
                if (sb_proc)
                        __put_super(sb_proc);
                sb_proc = sb;

         }
         if(sb_proc){
             __put_super(sb_proc);
             spin_unlock(orig_sb_lock);
         }
         else{
            spin_unlock(orig_sb_lock);
            printk("can't find the special sb\n");
         }
```
就发现确实两条冲突链都被释放了。
比如某个节点在释放前：
```
[3435957.357026] hlist_bl_head=ffffbd9d5a7a6cc0,count=34686
[3435957.357029] count=34686,d_name=net,d_len=3,name=core/somaxconn,hash=1701998435,len=4
[3435957.457039] IPVS: Creating netns size=2048 id=873057
[3435957.477742] hlist_bl_head=ffffbd9d429a7498,count=34686
[3435957.477745] count=34686,d_name=net,d_len=3,name=ipv6/conf/all/disable_ipv6,hash=913731689,len=4
[3435957.549173] hlist_bl_head=ffffbd9d5a7a6cc0,count=34687
[3435957.549176] count=34687,d_name=net,d_len=3,name=core/somaxconn,hash=1701998435,len=4
[3435957.667889] hlist_bl_head=ffffbd9d429a7498,count=34687
[3435957.667892] count=34687,d_name=net,d_len=3,name=ipv6/conf/all/disable_ipv6,hash=913731689,len=4
[3435958.720110] find proc sb=ffff9b647fdd4000-----------------------开始释放
[3435959.150764] shrinker_one count =259800,sb=ffff9b647fdd4000------释放结束

```
单独释放后：
```
[3436042.407051] hlist_bl_head=ffffbd9d466aed58,count=101
[3436042.407055] count=101,d_name=net,d_len=3,name=core/somaxconn,hash=1701998435,len=4
[3436042.501220] IPVS: Creating netns size=2048 id=873159
[3436042.591180] hlist_bl_head=ffffbd9d466aed58,count=102
[3436042.591183] count=102,d_name=net,d_len=3,name=core/somaxconn,hash=1701998435,len=4
[3436042.685008] hlist_bl_head=ffffbd9d4e8af728,count=101
[3436042.685011] count=101,d_name=net,d_len=3,name=ipv6/conf/all/disable_ipv6,hash=913731689,len=4
[3436043.957221] IPVS: Creating netns size=2048 id=873160
[3436044.043860] hlist_bl_head=ffffbd9d466aed58,count=103
[3436044.043863] count=103,d_name=net,d_len=3,name=core/somaxconn,hash=1701998435,len=4
[3436044.137400] hlist_bl_head=ffffbd9d4e8af728,count=102
[3436044.137403] count=102,d_name=net,d_len=3,name=ipv6/conf/all/disable_ipv6,hash=913731689,len=4
[3436044.138384] IPVS: Creating netns size=2048 id=873161
[3436044.226954] hlist_bl_head=ffffbd9d466aed58,count=104
[3436044.226956] count=104,d_name=net,d_len=3,name=core/somaxconn,hash=1701998435,len=4
[3436044.321947] hlist_bl_head=ffffbd9d4e8af728,count=103

```
上面可以看出两个细节：

1、释放前，hlist也是在增长的，释放后，hlist还是在增长。

2、释放后，net的dentry变了，所以hashlist的位置变化了。

综上所述，我们遍历热点慢，是因为snmpd所要查找init_net的ctl_table_set
和dcache中的其他dentry 归属的 ctl_table_set 不一致导致，而链表的长度则
是因为有人在销毁net_namespace的时候，还在访问ipv6/conf/all/disable_ipv6 以及
core/somaxconn 导致的，这两个dentry 都被放在了归属的super_block的 s_dentry_lru 
上。
最后一个疑问，是什么调用访问了这些dentry呢?触发的机制如下：
```
pid=16564,task=exe,par_pid=366883,task=dockerd,count=1958,d_name=net,d_len=3,name=ipv6/conf/all/disable_ipv6,hash=913731689,len=4,hlist_bl_head=ffffbd9d429a7498
hlist_bl_head=ffffbd9d5a7a6cc0,count=1960

pid=16635,task=runc:[2:INIT],par_pid=16587,task=runc,count=1960,d_name=net,d_len=3,name=core/somaxconn,hash=1701998435,len=4,hlist_bl_head=ffffbd9d5a7a6cc0
hlist_bl_head=ffffbd9d429a7498,count=1959
```
可以看到，其实就是 dockerd和runc 触发了这个问题，k8调用docker不停创建pause容器，
cni的网络参数填写不对，导致创建的net_namespace 很快被销毁，虽然销毁时调用了
unregister_net_sysctl_table，但同时 runc 和exe 访问了
该net_namespace下的两个dentry，导致这两个dentry被缓存在了 super_block的 
s_dentry_lru链表上。再因为整体内存比较充足，所以一直会增长。
而倒霉的snmpd因为一直要访问对应的链，
cpu就冲高了，使用手工 drop_caches 之后，立刻恢复，注意，线上的机器不能使用
drop_caches ，这个会导致sys 冲高，影响一些时延敏感型的业务。
#### 三、故障复现
1、内存空余的情况下，没有触发slab的内存回收，k8调用docker创建不同net_namespace
的pause容器，但因为cni的参数不对，会立刻销毁刚创建的net_namespace,如果你在dmesg
中频繁地看到如下日志：
```
IPVS: Creating netns size=2048 id=866615

```
则有必要关注一下 dentry的缓存情况。
#### 四、故障规避或解决

可能的解决方案是：

1、通过rcu的方式，读取 dentry_hashtable 的各个冲突链，大于一定程度，抛出告警。

2、通过一个proc参数，设置缓存的dentry的个数。

3、全局可以关注 /proc/sys/fs/dentry-state

4、局部的，可以针对super_block,读取s_nr_dentry_unused，超过一定数量，则告警，
示例代码可以参考shrink_slab函数的实现。

5、注意与 negative-dentry-limit 的区别。

6、内核中使用hash桶的地方很多，我们该怎么监控hash桶冲突链的长度呢？做成模块
扫描，或者找地方保存一个链表长度。

#### 五、感谢

在这个问题的解决过程中，k8的美女伟伟同学和网络的高工郑翔同学提供了很多帮助，在
此表示感谢。