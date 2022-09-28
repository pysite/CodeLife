## 2 Logical Clocks

Clock Condition：For any events a, b: if a$$\rightarrow$$b then C(a) < C(b).（其中C(x)是clock函数，就是给定event x，clock函数将给出该事件的order number）

> 如果满足下面的两个条件，则Clock Condition自然也能被满足

C1. If a and b are events in process $$P_i$$，and a comes before b, then $$C_i$$(a) < $$C_i$$(b).

C2. if a is the sending of a message by process $$P_i$$ and b is the receipt of that message by process $$P_j$$, then $$C_i$$(a) < $$C_j$$(b).

> 接下来介绍如何实现logical clock以满足C1和C2条件，假设$$C_i$$就是每个机器中的一个寄存器，一个counter。

IR1. Each process $$P_i$$ increments $$C_i$$ between any two successive events.

IR2. (a) If event a is the sending of a message m by process $$P_i$$, then the message m contains a timestamp $$T_m$$ = $$C_i$$(a)

​		(b) Upon receiving a message m, process $$P_j$$ sets $$C_j$$ greater then of equal to its present value and greater than $$T_m$$.



## 3 Ordering the Events Totally

> 通过使用满足之前$$Clock Condition$$的$$C_i$$，我们可以给分布式系统中的各个event编上一个全局的序列号

定义关系=>：（满足以下两个条件之一即可认为a=>b，其中a是进程$$i$$的事件，b是进程$$j$$的事件）

1. $$C_i$$(a) < $$C_j$$(b)
2. $$C_i$$(a) = $$C_j$$(b) and $$P_i$$ < $$P_j$$，其中进程之间的$$<$$是事先规定的任意的进程顺序。

关系=>可以看作是关系$$\rightarrow$$的拓展，用关系=>就可以给所有事件编序。



"给所有事件编上序号"这种能力可以用来解决很多分布式系统中的问题。论文中就展示了"多进程争取唯一的资源"问题。问题中的要求（需要我们设计算法来满足以下条件）：

$$I$$.	资源不可抢占，资源必须被释放后才可被下一进程使用。

$$II$$.	不同的请求(requests)要按照它们被发起的顺序进行满足。(the order in which they are made)

$$III$$.	如果每个进程保证不会永远占用资源，则最终每个进程的请求都会被满足。

> 首先是两个假设（这个假设不是本论文算法中的讨论范围，因为以下的假设可以靠其它机制来保证（例如msg编号以及ACK机制）

1. 消息最终都会被接受（不会丢失）
2. 进程A给进程B发送消息，则进程B接受消息的顺序和进程A发送消息的顺序一致。

> 接下来就是设计出的算法，该算法由5条规则组成（假设每个进程有一个request队列，假设时间戳Tm已经满足IR1和IR2）：

1. 当进程Pi发起对资源占用的请求时，将消息Tm:Pi（Tm是时间戳，就是前面的$$C_i$$)存入自己的请求队列并且也发送给所有其它进程。
2. 当进程Pj接受到消息Tm:Pi时，它会将其存入自己的请求队列中，并向Pi发送一条ACK消息。
3. 当正在占用资源的进程Pi想要释放资源时，会将队列中的Tm:Pi消息移除，并且向其它所有进程发送一条release的消息。
4. 当进程Pj接受到release的消息时，也将队列中的Tm:Pi消息移除。
5. 当满足以下两条件时，进程Pi开始占用资源：
   1. Tm:Pi消息在它自己的request队列头部；
   2. 进程Pi已经收到了来自所有进程的ACK消息；

> 要证明以上5条规则就能满足问题的3个条件很简单，不会的话就去看论文。关键点在于：因为事先的2条假设，所以收到ACK时，该进程之前的所有request肯定也已经发送过来了。而队列的具体实现作者没说，应该就是按照Tm排列的。

如果在这过程中有进程宕机了，则一个进程永远不会开始占用资源。具体如何处理有故障可能性的情况下的问题，作者说自己去看他的另一篇论文。



## 4 Anomalous Behavior反常行为

就是说，采用以上logical clock的系统仍然可能出现反常行为。例如进程A发起了请求，然后进程A给进程B在系统外打了个电话，然后进程B也发起了请求，结果很可能就是进程B的消息的时间戳比进程A的还小。（这就是因为系统外的时间导致了这种反常行为，如果不是经过系统外的电话，而是系统内的发消息，则不会引起此异常行为）

定义更强的"happended before"如下：

​		**a~~>b**：a在基于系统外的行为来看，也是发生在b之前的。

定义strong clock condition如下：

​		For any events a,b：如果a**~~>**b，则C(a) < C(b)

> 事实上strong clock condition的保证只能靠物理时钟保证



我们假设物理时钟满足以下两个条件：

PC1. 存在一个常数 k << 1，使得对于每个进程i，有$$|dC_t(t)/dt-1|<k$$。（即每个进程的时钟的时间增长速度都非常的逼近自然时间）

PC2. 存在一个常数 e，对于任意两个进程i和j，有$$|C_i(t)-C_j(t)|<e$$。（即两个进程的时钟最多差e）

另外假设两个进程之间通信的延迟最小是$$\mu$$，则要保证系统没有**反常行为**，即$$C_i(t+\mu)-C_j(t)>0$$，则必须要求以下条件满足（假设每个进程的物理时钟已经满足PC1和PC2）
$$
e/(1-k)\le\mu
$$


假设每个进程的物理时钟已经满足以下IR1和IR2的变种IR1'和IR2'，如何保证PC2呢？：

IR1. 任何时候（除开接受消息的那一瞬间），每个时钟的$$dC_i(t)/dt>0$$。

IR2. 接受消息Tm:Pi时，进程j会将自己的时钟作如下调整：$$C_j(t)=max(C_j(t), Tm+\mu_m)$$。（这个$$\mu_m$$就是之前的$$\mu$$一样的意义，最小的延时）



则通过数学证明可以得出以下定理：

在直径为d的全连接图中，已经满足IR1'和IR2'的物理时钟，且已经满足PC1，且每隔t秒有一条消息在每条边上传输，这条消息的实际延时也在一定范围的条件时，作者通过数学证明PC2中的$$e$$在一定范围内。（具体数学证明就不必去看了，反正知道PC2中的e存在就好）



> 复习的时候建议先看下Conclusion