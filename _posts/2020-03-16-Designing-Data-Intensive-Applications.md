---
layout: post
title: "Designing Data Intensive Applications"
author: fukua95
tags: ["distributed system"]
category: system
key: 100006
---

# Part1 Foundations of Data System  
 
## Ch1.Reliability, Scalability, Maintainability

## Ch2.Data Models and Query Languages
一个复杂的系统一般分成很多个layer，每一层提供给上层interface/data model, 以隐藏复杂度和执行细节. 比如：
1. 应用开发, 会把看到的数据(用户信息，组织，货物，粉丝列表...)建模成对象或数据结构.
2. 当应用开发想存储对象或数据结构，会把它们转化为更通用的数据模型，比如JSON，XML，数据库的table，图模型.
3. 存储系统/数据库工程师，负责把JSON/XML/table/graph转化为内存中的字节序列，保存在持久化设备上.
4. 硬件工程师，负责把字节序列转化为电流.
  
data model实际就是一种数据格式.  
本章会讲第2点，通用数据模型，第3章会讲第3点，存储引擎.  
// Todo  

## Ch3.Storage and Retrieval(检索)
应用开发会把数据存储到数据库/存储系统，所以一个数据库/存储系统需要考虑：**怎么保存数据？怎么检索数据？**  
传统的关系型数据库和NoSQL数据库，底下的存储引擎可以分为2类：
* log-structured storage engine, 日志结构
* page-oriented storage engine，面向页面
  
note: **The word log is often used to refer to application logs, where an application outputs text that describes 
what's happening. In this book, log is used in the more general sense: an append-only sequence of records. 
It doesn't to be human-readable; it might be binary and intended only for other programs to read**.  

以下几行代码就实现了一个简单的kv存储引擎：
```bash
#!/bin/bash

db_set() {
  echo "$1,$2" >> database
}
db_get() {
  grep "^$1," database | sed -e "s/^$1,//" | tail -n 1
}
```
db_set()的效率很高，但是db_get()的复杂度是O(n)的. 为了加快找到data，我们需要一个索引数据结构，本质上就是维护一些元数据(metadata)。
当然，多维护元数据会增加开销，db_set()时也需要写元数据，降低db_set()的效率.  
**This is an important trade-off in storage systems: well-chosen indexes speed up read queries, but every index slows down writes**.  

### Hash Indexes
上面的db_set(),db_get()数据库是只支持append写的，为了让db_get()更快，我们在内存中有一个hash_table<key, file_offset>(就是c++的unordered_map),
这样db_set()是O(1)的，db_get()log(n)就能索引到数据在磁盘的位置，  
适用的场景：key set不大的场景，因为hash_map是整个在内存中的，key太大了会导致内存不够.  
key set不大的情况下，如果每个key更新value的频率很高，也很浪费磁盘空间，对于每个key，我们只需要保留最新的value就行了，那之前的value怎么删除呢？  
把log file划分成多个固定大小的log块，每当一个块写满固定大小，就创建新的log块. 这样就方便删除旧的value，减少磁盘空间.  
同时，如果log块删除数据后变小了，还可以多个log块合并.   
当然，要工程实现这个idea，还是有很多细节需要考虑：
* log块文件的格式：最好是用二进制格式文件.
* crash recovery: 进程重启后，hash table丢失了，需要重建，遍历log块来重建很慢，更好的方式是把hash table的快照存在磁盘上.
* partially written records: 当一个写操作到一半时进程挂了导致数据损害，可以使用checksum来检查一个数据的完整性.
* 并发控制：只用一个write thread来写数据.
  
hash table index的局限性：
* key set不能太大，不然内存不够放hash table，hash table放在磁盘也不现实，因为会有大量的磁盘随机IO，速度很慢
* 范围查询很慢
  
### SSTables and LSM-Tree
SSTable, Sorted String Table是hash table index的一个优化，每个log块里面按照key排序，相比于hash table index的优点：
* 合并log块更简单，因为每个log块按照key排序了(归并排序)
* 可以不用把所有的key放到内存
  
