> 前面3部分全是介绍故事背景，真正的设计只在4、5章
>
> userfaultfd：https://www.anquanke.com/post/id/253835



## 故事背景

多个应用同时运行，则性能会互相影响，比如：

1. 在分配``swp_entry``时会加锁，因此多个应用同时运行时会有很严重的锁争用问题。
   - 应用在高峰期可能花70%的时间等待锁。
   - 单次swap out操作的latency可能高到几十us。
2. RDMA性能会明显下降。
   - 开了多线程的应用会分配到更多的RDMA带宽。
   - 单个应用内部，prefetch请求可能会因为等待demand请求（触发缺页的请求）而失效，作者做实验发现90%的prefetch页在70us内被用到。
3. 预取器性能会明显下降。
   - 作者拿Leap做实验，发现不同访问模式的应用一起运行的话会导致leap预取的命中率大幅度下降。



### 4 Swap System Isolation

1. Swap Partition Isolation

   Canvas为每个cgroup分配了一个swap partition、一个负责分配``swp_entry``的swap-entry manager、一个初始大小32MB的swap cache，即private cache。

   远程的内存空间是按需分配的（类似copy on write），一开始只是分配了远程内存的``swp_entry``空间。

2. RDMA Bandwidth Isolation

   Canvas为每个cgroup分配虚拟的RDMA QPs，即VQPs，应用把RDMA request push到VQP中以为自己push到的是真正的RDMA QP（PQP，physical QP），然后Canvas的调度器来负责从每个cgroup中pop RDMA request再发送给底层每个core的三个PQP中去进行发送。

   在Canvas中，用户可以为cgroup设置RDMA带宽上限。

3. Handling of Shared Pages

   多个进程间共享的内存页，即read-only的shared pages，其``struct page``中的mapcount标记了多少个进程正在共享它。Canvas中设置有global swap cache和global swap partition专门存储shared pages。在global swap partition中分配swap entry还是要加锁，不过因为shared pages数量不多，并不会给系统性能带来多大影响。



## 5 Isolation-Enabled Swap Optimizations

### 5.1 Adaptive Swap Entry Allocation

跑Memcached，核数从16增加到48，则swap entry allocation time从10us增加到130us。

1. Canvas首先为每个进程分配了单独的swap partition来避免锁争用（前面已经讲过）。但一些进程分配了很多线程，仍然可能会有大量的锁争用。

2. Canvas基于**冷热数据分类**的思想，开发了adaptive swap entry allocation。在``struct page``增加个字段entry ID来代表其对应的swap entry。每当swap out一个page时，就会将此次分配给他的swap entry记录到``struct page``中，如果下次swap out时这个entry ID还在则不用再次分配swap entry了。Canvas会在swap partition占用率超过一定阈值时启动一个周期性扫描线程，周期性扫描CPU的active LRU list，如果一个page连续几次被扫描到，则认为这个page是hot的，便将这个page的entry ID清零，释放对应的swap entry，以此来尽量减少swap entry分配次数。

   > Some of the recent patches submitted to the Linux community also attempt to reduce lock contention for swap entry allocation. A detailed description of how Canvas differs from these patches can be found in this paper's Appendix B.

### 5.2 Two-tier Adaptive Prefetching

> 之前Leap最多只实现了跟踪每个操作系统线程（即pid，每个线程一个pid）的访问历史，但是实际上很多用高级语言（Java、Go、C#）编写的现代应用很多访问都是使用指针，且很多操作系统线程内部有大量的应用线程（user space的线程）

Canvas实现了两层次prefetcher：

1. 第一层次还是使用操作系统自带的kernel prefetcher。用于处理array-based的访问。

2. 第二层次则是使用基于Linux userfaultfd实现的user space prefetcher，即每当触发缺页中断时，会最终被一个用户态线程进行处理。

   每当kernel prefetcher预取的命中率下降到一定阈值后，Canvas便会转而使用user space prefetcher。user space prefetcher当前只在JVM中实现了。

   user space prefetcher也分为两种：

   - **图的方法**。利用java的write barrier特性，每当给一个对象的字段赋值时，例如``a.f=b``，如果a和b在不同的连续内存页（page group）中，就会给这两个page group加条边。这样最后就会构建起一个图，点就是page group，边就是指针、引用，每当访问一个page group时，prefetcher会将所有3 hops以内的其它page group进行预取。
   - **多线程的方法**。即利用缺页中断的系统进程号pid去JVM中查询找到对应的java线程，Canvas为每个java线程维持历史信息，再利用Leap的majority识别方法，如果识别出majority则会进行预取。

   JVM中将所有超过1MB大小的array都组织成了一个大的search tree，在触发缺页时，如果发现这个应用的线程数量很多，且访问的是数组，则采用**2.多线程**的方法，否则采用**1.图的方法**进行预取。

### 5.3 Two-Dimensional RDMA Scheduling

> scheduling是指在大量的RDMA requests间进行schedule

1. Vertical: Fair Scheduling

   指用max-min  fairness算法为每个应用分配bandwidth，具体应该是体现在RDMA request VQP发送到PQP这一环节。

2. Horizontal: Priority Scheduling with Timeliness. 

   指在demand requests和prefetch requests之间，demand requests肯定是优先于prefetch requests被处理。

   此外，在每个``swp_entry``中增加个timestamp字段代表这个请求被压入VQP队列的时间。（如果是demand requests则不会填写这个字段，以区分demand和prefetch两种）Canvas会实时统计最近的prefetch request从发出到完成的耗时，在处理每个prefetch request时，会估计出这个prefetch request完成时间，如果已经超过阈值了（即认为这个prefetch request已经过期了），则会取消此次prefetch request，以节省RDMA带宽。