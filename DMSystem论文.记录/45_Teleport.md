## 论文动机

应用中有一些操作计算量不大但是访存量大，例如数据Select操作，在disaggregation memory中需要把memory node中的数据表读到compute node中，然后用一个filter去筛选数据到一个temporary table中；

而本文的想法就是将这种操作push到memory node中去执行，这样就避免将数据读到compute node，节省数据。



## 新的方法

本文基于LegoOS进行开发，LegoOS中的memory node中具有足够的metadata来本地执行（这一点和FastSwap这种架构不太相同），比如在LegoOS中进程的上下文全在remote（进程的数据段、页表、代码段等等）。

本地调用一个函数``fn``，在TELEPORT中就变为``pushdown(fn, arg, flags)``，会通过基于RDMA write实现的RPC将此次函数请求发送到memory node，然后在memory node中会根据进程的上下文**直接生成一个临时的线程**，这个临时的线程直接使用原始进程在memory node中的上下文（类似linux中vfork()调用，不过特殊的是page不会被设置为read-only，因此也不会有copy-on-write），原始进程会block在``pushdown(fn, arg, flags)``，直到返回结果。

如果一个进程有多个线程，且多个线程同时``pushdown(fn, arg, flags)``，则在memory node中会生成多个临时线程，这多个临时线程也是共享原始进程的上下文。

在memory node中执行出现exception的话，会把exception原封不动地返回给compute node。