实现：
1. 内存维护一个二叉平衡树(比如红黑树)，也叫memtable，每次write(k, v)就把(k, v)插入树，当树达到一个log块的大小，就把树的(k,v)按序写入log块，并新建一颗树
2. read(k)，先查找内存的树，没有再从最近的log块,...
  
这样实现的一个问题：当进程挂了，在内存中的tree中的(k,v)数据还没有落盘，就丢失了. 解决方案：
每次写(k,v)到内存中的tree的同时，把(k,v)append到一个额外临时的log块，这个log块不用按照key排序，当tree的数据落盘后，这个log块就可以删除了.  

实际上，上面描述的SSTable，叫做LSM-tree, Log-Structured Merge-Tree, LevelDB和RocksDB的底层数据存储结构就是LSM-tree.  
LSM-tre优化：
* 当查找一个数据库没有的key，需要先去memtable查找，再去最近的log块找，再往前找...,效率很低.  
  为了优化这类查询，存储引擎使用一个额外的数据结构Bloom filters(布隆过滤器).
* log块的压缩和合并的顺序/时机，同样有很多策略，
  
### B-Tree
ddia里面介绍的B-tree实际上是B+-tree, B和B+的区别：B的内部节点也会存数据，B+只有叶子节点会存数据，内部节点只存储引用；B+的叶子节点可以连接起来，看成一个排好序的链表.  
原因：
* 能让B+-tree变得"更矮更胖", 因为B+-tree的高度决定磁盘访问的次数，所以越矮越好
* B+的范围查询，不需要访问所有节点，只需要访问叶子节点，因为叶子节点本身就是一个排好序的链表
  
当B+-tree插入一个新数据，叶子节点空间不够用，就会分成2个page，然后修改parent-page，如果parent-page又不够用，又会分成2个page. 最后又需要把所有修改过的page写回磁盘.  
如果在这个过程中，进程挂了，就会损坏B+-tree的索引了. B+-tree是怎么处理这种场景呢？  
**WAL: B+-tree的实现一般都会写一个额外的数据结构到磁盘，a write-ahead log(也叫redo log)**.  
**WAL是一个append-only的文件，B+-tree的每一次修改会先记录到该文件，再修改B+-tree本身. 当进程挂掉重启时，使用WAL来重建b+-tree**.  
B+-tree的修改操作还需要考虑并发控制.  

B+-tree优化：
* 类似于持久化线段树，修改page->用新page
* 当key可以缩写时，让每个page放更多的子节点.
* 让叶子page顺序放在磁盘，当范围查询时可以顺序读写磁盘
  
### Comparing B-Tree and LSM-Tree
LSM-tree的写操作更快，B+-tree的读操作更快.  
写入LSM-tree的数据，在后来log块的压缩合并过程中，会被重新写到新的log块，也就是一共发生了多次磁盘写入，我们说LSM-tree有写放大的问题.  
**写放大：对数据库的一次写入导致在数据库的整个生命周期内对磁盘的多次写入**.  
LSM-tree的优点：
LSM-tree的写操作更快，因为LSM-tree是先聚集很多(k,v)在内存memtable中，再一次性顺序写入磁盘，LSM-tree更多是顺序写磁盘.  
而B+-tree每次写操作，都需要随机写磁盘.  
LSM-tree的缺点：
后台的压缩合并线程可能会影响到正常的read/write请求.  

### Other Indexing Structures

### Transaction Processing or Analytics?
### Column-Oriented Storage

## Ch4.Encoding and Evolution
### Formats for Encoding Data
### Modes of Dataflow

  
# Part2 Distributed System  
为什么需要分布式系统:  
**Scalability**: 数据容量,读写负载等超过一台机器的能力.  
**Fault tolerance/high availability**: 应用要求当若干台机器/网络/整个数据中心故障时,仍然可以提供服务.  
**Latency**: 假设应用的使用者遍布全球,在不同地区部署系统,能减少网络延时.  
  
