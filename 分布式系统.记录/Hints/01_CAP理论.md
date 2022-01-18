 ## 1 什么是CAP

C(Consistency)：一致性，指数据在多个副本之间能够保持一致的特性；

A(Availability)：可用性，指系统提供的服务一直可用，每次请求都能获取到非错的响应，但不保证获取的数据为最新数据。

P(Partitioning Tolerance)：分区容错性，是指网络分区出现故障时（不能联系上对方），这些子分区仍然能够对外提供服务，除非整个网络环境都发生故障。



## 2 三选二

CAP理论就是说，CAP这三点只能满足其中两点，不可能三点同时满足。

CA：满足原子和可用，放弃分区容错；

CP：满足原子和分区容错，放弃可用，也就是说，当数据更新时，必须让服务停机，等数据同步更新完成后才恢复服务；

AP：满足可用和分区容错，当数据更新时仍然继续对外服务，但可能不是最新的数据。



## 3 拓展：The Mystery of 'X'

Client每次操作(Read/Write)时，可以选择一个consistency level：

ANY：any server，这种方式最快，coordinator把write缓存起来并且快速回复给client，即使根本没把这个操作给任何node处理。

ALL：all server，这种方式最慢，确保强一致性（返回给client时，所有的replica的一致性都通过了）

ONE：有一个replica返回了ACK给coordinator，那么就算完成操作。

QUORUM：用法定数量的ACKs来确认操作的完成。Read就是读取n个replica，Write就是把新值写到n个replica。



假设：

W = write的法定ACK数量

R = read的法定ACK数量

N = 存储数据对应的replica的node数量

如果W+R>N，则R和W的场景总有交集，就能保证强一致性。否则要想保证强一致性，就一般用延迟的方法，要等待后台将更新传递给其它node中才能返回。