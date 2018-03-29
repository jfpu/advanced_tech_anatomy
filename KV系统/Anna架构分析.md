# Anna架构分析

---

分析人： jfpu
 
---

> 本文依据Anna公开资料与相关文章的测试数据，剖析世界最快的KVS数据库Anna在架构上的特点。相关内容未做深入研究，用于技术分享，仅供参考

---

[TOC]

---

## 1.Anna架构

Anna通过采用wait-free execution及coordination-free consistency（简化理解：异步执行及协调一致性）技术，实现高性能及高扩展性的可分区、多master备份的分布式KV系统。

### 1.1 Anna的设计目标

- Key空间分区： 系统的高性能不仅仅通过节点扩容达到，而且通过节点中的Actor Core扩容也能满足
- 多master备份： KV存储在多个master节点上（不区分主从模式）；针对单请求，随机选择一个core为master节点处理
- Wait-free execution：core绑定到固定CPU核上；core之间独立完成请求处理，不存在线程间CPU切换和调度开销
- Coordination-free consistency: core之间不存在数据同步互斥、一致性协同等开销

### 1.2 Anna关键技术

- Coordination-free Actors模型: 提供高性能的分布式协同架构，其性能优于采用共享内存与lock-free机制；同时也非常友好的支持系统弹性扩容
- Lattice-Powered Coordination-free Consistency: 基于分布式``格子``技术来满足最终一致性的需求

### 1.3 Anna角色分配