在分布式系统中,**Node**特指: In a distributed system, each machine or virtual machine running the database software is called a node. Each node uses its CPUs, RAM and Disks independently.  
  
## Ch5.Replication  
Replication: 复制,把数据的多个副本放到多台机器上,这些机器通过网络连接.  
好处:  
* 高可用性: 当node故障时, 可以从其他node获取数据.  
* 降低数据丢失的风险.  
* 多台机器可以同时处理read request, 增大系统read request的吞吐量.  
* 应用的使用者位于多个地方, 可以减少获取数据的延时.  
  
引入的问题:  
- 对于分布式Blob存储系统, 接口只有Put/Get/Delete，数据Put后不再改变, 保持多个副本的一致性比较简单.  
- 对于分布式数据库, 有各种数据Update操作, 保持多个副本的一致性比较难.  
  
保持多个副本的一致性的算法类型:  
* single-leader replication.  
* multi-leader replication.  
* leaderless replication.  
  
### Single-leader Replication  
**Replica**: Each **node** that stores a copy of the data.  
write/update request, 如何确保所有节点收到整个数据:  
1. 一个节点为leader, 所有写请求都通过这个leader.  
2. 其他节点为follower, leader把这个写请求(作为一条**replication log**)发给所有followers.  
  
leader和所有follower都能处理read request.  
一般分布式数据库/存储系统/消息队列都内置有这种复制的特性.  
  
#### Synchronous vs Asynchronous Replication  
sync: leader收到client的一个写请求, 转发给所有follower, 等到所有follower回复ok后, 再回复client.  
async: leader收到client的一个写请求, 转发给所有follower, 同时更新好数据后就回复client(不等follower的回复).  
semi-sync: leader收到写请求, 转发给所有follower, 等到一定数量(一般是一半)的follower回复ok后, 就回复client.  
  
#### Setting Up New Followers  
加新机器或者代替坏的机器时,需要设置新的follower, 一般通过**snapshot**让follower的**replication logs**跟上leader.  
  
#### Handling Node Outages  
当系统的node下线维护, 或因机器/网络故障不能提供服务时, 我们需要降低这个node对系统可用性的影响.  
+ Follower failure: Catch-up recovery  
  follower的本地disk保存有replication log, 只需要从leader获取后面的log.  
+ Leader failure: Failover(故障切换)  
  从followers中选出新leader, 这个过程一般是自动的(比如Raft).  
  
#### Implementation of Replication Logs  
// TODO: 待补. 这一节讲node把replication logs的操作写入底层的存储引擎, 等看完ch3再来补.  
  
### Problems with Replication Lag  
Leader-base Replication要求所有的write都经过Leader, 再由Leader给followers.假设Leader采用sync的复制方式, 系统的可用性会很差, 所以系统一般采用async/semi-sync的复制方式.  
但是async/semi-sync不可避免的会出现一个问题: **Replication Lag**. 对于系统的用户来说, 会看到**数据的不一致性**: 在某一个时刻, client_1的read request发给Leader, client_2的同一个请求发给某一个(replication lag的)follower, 这2个request返回不同的结果.  
**最终一致性**: 如果一个系统保证所有followers的数据最终都会和leader保持一致, 称这个系统具有"最终一致性"的能力.但这个"最终"有可能是任意长的时间.  
下面是3个比最终一致性强一点的一致性.  
  
#### Reading Your Own Writes  
Read-your-own-write-consistency: 保证用户会看到自己提交的任何更新.  
比如一个场景: 一个用户在一个网站更新自己的个人信息, 然后刷新页面, read-your-own-write保证用户会看到自己刚才更新的信息.  
实现:  
+ 当用户读取的信息是他能修改的, 从leader读. 所以这要求系统判断哪些信息是他能修改的.  
  比如: 用户在网站的个人信息, 一般只有他自己能修改, 所以用户读取自己的个人信息时, 从leader读取.  
