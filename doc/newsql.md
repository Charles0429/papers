近几年来，数据库领域出现了一种新的关系数据库类型，称为NewSQL，例如，Google的Spanner，Amazon的Aurora等等，这些数据库相对于传统数据库来讲，区别在哪里？[What's Really New with NewSQL?](http://db.cs.cmu.edu/papers/2016/pavlo-newsql-sigmodrec2016.pdf)给了很好的总结，本篇文章主要是总结该论文的观点，最后会有一个简单的讨论部分，全文的组织结构如下：

- 为什么需要NewSQL？
- NewSQL的分类
- NewSQL的技术挑战有哪些？
- 讨论

本文收录在我的github中[papers项目](https://github.com/Charles0429/papers)，papers项目旨在学习和总结分布式系统相关的论文。

# 为什么需要NewSQL？

数据库的发展通常是随着业务需求的变化，在2000年左右，随着互联网的兴起，有许多同时在线的用户，这对数据库领域带来了非常大的挑战，数据库通常会成为瓶颈，所以，此时业务针对数据库的需求，主要体现在可扩展上面。

这时期数据库的扩展性，往往采用如下两种方案：

1. 垂直扩展：使用更好的硬件，来做数据库的服务器
2. 水平扩展：采用中间件，做sharding的方式，即分库分表的方式

垂直扩展中使用更好的硬件意味者成本高，并且更换硬件后，需要把数据从老的机器迁移到新的机器，中间可能需要停服务，因此往往采用水平扩展，例如，Google's MySQL-based cluster。

采用中间件方式也有缺点，中间件一般要求轻量级，简单数据库操作可以搞定，但是，如果需要做分布式事务或者联表操作，会非常复杂，通常这些逻辑会放到应用层来做。

后续，NOSQL兴起，主要有几个原因：

1. 传统关系数据库更倾向于一致性，而在性能和可用性比较差
2. 全功能的关系型数据库太重
3. 关系模型对于简单的查询太重，不必要

NOSQL以Google’s BigTable 和 Amazon’s Dynamo为代表，开源版对应为HBase和Cassandra。

NOSQL往往是不保证强一致性的，而对于一些应用来讲（例如金融服务），是需要强一致性和事务的，因此，如果它们基于NOSQL系统来开发的话，应用层需要些大量的逻辑来处理一致性和事务相关的问题。此时，业务需求是拥有可扩展性的基础上，能够支持强一致性。

因此，这里有几条路：

1. 性能更好的单个服务器来做数据库服务器
2. 中间件层支持分布式事务

使用更好的单个服务器的话，不满足业务需求的可扩展性。

使用中间件的话，会有如下问题，例如：

1. 中间件层往往是比较轻量级的，要实现一致性，必须在中间件层实现分布式事务，这点是非常困难的
2. 中间件层本身的高可用很难保证

上面两条路都不能很好的满足应用的需求，因此，NewSQL出现了。

首先来看NEWSQL的定义：针对OLTP的读写，提供与NOSQL相同的可扩展性和性能，同时能支持满足ACID特性的事务。即保持NOSQL的高可扩展和高性能，并且保持关系模型。

NEWSQL的优点：

1. 轻松的获得可扩展性
2. 能够使用关系模型和事务，应用逻辑会简化很多

注意，此篇论文中的NEWSQL偏向于OLTP型数据库，和一些OLAP类型的数据库不同，OLAP数据库更偏向于复杂的只读查询，查询时间往往很长。

而NEWSQL数据库的特性如下，针对其读写事务：

1. 执行时间短
2. 一般只查询一小部分数据，通过使用索引来达到高效查询的目的
3. 一般执行相同的命令，使用不同的输入参数

# NewSQL的分类

分三大类：

1. 从头开始，使用新架构的系统
2. 中间件
3. DAAS，数据库即服务

## New Architectures

采用新架构的NewSQL有如下特点：

1. 无共享存储
2. 多节点的并发控制
3. 基于多副本做高可用和容灾
4. 流量控制
5. 分布式查询处理

优势：

1. 所有的部分都可以为分布式环境做优化，例如查询优化，通信协议优化。例如，所有的NEWSQL DBMS可以直接在节点间发送查询，而不是通过中心节点，例如中间件系统
2. 本身负责数据分区，因此，可以把查询发送给有数据的分区，而不是把数据发送给查询。
3. 拥有自身的存储，可以指定更复杂的多副本的方式

缺点：

1. 懂该数据库的人少，缺少专业的运维

代表产品：Spanner，CockroachDB

## Transparent Sharding Middleware

中间件负责的事情如下：

1. 对查询请求做路由
2. 分布式事务的协调者
3. 数据分布，数据多副本控制，数据分区

往往在各个数据库节点，需要装代理与中间件沟通，负责如下事情：

1. 在本地节点执行中间节点发来的情况，并且返回结果

优点：

1. 应用通常不需要做变化

缺点：

1. 各个节点还是运行传统数据库，即以磁盘为核心的数据库，对现有的大内存，多核服务器难以高效地利用
2. 重复的查询计划和查询优化，在中间件做一次，在各个DBMS做一次

备注：有研究表明，以磁盘为主要存储的传统DBMS，很难有效地利用非常多的核，以及更大的内存容量。

代表产品: MariaDB MaxScale, ScaleArc

## Database-as-a-Service

特点：

1. 用户可以按需使用
2. 数据库本身可能使用云产品，例如云存储等，可以较容易的实现可扩展性

代表产品：

1. Amazon Aurora
2. ClearDB

# NewSQL的技术挑战有哪些？

## Main Memory Storage

传统数据库都是以磁盘为存储中心的架构，读盘操作相对较慢，一般是内存中缓存页。

现在来讲，内存较便宜，容量大，能存储大量的数据。这些纯内存操作带来的好处是，读取和写入数据速度较快。

现有的大内存服务器，对数据库对内存的管理提出了新的要求，不再是像传统数据库那样，只是用来做页缓存，可以采用更高效地内存管理方式。

## Partitioning/Sharding

数据分区一般以某几列做hash或者range分区。

特点：

- 数据库需要能在多个分区执行SQL，并且合并数据结果的功能。
- 把同一个用户的数据可以放在一起，即使是不同数据表的数据，可以减少通信开销。
- 可以在线的添加或者删除机器。
- 可以在线的迁移或复制分区。

## Concurrency Control

数据库通过Concurrency Control来提供ACID中的Atomicity和Isolation。

### Atomicity

分布式场景下，一般采用类2PC的协议，根据事务是否需要中心节点，分为以下两类：

1. 中心节点：单点，容量限制
2. 非中心节点：需要时钟的同步

关于时钟同步，不同数据库也有不同的做法，Spanner和CroachDB在时钟同步上的不同选择：

```
But what makes Spanner differ- ent is that it uses hardware devices (e.g., GPS, atomic clocks) for high-precision clock synchronization. The DBMS uses these clocks to assign timestamps to transactions to enforce consistent views of its multi-version database over wide-area networks. CockroachDB also purports to provide the same kind of consistency for transactions across data centers as Span- ner but without the use of atomic clocks. They instead rely on a hybrid clock protocol that combines loosely synchronized hardware clocks and logical counters [41].
```

### Isolation

现有实现Isolation的技术主要包括：

- 2PL：two phase locking
- MVCC: Multiversion Concurrency Control
- OCC: Optimistic Concurrency Control

大部分的数据库还是在选择使用MVCC，例如CockroachDB；有些数据库使用2PL+MVCC，修改数据的时候，还是采用2PL，例如，InnoDB，Spanner

## Secondary Indexes

一般有两种实现方式：局部索引VS全局索引

局部索引：

1. 每个partition有一部分索引数据，每次修改索引，只需要修改一个节点，但查找数据需要可能涉及多个节点

全局索引：

1. 每个partition都有完整的索引数据，每次修改索引，都需要使用分布式事务，修改所有包含此索引副本的节点，查找数据只需要在一个节点

## Replication

两个需要考虑的点：

1. 如何保证一致性：Paxos和2PC（跨Partition）
2. 同步的方式：采用同步执行命令的方式，还是同步状态的方式

## Crash Recovery

如何最小化宕机时间？

采用主备切换

如何优化新加机器恢复到同步的时间？

一般手段为做checkpoint

# 讨论

可扩展性是NewSQL的一个非常重要的特点，对于中间件的方式，其上需要存路由信息，其本身的可扩展性比较难以解决，个人认为，其不应该算入NewSQL。

NewSQL的技术挑战除了上述提到的之外，还有如何实现多租户架构及租户之间的隔离，负载均衡等等问题。

从整篇论文中描述的内容可以看出，NewSQL中并没有开拓性的理论技术的创新，更多的是架构的创新，以及把现有的技术如何更好地适用于当今的服务器，适用于当前的分布式架构，使得这些技术有机的结合起来，形成高效率的整体，实现NewSQL高可用，可扩展，强一致性等需求。

PS:
本博客更新会在第一时间推送到微信公众号，欢迎大家关注。

![qocde_wechat](http://o8m1nd933.bkt.clouddn.com/blog/qcode_wechat.jpg)

# 参考文献

- [What's Really New with NewSQL?](http://db.cs.cmu.edu/papers/2016/pavlo-newsql-sigmodrec2016.pdf)