![](https://github.com/jfpu/advanced_tech_anatomy/blob/master/KV%E7%B3%BB%E7%BB%9F/anna_arch_pic/anna%E6%9E%B6%E6%9E%84.png)

Anna是支持分布式部署的KVS，系统由两部分组成： 客户端代理及Anna服务节点；但是，在单机s个服务进程上（比如：单个redis实例，单个TDE/tair实例），Anna与常见的其他KVS是有区别的，Anna服务节点里存在与CPU核数对等的Actro Core（可以理解为服务线程），其是Anna服务的基础组件；而Server只是一台物理机上部署Anna服务的逻辑概念。

举个例子，在TDE的使用中，可以在单机上启动多个data_server实例，每个实例/进程是TDE服务的基础；而Anna在一台物理机上只需要启动一个Anna Server，其内部默认（可配置）启动CPU核数对等的Actor Core，同时，Actor Core是绑定在对等CPU上，不会切换。

- client Proxy: 计算client请求路由，将KV请求hash后``随机分发``到key对应Actor Core组里的一个；同时，本地可以配置缓存（不同一致性策略时使用），已降低对Actor的压力
- Actor Core: Anna数据服务线程，也是KV数据存储、备份的核心组件。Actor Core采用私有哈希表，在共享内存中存储KV的备份；同时，单机的所有Actor Core也使用共享内存同步更新的KV数据（组播缓存）。
- Actor Hashtable: C++ STL的unordered map实现。
- Multicast Buffers: 需要同步的KV数据；接收端完成后，通过垃圾回收器回收存储空间

__Actor Core:__

单节点上Actor Core之间异步数据通信采用ZeroMQ的发布-订阅模式直接完成，而跨节点上Actor Core之间异步数据通信采用ZeroMQ tcp协议，数据协议采用PB完成。
Anna采用一致性HASH来分配存储KV的actor节点（参考：Dynamo），每一个actor线程具备独立的编号，Anna采用CRC32位HASH将actor分别到hash环。在KV路由选择时，通过key的hash值找到存储的actor节点，同时，将KV复制到顺时针方向的N-1（N为可配置的复制参数）个actor。

``个人理解：`` 此处说明了KEY的路由设置及查找、备份方法；文章其他部分提及处理用户请求时，是在KEY所属的actor组中随机选取一个作为处理的master actor；但是，此处没有见到相关说明

Anna支持PUT、GET及DELETE操作：
- GET： 不要求合并最新的值
- PUT： 触发actor之间的复制操作
- DELETE： 通过空VALUE段的特殊PUT操作实现；当DELETE KV的时间戳“主导/成为”key的时间戳后，actor自动回收KV占用的堆空间（可以理解，DELETE使用了临时申请的堆内存）；每个actor会维护一个时钟数组来存放接收到其他actor的最新时间戳，当时钟数组里最小的时间戳大于DELETE的时间戳时，actor才最终是否对应key的内存空间

__Client Proxy:__

客户端代理通过与actor线程的交互来服务用户请求。针对用户请求的PUT、GET及DELETE操作，客户端代理提供  __BEGIN_TRANSACTION__ 及 __END_TRANSACTION__ 两种状态来保证一致性。在一次事务处理过程中，处于同一状态的所有所有请求均会失败。

``个人理解：`` 大致意思是要保证事务开始和结束状态的一致性，但具体的维度可能会细分到KEY及用户ID粒度

同时，客户端代理通过本地缓存，缓存采用C++ STRL的unordered map实现；以此，来保障不同一致性级别下的高性能处理：

- Read Committed: 缓存PUT成功的数据，为读取操作提供服务
- Item-Cut Isolation: 缓存查询事务中的KV对

客户端代理与actor线程通信采用linux原生socket及PB实现。客户端代理到actor之间的路由采用KEY值同样的一致性HASH算法决策；在保证负载均衡方面，用户请求会被随机分配到actor组中的一个上。

在网络失败、延迟时，客户端代理通过超时机制，将请求分发的其他actor上处理。

__Actor加入与退出:__

参考Dynamo相关机制。

三个阶段：actor点到多点的状态通知及hash环调整、KV数据迁移及更新完成。其中，当数据开始迁移时，如果原始的actor接受到迁移key的请求，其会将请求转发到迁移目的actor处理，类似TDE中data_server失效后对KV请求的转发处理。

``个人理解：`` 工作节点的加入与退出与常规分布式KV系统并没有本质的区别。此处不详细介绍

### 1.4 Anna KV数据结构

![](https://github.com/jfpu/advanced_tech_anatomy/blob/master/KV%E7%B3%BB%E7%BB%9F/anna_arch_pic/anna%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84%E8%AE%BE%E8%AE%A1.png)

Anna在数据结构设计中，将KV数据与用户ID及修改的版本号一一关联起来。

MapLattice: KV哈希表，模板实现，key不可修改。Anna系统中常用两种： <key, PairLattice> 及 <user id, version id>
PairLattice: <MapLattice, ValueLattice>
ValueLattice: key的值，类型通过用户定义

PUT操作触发两种merge（通过handle函数）：
- MapLattice Merge: 合并Actor Core私有的哈希表与PUT传入的MapLattice哈希表，将共有的key（写入或更新）进行 ValueLattice 合并
- ValueLattice Merge: 文章未描述；猜测针对ValueLattice关联的userid及版本号（或者时间戳）信息，来判断最终一致性的value值，完成合并。其中，用户可以自己定义ValueLattice合并冲突的解决方法（类似：Cassandra、Bayou）

### 1.5 格子技术

- KV之间顺序保证：格子对合并更新时KV的顺序不敏感：Actor之间收到的KV副本的不同顺序均可以通过``格子``技术来保证最终顺序一致性
- KV块之间的顺序保证：简单的格子堆积块（理解为通过格子结构组织的KV数据块）可以组织成Coordination-free Consistency级别的KV数据段（范围）；其中，Coordination-free Consistency具备很多级别，如：线性化、串行化、读提交事务、读未提交事务等等，__一致性的不同类型，暂未深究，深入了解需要参考相关文章__

总之，文章通过定义阐释，格子技术可以满足交换性（commutativity）、关联性（associativity）及幂等性（idempotence）的需求

在格子技术的实现层面上，文章详细阐述了传统基于共享内存+同步机制的缺点，以及Anna采用消息传递机制的优点

- 共享内存+同步机制: 重点阐述同步机制（包括：lock、lock-free及原子CAS类技术等）对高竞争环境下，在扩展性（规模）及高性能之间存在不可调和的矛盾

``个人理解：`` 共享内存+同步机制在保证强一致性的情况下，高性能、高竞争与系统规模确实存在矛盾；但是，Anna所描述的解决方法是通过多actor之间的多master私有备份在提高系统规模同时保证高性能，其本质也是将强一致性要求降低为最终一致性

- 消息传递机制：actor私有数据/状态 + 异步数据同步队列
  - 单节点无备份: 数据无备份
  - 多节点备份: 在actor负载低时，触发定期的一部数据更新消息。

``个人理解：`` 无论是哪种备份机制，文章阐释中都未见说明采用什么机制；个人只能想到现有的lock、lock-free、CAS或者内存屏障等方法，所以，__不清楚具体的实现方法__

### 1.6 Anna Actor事件处理

文章V.A节阐释了Actor事件处理机制。

用户请求被proxy hash到一个master actor线程（其中，用户的请求不是固定hash到当个master线程，而是通过KEY的hash值，在对应处理master组中随机选取一个actor；__至于随机选取是否使用到用户ID的映射不是很清楚__），actor定期循检所以的请求报文，在每次循检结尾，将接受的请求信息存入局部changset。在共享内存的组播处理时，完成KV数据的合并

文章单独用一段内容阐释``发送方合并``技术，证明数据在发送方合并后能降低网络带宽的开销，同时也能保证不同actor接受到不同KV数据后，能够保证一致性

### 1.7 一致性分类

VI节重点阐释一致性分类。理解一致性分类级别需要需要深入阅读相关参考资料 __这里暂未深入研究__

- A.ACI Building Blocks: 通过格子有效性验证来保障一致性
- B.Anna Lattice Composition： ValueLattice Merge策略
- C.Lattice一致性案例：
  - Causal Consistency: 
  - Read Committed Consistency:
  - More Consistency: Read Uncommitted, item-cut isolation

## 2.Anna性能测试剖析

文章VIII节详细对比测试Anna与常规KV系统在不同场景的性能。

- 相关测试系统：
  - Anna: 采用了单备份、3个备份及全量备份三种配置的系统
  - TBB: 单核多线程并行、共享内存KV系统
  - Masstree: 多核并行、共享内存KV系统
  - IDEAL: 无一致性保证的多核线程、共享内存KV系统；其中，__IDEAL系统的性能是基于同步机制KV系统的理论上限__

- 竞争构造方式： 
  - zipfian

### 2.1 高竞争条件测试

- 竞争条件： zipf = 4

![](https://github.com/jfpu/advanced_tech_anatomy/blob/master/KV%E7%B3%BB%E7%BB%9F/anna_arch_pic/%E6%B5%8B%E8%AF%95-%E9%AB%98%E7%AB%9E%E4%BA%89.png)

在高竞争条件下，TBB、Mastree主要时耗用于多核之间数据互斥同步，因此，系统规模（线程数量）的增加，带来的性能提升有限。
IDEAL代表的是基于互斥机制的性能上限（数据不一致）
Anna本身将主要时耗用于请求处理上，因此，性能远高于传统KV系统

``个人理解：`` 在高竞争条件下，即KEY值冲突非常高时，Anna的性能主要通过多个actor master节点均可以处理用户请求来满足，这就解释了为什么Anna full replication模式性能最优。然后，其他传统KV系统为了保证强一致性，基本只有一个服务线程处理同一key的所有请求。

### 2.2 低竞争条件测试

- 竞争条件： zipf = 0.5

![](https://github.com/jfpu/advanced_tech_anatomy/blob/master/KV%E7%B3%BB%E7%BB%9F/anna_arch_pic/%E6%B5%8B%E8%AF%95-%E4%BD%8E%E7%AB%9E%E4%BA%89.png)

在低竞争条件下，TBB、Mastree及IDEAL的性能在一定程度上可以通过增加系统规模（线程数量）来提升，当然，提升的效果还是比较有限
而Anna在低竞争条件下，优于KEY值冲突较小，actor发送方merge功能的作用减小；而且需要组播更新的KV数量大幅度增加；导致备份的数量增大了系统负载。因此，Anna单备份在低竞争条件下性能表现最优

``个人理解：`` 在低竞争条件下，Anna系统（整体）性能优于基于互斥的KV系统，主要原因还是多master处理带来的收益

``一个测试经验就是，在低竞争的分布式KV系统上，配置较高的复制比例，会极大地影响系统的性能及吞吐量``

### 2.3 性能与集群规模测试

- Anna备份参数： 3
- 测试流量： 低竞争

![](https://github.com/jfpu/advanced_tech_anatomy/blob/master/KV%E7%B3%BB%E7%BB%9F/anna_arch_pic/%E6%B5%8B%E8%AF%95-%E6%89%A9%E5%B1%95%E6%80%A7.png)

Anna系统的性能与集群规模成线性正比

### 2.4 弹性可扩展测试

- 在流量爆发条件下，测试系统的弹性可扩展性能
- 测试流量： PUT、GET混合

测试方法：
  - 1.启动32个anna acoor，执行10s
  - 2.流量突增3倍的同时增加64个actor，执行20s
  - 3.流量和actor数量回调到步骤1的时候

![](https://github.com/jfpu/advanced_tech_anatomy/blob/master/KV%E7%B3%BB%E7%BB%9F/anna_arch_pic/%E6%B5%8B%E8%AF%95-%E5%BC%B9%E6%80%A7%E5%8F%AF%E6%89%A9%E5%B1%95%E5%9B%BE.png)

从测试数据看，Anna系统在弹性扩容方面的表现很优秀，集群吞吐量与资源容量几乎成正比。
当然，文章中没有针对集群扩缩容时服务可靠性的测试数据，__暂时不清楚Anna是否完全保证扩缩容时服务稳定性__

### 2.5 Anna与Redis单节点测试

- 测试流量： PUT、GET混合

![](https://github.com/jfpu/advanced_tech_anatomy/blob/master/KV%E7%B3%BB%E7%BB%9F/anna_arch_pic/%E6%B5%8B%E8%AF%95-%E8%AF%BB%E5%86%99%E5%B9%B6%E5%8F%91.png)

高竞争测试原因如上。
单节点上，在低竞争条件下，Anna与Redis性能并没有区别

### 2.6 分布式测试

- 备份参数： 3

![](https://github.com/jfpu/advanced_tech_anatomy/blob/master/KV%E7%B3%BB%E7%BB%9F/anna_arch_pic/%E6%B5%8B%E8%AF%95-%E5%88%86%E5%B8%83%E5%BC%8F.png)

高竞争测试原因如上。

## 3.Anna系统总结

综上所述，文章对Anna的主题架构说的很清楚，但对设计目标中的关键技术没有细说。针对这些疑问，结合笔者个人的有限经验，提出抛出一些问题：

### 3.1 Anna系统为什么有性能优势及应用场景？

Anna通过多master多备份机制，在保证最终一致性情况下，系统的扩展性及性能都优于基于互斥机制的KV系统
- 高竞争条件下，多备份（极限为全量备份）Anna配置最适合
- 低竞争条件下，少备份（机制为单个备份，常规为2）Anna配置最适合

那么，Anna性能优势来自何方？

- 宽松的一致性条件： 不满足强一致性
- actor绑定CPU
- actor多master
- 格子技术：暂未知
- actor合并技术： 暂未知

因此，Anna作为快速分布式KV系统，并不一定适合所有的场景。针对宽松一致性场景，Anna在各种竞争条件下，都具有独特的性能优势

### 3.2 Coordination-free技术是什么？

不太明白文章中的Coordination-free Consistency是否就是常见的最终一致性（暂时这么理解）
文章在II.A节中阐释过Coordination-free actor model，可以认为是分布式事件循环编程的扩展模式（个人简单理解为epoll接收报文的处理模式）；同时，给出类似参考：

- Hewitt's Actor model: 
- Erlang and Akka: 
- Bloom and CRDT: 

Anna采用的Coordination-free actor model具备独立的异步通信具备代理，但是与Bloom（一种分布式编程语言）及CRDT（未了解）中Actor模式采用的单向/单调编程模式有区别，__具体区别没有明说__

### 3.3 分布式格子技术是什么？

文章中没有详细阐述，只说明格子技术可以在不采用互斥、lock-free等常规同步方法下，通过merge操作来满足KV的最终一致性；
个人理解，即使后台merge任务通过用户ID、版本号或者时间戳来更新KV数据，应也会在merge操作中使用同步技术

-  交换性（commutativity）、关联性（associativity）及幂等性（idempotence）？

原理都好理解，就是具体的实现技术没有阐释，__暂未明白__

### 3.4 高/低竞争条件测试时Anna数据同步的消耗问题？

文章中没有阐释高竞争条件测试时，Anna actor对高冲突KEY的频繁广播及merge的开销分析
结合文章中阐释的actor发送端merge能保证一致性，同时能降低网络带宽。高竞争条件下，Anna后台组播任务能够很好的处理KEY的刷新合并，提升系统性能。这里的理解可以结合低竞争条件下Anna在3个备份及全量备份的性能低于单备份性能的表现可以分析到。

关键问题还是，__具体的算法思路没有透露__

### 3.5 Anna系统在数据可靠性方面的设计？

文章中阐释了Anna通过一致性HASH算法来决策KV在actor组之间的存储，并没有说明actor组里存在同一节点（物理机）上多个actor的情况，已经如何解决的算法，以及数据可靠性保障的程度

如果没有类似的机制保障，极端情况下，同一actor组全部在同一节点时，节点故障将导致数据丢失

假如，针对这种场景，加入数据快照机制、落地机制，是否能保障数据一定不丢失？以及actor在加入、退出时的性能影响？

### 3.6 Anna是否有利于降低KV系统的请求时耗？

文章中没有阐释，``个人理解``Anna主要针对解决系统扩容及吞吐量问题。具体到单个请求的时延，文章中仅提到client proxy的缓存机制（常规方案）。因此，Anna是否能降低请求时耗，具体取决于proxy - actor以及actor组播之间通信的代价

---

## 参考资料：

[伯克利推出世界最快的KVS数据库Anna：秒杀Redis和Cassandra](https://www.itcodemonkey.com/article/2628.html)
[The Declarative Imperative](http://db.cs.berkeley.edu/papers/sigrec10-declimperative.pdf)
[Coordination Avoidance in Database Systems](http://www.vldb.org/pvldb/vol8/p185-bailis.pdf)
[Highly Available Transactions: Virtues and Limitations](http://www.vldb.org/pvldb/vol7/p181-bailis.pdf)
[Anna: A KVS For Any Scale](https://arxiv.org/pdf/1309.3324.pdf)