+ client记录它最后一次write request的时间戳, 读取信息时从时间戳满足的node中读取.  
  时间戳具体可以是replication log的序列号, 或者system clock(会有误差), 或其他方式.  
  
当然, 在具体实现还需要考虑很多问题, 比如跨设备的场景: 在电脑上write, 在手机上读取信息, 很难用时间戳的方式实现read-your-own-write.  
  
#### Monotonic Reads  
Monotonic Read: 用户发了2个读请求, 保证第2个读请求的结果不会比第1次读请求的结果"更旧".  
比如一个场景: 用户第一次刷新页面, 看到页面的文章有10条评论; Monotonic reads保证用户第2次刷新页面, 不会出现文章只有5条评论的情况(除非删除评论, 但是, 有5条评论->10条评论->删除评论只剩5条, 这时候的5条评论是最新的数据，不是旧数据).  
一种实现:  
+ 用户总是从一个固定的节点处理它的读请求, 比如通过user_id的哈希值确定用户的读请求发往哪个节点.  
  但这种实现的问题是, 当这个node挂了时, 对应的用户的读请求被阻塞/转发到其他node.  
  
#### Consistent Prefix Reads  
...  
  
#### Solutions for Replication Lag  
当使用一个提供"最终一致性"的系统时, client需要考虑: 系统数据在某一个时刻的不一致状态, 会不会对client造成问题? 如果client不能接受这种不一致性:  
* client自己处理这种不一致状态, 但这样代码会很复杂和dirty, 不建议这样处理.  
* 改用能提供更强的一致性的系统.  
  
### Multi-leader Replication   
#### Use Cases for Multi-Leader Replication  
- Multi-datacenter operation  
  把数据副本放到多个datacenter的好处是, 可以容忍整个datacenter出现故障, 和减少用户访问系统的延时.  
  这时可以采用multi-leader模式, 每一个datacenter一个leader. 与single-leader模式的对比:  
  1.性能上, 一个write request发给它所在的datacenter的leader, leader再异步发给其他datacenter的leaders, 所以client不需要等待跨datacenter的网络延时.  
  2.single-leader模式的leader所在的datacenter故障, 需要重新选举, 但是multi-leader不需要.  
  3.不同datacenter通过公有网络, 而同一个datacenter里面一般是局域网络, 速度更快.  
- Clients with offline operation / Collaborative editing  
  如果一个应用在连接不上网络时照样需要工作, 或者像Google Docs这种允许多个人同时编辑一个doc的应用, 可以考虑multi-leader模式.  
  
#### Handling write conflicts  
Multi-leader replication的最大问题就是会出现写冲突.  
比如, 2个人同时编辑一个wiki页面, user_1把标题A改成B, user_2同时把它改成C, 2个改动都在他们访问的leader修改成功了, 在把改动异步复制给所有replica时, 就检测到写冲突了.  
* 最简单的策略: 避免冲突. 
* 给每一个write request分配一个unique id, 再出现写冲突时, 选取id最大的write request.  
* 系统把写冲突的解决方案留给使用者去写.  
  
#### Multi-Leader Replication Topologies  
在multi-leader replication系统中, 还需要考虑leader之间的复制顺序, 比如:  
- all-to-all topology, 每一个leader收到一个write request, 都会转发给所有leader.  
- circular topology, leader之间以固定的环状方式转发, 比如, leader1 -> leader2 -> leader3.  
- star topology, 类似菊花图, 以一个leader为中心.   
  
### Leaderless Replication  
Leaderless replication模式, 每一个replica都能直接接收client的write request.  
  
#### Writing to the Database When a Node Is Down  
#### Limitations of Quorum Consistency  
#### Sloppy Quorums and Hinted Handoff  
#### Detecting Concurrent Writes  
  
## Ch6.Partitioning  
Partition: 把数据库的数据划分成多个部分，每一个部分放到不同的node上.  
原因:  
- 数据量太大，单台机器放不下所有数据; 或者系统query throughput太高，单台机器承受不了.  
  
优点:  
* scalability: 把数据分割放到不同的机器上，分摊query load.  
  
需要考虑的点:  
- 怎么划分数据？比如k-v能不能按照k取模来划分？ 或者按照k的范围来划分？  
- 当集群节点加/减时，怎么做rebalance.  
  
### Partitioning and Replication  
一个数据库，划分成不同的partition, 每个partition会有多个副本，把副本放到不同的node. 所以，一个node会存储多个partition.  
以leader-follower replication model为例:  
* 每个partition的多个副本，会选出一个leader和多个follower，即leader/follower是以partition为单位选的.  
* 一个node有多个partition，所以这个node可能是一些partition的leader，另一些partition的follower.  
  **each node acts as leader for some partitions and follower for other partitions**.  
  
### Partitioning of Key-Value Data  
我们需要考虑: 怎么划分数据? 即每一条数据要存储到哪一个partition?  
分区的目的是让多个node分摊query load，但是不合理的划分方式让分区失去意义. 极端例子, 所有的热点数据都放到同一个partition中，导致这个partition的query load非常高，其他的非常低.  
* 最"简单"的划分方式就是随机  
  但随机的问题是：read query时很难确定数据在哪个partition中.  
* 以key range划分  
  优点: 支持range query  
  缺点: 怎么划分range? 和key的分布相关性大  
* 以 Hash of Key划分  
  优点: 选择好的hash function, 所有分区基本分摊query load.  
  缺点: 不支持range query  
  
### Skewed Workloads and Relieving Hot Spots  
hot spot: 如果绝大部分query都基中在一个partition中，称这个partition为hot spot.  
合理的划分方式可以减少hot spot，但无法完全避免. 场景: 所有的read/write集中在一个key上.  
例子: weibo某个大明星引发非常多的讨论(这时候key可能是大明星的user id, 或讨论主题的id).  
目前大部分数据库都无法自动处理这种非常倾斜的query， 需要数据库的使用者自己处理这种情况.  
做法举例: 使用者知道一小部分的key的查询频率特别高, 可以在这些key的开头拼接也给随机数, 如果随机数是2位数的话, 
也能把每个key的所有write操作分摊到100个key, 也就分摊到多个分区上. 不过, 对应的read也需要read 100个key并合并结果.  
  
### Partitioning and Secondary Indexes  
secondary index: A secondary index usually doesn't identify a record uniquely but rather is a way of searching for 
occurrences of a particular value: find all articles containing the word hogwash, find all cars whose color is read, and so on.  
* partitioning secondary indexes by document  
* partitioning secondary indexes by term  
  
### Rebalancing Partitions  
随着时间推移, 一个数据库会有以下变化:  
1. query吞吐量增加，需要加更多cpu来处理负载.  
2. 数据量增加，需要加更多磁盘内存来存储数据.  
3. 用新机器来替代坏了的机器.  
  
以上场景都要求data/request从一个node迁移到另一个node.  
把集群内的data/request从一个node迁移到另一个node的过程称为**rebalanci**ng.  
  
系统在rebalancing时要做到:  
* rebalancing后,系统的负载(data storage, read / write requests)在集群的nodes中比较均匀.  
* reblancing的过程，系统依然能够提供服务.  
* 非必要迁移的数据，保持不动，才能最小化网络/磁盘io的负载，加快迁移速度.  
  
#### Strategies for Rebalancing  
rebalancing的策略和系统的partitioning方式有很大的关系.  
  
### Request Routing  
当一个client想发一个request给系统，它怎么知道应该发给哪一个node呢？也就是，它怎么知道要连接哪一个ip，port呢？
称这种问题为service discovery.  
一般有以下几种方案:  
* 允许client连接任何node，如果一个node发现自己不能处理这个request，会转发给合适的node.  
* 所有reqeust都转发给一个中间服务器，由这个服务器转发request给合适的node.  
   
## Ch7.Transactions
### Transaction and ACID
一个数据库在运行时：  
* 系统的硬软件在任何时候都有可能故障，包括在一系列写操作的过程。  
* 系统的用户在任何时候都有可能失去连接(例如用户出现故障)，包括在用户端的一系列操作的过程。    
* 网络中断会切断系统与用户，系统节点之间的连接。   
* 多个用户端可能会同时对系统的某一个值进行修改。  
* 一个用户对某个值修改一半，有另一个用户想读该值。  
  
所以数据库编写者需要对以上，和所有可能导致出现问题的情况进行处理。  
事务是数据库的一个概念。支持事务的数据库把用户的一系列读写操作看作一个逻辑单元，称为一个事务。事务要么执行其全部内容，要么都不执行。
如果一个事务执行的过程失败了，那事务对数据库造成的任何可能的修改都要撤销。  
有了事务，用户访问数据库就简单很多，**用户不需要处理部分操作失败的问题**，因为数据库会帮用户处理.  
不是所有的数据库都支持事务，不是所有的用户都需要事务。  

事务有4个属性，简写为ACID，Atomicity原子性，Consistency一致性，Isolation隔离性，Durability持久性。  
#### Atomicity 原子性
一般说atomic, 意思是一个东西不能再被分解成更小的部分。**在cs的很多分支会出现atomic这个词，但是表达的是不同的意思**。  
比如，在多线程编程的atomic，意思是：If one thread executes an atomic operation, that means there is no way that 
another thread could see the half-finished result of the operation. The system can only be in the state it was 
before the operation or after the operation, not something in between.  
而在ACID中的atomic，和“并发”没有关系。  
ACID中的atomic，指的是**一个事务里的所有操作，要么全部执行成功，要么全部不执行，如果执行到一半有一个操作失败了，就必须撤销这个事务
对系统的所有修改**。好处：当一个事务失败了，client知道这个事务没有对系统有任何修改，不需要去考虑部分失败的场景.     

#### Consistency 一致性
**在cs的很多分支，consistency同样出现很多次, 表达不同的意思**。  
- ch5讨论的replica consistency, async replicated systems的eventual consistency.  
- consistent hashing, 是系统用来使负载均衡的一种方法.  
- CAP理论(ch9)中的C, consistency, 指的是linearizability.  
- ACID中的C.  
  
ACID的C指的是数据库的某些数据一直保持不变，在没有其他事务并发执行的情况下，一个事务保持数据库的一致性. **确保单个事务的一致性是编写该事务的应用程序员的责任**.  

#### Isolation 隔离性
尽管多个事务可能并发执行，但数据库保证，对于任何一对事务Ti,Tj，在Ti看来，Tj或者在Ti开始之前已经完成执行，或者在Ti完成之后才开始执行，
因此，每个事务都感觉不到系统中有其他事务在并发执行.  

要怎么实现隔离性呢？最简单的方式：让事务串行执行。  
现实中都是允许事务并发执行，原因：
* 提高吞吐量和资源利用率 (CPU和磁盘可以并行运作，所以IO操作可以和CPU处理并行执行)
* 减少事务的平均响应时间
  
所以数据库需要一个并发控制部件来调度事务的并行，使得在某种意义上等价于一个串行调度，称为可串行化调度.  
可串行化的代价是事务的并发度低，很多场景，client愿意为了提高性能，降低隔离性的级别.  
SQL标准规定的隔离性级别：可串行化, 可重复读, 已提交读, 未提交读. 这几个隔离级别，在下面有具体介绍.

#### Durability 持久性
一个事务完成后，它对数据库的修改必须是永久的，即使出现系统故障.  
单机数据库的持久性，一般指数据已经写到硬盘/SSD等存储设备了，它一般还包含一个WAL(write-ahead-log)或类似的机制，为了当磁盘上的数据损坏时能够恢复数据.  
多副本数据库，持久性指数据已经成功复制到其他数据节点了.  
绝对的持久性是不可能的：当你所有的磁盘/副本同时损坏，数据库也无能为力.  

#### Single-Object and Multi-Object Operations
假设你现在写一个20KB的JSON数据到数据库,你需要考虑：
* client只发送了10KB的数据后，网络连接断开了，数据库要怎么处理这10KB数据？
* 这20KB写入磁盘时，写一半时电源故障了，磁盘的新旧数据不久混在一起了？
* 当写入一半数据时，另一个client来读这个数据，会不会看到新旧值混在一起?
  
为了解决这些问题，**存储引擎基本都需要保证单个对象的原子性和隔离性**，原子性可以使用log来做故障恢复，隔离性可以使用锁.  

### Weak Isolation Levels
数据库提供的隔离性(包括原子性，持久性)，目的就是向客户端隐藏复杂度，让客户端不用考虑那些问题.  
可串行化对性能的影响很大，所以现实中很多数据库都提供弱一点的隔离级别.  
2个概念：
* dirty read: 脏读，即读到一个"未提交"的数据.  
  防止脏读：拥有这一行的row-lock的事务维护这一行的数据的新旧值，当有其他事务来读时，读的是旧值，直到事务提交.
* dirty write: 脏写，即修改一个"已经被另一个未提交的事务写入"的数据.   
  防止脏写一般用row-lock，一个事务想修改数据(一行或一个对象),需要先获得这个行锁，直到事务提交/中止后才释放行锁.  
  
隔离级别：
* 未提交读，read uncommitted: 允许脏读，其实等于没有隔离性, 这是SQL标准允许的最低级别.
* 已提交读，read committed: 只能读已提交数据(不能脏读)，换句话说，就是一个事务的写的数据，必须在事务提交后，新数据才能被其他事务看到.  
      很多数据库运行时的默认隔离级别为已提交读.  
* 可重复读，repeatable read: 只能读已提交数据，而且在一个事务2次读取一个数据项期间，其他事务不得更新该数据.
* 可串行化
  
**以上的隔离级别都不允许脏写**.  

#### Snapshot Isolation and Repeatable Read
快照隔离是一种并发控制机制，每一个事务都从某一个一致的数据库快照中读取数据. 我们把快照看成一个状态，数据库是一个状态机，
在事务T开始时看到的是状态1，则T的整个执行过程，看到的都是状态1.  
快照隔离适合的场景：运行时间长的只读查询，比如对数据库做备份.  
// TODO

#### Preventing Lost Updates
#### Write Skew and Phantoms

### Serializability
可串行化是最强的隔离级别. 单机数据库实现可串行化的3种技术：
* literally executing transaction in a serial order
* two-phase locking, which for several decades was the only viable option
* optimistic concurrency control techniques such as serializable snapshot isolation
  
#### Actual Serial Execution
实现可串行化的最简单方式：真的串行执行事务，不用并发.  
原因：
* RAM便宜了，出现了内存数据库，所以事务的平均执行时间很小
* OLTP事务的场景一般都是做小量的读写操作，串行执行事务性能也不错.
  
比如Redis就是单线程，串行执行事务

#### Two-Phase Locking (2PL)
注意，2PL和2PC(two-phase commit)是2个完全不同的东西，不要混了.  
MySQL(InnoDB)和SQL Server实现可串行化就是用2PL.  
2PL的实现：
* 每一个对象有一个读写锁(偏向读者还是偏向写者看场景)
* 最关键的一点：**一个事务获得的所有锁，包括读锁和写锁，都必须一直拿着，直到提交/中止后才释放**.  
  这也是2PL名字的含义：第一阶段，一个事务只拿锁不释放锁；第2阶段，事务在结束时才释放所有的锁.
  
由上可见，应该经常有发生死锁的情况，所以数据库需要有一个机制：检测死锁，在发生死锁时中止掉一些事务.
2PL的缺点：性能不太行，事务的执行时间不稳定.  

#### Serializable Snapshot Isolation (SSI)

### Summary

## Ch8.The Trouble with Distributed Systems  
在分布式系统中，任何可能出错的点都会出错.  
   

### Unreliable Networks  

### Unreliable Clocks  

### Knowledge, Truth, and Lies  

## Ch9.Consistency and Consensus
**consensus，共识：让所有node在某一件事上达成一致**.  
正如我们在ch7讲的，一个数据库的运行过程中可能会有各种问题，比如磁盘故障，进程中止等等，但是数据库内部处理了这些问题，
数据库向用户保证了原子性，隔离性，持久性，所以用户不需要去考虑这些问题.  
同样地，一个分布式系统会给用户一致性，共识等等的保证，让用户不需要去考虑这些问题.  
这一章讲的是，一个分布式系统能给用户哪些保证，哪些保证是系统做不到的？

### Consistency Guarantees
所有的分布式系统给用户的最低的一致性保证：最终一致性，也就是你向系统写的一个值，最终会在所有的replica上保持一致.  
当然，这个保证是非常弱的, 因为"最终"没有明确的时间限制，可能是非常非常久.  
系统也可以提供更强的一致性保证，代价是系统的更低的性能和更低的可用性，好处是用户需要考虑的异常场景更少.    

### Linearizability
linearizability, 是最强的一致性，在其他书上有其他名字，atomic consistency, strong consistency, immediate consistency, external consistency. 
意思：一旦一个client向系统写一个值成功，**所有client**从系统读这个值都是一致最新的，系统看上去就像只有一个replica.  

#### relying on linearizability
哪些场景必须使用保证了linearizability一致性的系统呢？
* 分布式锁和leader选举
  一个单leader多副本系统，需要选举leader，一种选举leader的方式：使用锁，每个node向一个锁系统请求锁，拿到锁的node成为leader.  
  这个锁系统必须是linearizability的，不然会发生脑裂.
* 唯一性约束
  唯一性约束在系统是很常见的，比如文件存储系统的一个文件名唯一确定一个文件，当2个client并发创建同样的文件名时，其中一个必须失败.  
  
#### the cost of linearizability 
Consistency(指强一致性)，Availability(高可用性), Partition tolerance(分区容忍性).  
高可用性不是看client发请求到收到回复的时间有多快，而是指在系统某一部分发生错误的情况下，整个系统依然可以使用，不需要停止服务.  
CAP定理：**either Consistent or Available when Partitioned**, 也就是，当出现网络分区时，强一致性和高可用性，系统只能保证一个.  

强一致性虽然是一个很好的保证，但实际上很少有系统默认强一致性.  
比如，现代multi-core CPU的RAM也不是强一致性的. 假设core1的线程A写一个值到一个内存位置addr，core2的线程B去读这个addr, 系统不保证B能读到刚才A写的值，
除非使用memory barrier. 原因是每个core有自己的cache, 而A写到cache的值是异步写到RAM的. cache的修改为什么不同步写到RAM呢？为了更好的性能.  
同样，很多分布式系统为什么不提供强一致性呢？为了性能.  

### ordering guarantees
// TODO
#### ordering and gausality
#### sequence number ordering 
#### total order broadcast

### distributed transaction and consensus
共识，让多个node对某一件事达成一致. 分布式系统的很多场景都需要共识：
* leader选举，单leader系统，所有node对谁是leader需要达成一致.
* atomic commit, 实现分布式事务的原子性
  
#### atomic commit and two-phase commit
// TODO
#### distributed transaction in practice
// TODO

#### fault-tolerant consensus
共识的标准定义：**one or more nodes may propose values, and the consensus algorithm decides on one of those values**.  


#### membership and coordination services

# Part3 Derived Data
// TODO


















